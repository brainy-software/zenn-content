---
title: "Sentryを入れるほどじゃないけど500エラーは通知したい ── Laravel専用エラー通知OSSを作った話"
emoji: "🚨"
type: "tech"
topics: ["laravel", "php", "slack", "discord", "oss"]
published: false
---

Webサービスや業務システム、受託案件の運用で避けて通れないのが、本番環境での500エラーや未処理例外の検知です。

ユーザーからの問い合わせで初めて障害に気づく事態は避けたいところです。とはいえ、Sentry や Datadog のような高機能 SaaS をすべての案件に入れられるかというと、コストと運用の両面でハードルがあります。

- 小規模なプロジェクトや社内ツールに、月額数千〜数万円の監視 SaaS を入れる予算がない
- チームで普段使っている Slack や Discord に、見やすいカード形式でエラーを流したいだけ
- Laravel 標準の Slack ログ（Monolog）はあるものの、レイアウトが素っ気ないうえ、**同一エラーが大量発生すると Slack がパンクします**（スロットリングがありません）
- リクエストパラメータに含まれるパスワードやトークンが、チャットに流出するのが怖い

こうした場面のために、**`.env` に Webhook URL を1行設定するだけで、Slack（Block Kit）や Discord（Rich Embed）へ通知する Laravel 専用の OSS パッケージ** を作りました。

- **GitHub**: [brainy-software/laravel-error-notifier](https://github.com/brainy-software/laravel-error-notifier)
- **Packagist**: `brainy/laravel-error-notifier`
- **ライセンス**: MIT License（商用利用・改変自由）

実際に届く通知はこのような見た目です。

![Slack に届いたエラー通知のカード](/images/laravel-error-notifier/slack-preview.png)
*Slack（Block Kit）*

![Discord に届いたエラー通知のカード](/images/laravel-error-notifier/discord-preview.png)
*Discord（Rich Embed）*

この記事では、既存ツールとの違い、設計でいちばん気を遣った「フェイルセーフと安全設計」、そしてブラウザ上で設定ファイルを生成できる GUI ジェネレーターの仕組みを紹介します。

---

## なぜ既存の通知パッケージでは足りなかったのか

Laravel で Slack や Discord にエラーを通知する手段は、これまでもいくつかありました。ただ、実務で安心して使うにはあと一歩足りない、というケースが多かったのです。

| 手法 / 既存ツール | メリット | 現場で直面した課題 |
| :--- | :--- | :--- |
| **Laravel標準 Monolog (Slack)** | 設定追加だけで動く | ・**スロットリング機能がない**ため、DB障害などで毎秒数百通の通知が届き Slack がフリーズする<br>・Discord に対応していない<br>・パスワード等の自動マスキングがない |
| **Spatie製 Slack Alerts** | 軽量で使いやすい | ・文字列を Webhook で送るラッパーであり、例外の捕捉・スタックトレース整形・コンテキスト収集は**すべて自前実装が必要** |
| **海外製の多機能通知パッケージ** | 多数のチャットに対応 | ・設定ファイルが巨大で難解<br>・日本語の情報がなく、メンテナンスが停滞しているものが多い |

欲しかったのは、**導入コストがほぼゼロで、設定は極限までシンプル。それでいて障害時にチャットをパンクさせず、セキュリティと安定性を担保できるツール**でした。

---

## パッケージのコア機能と特徴

### 1. Slack (Block Kit) と Discord (Embed) の両対応

エラー通知は、見た瞬間に状況が把握できることがいちばん重要です。Slack の Block Kit と Discord の Rich Embed を使い、初動で必要な情報を整理して通知します。

- **例外クラス名とメッセージ**（目立つ赤色アクセント）
- **発生ファイル名と行番号**
- **環境名**（LOCAL / STAGING / PRODUCTION）
- **リクエスト URL と HTTP メソッド**
- **クライアント IP とログイン中のユーザー ID**
- **アプリケーションコードに絞ったスタックトレース**（vendor ディレクトリの膨大なフレームを自動で除外）

### 2. 同一エラーの連打を自動で抑える

DB の接続エラーや外部 API のダウンが起きると、同一のエラーが短時間に数千件発生することがあります。通知ツール自体がチャットをパンクさせたり、Webhook のレートリミットに引っかかって止まっては本末転倒です。

そこで Laravel のキャッシュを使い、**例外クラス名・発生ファイル・行番号・エラーメッセージからハッシュを作って、同一エラーの連続通知を抑制**します（既定は10分間に1回のみ）。

### 3. 機密パラメータの自動マスキング

フォーム送信時のエラー通知には、リクエストパラメータが含まれることが多くなります。送信前にペイロードを再帰的に走査し、次のキーに該当する値は自動的に `********` へ置き換えます。

```php
'mask_keys' => [
    'password', 'password_confirmation', 'current_password',
    'token', 'secret', 'api_key', 'access_token',
    'credit_card', 'card_number', 'cvv',
],
```

### 4. 404やバリデーションエラーの自動除外

404（NotFoundHttpException）や422（ValidationException）、CSRF トークン不一致（TokenMismatchException）など、ユーザーの操作が起因で対応の必要がない例外は、既定で除外します。通知が鳴り止まない状態を防ぐためです。

---

## いちばん気を遣った「安全設計」

監視ツールや通知パッケージを入れるとき、最も怖いのは**通知パッケージ側の不具合でメインのアプリケーションが落ちること**です。本パッケージでは、次の2点を徹底しています。

1. **既存のログ出力を壊さない**
   Laravel の `ExceptionHandler::reportable()` に透過的にフックするため、`storage/logs/laravel.log` への書き込みや、Sentry 等の併用ツールと競合しません。
2. **通信障害が起きてもアプリは無傷**
   Slack や Discord の API 障害、タイムアウト（5秒）、ネットワーク寸断が起きても、通知処理の中で安全に例外を処理します。ユーザーへの HTTP レスポンスを巻き込んで500エラーになることはありません。

---

## 3分で終わるクイックスタート

動作要件は **PHP 8.2 以上 / Laravel 11・12・13** です。

### 1. インストール

```bash
composer require brainy/laravel-error-notifier
```

Laravel の Package Discovery に対応しているので、サービスプロバイダの登録は不要です。

### 2. .env に Webhook URL を指定

```env
# Slack Incoming Webhook
ERROR_NOTIFIER_SLACK_WEBHOOK=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK

# Discord Webhook (オプション)
ERROR_NOTIFIER_DISCORD_WEBHOOK=https://discord.com/api/webhooks/YOUR/DISCORD/WEBHOOK
```

### 3. 動作テスト

`artisan tinker` を1行叩けば、通知カードが届くことを確認できます。

```bash
php artisan tinker --execute="report(new \RuntimeException('テスト例外です！'));"
```

---

## ブラウザだけで完結する設定ジェネレーター

このパッケージには、**単一の HTML ファイルで動く設定ジェネレーター（`generator.html`）を同梱**しています。

設定ファイル（`config/error-notifier.php`）の各項目（メンションの有無、タイトルのカスタム、非表示にしたいフィールド、スロットリングの期間など）を、**Slack / Discord のプレビューを見ながらブラウザ上で生成**できます。

```bash
# パッケージ配下の generator.html をブラウザで開くだけ
open generator.html
```

- クライアント IP やリクエストパラメータを非表示にするトグルスイッチ
- `@channel` や `@here` メンションの入力
- `.env` のコピーと、設定ファイルのワンクリックダウンロード

外部サーバーとの通信は一切なく、完全にローカルで動くので、Webhook URL が外部に漏れる心配もありません。

---

## まとめと今後の予定

個人開発や社内ツール、受託開発の初期フェーズで、「手軽に、でも安全性と見た目には妥協せずエラー監視をしたい」という場面で役立てていただければ幸いです。

今後は次の機能も予定しています。

- **LINE 通知対応**（LINE Messaging API / LINE WORKS 連携）
- **Email 通知対応**（Laravel 標準 Mailer 連携）
- **汎用 Webhook (Custom JSON) 対応**（社内 API や PagerDuty 等への転送）
- **Microsoft Teams 対応**（Adaptive Cards）

バグ報告や機能要望、プルリクエストは GitHub リポジトリまでお寄せください。

- **GitHub**: [https://github.com/brainy-software/laravel-error-notifier](https://github.com/brainy-software/laravel-error-notifier)
- **開発元**: [合同会社ブレイニーソフトウェア (Brainy Software LLC)](https://www.brainy-software.co.jp/)
