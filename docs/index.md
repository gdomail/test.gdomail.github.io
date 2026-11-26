# GdoMail 公式ドキュメント

**GdoMail** は、普段お使いの Gmail 画面から、独自ドメインのメールアドレスを差出人（From）に指定して直接メールを送信できる Chrome 拡張機能です。

ユーザー自身が管理する Google Apps Script（GAS）や Cloudflare Workers などのバックエンドを経由し、外部メール配信API（Mailtrap 等）と連携して動作します。

* **[バックエンド連携仕様書（JSONプロトコル）](./spec.html)** — 独自バックエンドを構築する方向けの技術仕様
* **[プライバシーポリシー](./privacy.html)** — データの取り扱いとセキュリティについて
* **[サポート・お問い合わせ](./support.html)** — 不具合報告・ご質問・よくある確認事項

<br>

> ⚠️ **重要（Important）**
> 
> 本拡張機能は現在 Google Apps Script（GAS）にのみ対応しています。
> Cloudflare Workers 等の他環境への対応は今後のアップデートで予定しています。
<br>

---

## 主な特徴

* **Gmailの操作感そのまま**: Gmailの新規作成・返信画面に「独自ドメイン送信」の専用パネルを自動で追加します。
* **HTMLメール・添付ファイル対応**: Gmail上で作成した装飾テキスト（HTML）や複数ファイルの添付をそのまま送信できます。
* **高いプライバシーと透明性**: 開発者側のサーバーを一切経由しません。通信はすべてユーザー自身が構築したバックエンドURLとの間でのみ行われます。
* **送信済みトレイへの自動保存**: 送信したメールは、ご自身のGmail「送信済み」フォルダにも自動で記録されます。

---

## 導入手順（全3ステップ）

GdoMail を利用するには、メール配信サービスの準備と、中継役となるバックエンドの設置が必要です。
<br><br>
ここでは下記２つのサービス（無料）を利用する方法を説明しています。
<br>
<mark>※ 他の方法でシステムを独自に構築することも可能です。</mark>

* メール配信サービスは、 Mailtrap
* バックエンドサービスは、Google Apps Script（GAS）

### ステップ 1：Mailtrap の準備
1. [Mailtrap](https://mailtrap.io) にアカウント登録・ログインします。
2. 独自ドメインの認証（Sending Domains）を完了させます。
3. **API Tokens** 画面から、送信用の API Token を発行して控えておきます（※動作確認のみ行う場合は Sandbox の API Token と Inbox ID を控えます）。

---

### ステップ 2：Google Apps Script（GAS）の設置

1. [Google Apps Script](https://google.com) を開き、「新しいプロジェクト」を作成します。
2. エディタに最初から入っているコードをすべて消し、以下の **GASサンプルコード** を貼り付けて保存します。

#### GAS サンプルコード（Mailtrap 対応・最終版）

```javascript
/**
 * GdoMail バックエンド連携サンプルスクリプト（Mailtrap対応版）
 * 
 * 【セットアップ手順】
 * 1. 下部の「★設定エリア」に任意の認証パスワードと Mailtrap の API Token を入力してください。
 * 2. 左メニュー「サービス（＋）」から「Gmail API」を追加してください。
 * 3. 右上「デプロイ」＞「新しいデプロイ」＞種類「ウェブアプリ」を選択し、
 *    アクセスできるユーザーを「全員」にしてデプロイしてください。
 */

function doPost(e) {
  const props = PropertiesService.getScriptProperties();
  const AUTH_TOKEN = props.getProperty('AUTH_TOKEN');
  const API_URL = props.getProperty('API_URL');
  const API_KEY = props.getProperty('API_KEY');

  try {
    const data = JSON.parse(e.postData.contents);
    const expectedHash = AUTH_TOKEN ? sha256HexGAS(AUTH_TOKEN) : '';
    const isAuthorized =
      (data.authTokenHash && data.authTokenHash === expectedHash) ||
      (data.authToken && data.authToken === AUTH_TOKEN);

    if (!isAuthorized) {
      return createJsonResponse({ error: 'Unauthorized' });
    }

    const maxAttachmentMb = getMaxAttachmentMb(props);

    // Chrome拡張機能用：設定値のみ取得
    if (data.action === 'getConfig') {
      return createJsonResponse({
        success: true,
        maxAttachmentMb: maxAttachmentMb
      });
    }

    // 送信元アドレス データ取得（設定値もあわせて返却）
    if (data.action === 'getSenders') {
      const raw = props.getProperty('SENDERS');
      return createJsonResponse({
        success: true,
        senders: raw ? JSON.parse(raw) : [],
        maxAttachmentMb: maxAttachmentMb
      });
    }

    if (!API_URL || !API_KEY) {
      return createJsonResponse({ error: 'API_URL または API_KEY がスクリプトプロパティに設定されていません' });
    }

    // 添付ファイルサイズ上限チェック
    validateAttachmentSize(data.attachments, maxAttachmentMb);

    const requestSpec = buildProviderRequest(API_KEY, data);
    const options = {
      method: 'post',
      headers: requestSpec.headers,
      muteHttpExceptions: true
    };

    if (requestSpec.contentType) {
      options.contentType = requestSpec.contentType;
    }
    options.payload = requestSpec.payload;

    const response = UrlFetchApp.fetch(API_URL, options);
    const statusCode = response.getResponseCode();
    const result = response.getContentText() ? JSON.parse(response.getContentText()) : {};

    if (statusCode >= 400) {
      const errMsg = result.message || result.ErrorMessage || (Array.isArray(result.errors) ? result.errors.join(', ') : null) || 'API Error (' + statusCode + ')';
      return createJsonResponse({ error: errMsg, details: result });
    }

    saveToGmailSent(data);
    return createJsonResponse({ success: true});
  } catch (err) {
    return createJsonResponse({ error: err.message });
  }
}

/**
 * スクリプトプロパティから添付上限（MB）を取得（デフォルト: 10）
 */
function getMaxAttachmentMb(props) {
  const raw = props.getProperty('MAX_ATTACHMENT_MB');
  const parsed = Number(raw);
  return (raw && !isNaN(parsed) && parsed > 0) ? parsed : 10;
}

/**
 * 添付ファイルの合計サイズ検証（Base64長から元のバイト数を算出）
 */
function validateAttachmentSize(attachments, maxMb) {
  if (!attachments || attachments.length === 0) return;

  const maxBytes = maxMb * 1024 * 1024;
  let totalBytes = 0;

  attachments.forEach(function(att) {
    if (!att.base64) return;
    const padding = (att.base64.endsWith('==')) ? 2 : (att.base64.endsWith('=') ? 1 : 0);
    const bytes = (att.base64.length * 3) / 4 - padding;
    totalBytes += bytes;
  });

  if (totalBytes > maxBytes) {
    const totalMb = (totalBytes / (1024 * 1024)).toFixed(2);
    throw new Error('添付ファイルの合計サイズが上限（' + maxMb + 'MB）を超えています。（現在: ' + totalMb + 'MB）');
  }
}

/**
 * APIリクエスト（HTML / プレーンテキスト / 添付ファイル対応版）
 */
function buildProviderRequest(apiKey, data) {
  const toList = data.to || [];
  const ccList = data.cc || [];
  const bccList = data.bcc || [];
  const subject = data.subject || '(件名なし)';
  const isPlain = Boolean(data.isPlainText);
  const html = data.htmlContent || '';
  const text = data.textContent || '';
  const senderName = data.senderName || data.senderEmail;
  const attachments = data.attachments || []; // [{ name, type, base64 }]

  const payload = {
    from: { email: data.senderEmail, name: senderName },
    to: toList.map(function(e) { return { email: e }; }),
    subject: subject
  };
  if (isPlain) { payload.text = text; } else { payload.html = html; }
  if (ccList.length > 0) payload.cc = ccList.map(function(e) { return { email: e }; });
  if (bccList.length > 0) payload.bcc = bccList.map(function(e) { return { email: e }; });
  if (attachments.length > 0) {
    payload.attachments = attachments.map(function(att) {
      return { content: att.base64, filename: att.name, type: att.type || 'application/octet-stream', disposition: 'attachment' };
    });
  }
  return {
    contentType: 'application/json',
    headers: { 'Authorization': 'Bearer ' + apiKey },
    payload: JSON.stringify(payload)
  };
}

/**
 * Gmail 送信データ インポート（HTML / プレーンテキスト / 添付ファイル対応版）
 */
function saveToGmailSent(data) {
  const fromHeader = data.senderName
    ? '=?UTF-8?B?' + Utilities.base64Encode(data.senderName, Utilities.Charset.UTF_8) + '?= <' + data.senderEmail + '>'
    : data.senderEmail;

  const subjectHeader = '=?UTF-8?B?' + Utilities.base64Encode(data.subject || '(件名なし)', Utilities.Charset.UTF_8) + '?=';
  const attachments = data.attachments || [];
  const isPlain = Boolean(data.isPlainText);
  const bodyContentType = isPlain ? 'text/plain; charset=UTF-8' : 'text/html; charset=UTF-8';
  const bodyRawText = isPlain ? (data.textContent || '') : (data.htmlContent || '');
  const encodedBody = Utilities.base64Encode(bodyRawText, Utilities.Charset.UTF_8);

  const lines = [
    'From: ' + fromHeader,
    'To: ' + data.to.join(', '),
  ];

  if (data.cc && data.cc.length > 0) {
    lines.push('Cc: ' + data.cc.join(', '));
  }
  if (data.bcc && data.bcc.length > 0) {
    lines.push('Bcc: ' + data.bcc.join(', '));
  }

  lines.push('Subject: ' + subjectHeader);
  lines.push('MIME-Version: 1.0');

  if (attachments.length === 0) {
    lines.push('Content-Type: ' + bodyContentType);
    lines.push('Content-Transfer-Encoding: base64');
    lines.push('');
    lines.push(encodedBody);
  } else {
    const boundary = '----=_Part_' + Utilities.getUuid();
    lines.push('Content-Type: multipart/mixed; boundary="' + boundary + '"');
    lines.push('');

    lines.push('--' + boundary);
    lines.push('Content-Type: ' + bodyContentType);
    lines.push('Content-Transfer-Encoding: base64');
    lines.push('');
    lines.push(encodedBody);
    lines.push('');

    attachments.forEach(function(att) {
      const encodedFileName = '=?UTF-8?B?' + Utilities.base64Encode(att.name, Utilities.Charset.UTF_8) + '?=';
      const contentType = att.type || 'application/octet-stream';
      const foldedBase64 = att.base64.replace(/.{1,76}/g, '$&\r\n').trim();

      lines.push('--' + boundary);
      lines.push('Content-Type: ' + contentType + '; name="' + encodedFileName + '"');
      lines.push('Content-Disposition: attachment; filename="' + encodedFileName + '"');
      lines.push('Content-Transfer-Encoding: base64');
      lines.push('');
      lines.push(foldedBase64);
      lines.push('');
    });

    lines.push('--' + boundary + '--');
  }

  const rawMime = lines.join('\r\n');
  const encodedMessage = Utilities.base64EncodeWebSafe(rawMime);

  Gmail.Users.Messages.insert(
    {
      raw: encodedMessage,
      labelIds: ['SENT']
    },
    'me'
  );
}

/**
 * オブジェクト(配列)をJSON形式の文字列に変換
 */
function createJsonResponse(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

/**
 * ハッシュ照合
 */
function sha256HexGAS(text) {
  const rawHash = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, text, Utilities.Charset.UTF_8);
  return rawHash.map(function(byte) {
    const v = (byte < 0) ? byte + 256 : byte;
    return ('0' + v.toString(16)).slice(-2);
  }).join('');
}
```

#### スクリプト プロパティの設定
APIトークンやパスワードをコード内に直接書かず安全に管理するため、GASの「プロジェクトの設定」から以下のプロパティを登録します。

1. GASエディタ左側の歯車アイコン **「プロジェクトの設定」** をクリックします。
2. 画面一番下の **「スクリプト プロパティ」＞「スクリプト プロパティを追加」** から、以下の項目を設定して保存します。

| プロパティ名 | 必須/任意 | 設定する値（例） | 説明 |
| :--- | :--- | :--- | :--- |
| `AUTH_TOKEN` | **必須** | `my-secret-password-123` | Chrome拡張機能側と一致させる任意の認証パスワード |
| `API_URL` | **必須** | `https://send.api.mailtrap.io/api/send` | 本番送信用のURL<br><br>※Sandboxテスト時は下記<br> `https://sandbox.api.mailtrap.io/api/send/{INBOX_ID}` |
| `API_KEY` | **必須** | `a1b2c3d4e5f6...` | Mailtrap で発行した API Token |
| `MAX_ATTACHMENT_MB` | 任意 | `10` | 添付ファイル合計サイズの上限（MB単位・未設定時はデフォルト `10`） |
| `SENDERS` | 任意 | `[{"email":"info@example.com","name":"表示名"}]` | 拡張機能側へ自動取得させる差出人リスト（JSON配列形式） |

#### GAS の有効化とデプロイ手順
1. GASエディタ左側の **「サービス（＋）」** をクリックし、一覧から **「Gmail API」** を選択して「追加」を押します。
2. 画面右上の **「デプロイ」＞「新しいデプロイ」** をクリックします。
3. 種類の選択（歯車アイコン）から **「ウェブアプリ」** を選び、以下のように設定します。
   * **次のユーザーとして実行**: 自分
   * **アクセスできるユーザー**: 全員
4. 「デプロイ」をクリックし、アクセス承認画面が出たらご自身のGoogleアカウントで許可します。
5. 発行された **「ウェブアプリ URL」** をコピーします。

---

### ステップ 3：Chrome 拡張機能「GdoMail」の設定

1. Chrome ウェブストアから **GdoMail** をインストールします。
2. Chrome ツールバーの拡張機能アイコンから GdoMail の **「オプション（設定画面）」** を開きます。
3. 以下の項目を入力して保存します。
   * **GAS ウェブアプリ URL**: ステップ2で発行した URL
   * **認証パスワード**: スクリプトプロパティに設定した `AUTH_TOKEN` と同じ文字列
   * **差出人（From）リスト**: 送信に使う独自ドメインのメールアドレスと表示名
4. Gmail を開き、「作成」ボタンを押すと表示される GdoMail パネルから送信をお試しください。

---

## サポート・お問い合わせ

不具合の報告やご質問、よくある確認事項については、以下の専用ページをご確認ください。

* **[サポート・お問い合わせページはこちら](./support.html)**
