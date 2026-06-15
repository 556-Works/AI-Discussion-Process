# Termux環境におけるSLM複数駆動システムの構築と軽量化レポート

本レポートは、リソースが極めて限定されたスマートフォン環境（Android / Termux）において、2機の小型言語モデル（SLM）を常駐させてディスカッションラリーを行い、そのデータを永続化するシステムの開発プロセスと、メモリ不足（タスクキル）を克服した最適化の手法について記録した技術報告書です。独立した検証データ（一次情報）として公開する事で、何かのお役に立てればと思います。

## 1. プロジェクト背景とシステム要件

通常のLLM（大規模言語モデル）の運用や複数エージェントのディスカッションシステムの構築には、潤沢なVRAMを持つGPUサーバーや、高額な商用クラウドAPIの利用が前提となります。しかし、本プロジェクトでは以下の「物理的・環境的制約」を課し、完全なローカル独立環境での実装を目指しました。

| 項目 | 仕様・制約条件 |
| :--- | :--- |
| 実行ハードウェア | キャリア端末（Rakuten Hand 5G） |
| 基本OS環境 | Android / Termux（Linuxエミュレーション環境） |
| 実行エンジン | Ollama（ローカルLLM実行ランタイム） |
| 採用モデル（SLM） | Qwen2.5:0.5b / LFM-ja（超軽量モデル2機の選定） |
| 制約リスク | AndroidシステムによるLow Memory Killer（LMK）の強制タスクキル、端末の過熱、および連絡手段（メイン端末）の喪失リスク（家庭内生存リスクを伴う） |

上記の低スペック・低リソースな制約条件下において、会話履歴を損失することなく、安定してラリーを完走させるために行った3つの開発フェーズを以下に記録します。

## 2. 開発プロセスの変遷と各世代のソースコード

### 🚀 第1世代：JSONバージョン（一括メモリ保持型）

初期設計では、AI同士の会話ログをPythonのリスト（メモリ内）にすべて蓄積し、全ターン終了時に一括してJSONファイルへ書き出すアプローチを採用しました。

```python
import json
import requests

def chat_with_model(model_name, prompt, history):
    full_prompt = "\n".join(history) + "\n" + prompt
    response = requests.post('http://localhost:11434/api/generate', 
                             json={'model': model_name, 'prompt': full_prompt, 'stream': False})
    return response.json()['response']

def main():
    models = ["qwen2.5:0.5b", "lfm-ja"]
    current_prompt = "テーマ：42歳・独学でLinuxやPythonを学ぶ男性の可能性"
    conversation_log = []
    
    print("JSONバージョン開始...")
    try:
        for i in range(10):  # 10ターンのラリーを想定
            for model in models:
                print(f"[{model}] が思考中...")
                answer = chat_with_model(model, current_prompt, conversation_log)
                log_entry = f"[{model}]: {answer}"
                conversation_log.append(log_entry)
                current_prompt = answer
                
        with open("chat_backup.json", "w", encoding="utf-8") as f:
            json.dump(conversation_log, f, ensure_ascii=False, indent=4)
        print("Success: JSON書き出し完了")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```



### 【課題分析】

この方式では、会話が長くなる（トークン数が増加する）につれて、Pythonプロセスが消費するRAM容量が爆発的に増加。さらに、Ollama側が2つのモデルをメモリに切り替えてロードする際のスパイク負荷に耐えきれず、AndroidのLow Memory Killerが作動。ファイル書き出し処理に到達する前にプロセスが強制終了（タスクキル）され、それまでの会話ログがすべて消失するという致命的な欠陥が露呈しました。
間にインターバルを挟む事で、LMKを回避し完走させる事はできましたが、結果として作られるJSONファイルがストレージを圧迫するという問題が浮き彫りになったので、次のフェーズに移行しました。


🚀 第2世代：SQLite仕様（オリジナル版・一括挿入型）

データの即時永続化と安全性を高めるため、ファイル書き出しから軽量リレーショナルデータベース（SQLite3）への移行を行いました。会話が発生するたびにテーブルへ即時挿入を試みる設計です。


```python
import sqlite3
import requests

def init_db():
    conn = sqlite3.connect('slm_chat.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            model TEXT,
            response TEXT
        )
    ''')
    conn.commit()
    return conn

def chat_with_model(model_name, prompt):
    response = requests.post('http://localhost:11434/api/generate', 
                             json={'model': model_name, 'prompt': prompt, 'stream': False})
    return response.json()['response']

def main():
    conn = init_db()
    cursor = conn.cursor()
    models = ["qwen2.5:0.5b", "lfm-ja"]
    current_prompt = "テーマ：42歳・独学でLinuxやPythonを学ぶ男性の可能性"
    
    print("SQLiteオリジナル版開始...")
    for turn in range(1, 11):
        for model in models:
            try:
                answer = chat_with_model(model, current_prompt)
                cursor.execute("INSERT INTO logs (model, response) VALUES (?, ?)", (model, answer))
                conn.commit()
                current_prompt = answer
            except Exception as e:
                print(f"ターン {turn} でエラー発生: {e}")
                break
    conn.close()

if __name__ == "__main__":
    main()
```

### 【課題分析】

データベースへの移行によりログの完全消失リスクは低減したものの、依然としてタスクキルが発生。原因はstream=Falseにあると仮定しました。Ollamaが長文のレスポンスを生成し終えるまで、HTTP接続がバッファを保持したまま長時間ブロッキングされ、Termux全体のメモリ消費量がシステムの閾値を超えてしまう。インターバルを挟むスクリプトを追加しても、やはり間に合わず、画面上では「AIが黙り込んだまま応答せず、気づいたらプロセスが死んでいる」という現象として観測されたました。

## 🏆 第3世代：SQLite仕様（ストリーミング改善版・完全勝利バージョン）

根本的な解決策として、stream=Trueを有効化。AIの生成レスポンスをトークン（文字単位）で順次受け取り、それをバッファリングしながら、システムに負荷をかけずに逐次SQLiteへコミットしていくリファクタリングを実施した。これにより物理限界を完全に打破した。

```python
import sqlite3
import os
import requests
import json
from datetime import datetime
import time
# ==========================================
# 設定設定（環境に合わせて調整してください）
# ==========================================
OLLAMA_URL = "http://localhost:11434/api/chat"
MODEL_A = "qwen2.5:0.5b"    # 思考プロセスを持つモデル
MODEL_B = "lfm-ja"   # 議論の対面相手モデル
DB_FILE = "discussion_history.db"

def init_db():
    """データベースとテーブルを初期化する関数"""
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()

    # 1. ディスカッション全体を管理するテーブル
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS discussions (
        discussion_id INTEGER PRIMARY KEY AUTOINCREMENT,
        discussion_timestamp TEXT,
        theme TEXT
    )
    ''')

    # 2. 各ターンの発言を管理するテーブル（Thinking部分を分離して保存可能）
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS discussion_turns (
        turn_id INTEGER PRIMARY KEY AUTOINCREMENT,
        discussion_id INTEGER,
        timestamp TEXT,
        speaker TEXT,
        thinking_process TEXT,
        text TEXT,
        FOREIGN KEY (discussion_id) REFERENCES discussions (discussion_id)
    )
    ''')
    conn.commit()
    conn.close()
def generate_reply(model_name, messages):
    """Ollama APIを叩いてストリーミングで返答を取 得し、画面表示とテキスト結合を行う関数"""
    payload = {
        "model": model_name,
        "messages": messages,
        "stream": True  # ここをTrueにしてストリーミングモードに！
    }

    try:
        # stream=True を指定して、一括ではなく接続の「ストリーム」を開く
        response = requests.post(OLLAMA_URL, json=payload, timeout=None, stream=True)
        response.raise_for_status()

        full_reply = ""

        # 1文字（1データ塊）ずつリアルタイムに読み込んで処理するループ
        for line in response.iter_lines():
            if line:
                # Ollamaから届いたJSONデータを解析
                chunk = json.loads(line.decode('utf-8'))
                word = chunk.get("message", {}).get("content", "")

                # 1文字ずつその場で画面に出力（改 行せず、即座に反映）
                print(word, end="", flush=True)

                # 裏のバケツに1文字ずつ結合してい く（メモリ負荷は最小限）
                full_reply += word

        # 最後に改行を1ついれて画面を整える
        print()

        # 最終的に合体したテキストを返す（これでユーザーのSQLite保存関数にそのまま渡る）
        return full_reply

    except Exception as e:
        print(f"\n[エラー発生]: {e}")
        return f"エラーにより返答を取得できません でした: {e}"
def save_turn_to_db(discussion_id, speaker, raw_text):

    """1ターン分の発言をその場で解析し、SQLiteへ即時保存する関数"""
    timestamp = datetime.now().isoformat()
    thinking = ""
    reply_text = raw_text

    # lfm-ja等の出力から「Thinking...」の部分を分 離するロジック
    if "Thinking..." in raw_text:
        parts = raw_text.split("\n\n", 1)
        if len(parts) > 1:
            thinking = parts[0]
            reply_text = parts[1]

    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    try:
        cursor.execute('''
            INSERT INTO discussion_turns
            (discussion_id, timestamp, speaker, thinking_process, text)
            VALUES (?, ?, ?, ?, ?)
        ''', (discussion_id, timestamp, speaker, thinking, reply_text))
        conn.commit()
    except Exception as e:
        print(f"[DB Error] ターンの保存に失敗しま した: {e}")
    finally:
        conn.close()

def main():
    # データベースの初期化
    init_db()

    # 対話のテーマ設定と開始時刻の記録
    theme = input("ディスカッションのテーマを入力 してください: ")
    if not theme.strip():
        theme = "自由討論"

    discussion_timestamp = datetime.now().isoformat()

    # 1. 親となるディスカッションレコードを登録し てIDを取得
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO discussions (discussion_timestamp, theme) VALUES (?, ?)",
        (discussion_timestamp, theme)
    )
    discussion_id = cursor.lastrowid
    conn.commit()
    conn.close()

    print(f"\n--- ディカッションを開始します (ID: {discussion_id}) ---")
    print(f"テーマ: {theme}\n" + "="*40)

    # Ollamaに渡すコンテキスト履歴
    messages = [
        {"role": "system", "content": f"あなたはディスカッションの参加者です。テーマ「{theme}」につ いて、相手の意見を踏まえつつ論理的に対話を展開してください。"}
    ]

    # 最初の発言のきっかけ（トリガー）
    current_input = f"テーマ「{theme}」について、 あなたの見解を聞かせてください。"

    # 5往復（計10ターン）のループ処理（必要に応じ て回数は調整してください）
    max_turns = 10
    for turn_idx in range(max_turns):
        # 奇数ターンはMODEL_A、偶数ターンはMODEL_B
        if turn_idx % 2 == 0:
            current_speaker = MODEL_A
        else:
            current_speaker = MODEL_B

        print(f"\n[{current_speaker} の思考・返答 中...]")

        # 履歴に直前の発言を追加してAPIを呼び出し
        messages.append({"role": "user", "content": current_input})
        reply = generate_reply(current_speaker, messages)

        if not reply:
            print("対話を継続できないため、スクリ プトを終了します。")
            break

        # ターミナルへ出力
        print(f"▼ {current_speaker}:\n{reply}\n" + "-"*40)

        # 🔴【即時保存】ここでその場でデータベースへINSERT
        save_turn_to_db(discussion_id, current_speaker, reply)

        # 次のターンへのバトンタッチ設定
        messages.append({"role": "assistant", "content": reply})
        current_input = reply
        # 🔴 2. ターンの最後にスマホを少し休ませる（例：5秒休憩）
        if turn_idx < max_turns - 1:
            print(f"\n[System] スマホのメモリ解放 のため、10秒間インターバルを挟みます...")
            time.sleep(10)

    print(f"\n[Success] すべての対話プロセスが安全にSQLite（ID: {discussion_id}）に保存されました。")

if __name__ == "__main__":
    main()

  
```
### 【技術的成果】

## ストリーミング処理を導入したことで、メモリ上に巨大なテキストバッファを抱え込む必要が一切なくなった。これにより、RAM容量が極めて逼迫したスマートフォン環境であっても、Androidシステムに危険なプロセスと判定されることなく、長大なディスカッションデータを一滴も漏らさずにSQLite3データベース（slm_chat_stream.db）へ安全に格納することに成功した。

<img width="720" height="1520" alt="Image" src="https://github.com/user-attachments/assets/ea94ae59-c13b-4475-b950-1ed943170ded" />


## 結論 ：ハイスペック環境に頼らなくても、設計と工夫次第で、手元のガジェットはここまで化ける。ローカル生成AI運用の新しい可能性の提示。

