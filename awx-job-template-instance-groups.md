# AWX API でジョブテンプレートのインスタンスグループを変更する

## 概要

ジョブテンプレートのインスタンスグループは、サブリソース
`/api/v2/job_templates/{ジョブテンプレートID}/instance_groups/` に対する **POST** で変更します。

- 追加（関連付け）: `{"id": <インスタンスグループID>}`
- 削除（関連付け解除）: `{"id": <インスタンスグループID>, "disassociate": true}`
- 認証: ユーザーのトークンを `Authorization: Bearer <TOKEN>` ヘッダーで渡す

## 事前準備

```bash
TOKEN="xxxxxxxx"
AWX="https://awx.example.com"
```

## 手順

### 1. ID を調べる

```bash
# ジョブテンプレートの ID（名前で検索）
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AWX/api/v2/job_templates/?name=MyTemplate" | jq '.results[] | {id, name}'

# インスタンスグループの ID
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AWX/api/v2/instance_groups/" | jq '.results[] | {id, name}'
```

出力例:

```json
{ "id": 1, "name": "default" }
{ "id": 2, "name": "controlplane" }
{ "id": 3, "name": "gpu-nodes" }
```

Web UI では、「インスタンスグループ」画面で対象を開いたときの URL
（例: `.../instance_groups/3/details`）に含まれる数字が ID です。

### 2. 現在の設定を確認する

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$AWX/api/v2/job_templates/10/instance_groups/" | jq '.results[] | {id, name}'
```

### 3. 追加する（関連付け）

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id": 3}' \
  "$AWX/api/v2/job_templates/10/instance_groups/"
```

| 箇所 | 意味 |
|---|---|
| URL の `10` | 変更対象の**ジョブテンプレート**の ID |
| ボディの `{"id": 3}` | 関連付ける**インスタンスグループ**の ID |

成功すると `204 No Content` が返ります。

### 4. 外す（関連付け解除）

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"id": 3, "disassociate": true}' \
  "$AWX/api/v2/job_templates/10/instance_groups/"
```

## 注意点

- **置き換え**: PATCH による一括置換はできません。既存のグループを `disassociate` してから、新しいグループを関連付けます。
- **優先順位**: 複数設定した場合は、関連付けた順番が優先順位になります。順番を変えたいときは、一度すべて外してから優先したい順に関連付け直します。
- **権限**: トークンのユーザーには、ジョブテンプレートの Admin 権限と、対象インスタンスグループの Use 権限が必要です。不足していると `403` になります。

## 補足

### 実行時だけインスタンスグループを指定する

テンプレート自体は変更せずに、その実行だけ別のインスタンスグループを使う方法です（比較的新しい AWX で利用可能）。テンプレートで `ask_instance_groups_on_launch` を `true` にしておき、起動時に指定します。

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"instance_groups": [3]}' \
  "$AWX/api/v2/job_templates/10/launch/"
```

### Ansible で宣言的に管理する

`awx.awx.job_template` モジュールの `instance_groups` パラメータでも設定できます。
