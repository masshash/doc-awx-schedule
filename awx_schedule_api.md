# AWX スケジュールを API から設定する

## API から設定する例（アクセストークン使用）

curl でアクセストークン（Bearer）を使ってスケジュールを作成する例です。`AWX_HOST`、`AWX_TOKEN`、ジョブテンプレート ID は環境に合わせて置き換えてください。

### 1. 事前にルールをプレビューする（任意）

登録前に、実際にどの時刻で実行されるかを確認できます。

```bash
AWX_HOST="https://awx.example.com"
AWX_TOKEN="xxxxxxxxxxxxxxxxxxxx"

RRULE='DTSTART;TZID=Asia/Tokyo:20261012T090000 RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=9,10,11,12,13,14,15;BYMINUTE=0,30 RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=16;BYMINUTE=0'

curl -s -X POST "${AWX_HOST}/api/v2/schedules/preview/" \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"rrule\": \"${RRULE}\"}"
```

レスポンスの `local` に直近の実行予定時刻が並ぶので、09:00〜16:00 の 30 分おきになっているか確認します。

### 2. スケジュールを作成する

ジョブテンプレート ID が `10` の場合の例です。

```bash
JT_ID=10

curl -s -X POST "${AWX_HOST}/api/v2/job_templates/${JT_ID}/schedules/" \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
        \"name\": \"daytime-every-30min\",
        \"description\": \"毎日 09:00-16:00 に 30 分おき実行\",
        \"rrule\": \"${RRULE}\",
        \"enabled\": true
      }"
```

`/api/v2/schedules/` に POST し、ボディに `"unified_job_template": 10` を含める形でも同じように作成できます。レスポンスの `id` がスケジュール ID、`next_run` が次回実行時刻（UTC 表記）です。

### 3. 作成後の確認・変更

```bash
SCHEDULE_ID=25

# 内容と次回実行時刻の確認
curl -s "${AWX_HOST}/api/v2/schedules/${SCHEDULE_ID}/" \
  -H "Authorization: Bearer ${AWX_TOKEN}"

# 時間帯などを変更する（rrule を差し替え）
curl -s -X PATCH "${AWX_HOST}/api/v2/schedules/${SCHEDULE_ID}/" \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"rrule": "DTSTART;TZID=Asia/Tokyo:20261012T090000 RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=9,10,11,12,13,14,15;BYMINUTE=0,30;BYDAY=MO,TU,WE,TH,FR"}'

# 一時停止（無効化）
curl -s -X PATCH "${AWX_HOST}/api/v2/schedules/${SCHEDULE_ID}/" \
  -H "Authorization: Bearer ${AWX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'
```

### 補足

- トークンには、対象ジョブテンプレートに対して**スケジュールを作成できる権限**（通常は Execute 以上＋スケジュール作成が可能なロール）を持つユーザーのものを使ってください。スコープは `Write` が必要です。
- `DTSTART` の日付は開始日です。過去日付でも構いませんが、その日の 09:00 が最初の基準になります。
- 古い AWX で複数の `RRULE` を 1 つにまとめられずエラーになる場合は、16:00 の分だけ別スケジュールとして同じ要領で POST してください。

---

## RRULE の記載内容の詳細解説

例の RRULE を分解して、1 つずつ説明します。

```
DTSTART;TZID=Asia/Tokyo:20261012T090000 RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=9,10,11,12,13,14,15;BYMINUTE=0,30 RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=16;BYMINUTE=0
```

### 全体の構造

この文字列は、スペースで区切られた 3 つの部品でできています。

1. `DTSTART;TZID=Asia/Tokyo:20261012T090000` … いつから・どのタイムゾーンで数えるか
2. `RRULE:FREQ=DAILY;...;BYMINUTE=0,30` … ルール 1（09:00〜15:30 の 30 分おき）
3. `RRULE:FREQ=DAILY;...;BYMINUTE=0` … ルール 2（16:00 の 1 回）

本来の iCalendar 形式では改行で区切りますが、AWX では 1 行にスペース区切りで書く形式を使います。各 RRULE の中は `項目=値` を `;`（セミコロン）でつなぎ、値が複数あるときは `,`（カンマ）でつなぎます。

### 1. DTSTART（開始日時）

`DTSTART;TZID=Asia/Tokyo:20261012T090000`

- **`DTSTART`** … スケジュールの起点となる日時です。AWX では必須です。
- **`;TZID=Asia/Tokyo`** … 時刻をどのタイムゾーンで解釈するかの指定です。これがあるので、以降の「9 時」「16 時」はすべて日本時間として扱われます。省略して末尾に `Z` を付けると UTC 扱いになり、日本時間では 9 時間ずれるので注意してください。
- **`:20261012T090000`** … 日時本体で、`YYYYMMDD` + `T` + `HHMMSS` の形式です。この例は「2026 年 10 月 12 日 09:00:00」です。この日から実行が始まります。

また、ルールで指定しなかった要素は DTSTART から引き継がれます。例えば秒は指定していないので、DTSTART の「00 秒」が使われます。

### 2. ルール 1（09:00〜15:30 を 30 分おき）

`RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=9,10,11,12,13,14,15;BYMINUTE=0,30`

- **`RRULE:`** … ここから繰り返しルールが始まる、という目印です。
- **`FREQ=DAILY`** … 繰り返しの基本単位で「日ごと」です。ほかに `MINUTELY`（分）、`HOURLY`（時）、`WEEKLY`（週）、`MONTHLY`（月）などがあります。
- **`INTERVAL=1`** … 基本単位を何個おきに繰り返すかです。`1` なので「毎日」。`2` にすると「1 日おき」になります。省略時の既定値も 1 ですが、AWX では明記しておくのが無難です。
- **`BYHOUR=9,10,11,12,13,14,15`** … その日のうち、何時台に実行するかの一覧です（0〜23 で指定）。範囲指定（9-15 のような書き方）はできないので、1 つずつ列挙します。
- **`BYMINUTE=0,30`** … その時間のうち、何分に実行するかの一覧です（0〜59 で指定）。

ポイントは、**BYHOUR と BYMINUTE はすべての組み合わせに展開される**ことです。7 つの時 × 2 つの分 = 14 通りになります。

| 時 | 0 分 | 30 分 |
|---|---|---|
| 9 | 09:00 | 09:30 |
| 10 | 10:00 | 10:30 |
| 11 | 11:00 | 11:30 |
| 12 | 12:00 | 12:30 |
| 13 | 13:00 | 13:30 |
| 14 | 14:00 | 14:30 |
| 15 | 15:00 | 15:30 |

### 3. ルール 2（16:00 の 1 回）

`RRULE:FREQ=DAILY;INTERVAL=1;BYHOUR=16;BYMINUTE=0`

意味は「毎日、16 時 0 分」です。

なぜ別ルールにしているかというと、ルール 1 の BYHOUR に 16 を足すと、組み合わせ展開によって 16:00 だけでなく **16:30 も生成されてしまう**からです。16:00 だけを足すために、独立したルールにしています。

### 4. 複数ルールの合成

RRULE が複数ある場合、**各ルールが生成した時刻をすべて合わせたもの（和集合）**が実行時刻になります。結果として、1 日の実行時刻は次の 15 回です。

09:00, 09:30, 10:00, 10:30, 11:00, 11:30, 12:00, 12:30, 13:00, 13:30, 14:00, 14:30, 15:00, 15:30, 16:00

### よく使う追加オプション

- **`BYDAY=MO,TU,WE,TH,FR`** … 実行する曜日を限定します（MO=月, TU=火, WE=水, TH=木, FR=金, SA=土, SU=日）。平日のみにしたいときは、両方のルールに付けます。
- **`UNTIL=20261231T150000Z`** … この日時で繰り返しを終了します。AWX では UTC（末尾 `Z`）で書くのが一般的で、この例は日本時間 2027 年 1 月 1 日 0:00 にあたります。
- **`COUNT=100`** … 実行回数の上限です。UNTIL とは同時に使えません。

平日のみ・年末で終了、とする場合はこのようになります。

```
DTSTART;TZID=Asia/Tokyo:20261012T090000 RRULE:FREQ=DAILY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR;BYHOUR=9,10,11,12,13,14,15;BYMINUTE=0,30;UNTIL=20261231T150000Z RRULE:FREQ=DAILY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR;BYHOUR=16;BYMINUTE=0;UNTIL=20261231T150000Z
```

書き換えたときは、`/api/v2/schedules/preview/` で実行予定を確認してから登録すると確実です。
