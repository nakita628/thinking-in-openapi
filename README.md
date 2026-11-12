# thinking-in-openapi

OpenAPI の文法と、それを使った API 設計のベストプラクティスをまとめます。
対象は OpenAPI 3.1 です。3.0 と異なる点は「3.0」と注記します。

## 0. 書き始める前に

OpenAPI は設計の結論を書く場所です。先に次を決めます。

- 誰が使うか（コンシューマ、エンドユーザー）
- 何をしたいか（ゴール）
- そのために何が必要で、何が返るか（入力と出力）

ゴールから OpenAPI への対応は次のとおりです。

| 決めたこと | OpenAPI の置き場所             |
| ---------- | ------------------------------ |
| 誰が       | `security` のスコープ          |
| 何を       | `tags`                         |
| ゴール     | パス＋HTTP メソッド、`summary` |
| 入力       | `parameters`、`requestBody`    |
| 出力       | `responses`                    |

- テーブル名や内部モジュールの形をそのまま `paths` にしない。
- コンシューマの言葉で書く。内部の仕組みは書かない。

## 1. ドキュメントの構造

```yaml
openapi: 3.1.0
info: {}
jsonSchemaDialect: https://spec.openapis.org/oas/3.1/dialect/base
servers: []
security: []
tags: []
paths: {}
webhooks: {}
components: {}
externalDocs: {}
```

| フィールド   | 必須 | 内容               |
| ------------ | ---- | ------------------ |
| `openapi`    | ○    | 仕様のバージョン   |
| `info`       | ○    | API の情報         |
| `paths`      | △    | エンドポイント     |
| `webhooks`   | △    | API が送るイベント |
| `components` | △    | 再利用する部品     |
| `servers`    |      | 接続先             |
| `security`   |      | 既定の認証         |
| `tags`       |      | 分類と並び順       |

- △は「少なくとも1つ必須」である。3.0 では `paths` が必須で、`webhooks` はない。
- フォーマットは YAML にする。コメントが書け、差分が読みやすい。
- 大きくなったらファイルを分割し、`$ref` で参照する。

## 2. info

```yaml
info:
  title: Banking API
  version: 1.2.0
  summary: 口座と振替を扱う
  description: |
    ## ユースケース
    1. 依頼人を選ぶ
    2. 受取人を選ぶ
    3. 金額を指定して振り替える
  contact:
    name: Banking API チーム
    url: https://developer.example.com
  license:
    name: Apache 2.0
    identifier: Apache-2.0
```

- `openapi` は仕様のバージョン、`info.version` は API のバージョンである。
- `info.version` はセマンティックバージョニングにする。
- `description` は Markdown で、ユースケース・認証方法・共通ルール・変更ログを書く。
- `summary` と `license.identifier` は 3.1 から使える。

## 3. servers

```yaml
servers:
  - url: https://api.example.com/v1
    description: 本番
  - url: https://{env}.example.com/v1
    description: 検証
    variables:
      env:
        enum:
          - staging
          - sandbox
        default: sandbox
```

- バージョンをパスに含めるなら、`servers.url` に書く。各パスには書かない。
- 変数には `enum` と `default` を付ける。

## 4. paths

```yaml
paths:
  /accounts:
    get: {}
  /accounts/{accountId}:
    parameters:
      - $ref: "#/components/parameters/accountId"
    get: {}
    delete: {}
```

| 良い例                               | 避ける例                   | 理由                       |
| ------------------------------------ | -------------------------- | -------------------------- |
| `/accounts`                          | `/account`、`/getAccounts` | コレクションは複数形の名詞 |
| `/accounts/{accountId}`              | `/accounts/{id}`           | 何の ID かを名前で示す     |
| `/delayed-transfers`                 | `/delayedTransfers`        | kebab-case                 |
| `/accounts/{accountId}/transactions` | 4階層以上のネスト          | 階層は浅くする             |

- 末尾のスラッシュは付けない。
- 同じ形のパスを変数名だけ変えて並べない（`/a/{x}` と `/a/{y}`）。エラーになる。
- 動詞のパス（`POST /cart/check-out`）は最後の手段にする。
- 氏名などのセンシティブな値をパスやクエリに置かない。URL はログに残る。

## 5. オペレーション

```yaml
paths:
  /transfers:
    post:
      operationId: createTransfer
      summary: 振替
      description: |
        `source` から `destination` へ `amount` を送金する。
        `date` を指定すると期日指定振替になる。
      tags:
        - Transfers
      security:
        - oauth:
            - transfers:write
      requestBody: {}
      responses: {}
```

| HTTP メソッド | 用途                         | ボディ |
| ------------- | ---------------------------- | ------ |
| GET           | 取得・検索                   | なし   |
| POST          | 作成、他に当てはまらない操作 | あり   |
| PUT           | 全体の置き換え               | あり   |
| PATCH         | 一部の更新                   | あり   |
| DELETE        | 削除・キャンセル             | なし   |

- `operationId` は API 全体で一意にし、camelCase の「動詞＋名詞」にする。コード生成で関数名になる。
- `summary` は短いゴール名にする。メニューに表示される。
- `description` に振る舞い・前提・制約を書く。
- `tags` は1つにする。複数付けるとメニューが重複する。
- メソッドの並び順を全パスでそろえる。

## 6. parameters

```yaml
components:
  parameters:
    accountId:
      name: accountId
      in: path
      required: true
      description: 10桁の口座番号
      schema:
        type: string
        pattern: ^\d{10}$
    pageSize:
      name: pageSize
      in: query
      required: false
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
```

| `in`     | 用途                         | 注意                                                 |
| -------- | ---------------------------- | ---------------------------------------------------- |
| `path`   | リソースの識別               | `required: true` が必須                              |
| `query`  | 絞り込み・ページング・ソート | オブジェクトを渡さない                               |
| `header` | 補助情報                     | `Accept`・`Content-Type`・`Authorization` は書かない |
| `cookie` | ブラウザ向け                 | API では避ける                                       |

配列の渡し方は `style` と `explode` で決まります。

| 書き方                                         | URL                  |
| ---------------------------------------------- | -------------------- |
| `style: form`、`explode: true`（query の既定） | `?status=a&status=b` |
| `style: form`、`explode: false`                | `?status=a,b`        |

- 共通のパラメータは `components.parameters` に置き、`$ref` で使う。
- パスパラメータはパス直下に置き、全メソッドに適用する。
- `required: false` も明示する。
- 範囲は `minimum`・`maximum`、既定値は `default` で書く。

## 7. requestBody

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: "#/components/schemas/TransferRequest"
      examples:
        immediate:
          summary: 当日振替
          value:
            source: "0001234567"
            destination: "0045678910"
            amount: 100.5
        delayed:
          summary: 期日指定振替
          value:
            source: "0001234567"
            destination: "0045678910"
            amount: 100.5
            date: "2030-06-09"
```

- `required: true` を書く。既定値は `false` である。
- スキーマは `$ref` にする。インラインだとコード生成で名前のない型になる。
- ユースケースごとに `examples` を用意する。

## 8. responses

```yaml
responses:
  "201":
    description: 振替を実行した
    headers:
      Location:
        description: 作成された振替の URL
        schema:
          type: string
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/Transfer"
  "202":
    description: 期日指定振替を受け付けた
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/Transfer"
  "400":
    $ref: "#/components/responses/BadRequest"
  "401":
    $ref: "#/components/responses/Unauthorized"
  "403":
    $ref: "#/components/responses/Forbidden"
```

| コード | 使う場面                                     |
| ------ | -------------------------------------------- |
| 200    | 取得・更新に成功                             |
| 201    | 作成した（`Location` を返す）                |
| 202    | 受け付けた、後で処理する                     |
| 204    | 成功、返すものがない（`content` を書かない） |
| 400    | リクエストが不正、ビジネスルール違反         |
| 401    | 認証されていない                             |
| 403    | スコープや権限が足りない                     |
| 404    | リソースがない、または見せられない           |
| 409    | 状態が競合している                           |
| 422    | 形式は正しいが処理できない                   |
| 429    | リクエストが多すぎる                         |
| 500    | サーバーの想定外のエラー                     |

- ステータスコードは文字列（`"201"`）で書く。
- `description` は必須である。
- 成功だけでなく、起こりうるエラーをすべて書く。
- 共通のエラーは `components.responses` に置く。
- `default` は最後の受け皿にし、具体的なコードの代わりにしない。

## 9. スキーマ（JSON Schema）

### 9.1 型

| 値                      | 書き方                                      |
| ----------------------- | ------------------------------------------- |
| 文字列                  | `type: string`                              |
| 整数                    | `type: integer`（`format: int32`／`int64`） |
| 数値                    | `type: number`（`format: float`／`double`） |
| 真偽値                  | `type: boolean`                             |
| 配列                    | `type: array`、`items`                      |
| オブジェクト            | `type: object`、`properties`                |
| 日付                    | `type: string`、`format: date`              |
| 日時                    | `type: string`、`format: date-time`         |
| URL                     | `type: string`、`format: uri`               |
| 先頭の0に意味がある番号 | `type: string`、`pattern`                   |
| 区分                    | `type: string`、`enum`                      |
| 通貨・国・言語          | ISO 4217・ISO 3166・BCP 47 のコード         |

- 日時に UNIX タイムスタンプを使わない。
- 区分に数値コードを使わない（`2` ではなく `checking`）。

### 9.2 制約

| キーワード               | 対象         | 例                                 |
| ------------------------ | ------------ | ---------------------------------- |
| `minLength`／`maxLength` | 文字列       | `maxLength: 140`                   |
| `pattern`                | 文字列       | `pattern: ^\d{10}$`                |
| `minimum`／`maximum`     | 数値         | `minimum: 0`                       |
| `exclusiveMinimum`       | 数値         | `exclusiveMinimum: 0`              |
| `minItems`／`maxItems`   | 配列         | `maxItems: 100`                    |
| `uniqueItems`            | 配列         | `uniqueItems: true`                |
| `required`               | オブジェクト | 必須プロパティの一覧               |
| `additionalProperties`   | オブジェクト | `false` で未定義のプロパティを禁止 |

- 3.0 の `exclusiveMinimum` は真偽値で、`minimum` と組み合わせる。
- 文字列と配列には上限を付ける。上限のない入力は約束できない。

### 9.3 null

```yaml
# 3.1
nickname:
  type:
    - string
    - "null"

# 3.0
nickname:
  type: string
  nullable: true
```

- 「値がない」はプロパティの省略で表し、`null` は意味がある場合だけ使う。

### 9.4 説明と例

```yaml
amount:
  type: number
  exclusiveMinimum: 0
  description: 振替金額。通貨は口座の通貨に従う。
  examples:
    - 100.5
```

- 3.1 のスキーマでは `examples`（配列）を使う。`example` は非推奨である。
- 名前で自明なプロパティに説明は書かない（`amount: 金額` は不要）。

### 9.5 入力と出力の共有

```yaml
Transfer:
  type: object
  required:
    - source
    - destination
    - amount
  properties:
    id:
      type: string
      readOnly: true
    status:
      type: string
      enum:
        - pending
        - executed
      readOnly: true
    source:
      type: string
    destination:
      type: string
    amount:
      type: number
```

- `readOnly` はレスポンスだけ、`writeOnly` はリクエストだけに現れる。
- 差が大きいならスキーマを分ける（`TransferRequest` と `Transfer`）。

### 9.6 組み合わせ

| キーワード | 意味                                  |
| ---------- | ------------------------------------- |
| `allOf`    | すべてを満たす（継承・拡張）          |
| `oneOf`    | ちょうど1つを満たす（種類の切り替え） |
| `anyOf`    | 1つ以上を満たす                       |
| `not`      | 満たさない                            |

```yaml
Payment:
  oneOf:
    - $ref: "#/components/schemas/CardPayment"
    - $ref: "#/components/schemas/BankPayment"
  discriminator:
    propertyName: method
    mapping:
      card: "#/components/schemas/CardPayment"
      bank: "#/components/schemas/BankPayment"
```

- `oneOf` には `discriminator` を付け、どの型かを判別できるようにする。
- 判別用のプロパティ（`method`）は各スキーマで `required` にする。
- 組み合わせは浅く保つ。深いとコード生成と読解が難しくなる。

### 9.7 設計のルール

- スキーマ名は PascalCase、プロパティ名は camelCase にする。
- 型を表す接尾辞（`Dto`、`Bln`）や略語を使わない。
- 同じ概念には API 全体で同じ名前・型・フォーマットを使う。
- `required` を必ず書く。すべてオプションのスキーマは疑う。
- 関連するプロパティはオブジェクトにまとめる。入れ子は3階層までにする。
- 一覧は配列を直接返さず、`items` を持つオブジェクトで返す。

```yaml
TransferList:
  type: object
  required:
    - items
  properties:
    items:
      type: array
      items:
        $ref: "#/components/schemas/Transfer"
    nextCursor:
      type: string
```

## 10. エラー

```yaml
components:
  responses:
    BadRequest:
      description: リクエストに誤りがある
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
  schemas:
    Problem:
      type: object
      required:
        - type
        - title
        - status
      properties:
        type:
          type: string
          format: uri
        title:
          type: string
        status:
          type: integer
        detail:
          type: string
        errors:
          type: array
          items:
            $ref: "#/components/schemas/ProblemItem"
    ProblemItem:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          examples:
            - MISSING_MANDATORY_PROPERTY
        pointer:
          type: string
          description: 原因の JSON Pointer
          examples:
            - /amount
        message:
          type: string
```

- 形式は RFC 9457（Problem Details）に合わせる。
- 複数のエラーは配列でまとめて返す。
- エラーコードは汎用的な名前にし、一覧を `description` に書く。
- スタックトレース・サーバー名・内部コードを含めない。
- 権限のないリソースは 403 ではなく 404 にし、存在を漏らさない。

## 11. セキュリティ

```yaml
components:
  securitySchemes:
    oauth:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            accounts:read: 口座と取引の参照
            transfers:write: 振替の実行とキャンセル
security:
  - oauth:
      - accounts:read
paths:
  /health:
    get:
      security: []
```

| `type`          | 用途                              |
| --------------- | --------------------------------- |
| `oauth2`        | 委譲された認可（スコープを使う）  |
| `openIdConnect` | 認証（`openIdConnectUrl` を指定） |
| `http`          | `scheme: bearer` や `basic`       |
| `apiKey`        | API キー（ヘッダーで渡す）        |
| `mutualTLS`     | クライアント証明書（3.1 から）    |

- トップレベルの `security` を既定にし、必要なオペレーションだけ上書きする。
- 認証不要のオペレーションは `security: []` で明示する。
- 配列の要素は OR、1要素内の複数スキームは AND である。
- スコープは「何を」単位にし、説明を付ける。同意画面に表示される。
- implicit フローは使わない。
- `apiKey` をクエリで渡さない。

## 12. tags

```yaml
tags:
  - name: Accounts
    description: 口座と取引の参照
  - name: Transfers
    description: 振替と受取人の管理
```

- 並び順がそのままメニューの順になる。重要なものから並べる。
- パスの形ではなく、ユーザーがしたいことで分類する。

## 13. 再利用と $ref

| 置き場所                     | 用途                   |
| ---------------------------- | ---------------------- |
| `components.schemas`         | データモデル           |
| `components.parameters`      | パラメータ             |
| `components.responses`       | 共通のレスポンス       |
| `components.requestBodies`   | 共通のリクエストボディ |
| `components.headers`         | 共通のヘッダー         |
| `components.examples`        | 共通の例               |
| `components.securitySchemes` | 認証方式               |
| `components.pathItems`       | 共通のパス（3.1 から） |

```yaml
# 同じファイル内
$ref: "#/components/schemas/Transfer"
```

```yaml
# 別ファイル
$ref: "./schemas/transfer.yaml"
```

```yaml
# 別ファイルの一部
$ref: "./common.yaml#/components/responses/BadRequest"
```

- `$ref` は `components` か外部ファイルを指す。
- 3.1 では `$ref` と同じ階層に `summary`・`description` を書ける。3.0 では無視される。
- 使われていないコンポーネントは削除する。

## 14. webhooks とコールバック

```yaml
webhooks:
  transferExecuted:
    post:
      summary: 振替の実行を通知する
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/TransferEvent"
      responses:
        "204":
          description: 受け取った
```

- `webhooks` は API が送るイベントを書く（3.1 から）。
- リクエストごとに登録される通知は `callbacks` で書く。
- イベントは軽く保ち、詳細は API で取得させる。
- 届く順序やリトライの方針は `description` に書く。

## 15. 進化

- 非推奨には `deprecated: true` を付け、代替手段を `description` に書く。
- 破壊的変更でメジャーバージョンを上げる。
- スキーマを変えるときは、入力と出力のどちらで使われるかを確認する。

| 変更                           | レスポンス       | リクエスト |
| ------------------------------ | ---------------- | ---------- |
| プロパティの削除・名前変更     | 破壊的           | 破壊的     |
| 型・フォーマットの変更         | 破壊的           | 破壊的     |
| `required` から外す            | 破壊的           | 安全       |
| `required` に加える            | 安全             | 破壊的     |
| `enum` に値を加える            | 破壊的になりうる | 安全       |
| `maxLength` を大きくする       | 破壊的になりうる | 安全       |
| オプションのプロパティを加える | 安全             | 安全       |
| オペレーションを加える         | 安全             | 安全       |

- 拡張しやすい形にしておく。真偽値より `status` の文字列、配列よりオブジェクト。

## 16. 拡張

```yaml
x-implementation:
  source: core-banking.ZBAL0.RTBAL
```

- 独自情報は `x-` で始まるプロパティに書く。標準パーサーは無視する。
- 実装者向けの情報は、公開前に取り除く。

## 17. OpenAPI に書けないこと

| 書けないこと             | 補う方法                       |
| ------------------------ | ------------------------------ |
| 呼び出しの順序（フロー） | `info.description`、図         |
| パーミッション制御       | `description`、実装者向け資料  |
| キャッシュの方針         | `Cache-Control` ヘッダーの説明 |
| 応答時間や並び順の保証   | 保証する／しないを明記する     |

- 書けないことは文章で補う。書かなかった振る舞いにも、コンシューマは依存する。

## 18. 書く手順

| 手順 | やること                                               | 書く場所                         |
| ---- | ------------------------------------------------------ | -------------------------------- |
| 1    | 誰が・何を・どのようにを表にする                       | ゴールキャンバス（OpenAPI の外） |
| 2    | ゴールの名詞からリソース、動詞からアクションを取り出す | メモ                             |
| 3    | リソースをパス、アクションを HTTP メソッドにする       | `paths`                          |
| 4    | `operationId`・`summary`・`tags` を付ける              | オペレーション                   |
| 5    | 入力と出力のスキーマを作る                             | `components.schemas`             |
| 6    | パラメータとボディを決める                             | `parameters`、`requestBody`      |
| 7    | 成功とエラーのレスポンスをすべて書く                   | `responses`                      |
| 8    | スコープを決めて割り当てる                             | `securitySchemes`、`security`    |
| 9    | 説明と例を足す                                         | `description`、`examples`        |
| 10   | リンターにかける                                       | CI                               |
| 11   | プロバイダとコンシューマの視点でレビューする           | プルリクエスト                   |

- 手順は行き来してよい。新しい発見があれば前に戻る。
- 手順1を飛ばさない。飛ばすと内部の形がそのまま API になる。

## 19. ユースケース別の書き方

### 19.1 一覧を返す（ページング）

- メソッドは GET、パスはコレクション（`/transfers`）にする。
- 件数が増え続けるデータは、カーソル方式にする。ページ番号方式は途中の追加で重複する。
- レスポンスはオブジェクトで返し、`items` と次の位置を含める。

```yaml
/transfers:
  get:
    operationId: listTransfers
    parameters:
      - name: cursor
        in: query
        schema:
          type: string
      - name: limit
        in: query
        schema:
          type: integer
          minimum: 1
          maximum: 100
          default: 20
    responses:
      "200":
        description: 振替の一覧
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/TransferList"
```

### 19.2 絞り込みと並べ替え

- 絞り込みはプロパティ名と同じ名前のクエリパラメータにする（`?status=pending`）。
- 範囲は `From`／`To` を付ける（`createdFrom`、`createdTo`）。
- 並べ替えは `sort` 1つにまとめ、`-` で降順を表す（`?sort=-amount,createdAt`）。
- 検索条件にセンシティブな値が入るなら、`POST /transfers/search` にしてボディで渡す。

### 19.3 作成する

- `POST /コレクション` にする。
- すぐ作成できたら 201 と `Location`、後で処理するなら 202 を返す。
- 二重送信を防ぐなら、`Idempotency-Key` ヘッダーを受け取る。

```yaml
parameters:
  - name: Idempotency-Key
    in: header
    required: false
    description: 同じキーの再送は1回だけ処理する
    schema:
      type: string
      maxLength: 64
```

### 19.4 一部を更新する

- `PATCH /コレクション/{id}` にする。
- ボディのプロパティはすべてオプションにし、空のボディを禁止する。
- 更新できないプロパティ（`id`、`createdAt`）を含めない。

```yaml
UpdateTransferRequest:
  type: object
  minProperties: 1
  additionalProperties: false
  properties:
    comment:
      type: string
      maxLength: 140
```

### 19.5 削除・キャンセルする

- `DELETE /コレクション/{id}` にし、成功は 204 にする。
- 物理削除でなくても DELETE を使ってよい（キャンセル、無効化）。
- すでに処理済みで消せないなら 409 を返す。

### 19.6 CRUD に当てはまらない操作

1. 状態の変化として表せないかを考える（`PATCH` で `status` を変える）。
2. 作成として表せないかを考える（チェックアウト → `POST /orders`）。
3. どちらも不自然なら、アクションリソースにする（`POST /transfers/{id}/cancel`）。

- 3 を選ぶときは、コンシューマにとってわかりやすいかを優先する。

### 19.7 時間のかかる処理

- 202 を返し、状態を確認できるリソースを用意する。
- 状態は `enum` の文字列にする（`pending`・`validated`・`executed`）。
- 完了の通知が必要なら `webhooks` も書く。

```yaml
"202":
  description: 受け付けた。status で進捗を確認する。
  headers:
    Location:
      description: 状態を確認する URL
      schema:
        type: string
```

### 19.8 まとめて処理する

- 1件ずつのリクエストと同じデータに `id` を足したものを、`items` で受け取る。
- 件数に上限（`maxItems`）を付ける。
- 一部だけ成功しうるなら 207 を返し、要素ごとに `status` と `body` を返す。
- 一部の成功が許されないなら、全件成功で 200、1件でも失敗なら 400 にする。

### 19.9 ファイルを受け取る

```yaml
requestBody:
  required: true
  content:
    multipart/form-data:
      schema:
        type: object
        required:
          - file
        properties:
          file:
            type: string
            contentMediaType: application/pdf
          description:
            type: string
```

- 3.0 では `format: binary` で表す。
- サイズの上限は `description` に書き、超えたら 413 を返す。

### 19.10 ファイルを返す

```yaml
"200":
  description: 取引明細の PDF
  content:
    application/pdf: {}
```

- 3.1 ではバイナリのスキーマを省略できる。3.0 では `type: string`、`format: binary` を書く。

### 19.11 複数のフォーマットで返す

- 同じパスで、`content` に複数のメディアタイプを並べる。
- コンシューマは `Accept` ヘッダーで選ぶ。対応しないものには 406 を返す。

```yaml
content:
  application/json:
    schema:
      $ref: "#/components/schemas/TransactionList"
  text/csv:
    schema:
      type: string
```

### 19.12 キャッシュと条件付きリクエスト

- レスポンスに `ETag` と `Cache-Control` を書く。
- リクエストで `If-None-Match` を受け取り、変化がなければ 304 を返す。
- キャッシュしてよい期間は、データの変わりやすさで決めて `description` に書く。

### 19.13 レート制限

- 超えたら 429 を返し、`Retry-After` ヘッダーで再試行の時期を伝える。
- 上限は `info.description` に書く。

### 19.14 多言語対応

- `Accept-Language` をヘッダーパラメータで受け取り、`Content-Language` を返す。
- 対応しない言語には既定の言語で返すか 406 を返し、どちらかを明記する。

### 19.15 センシティブなデータ

- パスワードなど入力専用の値は `writeOnly: true` にする。
- カード番号などは末尾4桁にするか、返さない。
- 本当に必要なら専用のオペレーションと専用のスコープにする。

### 19.16 非推奨にする

- `deprecated: true` を付け、代替手段と廃止予定日を `description` に書く。
- 廃止予定日は `Sunset` ヘッダー（RFC 8594）でも返す。
- 変更ログに記録する。

## 20. チェックリスト

本書の第13章に従い、レビューを5段階に分けます。
本書にない項目には「（独自）」を付けています。

| 段階                    | いつ               | 見るもの                     |
| ----------------------- | ------------------ | ---------------------------- |
| 20.1 ニーズ             | OpenAPI を書く前   | 何を作るべきか               |
| 20.2 リント             | OpenAPI を書いた後 | 形式・ガイドライン・一貫性   |
| 20.3 プロバイダの視点   | リントの後         | ニーズを満たし、実装できるか |
| 20.4 コンシューマの視点 | リントの後         | 理解しやすく使いやすいか     |
| 20.5 実装の検証         | 実装の後           | OpenAPI どおりに動くか       |

### 20.1 ニーズ（表13-1）

#### 20.1 ニーズの分析

- [ ] 何をしたいか、コンテキストは何かを確認した
- [ ] 「それ」がどう使われるかを確認した
- [ ] ターゲットコンシューマが誰かを確認した（業界規格・セキュリティに影響）
- [ ] ターゲットエンドユーザーが誰かを確認した（国際化・セキュリティに影響）
- [ ] 「5つのなぜ」で本当のニーズを掘り下げた
- [ ] ゴールキャンバスを埋めた

#### 20.1 コンシューマのコンテキスト

- [ ] コンシューマの制約を確認した（XML しか使えない、PATCH が使えないなど）
- [ ] 業界規格の有無と、ターゲットが採用しているかを確認した
- [ ] ネットワーク環境を確認した（BFF やエクスペリエンス API が要るか）

#### 20.1 プロバイダのコンテキスト

- [ ] 依存するシステム・API・チーム・パートナーを洗い出した
- [ ] 技術的な制限を確認した（24時間動かない、非同期のみ、遅い、スケールしない）
- [ ] 機能的な制限を確認した（既存のビジネスルールがニーズに合わない）
- [ ] 人間の関与する処理がないかを確認した
- [ ] 必要なコミュニケーションの種類を決めた
- [ ] 必要な API の種類を決めた（REST、gRPC、GraphQL、イベント）
- [ ] API がソリューションか、その一部にすぎないかを確認した

#### 20.1 セキュリティ

- [ ] センシティブなデータやアクションを扱うかを確認した

### 20.2 リント（表13-2）

#### 20.2 全体

- [ ] 破壊的変更がないか、あるなら本当に必要でバージョニングポリシーを適用した
- [ ] 文法エラーがない（独自）
- [ ] `info.title`・`info.version`・`info.description` がある（独自）

#### 20.2 セキュリティ定義

- [ ] 各セキュリティ定義が使われている（未使用なら、付け忘れか確認して削除する）
- [ ] セキュリティタイプに応じた情報（フロー、URL）がガイドラインどおりにある
- [ ] 各スコープに適切な説明がある

#### 20.2 ゴールフロー

- [ ] フロー（ユースケース）と振る舞いがガイドラインに準拠し、既存の要素と一貫している
- [ ] フローが `info.description` かゴールの説明に明確に書かれている

#### 20.2 モデル（`components.schemas`）

- [ ] 各モデルが使われている（未使用なら、付け忘れか確認して削除する）
- [ ] 名前がガイドラインに準拠し、既存の要素と一貫している
- [ ] 構造と深さがガイドラインに準拠し、既存の要素と一貫している
- [ ] 必要な所に正確な `description` がある

#### 20.2 モデルのプロパティ

- [ ] 名前がガイドラインに準拠し、既存の要素と一貫している
- [ ] 必須かどうか（`required`）がガイドラインに準拠し、既存の要素と一貫している
- [ ] すべてオプションになっていない（なっているなら理由がある）
- [ ] 型とフォーマットが名前・値・説明に合い、既存の要素と一貫している
- [ ] `description` と制約（`minimum`、`maxLength` など）が正確で、一貫している

#### 20.2 ゴール（オペレーション）

- [ ] `tags` で分類されている
- [ ] パスのフォーマットがガイドラインに準拠し、既存の要素と一貫している
- [ ] パスが成功のレスポンスの型に合っている（コレクションなら一覧を返すなど）
- [ ] パスパラメータが定義されている
- [ ] HTTP メソッドがゴールにふさわしく、既存の要素と一貫している
- [ ] 成功とエラーに必要なレスポンスがすべてある
- [ ] セキュリティでカバーされている
- [ ] `operationId` が一意で命名規則に従っている（独自）

#### 20.2 ゴールのパラメータ

- [ ] 名前がガイドラインに準拠し、既存の要素と一貫している
- [ ] 必須かどうかが位置に合っている（パスパラメータは必須）
- [ ] 作成・置換のボディがすべてオプションになっていない（なっているなら理由がある）
- [ ] 型とフォーマットが位置に合っている（クエリにオブジェクトを渡さない、パスに URL で禁止された文字を含めない）
- [ ] `description` と制約が正確で、一貫している
- [ ] 位置が HTTP メソッドに合っている（DELETE にボディを付けない）
- [ ] コンシューマが値を用意できる（知っているか、別のゴールから得られる）

#### 20.2 ゴールのレスポンス

- [ ] ステータスコードがレスポンスの種類（成功、コンシューマのエラー、プロバイダのエラー）に合っている
- [ ] ボディの型とフォーマットがステータスコードに合っている（4XX・5XX の標準エラーモデルなど）
- [ ] ヘッダーの名前・型・フォーマットが一貫している
- [ ] 返すデータが実際に使われる
- [ ] 成功の `description` とスキーマが正確で、一貫している
- [ ] エラーの `description` に起こりうるエラーの詳細がある（可能ならマシンリーダブルにする）

#### 20.2 特に注意する検証

- [ ] ドキュメントが包括的で、レビューに使える
- [ ] フローが実際につながる（各パラメータを前のゴールから得られる）
- [ ] 既存の要素と一貫している（住所・国・金額・電話番号の表し方がばらばらでない）
- [ ] 使える標準規格（ISO 8601、ISO 4217、E.164 など）を使っている

### 20.3 プロバイダの視点（表13-3）

#### 20.3 セキュリティ定義

- [ ] スコープがセキュリティ分割のニーズに合っている

#### 20.3 ゴールフロー

- [ ] 目的がニーズに合っている
- [ ] フロー全体のパフォーマンス推定値がニーズに合っている
- [ ] 将来の進化を考慮している

#### 20.3 モデル

- [ ] 構造がニーズに合い、期待するコンセプトを表している
- [ ] 将来の展開を考慮している

#### 20.3 モデルのプロパティ

- [ ] 必須かどうか・型とフォーマット・値がニーズに合っている
- [ ] センシティブなデータを不当に公開せず、安全な表現になっている
- [ ] 値を実装で（直接または依存先から）用意できる。パラメータなら受け取れる
- [ ] 将来の進化を考慮している

#### 20.3 ゴール

- [ ] 目的がニーズに合っている
- [ ] パスにセンシティブなデータがない
- [ ] 期待される成功レスポンスがすべて定義されている
- [ ] エラーが網羅されている（サーフェスコントロール、機能的、セキュリティ、技術的）
- [ ] 想定するスコープでカバーされ、不当なアクセスを許さない
- [ ] 実装できる（直接または依存先を通じて）
- [ ] パフォーマンス推定値がニーズに合っている
- [ ] 将来の進化を考慮している

#### 20.3 ゴールのパラメータ

- [ ] 必須かどうか・型とフォーマットがニーズに合っている
- [ ] センシティブなデータを不当に要求せず、安全な表現・位置で受け取る
- [ ] 位置がセキュリティ基準に合っている（URL にセンシティブなデータを置かない）
- [ ] 将来の進化を考慮している

#### 20.3 ゴールのレスポンス

- [ ] 成功のデータがニーズに合い、安全で、実装で用意できる
- [ ] エラーのデータがニーズに合い、網羅的で、安全で、実装で用意できる
- [ ] 将来の進化を考慮している

### 20.4 コンシューマの視点（表13-4）

#### 20.4 セキュリティ定義

- [ ] スコープがコンシューマにとって意味がある

#### 20.4 ゴールフロー

- [ ] エラーを防ぐゴールがある（依頼人ごとの有効な受取人を返すなど）
- [ ] ステップが可能な限り少ない
- [ ] パフォーマンス推定値がコンシューマのニーズに合っている（基本と複雑なユースケースの両方）
- [ ] 各ステップがコンシューマにとって意味がある

#### 20.4 モデル

- [ ] 構造・深さ・詳細さが理解しやすく使いやすい
- [ ] パラメータのプロパティが最小限である
- [ ] 名前がわかりやすい（`aDTO` ではなく `Account`）

#### 20.4 モデルのプロパティとパラメータ

- [ ] 必須のプロパティが最低限である
- [ ] 名前がわかりやすい（`ts` ではなく `source`）
- [ ] 型とフォーマットがわかりやすい（UNIX タイムスタンプではなく ISO 8601、`2` ではなく `CHECKING`）
- [ ] 値がすぐ使える（年齢の追加、ラベルの追加、テクニカル ID ではなく口座番号）
- [ ] コンシューマがパラメータを用意できる

#### 20.4 ゴール

- [ ] パスの構造・名前・パラメータがわかりやすい
- [ ] ゴールがコンシューマにとって意味がある
- [ ] パフォーマンス推定値がコンシューマのニーズに合っている

#### 20.4 ゴールのレスポンス

- [ ] 成功時に、必要なデータと情報的価値のあるデータ（何が起きたか、次に何をすべきか）を返す
- [ ] エラー時に、問題の解決を助ける包括的なフィードバックを返す

#### 20.4 全体

- [ ] 内部の仕組み（テーブル、モジュール、部署）が見えていない
- [ ] プロバイダの専門用語を使っていない
- [ ] API を知らない人になりきって読んだ
- [ ] チームの外の人や、実際のコンシューマにレビューしてもらった

### 20.5 実装の検証（13.3.5 項）

#### 20.5 セキュリティ

- [ ] 登録済みのコンシューマだけがアクセスできる
- [ ] 許可されたスコープの外では何もできない
- [ ] エンドユーザーのパーミッションどおりのデータだけを返す（他人の口座は返さない）
- [ ] センシティブなデータを不当に返さず、必要なら保護している

#### 20.5 コントラクト

- [ ] 生成された OpenAPI との比較は、アノテーションと無関係な要素に限って信頼している
- [ ] 実行時に振る舞いをテストした（必須プロパティがないと 400 になるか）
- [ ] 必須のプロパティが常に返る
- [ ] 最小値・最大値・要素数などの制約を守っている

#### 20.5 ネットワーク

- [ ] ファイアウォール・プロキシ・VIP・API ゲートウェイを含む経路でテストした
- [ ] DELETE などのメソッドがブロックされない
- [ ] 5XX が 500 に置き換わらない
- [ ] エラー時に HTML ではなく API のエラー形式が返る
- [ ] ゲートウェイがエラーを1つずつ返したり、独自形式に変えたりしない

### 20.6 公開前（独自）

- [ ] 公開版に `x-` の内部情報が残っていない
- [ ] 変更ログを更新した
- [ ] 非推奨の要素に `deprecated: true` と代替手段がある

形式のチェックはリンター（Spectral、Redocly CLI など）で自動化し、人は 20.1・20.3・20.4 に集中します。
