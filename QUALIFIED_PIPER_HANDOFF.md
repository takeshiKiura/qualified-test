# Qualified Piper × Salesforce Experience Cloud 引き継ぎ

## 目的

Salesforce Experience Cloud サイトに Qualified の Piper（Multi-modal AI SDR）を表示し、指定URLで Automatic Experience を起動する。

対象サイト：

```text
https://trailsignup-c62524c336c869.my.site.com/consumer/s/
```

## 現在の状態

### Qualified 側

- Agent Studio の Piper Profile は作成済み（`Default`）。
- Automatic Experience `PiperTest` は **Active**。
- アクションは `Start Multi-modal AI SDR`、Profile は `Default`。
- 起動条件は現在、`Current page contains` に Experience Cloud のURLを指定している。
- Website domains に次のホスト名を追加済み。

```text
trailsignup-c62524c336c869.my.site.com
```

### Salesforce 側

Experience Builder の Head Markup に Qualified snippet を追加済み。

Experience Builder → Settings → Security & Privacy：

- Security Level: `Relaxed CSP: Permit Access to Inline Scripts and Allowed Hosts`
- Trusted Sites for Scripts:
  - `https://js.qualified.com`
  - `https://app.qualified.com`
  - `https://*.qualified.com`
  - `wss://*.qualified.com` （WebSocket 用。Piper が `wss://ws1.qualified.com` に接続するため必須）

Salesforce Setup → Trusted URLs（**Experience Builder の Trusted Sites for Scripts にも同じものを登録**）：

| # | API Name | URL | 用途 |
|---|---|---|---|
| 1 | Qualified | `https://*.qualified.com` | Qualified 本体（js / app / voice-calls など全サブドメイン） |
| 2 | QualifiedWSS | `wss://*.qualified.com` | Piper のリアルタイム通信（ws1.qualified.com など） |
| 3 | DailyCo | `https://*.daily.co` | Piper 音声通話バックエンド |
| 4 | DailyCoWSS | `wss://*.daily.co` | Daily.co WebSocket |
| 5 | Sentry | `https://*.ingest.sentry.io` | Qualified のエラー送信先 |

**各エントリで CSP Directive を全てチェック**:
connect-src / frame-src / img-src / style-src / font-src / **media-src** / script-src

注意点:
- CSP の `connect-src` はスキーム込みでマッチするため、`https://` の許可だけでは `wss://` はブロックされる。wss を別エントリで登録すること。
- Piper は `js.qualified.com` から mp3（メッセージ通知音）を読むため **media-src の許可が必須**。`media-src` は Experience Builder の Trusted Sites for Scripts では制御できないので、Setup → Trusted URLs 側で directive をチェックする。
- 変更後は Experience Builder を **Publish** してからシークレットウィンドウで確認。
- BuilderのCSP画面は最終的に `Issue free for CSP` になった。

## 確認済みの事実

公開ページのDOMには以下が存在する。

- Qualified のインライン初期化コード
- `https://js.qualified.com/qualified.js?token=...`

一方、15秒待っても以下は確認できていない。

- Qualified messenger の iframe
- Qualified の表示用DOM
- Piper の表示

Qualified の `PiperTest` Experience Analytics も **Triggers = 0** のまま。

これは「サイト訪問がゼロ」という意味ではなく、Qualifiedが訪問を認識して、かつPiperTestの条件に一致した回数がゼロという意味。

## 有力な原因：Experience Cloud のSPA遷移

Experience Cloud はSPA的な動きをするため、Qualifiedは通常のscript読み込みだけでは Current page 条件を再判定できない可能性がある。

Qualified公式のSPA向け案内に従い、画面読込／ルート遷移後に次を実行する。

```js
qualified("page");
```

参考：<https://university.qualified.com/api-49/client-side-api-95>

## VS Codeで実装する推奨方針

### 1. まず最小HTMLでQualified単体を検証

同ディレクトリに次のファイルを作成済み。

```text
qualified-piper-test.html
```

このファイルはQualified snippetと、ロード後に `window.qualified("page")` を呼ぶ最小ページ。

実行前に以下を行う。

1. `YOUR_QUALIFIED_TOKEN` を Qualified Setup に表示される実トークンへ置換する。
2. Qualified → Settings → App Settings → Setup → Website domains に `localhost` を追加する。
3. `PiperTest` のURL条件を一時的に以下へ変更する。

```text
Current page contains http://localhost:8080/qualified-piper-test.html
```

4. ローカルサーバーを起動する。

```bash
python3 -m http.server 8080
```

5. シークレットウィンドウで開く。

```text
http://localhost:8080/qualified-piper-test.html
```

期待結果：PiperTestのTriggersが `1` 以上になり、Piperが表示される。

> 注意：トークンは秘密情報として扱い、Gitへコミットしないこと。

### 2. 最小HTMLで動いた場合：Experience Cloudへ反映

Experience BuilderのHead Markupには、Salesforce Embedded Messagingのコードを混在させず、Qualifiedだけを配置して検証する。

```html
<!-- Qualified -->
<script>
  (function (w, q) {
    w.QualifiedObject = q;
    w[q] = w[q] || function () {
      (w[q].q = w[q].q || []).push(arguments);
    };
  })(window, "qualified");
</script>

<script async src="https://js.qualified.com/qualified.js?token=YOUR_QUALIFIED_TOKEN"></script>

<script>
  window.addEventListener("load", function () {
    window.setTimeout(function () {
      window.qualified("page");
    }, 1000);
  });
</script>
<!-- End Qualified -->
```

保存後はExperience BuilderをPublishし、シークレットウィンドウで確認する。

### 3. SPAのページ遷移も追従させる場合

Experience Cloud内で画面遷移した際にも、遷移完了後に以下を一度だけ呼ぶ実装を追加する。

```js
window.qualified("page");
```

実装方法は利用しているExperience Cloudテンプレート／LWC構成に合わせる。たとえばルート変更を検知するLWCまたはページ用の共通スクリプトから呼び出す。

## 重要な切り分け順

1. 最小HTMLでPiperが出るか。
2. 出るなら、Experience Cloud側のSPAイベント実装またはHead Markup実行順が原因。
3. 最小HTMLでもTriggersが0なら、Qualified側のExperience条件、Website domain、またはトークンと組織の組合せを再確認する。

## 補足

- StreamはLive View／営業担当へのルーティング用であり、PiperのPounce起動そのものの必須設定ではない。
- Pounce設定画面のAuto Pounceは、主に有人営業担当向けの旧来のPounce設定。Piperの開始はAutomatic Experienceの `Start Multi-modal AI SDR` が担う。
- Cookie制限は検証中不要。Experience Builderの `Allowed Cookies` はOffのままにしている。
- CSPポップアップの `app.qualified.com/.../sentry/proxy` はQualifiedのエラー送信先。現在BuilderのCSP画面は問題なし表示。
