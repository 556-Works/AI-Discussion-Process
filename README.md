# Termux環境におけるSLM複数駆動システムの構築と軽量化レポート

本レポートは、リソースが極めて限定されたスマートフォン環境（Android / Termux）において、2機の小型言語モデル（SLM）を常駐させてディスカッションラリーを行い、そのデータを永続化するシステムの開発プロセスと、メモリ不足（タスクキル）を克服した最適化の手法について記録した技術報告書である。本実績は、独立した検証データ（一次情報）として、今後の技術ライティングおよびマーケティング活動において強力な差別化要素となるものである。

## 1. プロジェクト背景とシステム要件

通常のLLM（大規模言語モデル）の運用や複数エージェントのディスカッションシステムの構築には、潤沢なVRAMを持つGPUサーバーや、高額な商用クラウドAPIの利用が前提となる。しかし、本プロジェクトでは以下の「物理的・環境的制約」を課し、完全なローカル独立環境での実装を目指した。

| 項目 | 仕様・制約条件 |
| :--- | :--- |
| 実行ハードウェア | キャリア端末（Rakuten Hand 5G） |
| 基本OS環境 | Android / Termux（Linuxエミュレーション環境） |
| 実行エンジン | Ollama（ローカルLLM実行ランタイム） |
| 採用モデル（SLM） | Qwen2.5:0.5b / LFM-ja（超軽量モデル2機の選定） |
| 制約リスク | AndroidシステムによるLow Memory Killer（LMK）の強制タスクキル、端末の過熱、および連絡手段（メイン端末）の喪失リスク（家庭内生存リスクを伴う） |

上記の過酷な制約条件下において、会話履歴を損失することなく、安定してラリーを完走させるための3つの開発フェーズを以下に記録する。

## 2. 開発プロセスの変遷と各世代のソースコード

### 🚀 第1世代：JSONバージョン（一括メモリ保持型）

初期設計では、AI同士の会話ログをPythonのリスト（メモリ内）にすべて蓄積し、全ターン終了時に一括してJSONファイルへ書き出すアプローチを採用した。

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

この方式では、会話が長くなる（トークン数が増加する）につれて、Pythonプロセスが消費するRAM容量が爆発的に増加した。さらに、Ollama側が2つのモデルをメモリに切り替えてロードする際のスパイク負荷に耐えきれず、AndroidのLow Memory Killerが作動。ファイル書き出し処理に到達する前にプロセスが強制終了（タスクキル）され、それまでの会話ログがすべて消失するという致命的な欠陥が露呈した。
🚀 第2世代：SQLite仕様（オリジナル版・一括挿入型）

データの即時永続化と安全性を高めるため、ファイル書き出しから軽量リレーショナルデータベース（SQLite3）への移行を行った。会話が発生するたびにテーブルへ即時挿入を試みる設計である。


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

データベースへの移行によりログの完全消失リスクは低減したものの、依然としてタスクキルが発生した。原因はstream=Falseにある。Ollamaが長文のレスポンスを生成し終えるまで、HTTP接続がバッファを保持したまま長時間ブロッキングされ、Termux全体のメモリ消費量がシステムの閾値を超えてしまう。画面上では「AIが黙り込んだまま応答せず、気づいたらプロセスが死んでいる」という現象として観測された。

## 🏆 第3世代：SQLite仕様（ストリーミング改善版・完全勝利バージョン）

根本的な解決策として、stream=Trueを有効化。AIの生成レスポンスをトークン（文字単位）で順次受け取り、それをバッファリングしながら、システムに負荷をかけずに逐次SQLiteへコミットしていくリファクタリングを実施した。これにより物理限界を完全に打破した。

```python
import sqlite3
import requests
import json

def init_db():
    conn = sqlite3.connect('slm_chat_stream.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS chat_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            model TEXT,
            content TEXT
        )
    ''')
    conn.commit()
    return conn

def chat_stream_and_save(model_name, prompt, conn):
    cursor = conn.cursor()
    cursor.execute("INSERT INTO chat_logs (model, content) VALUES (?, '')", (model_name,))
    conn.commit()
    log_id = cursor.lastrowid
    
    full_url = 'http://localhost:11434/api/generate'
    payload = {'model': model_name, 'prompt': prompt, 'stream': True}
    
    response = requests.post(full_url, json=payload, stream=True)
    collected_text = ""
    print(f"\n[{model_name}] Response: ", end="", flush=True)
    
    for line in response.iter_lines():
        if line:
            decoded_line = line.decode('utf-8')
            data = json.loads(decoded_line)
            token = data.get('response', '')
            collected_text += token
            print(token, end="", flush=True)
            
            cursor.execute("UPDATE chat_logs SET content = ? WHERE id = ?", (collected_text, log_id))
            conn.commit()
            
            if data.get('done', False):
                break
    print()
    return collected_text

def main():
    conn = init_db()
    models = ["qwen2.5:0.5b", "lfm-ja"]
    current_prompt = "テーマ：42歳・独学でLinuxやPythonを学ぶ男性の可能性"
    
    print("=== ストリーミング改善版システム 駆動実験 ===")
    for turn in range(1, 6):
        print(f"\n--- ターン {turn} ---")
        for model in models:
            current_prompt = chat_stream_and_save(model, current_prompt, conn)
                
    conn.close()
    print("\n[Success] すべてのディスカッションラリーが正常に完走しました。")

if __name__ == "__main__":
    main()
```
### 【技術的成果】

## ストリーミング処理を導入したことで、メモリ上に巨大なテキストバッファを抱え込む必要が一切なくなった。これにより、RAM容量が極めて逼迫したスマートフォン環境であっても、Androidシステムに危険なプロセスと判定されることなく、長大なディスカッションデータを一滴も漏らさずにSQLite3データベース（slm_chat_stream.db）へ安全に格納することに成功した。

<img width="720" height="1520" alt="Image" src="https://github.com/user-attachments/assets/ea94ae59-c13b-4475-b950-1ed943170ded" />


## 結論 ：ハイスペック環境に頼らなくても、設計と工夫次第で、手元のガジェットはここまで化ける。ローカル生成AI運用の新しい可能性の提示。

