# バックエンド連携仕様書（JSON プロトコル）

**GdoMail** は、ユーザーが指定した任意のエンドポイント URL に対して **HTTP POST リクエスト（JSON 形式）** を送信するオープンな連携仕様を採用しています。

標準の Google Apps Script（GAS）だけでなく、本仕様に準拠した JSON レスポンスを返す仕組みを実装すれば、**Cloudflare Workers** や任意のサーバーレス環境・独自サーバーでもバックエンドとして動作させることができます。

* [← トップページ（導入手順・GASサンプルコード）へ戻る](./index.html)
* [プライバシーポリシー](./privacy.html)
* [サポート・お問い合わせ](./support.html)

<br>

> ⚠️ **重要（Important）**
> 
> 本拡張機能は現在 Google Apps Script（GAS）にのみ対応しています。
> Cloudflare Workers 等の他環境への対応は今後のアップデートで予定しています。
<br>

---

## 1. 通信の基本仕様

* **メソッド**: `POST`
* **リクエストヘッダー**: `Content-Type: application/json`
* **レスポンス形式**: `application/json`（HTTP ステータスコード `200` で JSON を返却）
* **認証方式**: リクエストボディ内の `authTokenHash`（パスワードの SHA-256 16進数ハッシュ値）または `authToken`（平文パスワード）による照合

---

## 2. アクション一覧（リクエストとレスポンス）

GdoMail から送信される POST リクエストは、JSON 内の `action` フィールドによって以下の **3種類** に分かれます。

### ① 設定値の取得（`action: "getConfig"`）
拡張機能の設定画面や Gmail 画面の初期化時に、添付ファイルサイズの上限値を取得するために送信されます。

#### リクエスト JSON 例
```json
{
  "action": "getConfig",
  "authTokenHash": "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"
}
```

#### レスポンス JSON（成功時）
```json
{
  "success": true,
  "maxAttachmentMb": 10
}
```

---

### ② 差出人リストの取得（`action: "getSenders"`）
バックエンド側に登録されている差出人（From）アドレス一覧と設定値を取得するために送信されます。

#### リクエスト JSON 例
```json
{
  "action": "getSenders",
  "authTokenHash": "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8"
}
```

#### レスポンス JSON（成功時）
```json
{
  "success": true,
  "senders": [
    {
      "email": "info@example.com",
      "name": "サポート窓口"
    },
    {
      "email": "sales@example.com",
      "name": "営業担当"
    }
  ],
  "maxAttachmentMb": 10
}
```

---

### ③ メールの送信（`action` 省略または未指定）
Gmail 画面から「独自ドメイン送信」を実行した際に送信されます。件名・本文・宛先・Base64エンコードされた添付ファイルが含まれます。

#### リクエスト JSON 例
```json
{
  "authTokenHash": "5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8",
  "senderEmail": "info@example.com",
  "senderName": "サポート窓口",
  "to": ["recipient@example.net"],
  "cc": ["cc@example.net"],
  "bcc": [],
  "subject": "お問い合わせありがとうございます",
  "isPlainText": false,
  "htmlContent": "<p>いつもお世話になっております。</p>",
  "textContent": "いつもお世話になっております。",
  "attachments": [
    {
      "name": "document.pdf",
      "type": "application/pdf",
      "base64": "JVBERi0xLjQKJ..."
    }
  ]
}
```

#### リクエスト各項目の説明

| キー名 | 型 | 説明 |
| :--- | :--- | :--- |
| `authTokenHash` | `string` | 設定された認証パスワードを SHA-256 でハッシュ化した小文字16進文字列 |
| `senderEmail` | `string` | 差出人（From）のメールアドレス |
| `senderName` | `string` | 差出人（From）の表示名 |
| `to` | `string[]` | 宛先（To）メールアドレスの配列 |
| `cc` | `string[]` | CC メールアドレスの配列（任意） |
| `bcc` | `string[]` | BCC メールアドレスの配列（任意） |
| `subject` | `string` | メールの件名 |
| `isPlainText` | `boolean` | プレーンテキストモードの場合は `true`、HTMLモードの場合は `false` |
| `htmlContent` | `string` | HTML形式のメール本文 |
| `textContent` | `string` | プレーンテキスト形式のメール本文 |
| `attachments` | `object[]` | 添付ファイル情報の配列（ファイル名 `name`、MIMEタイプ `type`、Base64文字列 `base64`） |

#### レスポンス JSON（成功時）
```json
{
  "success": true
}
```

---

## 3. 添付ファイルのサイズ制限仕様

GdoMail は、バックエンドやメール配信 API の制限超過を防ぐため、**拡張機能（フロントエンド）側とバックエンド側の双方で添付ファイル合計サイズの検証**を行います。

* **デフォルト上限値**: `10 MB`（バックエンドの `MAX_ATTACHMENT_MB` 設定により変更可能）
* **サイズ算出基準**: Base64 エンコード前の元データ（バイナリ）の合計バイト数で判定します。
  * バックエンド側で Base64 文字列から検証する場合は、パディング（`=` / `==`）を除いた元のバイト数 `Math.floor((base64.length * 3) / 4) - padding` を用いて計算します。
* **上限超過時の挙動**:
  1. **拡張機能側**: `getConfig` または `getSenders` で取得した `maxAttachmentMb` を超えるファイルが選択された場合、送信リクエストを行わず画面上に警告を表示します。
  2. **バックエンド側**: 万一上限を超えるリクエストを受信した場合はメール配信を実行せず、以下のエラー JSON を返却します。

```json
{
  "error": "添付ファイルの合計サイズが上限（10MB）を超えています。（現在: 12.34MB）"
}
```

## 4. エラー時のレスポンス仕様

認証失敗、サイズ上限超過、外部配信APIエラーなどが発生した場合は、以下の形式で `error` キーにエラーメッセージを含めて返却します。GdoMail 画面上にこのメッセージが表示されます。

```json
{
  "error": "Unauthorized"
}
```

---

## 5. Cloudflare Workers 等での独自実装について

Google Apps Script 以外の環境（Cloudflare Workers、AWS Lambda、独自サーバーなど）でバックエンドを構築する場合は、以下の要件を満たすように実装してください。

1. **POST リクエストの受信**: `Content-Type: application/json` のボディをパースし、`authTokenHash` を環境変数のパスワードハッシュと照合する。
2. **`getConfig` / `getSenders` への応答**: `data.action` に応じて、上記の JSON フォーマット（`success: true` 等）を返す。
3. **メール配信 API への転送**: `action` が指定されていない場合は、受け取った `senderEmail`・`to`・`subject`・`htmlContent`（または `textContent`）・`attachments` を、お好みのメール配信サービス（Mailtrap、Resend、Brevo、SendGrid 等）の API 仕様に変換して送信し、`{ "success": true }` を返す。
4. **CORS ヘッダーの付与（ブラウザ直接通信時）**: 必要に応じて `Access-Control-Allow-Origin: *` および `OPTIONS` プリフライトリクエストへの応答を設定する。

---

* [← トップページ（導入手順・GASサンプルコード）へ戻る](./index.html)
