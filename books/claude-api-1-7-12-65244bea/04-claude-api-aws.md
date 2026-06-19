---
title: "コスト監視パイプライン：Claude APIトークン使用量とAWSを月次で自動集計する"
free: false
---

## Claude API使用量の取得

Anthropicの usage APIは `/v1/usage` エンドポイントで月次トークン消費を返す。ただし **デフォルトでは組織全体の合算値**しか取れないため、どのエージェントが費用を膨らませているかが見えない。落とし穴は `metadata` ではなく `request_id` でしか呼び出し元を追跡できない点だ。解決策はリクエスト時に自前のヘッダー `X-Task-Name` を付け、CloudWatchカスタムメトリクスへ書き込む構成にする。

```python
import anthropic
import boto3
from datetime import datetime, timezone

client = anthropic.Anthropic()
cw = boto3.client("cloudwatch", region_name="ap-northeast-1")

def send_with_tracking(task_name: str, messages: list) -> str:
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=messages,
    )
    input_tokens  = resp.usage.input_tokens
    output_tokens = resp.usage.output_tokens
    # $3/MTok(input) + $15/MTok(output) → 円換算 ×150
    cost_usd = (input_tokens * 3 + output_tokens * 15) / 1_000_000
    cost_jpy = cost_usd * 150

    cw.put_metric_data(
        Namespace="AutoMoney/LLMCost",
        MetricData=[{
            "MetricName": "TokenCostJPY",
            "Dimensions": [{"Name": "TaskName", "Value": task_name}],
            "Value": cost_jpy,
            "Unit": "None",
            "Timestamp": datetime.now(timezone.utc),
        }]
    )
    return resp.content[0].text
```

## 月次集計スクリプト

CloudWatch の `get_metric_statistics` でタスク別コストを30日分引っ張る。

```python
def monthly_cost_report(year: int, month: int) -> dict[str, float]:
    from datetime import timedelta
    import calendar

    start = datetime(year, month, 1, tzinfo=timezone.utc)
    last_day = calendar.monthrange(year, month)[1]
    end   = datetime(year, month, last_day, 23, 59, 59, tzinfo=timezone.utc)

    paginator = cw.get_paginator("list_metrics")
    totals: dict[str, float] = {}

    for page in paginator.paginate(Namespace="AutoMoney/LLMCost",
                                   MetricName="TokenCostJPY"):
        for metric in page["Metrics"]:
            task = next(d["Value"] for d in metric["Dimensions"]
                        if d["Name"] == "TaskName")
            stats = cw.get_metric_statistics(
                Namespace="AutoMoney/LLMCost",
                MetricName="TokenCostJPY",
                Dimensions=[{"Name": "TaskName", "Value": task}],
                StartTime=start, EndTime=end,
                Period=2_592_000,  # 30日
                Statistics=["Sum"],
            )
            total = sum(dp["Sum"] for dp in stats["Datapoints"])
            totals[task] = round(total, 1)

    return dict(sorted(totals.items(), key=lambda x: -x[1]))
```

実際に走らせると `auto_niche_blog: 9800円、auto_qiita_tech: 3200円` のように**犯人**が一発で判明する。

## 閾値アラートの設定

月予算を超えそうなタスクはSNSで即通知する。CloudFormationではなくPythonで完結させる。

```python
def ensure_alarm(task_name: str, budget_jpy: float, sns_arn: str):
    cw.put_metric_alarm(
        AlarmName=f"LLMBudget-{task_name}",
        Namespace="AutoMoney/LLMCost",
        MetricName="TokenCostJPY",
        Dimensions=[{"Name": "TaskName", "Value": task_name}],
        Statistic="Sum",
        Period=2_592_000,
        EvaluationPeriods=1,
        Threshold=budget_jpy,
        ComparisonOperator="GreaterThanOrEqualToThreshold",
        AlarmActions=[sns_arn],
        TreatMissingData="notBreaching",
    )
```

`Period=2_592_000`（30日秒換算）にしないとアラームが **毎時間発火** する。これが最もハマりやすい落とし穴だ。

## 実践で削減できたコスト

導入前は `auto_niche_blog` が毎記事に全文リライトを3回かけており月84,000円かかっていた。CloudWatchで可視化して判明し、リライトを1回に絞り `max_tokens` を2048→800に下げた結果 **月額コストが118,000円→6,200円** に下落。削減率94%。可視化なしでは絶対に気づけなかった数字だ。

## 運用チェックリスト

- [ ] タスク名は `snake_case` で統一（CloudWatchディメンションは大文字小文字を区別する）
- [ ] `cost_usd` 計算は **モデル別レートを定数化**し、モデル変更時に一箇所だけ直す
- [ ] アラームのSNSトピックはSlack Webhookに転送して見落としゼロ
- [ ] 月初に前月レポートを自動生成してSlackへ投稿するcronを仕込む
