---
title: "ポートフォリオに DL を出したら、Star 同期が一度も動いていなかった"
emoji: "📈"
type: "tech"
topics: ["javascript", "githubpages", "oss", "claudecode", "ポートフォリオ"]
published: true
---

![](/images/2026-06-22_portfolio-dl-pill-and-broken-star-sync_hero.png)

自作の個人 OSS を並べた GitHub Pages のポートフォリオがあって、いままで各カードには「★ 4」のような Star 数だけ出していました。

そこに Releases のダウンロード数も足したいと思っていました。実用以前に「触る前にどれくらい使われているか」が見えるかどうかで、知らない人の印象は全然変わるはずで、社会的証明として欲しかった、というのが本音です。

dl-stats という自前の集計 API（Cloudflare Workers + D1）はすでに動いていて、Stars もそこから持ってきている **つもり** でした。なのでこれは「DL も同じパスで足す」だけの軽い作業のはずだった。実際は、まったくそうではなかったです。

検索用に置いておくと: GitHub Pages, ポートフォリオ, Releases ダウンロード数, JavaScript, ts-check, dl-stats, sparkline, 個人 OSS。

## 数字を足す前に、★ 同期が一度も動いていなかった

モックで「★ 4 · ↓ 164 · 30 日 sparkline」のセットを並べてみて、実装に入る前に既存の `fetchDlStats()` を読み直しました。

そこで気になったのが、ある OSS のカードに表示されている Star 数が、現実より 1 つ少なかったことです。GitHub 上では ★ 10 なのに、カードは ★ 9。最近 Star が増えたばかりだから、まあ間に合っていないだけだろう、と一度はスルーしました。

ただ、dl-stats のダッシュボード側を見たら ★ 10 になっています。dl-stats からは ★ 10 が返ってきているのに、ポートフォリオ側が ★ 9 のまま。これは「間に合っていない」ではなく「反映されていない」だ、とそこで気付きました。

## 既存コードの比較式が、一度も一致しなかった

`fetchDlStats()` の中身はだいたいこうなっていました。

```js:assets/app.js
const w = WORKS.find((x) => x.repo === `https://github.com/${tool.repo}`);
if (w && typeof tool.metrics?.stars === "number") {
  w.stars = tool.metrics.stars;
}
```

API の `tool.repo` を `https://github.com/` の後ろに連結して、`WORKS` 配列にある完全な GitHub URL と完全一致で比較する、という形です。

`WORKS[].repo` は `"https://github.com/ishizakahiroshi/many-ai-cli"` のようなフル URL。これに対して API 側がどう返してくるかというと、

```json
{ "tools": [ { "repo": "many-ai-cli" } ] }
```

owner プレフィックスがありません。連結すると `https://github.com/many-ai-cli` になります。これと `https://github.com/ishizakahiroshi/many-ai-cli` を `===` で比較しても、当然一度も一致しません。`WORKS.find(...)` は毎回 `undefined` を返し、`if (w && ...)` の中には入らない。Star 数の更新は **一度も実行されていなかった** わけです。

![](/images/2026-06-22_portfolio-dl-pill-and-broken-star-sync_fig1.png)

:::message
途中で dl-stats のディスカバリ仕様が変わって `tool.repo` から owner が落ちた可能性が高いですが、いつどう変えたかは追えませんでした。「同期は動いている」という思い込みのまま、ずっと静的に書いた `stars: 9` を出し続けていました。
:::

## マッピングを末尾セグメントに直したら、いままでの Star 増分が一気に反映された

plan には「マッピング前提: `WORKS[].repo` の末尾セグメント ↔ `tools[].repo`」と書いてありました。設計時点ではこちらが意図だったのに、実装が完全 URL 比較になっていた、ということになります。

末尾セグメント比較に直すだけです。

```js:assets/app.js
const toolSlug = tool.repo.split("/").pop();
const w = WORKS.find((x) => x.repo.split("/").pop() === toolSlug);
```

これを当ててリロードした瞬間、件のカードが `★ 9` から `★ 10` になりました。**Star 同期は今この瞬間に初めて動きました**。1 つ 1 つ確認したわけではないけれど、おそらく他のカードもそれまで GitHub の実値とは一致していなかったはずです。

## 数字の上に、矢印アイコンと 30 日 sparkline を足す

ここまでで Star は直りました。本題の DL 表示を足します。

決めごとはモックで固めてあって、

- `↓ N` のピル（白地・矢印アイコン）。「DL」というラベルは付けない
- DL が 0 でも `↓ 0` を出す（欠損か未配布かの区別）
- npm 主体カード向けには `↻ N 30d` を並列に。Releases と両方ある場合は `↓ N · ↻ N 30d`
- カード末尾に 30 日 sparkline。色はカードのテーマカラー（`var(--c)`）に揃える
- CTA「アプリDL数」カードの説明文を「のべ N DL / ★ M」に動的化

sparkline は外部ライブラリを入れたくなかったので、80×20px の軽量 SVG `<polyline>` をその場で組み立てるヘルパに収めました。値が空 / 非数値 / すべて同値ならフラット線、`undefined` を含むなら描かない、というだけの 20 行ぐらいの関数です。

```js:assets/app.js
function sparklineSvg(values, color) {
  if (!Array.isArray(values) || values.length === 0) return "";
  for (const v of values) {
    if (typeof v !== "number" || !Number.isFinite(v)) return "";
  }
  // ...polyline points をその場で計算して SVG 文字列を返す
}
```

CTA 側は `dlStatsCache`（モジュールスコープの変数）に `cumulativeInstalls` と `totals.stars` を保存しておいて、`renderWorks(lang)` が呼ばれた時点でキャッシュがあれば「のべ 314 DL / ★ 16」に差し替え、無ければ既存の固定文に戻る、という分岐にしました。`fetch` 自体は初回 1 回だけで、日本語と英語の切替で再 fetch が走らないようにしたかったので。

## 「カードと詳細を同じ仕様に」と言ったのは、自分

カードの表示が一通り入って、本番にデプロイして、目視で確認して、「OK じゃない」と思いました（チャットにそう書いたら、AI 側は不安そうにしていました）。

その直後にもう 1 つ気付いて、自分から「カードをクリックした後の詳細ページにも同じ仕様にしないとだめだよね」とフィードバックを入れました。詳細ページ（`work.html?id=…`）は同じ作品データを別レイアウトで見せている画面で、カードに `↓ 164` が出ているのに詳細を開いたら `★ 4` しか無い、というのは情報が削れるだけです。

ここで初めて、「カードと詳細でメタ部分の HTML を別々に書いている」状態が見えました。

![](/images/2026-06-22_portfolio-dl-pill-and-broken-star-sync_fig2.png)

やったことはシンプルで、`renderWorkStats(w, lang)` というヘルパを 1 つ作って、メタ行と sparkline を `{ metaHtml, sparkHtml }` で返す形にしました。カード側も詳細側も、これを呼んで `innerHTML` に差し込むだけです。

CSS も合わせて整理しました。これまで `.card .meta` / `.card .dl` / `.card .spark` のように `.card` スコープでぶら下がっていたのを、`.meta` / `.dl` / `.spark` / `.sep` に剥がして、カード固有の `margin-bottom: 12px` だけ `.card .meta` 側に残す形に。

:::details `.card .X` を `.X` に剥がす時のチェック
他の場所で `.meta` `.dl` `.spark` `.sep` が使われていないかは grep で確認しました。プロジェクト固有のクラス名で衝突しなさそうなものだけ剥がしています。`.star` `.go` のような汎用名は `.card .star` のままにしてあります（剥がしたら別の場所のスタイルにぶつかるので）。
:::

## 「同じ仕様」は実装で揃える

これは reviewer 不在の自分のリポなので、最初から「カードと詳細で同じデータを出す」ことに気付いて 1 本のヘルパで書いておけば良かった話です。

最初に書いた時点では「カードに何を出すか」しか考えていなかった。詳細ページは別件、と頭で勝手に分けていました。カードで実装して、本番に上げて、見て、「これ、詳細ページにも同じ仕様にしないとだめだよね」と自分で気付くまで、間が空いていました。

書いてあるルールは単純で、「同じ見た目を同じ HTML で出す」「カードと詳細で別々のクラス名を使わない」。それを最初から守るのは難しくて、もう 1 階レイヤを足してから初めて見える、というケースが多い気がしています。

## 学んだこと

- 自前 API の境界では「型」だけでなく「形（命名）」も契約として扱う。`tool.repo` が `"many-ai-cli"` なのか `"owner/many-ai-cli"` なのかで、完全 URL 比較は破綻する
- 「同期は動いているはず」の前提は、実値と画面の差で 1 回は確認しておく。今回は ★9 と ★10 の 1 だけの差で気付けたけど、もっと前に Star が動いていなかったら気付かないままだった
- 同じデータを 2 画面に出すなら、最初から共通ヘルパに切り出す。後付けでリファクタすると遅い

並行して前日に Claude Code 側の context を 14% ほど削っていて、その記事はこちらに書きました。

https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet

ポートフォリオ側の本番はこちらで動いています。

https://ishizakahiroshi.github.io/

小さく。公開している数字に責任を持つ、を、これからも習慣にしていきます。

---

書いた人: ishizakahiroshi
田舎在宅エンジニア（バックエンド・インフラ・AI連携）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X : https://x.com/ishizakahiroshi
