---
title: "登録不要・完全ローカルの請求書作成ツールを作った（インボイス・源泉徴収対応）"
emoji: "🧾"
type: "tech"
topics: ["react", "typescript", "個人開発", "インボイス", "vite"]
published: true
---

フリーランスや副業で請求書を発行しはじめたとき、使えるツールを探してみると意外と選択肢が少なかったです。

- freeeや弥生は機能が豊富すぎて料金もそれなりにかかる
- 無料のPDFツールはインボイス制度の記載要件に対応していない
- Googleスプレッドシートのテンプレートは源泉徴収の計算が手動になる

「登録不要で、ブラウザだけで使えて、インボイスの要件を満たしたPDFが出力できる」ものが見つからなかったので、作りました。

**サクッと請求書**というWebアプリです。データは一切外部に送信せず、ブラウザの中だけで動作します。

→ https://sakutto-invoice.pages.dev

---

## プライバシー設計を技術的な売りにする

最近のWebサービスでは「無料で使えます」と言いつつ、使い方の分析データやフォームの入力内容をどこかに送っているケースがあります。請求書に記載する取引先名・金額・登録番号は、外に出したくない情報の代表格です。

サクッと請求書はその点を技術的な制約として明確にしました。

**外部通信がゼロ**です。バックエンドサーバーを持ちません。アナリティクス（Google Analytics等）も搭載していません。CDNから届いたHTMLとJavaScriptがブラウザの中だけで動作し、入力したデータは `localStorage` に保存されるだけです。

Vite + React + TypeScriptで作った完全静的なSPA（Single Page Application）で、ビルド成果物は静的ファイルのみです。Cloudflare Pagesにデプロイしており、サーバーサイドのコードは一行も存在しません。

```
ユーザーの入力 → localStorage → PDF出力（ブラウザのprint機能）
                              ↑
                   ここまで外部通信ゼロ
```

「データを送らない」を実現するためにバックエンドを作らない、という逆転の発想で、むしろ実装がシンプルになっています。

## インボイス制度の記載要件を純関数で実装する

2023年10月から始まったインボイス制度（適格請求書等保存方式）では、請求書に記載すべき項目が細かく定められています。

- 発行者の氏名・名称と登録番号（T + 13桁）
- 取引年月日
- 取引内容（軽減税率対象の場合はその旨）
- 税率ごとに区分した合計額と消費税額
- 受領者の氏名・名称

このうち「税率ごとに区分した消費税計算」が実装のポイントになります。インボイス制度では、1枚の請求書につき税率区分ごとに1回だけ端数処理を行うとされています。行ごとに切り捨てて合算するのは要件違反です。

計算ロジックは `src/lib/calc.ts` に純関数として切り出しています。

```typescript
export function calcTaxByRate(
  items: LineItem[],
  roundingMode: RoundingMode,
): CalcResult['byTaxRate'] {
  const rateGroups = new Map<TaxRate, number>();

  // まず税率区分ごとに税抜合計を集計する
  for (const item of items) {
    const subtotal = calcLineSubtotal(item);
    const current = rateGroups.get(item.taxRate) ?? 0;
    rateGroups.set(item.taxRate, current + subtotal);
  }

  // 集計後に1回だけ端数処理をかける
  const result: CalcResult['byTaxRate'] = [];
  for (const [rate, subtotal] of rateGroups) {
    const taxAmount = applyRounding(subtotal * rate, roundingMode);
    result.push({ rate, subtotal, taxAmount, total: subtotal + taxAmount });
  }
  return result;
}
```

端数処理方式は切捨て・四捨五入・切上げの3方式を設定で選択できます。デフォルトは切捨て（消費税法の一般的な慣行）です。

純関数に切り出しているのでvitestでのユニットテストが書きやすく、境界値（標準10%・軽減8%の混在、端数処理3方式）をすべてカバーしています。

## 源泉徴収の計算: 100万円の境界値

フリーランスのデザイナー・ライター・エンジニアなど、特定の職種では請求額から源泉徴収税が差し引かれます。確定申告のとき還付される税金ですが、控除額が請求書に明記されていないとトラブルになることがあります。

源泉徴収の税率計算には100万円という境界値があります。

- 税抜報酬の100万円以下の部分: **10.21%**（所得税10% + 復興特別所得税0.21%）
- 100万円を超える部分: **20.42%**（所得税20% + 復興特別所得税0.42%）

```typescript
export const WITHHOLDING_THRESHOLD = 1_000_000;
export const WITHHOLDING_RATE_LOW  = 0.1021;
export const WITHHOLDING_RATE_HIGH = 0.2042;

export function calcWithholding(taxExcludedTotal: number): number {
  if (taxExcludedTotal <= WITHHOLDING_THRESHOLD) {
    return Math.floor(taxExcludedTotal * WITHHOLDING_RATE_LOW);
  }
  const low  = Math.floor(WITHHOLDING_THRESHOLD * WITHHOLDING_RATE_LOW);
  const high = Math.floor((taxExcludedTotal - WITHHOLDING_THRESHOLD) * WITHHOLDING_RATE_HIGH);
  return low + high;
}
```

源泉徴収の端数処理は法定で1円未満切捨てなので、ここだけは設定に関わらず `Math.floor` を使います。チェックボックスをONにすると請求書に控除額が自動表示されます。

## PDF出力はprint CSSで

「PDF出力」と聞くとjsPDFやPuppeteerを思い浮かべるかもしれませんが、日本語フォントの埋め込みが複雑になるので今回は採用しませんでした。

代わりに **print CSS方式**を使います。ブラウザの印刷機能（Ctrl+P / Cmd+P）を使ってPDF保存する方法です。

```css
@media print {
  .no-print { display: none !important; }
  
  body {
    background: white;
    font-size: 10pt;
  }

  .invoice-page {
    width: 210mm;
    min-height: 297mm;
    padding: 15mm;
    page-break-after: always;
  }
}
```

ブラウザが日本語フォントのレンダリングを全部やってくれるので、文字化けや文字欠けの心配がありません。印刷プレビューで最終確認できるのも利点です。欠点は「ブラウザの印刷ダイアログを経由する」という一手間ですが、フリーランスの請求書発行頻度を考えるとそこまで不便ではないと思っています。

## 登録番号の形式バリデーション

適格請求書発行事業者の登録番号は「T + 13桁の数字」という形式です。未登録の方は登録番号なしで発行できる猶予期間がありましたが、フォームの入力補助として形式チェックは入れています。

```typescript
export function isValidRegistrationNumber(value: string): boolean {
  return /^T\d{13}$/.test(value.trim());
}
```

未登録の場合は登録番号欄を空欄にして発行できます。適格請求書にはなりませんが、簡易的な請求書として使えます。

## 作ってみての感想

「バックエンドなし・完全ローカル」という制約を最初に決めたことで、実装の選択肢がシンプルになりました。セキュリティの考慮も「外部通信しない」で大部分が解決できています。

インボイス制度と源泉徴収は税法の話なので、実装する前に国税庁のWebサイトで要件を確認しました。ルールが複雑に見えますが、計算ロジック自体は純関数に落とし込めるので、テストを書きながら進めると意外とすっきり実装できます。

---

*サクッと請求書は https://sakutto-invoice.pages.dev で公開しています。ブラウザで開くだけで使えます（登録不要）。*

*本ツールは請求書の作成をサポートするものであり、税務アドバイスを提供するものではありません。源泉徴収の要否・税率の適用などについては税理士または税務署にご確認ください。*
