---
title: "YouTube Data API v3で毎日0:00に自動アップロード — OAuth 2.0トークン管理とWindows Task Scheduler/cron全設定"
free: false
---

## Google Cloud ConsoleでOAuth 2.0クライアントIDを10分で取得する

[Google Cloud Console](https://console.cloud.google.com/) でプロジェクトを作成後、以下の順で設定する。

1. **APIs & Services → Enable APIs** → "YouTube Data API v3" を検索して有効化
2. **OAuth同意画面 → User Type: External** → アプリ名と連絡先メールを入力
3. **Credentials → Create Credentials → OAuth client ID → Desktop app** → JSONをダウンロード
4. ダウンロードしたJSONを `credentials/channel_A.json` として保存

```bash
pip install google-auth google-auth-oauthlib google-api-python-client python-dotenv requests
```

初回認証は手動でブラウザを開く必要があるが、以降はトークンファイルで無人継続できる。

---

## video.insertクォータ逆算 — 1日10,000ユニットでShortsは最大6本

YouTube Data API v3の1日クォータは **10,000ユニット**。`videos.insert` は1回 **1,600ユニット** 消費する。

| 操作 | コスト |
|------|--------|
| `videos.insert` | 1,600 |
| `videos.list` | 1 |
| `thumbnails.set` | 50 |
| `playlistItems.insert` | 50 |

```python
QUOTA_PER_DAY = 10_000
UPLOAD_COST   = 1_600
MAX_UPLOADS   = QUOTA_PER_DAY // UPLOAD_COST   # → 6
SAFE_UPLOADS  = 4  # サムネイル・プレイリスト操作込みで安全ライン
```

サムネイル設定（50ユニット×本数）や再生リスト追加を加算すると、実運用では **1日4〜5本** が枯渇しない上限。

---

## Pythonでvideo.insertを実装 — アクセストークン自動リフレッシュ付き

```python
# upload_short.py
import os, pickle
from pathlib import Path
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request

SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]

def get_credentials(creds_file: str, token_file: str):
    creds = None
    if Path(token_file).exists():
        with open(token_file, "rb") as f:
            creds = pickle.load(f)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())          # 期限切れを自動更新
        else:
            flow = InstalledAppFlow.from_client_secrets_file(creds_file, SCOPES)
            creds = flow.run_local_server(port=0)
        with open(token_file, "wb") as f:
            pickle.dump(creds, f)
    return creds

def upload_short(video_path: str, title: str, creds_file: str, token_file: str) -> str:
    creds   = get_credentials(creds_file, token_file)
    youtube = build("youtube", "v3", credentials=creds)
    body = {
        "snippet": {
            "title": title,
            "description": "#Shorts",
            "tags": ["Shorts"],
            "categoryId": "22",
        },
        "status": {"privacyStatus": "public", "selfDeclaredMadeForKids": False},
    }
    media    = MediaFileUpload(video_path, chunksize=-1, resumable=True)
    request  = youtube.videos().insert(part="snippet,status", body=body, media_body=media)
    response = request.execute()
    return response["id"]

if __name__ == "__main__":
    vid_id = upload_short(
        video_path="output/short_final.mp4",
        title="【AI生成】今日の一言 #Shorts",
        creds_file="credentials/channel_A.json",
        token_file="tokens/channel_A.pkl",
    )
    print(f"https://youtu.be/{vid_id}")
```

`creds.refresh(Request())` がトークン期限切れを自動延長する。初回のみブラウザ認証が走り、`tokens/channel_A.pkl` に保存後は以降の実行で再認証不要になる。

---

## 複数チャンネルのクレデンシャルを.envで分離するパターン

チャンネルごとにOAuthクライアントIDを別発行することで、片方の停止が他方に波及しない。

```bash
# .env
CHANNEL_A_CREDS=credentials/channel_A.json
CHANNEL_A_TOKEN=tokens/channel_A.pkl
CHANNEL_B_CREDS=credentials/channel_B.json
CHANNEL_B_TOKEN=tokens/channel_B.pkl
```

```python
# run_upload.py
import os
from dotenv import load_dotenv
from upload_short import upload_short

load_dotenv()

CHANNELS = {
    "A": (os.environ["CHANNEL_A_CREDS"], os.environ["CHANNEL_A_TOKEN"]),
    "B": (os.environ["CHANNEL_B_CREDS"], os.environ["CHANNEL_B_TOKEN"]),
}

VIDEO = "output/short_final.mp4"

for name, (creds, token) in CHANNELS.items():
    vid = upload_short(VIDEO, f"Shorts {name}", creds, token)
    print(f"[{name}] https://youtu.be/{vid}")
```

チャンネルを追加する場合は `.env` に2行足すだけ。コード変更不要。

---

## Windows Task Scheduler/cronで毎日0:00に起動する全設定

**Windows — XMLインポート方式**

```xml
<!-- tasks/upload_short.xml -->
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <Triggers>
    <CalendarTrigger>
      <StartBoundary>2026-06-25T00:00:00</StartBoundary>
      <ScheduleByDay><DaysInterval>1</DaysInterval></ScheduleByDay>
    </CalendarTrigger>
  </Triggers>
  <Actions Context="Author">
    <Exec>
      <Command>C:\auto-money\.venv\Scripts\python.exe</Command>
      <Arguments>C:\auto-money\run_upload.py</Arguments>
      <WorkingDirectory>C:\auto-money</WorkingDirectory>
    </Exec>
  </Actions>
  <Settings>
    <ExecutionTimeLimit>PT30M</ExecutionTimeLimit>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
  </Settings>
</Task>
```

```powershell
schtasks /Create /XML tasks\upload_short.xml /TN "AutoShorts\Upload" /F
```

**Linux/WSL2 — crontab**

```bash
# crontab -e
0 0 * * * cd /home/user/auto-money && \
  /home/user/auto-money/.venv/bin/python run_upload.py >> logs/upload.log 2>&1
```

`ExecutionTimeLimit` を `PT30M`（30分）に設定しておくと、ハング時に自動強制終了できる。

---

## アップロード失敗時にSlack Webhookで即時通知する

クォータ超過（`HttpError 403 quotaExceeded`）や認証エラーを無言スキップすると翌日に気付けない。`SLACK_WEBHOOK_URL` を `.env` に1行追加するだけで完結する。

```python
# notify.py
import requests, os

_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL", "")

def notify_success(video_id: str, channel: str) -> None:
    if not _WEBHOOK:
        return
    requests.post(_WEBHOOK, json={"text": f":white_check_mark: [{channel}] https://youtu.be/{video_id}"})

def notify_failure(error: str, channel: str) -> None:
    if not _WEBHOOK:
        return
    requests.post(_WEBHOOK, json={"text": f":x: [{channel}] FAILED — {error}"})
```

```python
# run_upload.py（通知付き最終版）
from dotenv import load_dotenv
load_dotenv()

import os
from upload_short import upload_short
from notify import notify_success, notify_failure

CHANNELS = {
    "A": (os.environ["CHANNEL_A_CREDS"], os.environ["CHANNEL_A_TOKEN"]),
}
VIDEO = "output/short_final.mp4"

for name, (creds, token) in CHANNELS.items():
    try:
        vid = upload_short(VIDEO, f"Shorts {name}", creds, token)
        notify_success(vid, name)
    except Exception as e:
        notify_failure(str(e), name)
```

Slack通知がない日はクォータ超過か認証切れを疑う。`HttpError 403` のメッセージがそのままSlackに飛ぶため、エラー種別を目視で判断できる。
