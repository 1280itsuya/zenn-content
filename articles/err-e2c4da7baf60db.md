---
title: "0/3 nodes are available: insufficient cpu"
emoji: "🐛"
type: "tech"
topics: ["Kubernetes", "コンテナ", "インフラ"]
published: true
---

## 発生条件

- Kubernetes クラスターへ Pod をデプロイした際、`0/3 nodes are available: insufficient cpu` というメッセージとともに Pod が `Pending` 状態で止まる
- Deployment や Job の `resources.requests.cpu` に対して、クラスター内の全ノードの空き CPU が不足している
- HPA によるスケールアウト時や、複数の Deployment が同時にデプロイされてノードリソースが逼迫している場合にも発生する

## 原因

スケジューラーは `requests.cpu` の値をもとにノードへの配置可否を判断する。以下のような過大なリクエストを設定していると、空きノードが見つからずエラーになる。

```yaml
# 問題のある Deployment 例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          resources:
            requests:
              cpu: "4"       # ノード1台の全CPUを超えるリクエスト
              memory: "8Gi"
            limits:
              cpu: "8"
              memory: "16Gi"
```

3レプリカ × `cpu: "4"` を要求すると、合計12コア分の空きが必要になる。ノードが3台でも各ノードに他の Pod が乗っていれば `insufficient cpu` が発生する。

`kubectl describe pod <pod-name>` を実行すると Events 欄に `0/3 nodes are available: insufficient cpu` が明示される。

## 修正方法

実際のアプリが消費する CPU を `kubectl top pod` で計測し、実態に合った値へ下げる。

```yaml
# 修正後の Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          resources:
            requests:
              cpu: "250m"    # 実測ベースで設定
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
```

要点は以下の通り。

- **requests は実測値に合わせる。** `kubectl top pod` の平均値を基準にし、そこへ余裕を持たせた値を設定する。limits は requests の2倍程度が目安だが、アプリの特性による。
- **ノード増設が必要な場合は先に計画する。** クラスターの総空き CPU が `replicas × requests.cpu` を下回る限り、設定を下げずにノードを追加しても同じエラーが再発する。
- **HPA を使っている場合は `minReplicas` と `requests.cpu` の積がノード収容上限を超えないか確認する。**

```bash
# ノードごとの割り当て可能リソースを確認
kubectl describe nodes | grep -A 5 "Allocatable"

# 現在の Pod リソース消費量を確認
kubectl top pods -n <namespace>
```

`0/3 nodes are available` の数字はクラスター内のノード台数を示している。この数字が増えていない場合はノード増設が行われていないか、増設されても requests が大きすぎて配置できていないことを意味する。設定変更後は `kubectl get pods -w` でステータスが `Running` に遷移するか確認すること。

## 似たエラーの解決記事

- [CrashLoopBackOff](https://zenn.dev/articles/err-244066f0ce9690)
- [ImagePullBackOff](https://zenn.dev/articles/err-7940487a277fca)


---

📌 このエラーの要点だけ知りたい方は [エラー即答ページ](https://1280itsuya.github.io/devtools/error/e/7879e1fa2200/) へ。別のエラーで困ったら [エラーメッセージ検索ツール](https://1280itsuya.github.io/devtools/error/)（貼るだけで原因と直し方を表示）も使えます。


---

## このエラーで時間を溶かさないために
- 設定系の同種エラーを貼るだけでAI診断する[JS/TS設定エラー診断キット](https://itsuya.gumroad.com/l/aikit260623)- 原因を根本から理解するなら体系的な技術書（Amazon: [Kubernetes 実践](https://www.amazon.co.jp/s?k=Kubernetes%20%E5%AE%9F%E8%B7%B5&tag=1280itsuya22-22)）
