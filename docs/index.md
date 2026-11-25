# GdoMail 公式ドキュメント

**GdoMail** は、普段お使いの Gmail 画面から、独自ドメインのメールアドレスを差出人（From）に指定して直接メールを送信できる Chrome 拡張機能です。

ユーザー自身が管理する Google Apps Script（GAS）や Cloudflare Workers などのバックエンドを経由し、外部メール配信API（Mailtrap 等）と連携して動作します。

* **[バックエンド連携仕様書（JSONプロトコル）](./spec.html)** — 独自バックエンド（Cloudflare Workers 等）を構築する方向けの技術仕様
* **[プライバシーポリシー](./privacy.html)** — データの取り扱いとセキュリティについて

---

## 主な特徴

* **Gmailの操作感そのまま**: Gmailの新規作成・返信画面に「独自ドメイン送信」の専用パネルを自動で追加します。
* **HTMLメール・添付ファイル対応**: Gmail上で作成した装飾テキスト（HTML）や複数ファイルの添付をそのまま送信できます。
* **高いプライバシーと透明性**: 開発者側のサーバーを一切経由しません。通信はすべてユーザー自身が構築したバックエンドURLとの間でのみ行われます。
* **送信済みトレイへの自動保存**: 送信したメールは、ご自身のGmail「送信済み」フォルダにも自動で記録されます。

---

## 導入手順（全3ステップ）

GdoMail を利用するには、メール配信サービス（Mailtrap）の準備と、中継役となる Google Apps Script（GAS）の設置が必要です。

### ステップ 1：Mailtrap の準備
1. [Mailtrap](https://mailtrap.io) にアカウント登録・ログインします。
2. 独自ドメインの認証（Sending Domains）を完了させます。
3. **API Tokens** 画面から、送信用の API Token を発行して控えておきます（※動作確認のみ行う場合は Sandbox の API Token も利用できます）。

---

### ステップ 2：Google Apps Script（GAS）の設置

1. [Google Apps Script](https://google.com) を開き、「新しいプロジェクト」を作成します。
2. エディタに最初から入っているコードをすべて消し、以下の **GASサンプルコード** を貼り付けます。
3. コード冒頭の設定エリアにある `AUTH_TOKEN` と `MAILTRAP_TOKEN` を、ご自身の任意のパスワードとステップ1で取得したトークンに書き換えます。

#### GAS サンプルコード（Mailtrap 連携版）

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

// ================= ★設定エリア =================
const CONFIG = {
  // Chrome拡張機能側と一致させる任意の認証パスワード
  AUTH_TOKEN: 'your-secret-password-here',

  // Mailtrap の API Token（Live または Sandbox）
  MAILTRAP_TOKEN: 'your-mailtrap-api-token-here',

  // 送信モード（'live' = 本番送信 / 'sandbox' = テスト送信）
  MODE: 'live',

  // Sandboxモード時に使用する Inbox ID（MODE が 'sandbox' の場合のみ必須）
  SANDBOX_INBOX_ID: ''
};
// ================================================

function doPost(e) {
  try {
    const params = JSON.parse(e.postData.contents);

    // 1. 認証チェック
    if (!params.authToken || params.authToken !== CONFIG.AUTH_TOKEN) {
      return createJsonResponse({ status: 'error', message: '認証エラー: パスワードが一致しません。' });
    }

    // 2. パラメータ検証
    if (!params.from || !params.to || !params.subject) {
      return createJsonResponse({ status: 'error', message: '必須項目（差出人・宛先・件名）が不足しています。' });
    }

    // 3. Mailtrap API へ送信リクエスト
    const sendResult = sendViaMailtrap(params);
    if (!sendResult.success) {
      return createJsonResponse({ status: 'error', message: '配信APIエラー: ' + sendResult.error });
    }

    // 4. Gmail「送信済み」トレイへ控えを保存（エラーが出ても送信自体は成功扱いとする）
    try {
      saveToGmailSent(params);
    } catch (gmailErr) {
      console.warn('Gmail送信済み保存スキップ:', gmailErr);
    }

    return createJsonResponse({ status: 'success', message: '送信が完了しました。' });

  } catch (err) {
    return createJsonResponse({ status: 'error', message: 'サーバーエラー: ' + err.toString() });
  }
}

/**
 * Mailtrap API 送信処理
 */
function sendViaMailtrap(params) {
  const endpoint = (CONFIG.MODE === 'sandbox')
    ? `https://mailtrap.io{CONFIG.SANDBOX_INBOX_ID}`
    : 'https://mailtrap.io';

  const payload = {
    from: {
      email: params.from,
      name: params.fromName || ''
    },
    to: parseRecipients(params.to),
    subject: params.subject
  };

  if (params.cc) payload.cc = parseRecipients(params.cc);
  if (params.bcc) payload.bcc = parseRecipients(params.bcc);

  if (params.isHtml) {
    payload.html = params.body;
    payload.text = stripHtml(params.body);
  } else {
    payload.text = params.body;
  }

  if (params.attachments && params.attachments.length > 0) {
    payload.attachments = params.attachments.map(att => ({
      content: att.data,
      filename: att.name,
      type: att.mimeType || 'application/octet-stream',
      disposition: 'attachment'
    }));
  }

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: {
      'Authorization': 'Bearer ' + CONFIG.MAILTRAP_TOKEN
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  const response = UrlFetchApp.fetch(endpoint, options);
  const code = response.getResponseCode();
  const body = response.getContentText();

  if (code >= 200 && code < 300) {
    return { success: true };
  } else {
    return { success: false, error: `HTTP ${code}: ${body}` };
  }
}

/**
 * Gmail API を使用して「送信済み」ラベル付きで保存
 */
function saveToGmailSent(params) {
  const boundary = '----=_Part_' + new Date().getTime();
  let rawLines = [
    `From: ${params.fromName ? `"params.fromName" <{params.from}>` : params.from}`,
    `To: ${params.to}`,
    params.cc ? `Cc: ${params.cc}` : '',
    `Subject: =?UTF-8?B?${Utilities.base64Encode(params.subject, Utilities.Charset.UTF_8)}?=`,
    'MIME-Version: 1.0',
    `Content-Type: multipart/mixed; boundary="${boundary}"`,
    '',
    `--${boundary}`,
    params.isHtml
      ? 'Content-Type: text/html; charset=UTF-8'
      : 'Content-Type: text/plain; charset=UTF-8',
    'Content-Transfer-Encoding: base64',
    '',
    Utilities.base64Encode(params.body || '', Utilities.Charset.UTF_8)
  ].filter(line => line !== '');

  if (params.attachments && params.attachments.length > 0) {
    params.attachments.forEach(att => {
      rawLines.push(
        `--${boundary}`,
        `Content-Type: ${att.mimeType || 'application/octet-stream'}; name="${att.name}"`,
        'Content-Transfer-Encoding: base64',
        `Content-Disposition: attachment; filename="${att.name}"`,
        '',
        att.data
      );
    });
  }

  rawLines.push(`--${boundary}--`);
  const rawEmail = rawLines.join(\('\r\\)n');
  const encodedEmail = Utilities.base64EncodeWebSafe(rawEmail);

  Gmail.Users.Messages.insert({
    raw: encodedEmail,
    labelIds: ['SENT']
  }, 'me');
}

function parseRecipients(str) {
  return str.split(',')
    .map(s => s.trim())
    .filter(s => s.length > 0)
    .map(email => ({ email: email }));
}

function stripHtml(html) {
  return (html || '').replace(/<[^>]*>?/gm, '');
}

function createJsonResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

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
   * **認証パスワード**: GASコード内に設定した `AUTH_TOKEN` と同じ文字列
   * **差出人（From）リスト**: 送信に使う独自ドメインのメールアドレスと表示名
4. Gmail を開き、「作成」ボタンを押すと表示される GdoMail パネルから送信をお試しください。

---

## サポート・お問い合わせ

不具合の報告やご質問は、Chrome ウェブストアのサポートタブ、または下記連絡先までお問い合わせください。

* **開発者**: （※ここにデベロッパー名を入力）
* **連絡先**: （※ここにサポート用メールアドレスを入力）
