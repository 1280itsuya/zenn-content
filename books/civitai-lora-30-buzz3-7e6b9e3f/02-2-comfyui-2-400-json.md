---
title: "第2章 ComfyUIで2,400枚を量産：シード固定とバッチ生成のワークフローJSON"
free: false
---

# 第2章 ComfyUIで2,400枚を量産：シード固定とバッチ生成のワークフローJSON

この章を読み終えると、CSV1枚からComfyUIのAPIモードを叩いて2,400枚を無人生成し、各画像のシード・CFG・プロンプトをSQLiteへ自動記録するスクリプトが手に入る。後段(第3章)のCLIPスコア選定はこのメタデータを参照するため、ここでの記録設計が選定精度を決める。

## ComfyUI APIモードを8188ポートで起動しworkflow.jsonを取得する

GUIではなく`--listen`付きのAPIモードで起動する。Web UIの「Save (API Format)」で吐いたJSONがそのままPOSTペイロードになる。

```bash
python main.py --listen 127.0.0.1 --port 8188 --highvram
# 別ターミナルでノード構成を確認
curl -s http://127.0.0.1:8188/object_info | python -m json.tool | head -40
```

`--highvram`はRTX5090の32GB前提。8GB機では`--normalvram`に落とさないと起動直後にOOMする。

## CSV駆動でprompt・seed・CFGをバッチ投入する

30体×80枚=2,400行のCSVを読み、ノードID `"3"`(KSampler)へseedとCFGを差し込む。`workflow_api.json`のキーは自分の構成に合わせて書き換える。

```python
import csv, json, copy, requests

base = json.load(open("workflow_api.json", encoding="utf-8"))
URL = "http://127.0.0.1:8188/prompt"

with open("jobs.csv", encoding="utf-8") as f:
    for row in csv.DictReader(f):  # columns: lora,prompt,seed,cfg
        wf = copy.deepcopy(base)
        wf["6"]["inputs"]["text"] = row["prompt"]
        wf["3"]["inputs"]["seed"] = int(row["seed"])
        wf["3"]["inputs"]["cfg"]  = float(row["cfg"])
        requests.post(URL, json={"prompt": wf}).raise_for_status()
```

seedを固定列で渡すことで、Buzzした構図を後から1ピクセル差で再現できる。乱数任せにすると第4章の再現投稿が不能になる。

## バッチサイズ6でVRAM32GBを使い切る最適値

`EmptyLatentImage`の`batch_size`を変えて計測した。RTX5090(32GB)では6が上限。1枚あたり2.1秒、2,400枚で約84分だった。

```python
# batch_size別のVRAM実測 (SDXL 1024x1024)
benchmarks = {
    4: {"vram_gb": 21.8, "sec_per_img": 2.4},
    6: {"vram_gb": 29.6, "sec_per_img": 2.1},  # 採用
    8: {"vram_gb": 33.1, "sec_per_img": "OOM"},
}
wf["5"]["inputs"]["batch_size"] = 6
```

8で落ちたのはVAEデコードが一括展開でピークを踏むため。`--lowvram`ではなくバッチを6に下げる方が総時間は速い。

## OOMで3回落ちた原因をqueue分割で潰す

2,400枚を一度にキュー投入すると、3回ともVAEノードでメモリ断片化によりクラッシュした。300枚ごとに区切り、`/queue`が空になるまで待つ。

```python
import time
def wait_empty():
    while requests.get("http://127.0.0.1:8188/queue").json()["queue_running"]:
        time.sleep(5)

for i, batch in enumerate(chunks(rows, 300)):  # 8分割
    for r in batch: post(r)
    wait_empty()
    requests.post("http://127.0.0.1:8188/free", json={"unload_models": True})
```

`/free`でモデルをアンロードし断片化をリセットすると、8分割の完走率が100%になった。

## 生成メタデータをSQLiteへ記録し選定で参照する

ComfyUIの履歴APIからファイル名を回収し、seed・cfg・loraを1行に正規化する。第3章のCLIPスコア閾値判定はこのテーブルをJOINして動く。

```python
import sqlite3
db = sqlite3.connect("gen_meta.db")
db.execute("""CREATE TABLE IF NOT EXISTS images(
  filename TEXT PRIMARY KEY, lora TEXT, seed INTEGER,
  cfg REAL, prompt TEXT, clip_score REAL DEFAULT NULL)""")

hist = requests.get("http://127.0.0.1:8188/history").json()
for pid, h in hist.items():
    img = h["outputs"]["9"]["images"][0]["filename"]
    inp = h["prompt"][2]
    db.execute("INSERT OR IGNORE INTO images VALUES(?,?,?,?,?,NULL)",
        (img, inp["10"]["inputs"]["lora_name"],
         inp["3"]["inputs"]["seed"], inp["3"]["inputs"]["cfg"],
         inp["6"]["inputs"]["text"]))
db.commit()
```

`clip_score`列をNULLで先に確保しておくと、選定フェーズはUPDATE1本で済み、2,400枚の評価結果を再生成なしで上書きできる。

---

topics: ["comfyui", "python", "stablediffusion", "automation", "sqlite"]
