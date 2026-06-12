---
title: "ChatGPT/Claude/Geminiを横断するプロンプト管理Chrome拡張を作った"
emoji: "🃏"
type: "tech"
topics: ["chrome拡張", "react", "typescript", "個人開発", "vite"]
published: true
---

仕事でAIを使いはじめてから、ずっと気になっていたことがありました。

「同じようなプロンプトを毎回ゼロから書いているな」

ChatGPTでメールの下書きを頼むとき、Claudeでコードレビューを依頼するとき、毎回似たような前置きを打ち込む。コピペ元のメモがどこかにあるはずなのに、デスクトップに .txt ファイルが増えていく一方で整理が追いつかない。

そういう「プロンプトのコピペ地獄」を解消したくて、**プロンプトデッキ**というChrome拡張を作りました。日本語ビジネスプロンプト50本を同梱しつつ、自分のプロンプトも管理できます。

現在Chrome Web Storeでの公開準備中です。製品ページはこちら: https://prompt-deck.pages.dev

---

## 設計で一番こだわったこと: Side Panel APIの採用

Chrome拡張でテキスト入力補助をやろうとすると、真っ先に思いつくのが**コンテンツスクリプトによるDOM注入**です。ページにUIを差し込んで、入力欄を直接操作する方法ですね。

でも今回はこれをやめました。理由は単純で、**壊れるから**です。

ChatGPTもClaudeもGeminiも、UIの改修を頻繁にやっています。昨日まで動いていたセレクタが今日突然消えた、という話は拡張作者にとって悪夢です。サポートの問い合わせが来るたびに修正をリリースする、というイタチごっこに巻き込まれるのが目に見えていました。

そこで採用したのが **Chrome Side Panel API** です。Manifest V3で安定したこのAPIを使うと、ブラウザウィンドウの横にパネルとして自前UIを表示できます。対象サイトのDOMとは完全に独立しているので、ChatGPTがどれだけUIを変えても壊れません。

挿入時だけは対象ページにアクセスが必要ですが、ここも「挿入スクリプト（scripting API）＋失敗時はクリップボードコピーにフォールバック」という設計にしました。挿入に失敗しても「コピーしました。貼り付けてください」というトーストが出るので、ユーザーは詰まりません。新しいAIサービスへの対応もフォールバックがあるおかげで即日できます。

## ストレージの設計: `chrome.storage.sync` の100KB制限

Chrome拡張で「複数端末で使えます」を実現するには `chrome.storage.sync` を使いたいところです。ただしこれ、容量が **100KB** という厳しい制限があります。プロンプト集を本格的に使い込むと、すぐに超えそうです。

今回の設計はこうしました。

- **プロンプト本文は `chrome.storage.local`（上限 ~10MB）に保存**
- **設定（テーマ/ライセンスキー/非表示フラグ）だけ `chrome.storage.sync` に保存**

端末間のプロンプト同期はJSONエクスポート/インポートで対応し、設定だけは自動的に同期されます。これで100KB制限を回避しつつ、「テーマ設定が別端末でも引き継がれる」という体験は確保できます。

コード上はこのように分かれています。

```typescript
const LOCAL_KEY = 'promptDeck.templates'; // chrome.storage.local
const SYNC_KEY  = 'promptDeck.settings';  // chrome.storage.sync
```

ストレージの分散は複雑さを増しますが、制限を正しく把握した上での設計なので、後から「容量オーバーでデータが保存できない」というバグを踏まずに済んでいます。

## 変数機能: `{{変数名}}` の展開

同梱テンプレートの多くは `{{宛先}}` や `{{案件名}}` のようなプレースホルダを含んでいます。挿入ボタンを押したとき、変数を含むテンプレートはそのまま流し込まずに**変数入力フォーム**をポップアップします。

```typescript
export function extractVariables(body: string): string[] {
  const matches = body.matchAll(/\{\{([^}]+)\}\}/g);
  return [...new Set([...matches].map(m => m[1].trim()))];
}
```

シンプルな正規表現ですが、重複を `Set` で除去しているので同じ変数が複数回出てきても入力欄は1つだけ表示されます。入力後はリアルタイムでプレビューが更新されるので、送信前に確認できます。

## Freemiumのライセンス設計

Pro機能（自作テンプレートの無制限保存。無料プランは20件まで）の決済には、Merchant of Record型の **Polar.sh** を使っています。海外ユーザーが購入する場合の消費税・VAT処理がPolar側に乗るので、個人開発者にとって法務リスクが減ります。

購入するとPolarがライセンスキーを発行し、拡張はそのキーをPolarの公開API（customer-portal）に `organization_id` を添えて問い合わせて検証します。設計で重視したのは2点です。

**1. 形式チェック単独でProを付与しない。** キーの形式（プレフィックスや大文字小文字の扱い）はPolar側の設定に依存するので、クライアントの形式チェックは緩い妥当性確認に留めています。

```typescript
export function isValidLicenseFormat(key: string): boolean {
  // 形式は緩い妥当性チェックに留める。真の検証はPolar APIが行う。
  // キーの大文字小文字は改変しない（Polar側で区別される可能性があるため）。
  const trimmed = key.trim();
  return /^[A-Za-z0-9_-]{8,64}$/.test(trimmed);
}
```

Proかどうかの判定は、APIの実検証で `status === 'granted'` だった結果をキャッシュに書き、**そのキャッシュだけを信頼**します。

```typescript
// isPro は service worker の実検証キャッシュのみを信頼する
const isPro = useMemo(() => {
  if (DEV_FORCE_PRO) return true;
  return isProStatus(
    licenseStatusFromCache(settings.licenseKey, licenseCache, Date.now()),
  );
}, [settings.licenseKey, licenseCache]);
```

**2. `organization_id` でスコープする。** 検証APIに自組織のIDを渡すことで、形式上は正しい「他組織が発行したキー」を誤って受け入れる事故を防いでいます。

## 権限の最小化

Chrome Web Storeの審査で落ちる理由の一つが「過剰な権限」です。今回申請する権限はこれだけです。

```json
"permissions": ["sidePanel", "storage", "activeTab", "scripting", "clipboardWrite"]
```

`host_permissions` は `chatgpt.com`、`claude.ai`、`gemini.google.com` の3ドメインのみ。`<all_urls>` は使いません。「なぜこの権限が必要か」が1行で説明できる範囲に収めるのが、審査を通す上での経験則です。

## テストの方針

ストレージ操作やDOM操作はテストしづらいので、純関数に切り出してvitestでユニットテストを書いています。変数展開・検索フィルタ・エクスポート形式・ライセンス状態遷移の検証などが該当します。拡張特有の `chrome.*` APIはモックを使わずに済む設計にしました。

## 振り返って

「壊れにくい設計」と「できるだけ早くストアに出す」のバランスが難しかったです。Side Panel APIの採用はその意味では正解で、対象サイトのUIに依存していないぶん、審査後の修正コストが読めています。

変数機能とフォールバックのコピー動作あたりが実際に使ってみてよく使う機能で、設計時に「これは面倒だから後回し」と言わなくてよかったと思っています。

もし同じような拡張を作っている方がいたら、ストレージの容量設計と権限の説明文は早めに考えておくことをおすすめします。審査担当者が読む前提で書いておくと、後で慌てずに済みます。

---

プロンプトデッキの製品ページはこちらです（Chrome Web Storeは近日公開予定）。
https://prompt-deck.pages.dev

バグ報告・フィードバックは歓迎しています。
