---
title: "X の複数アカウント運用: Chrome 拡張で挫折 → Firefox Multi-Account Containers に流れ着いた記録"
emoji: "🦊"
type: "tech"
topics: ["firefox", "chrome", "browserextension", "manifestv3", "cookie"]
published: false
---

![](/images/2026-07-02_x-multi-account-firefox-container_hero.png)

X で複数アカウントを持っていて、それを 1 ウィンドウの中で行ったり来たりしたい。長らくこの当たり前の要望を満たせる手が見つからなくて、拡張を 1 本自作して、それが負けて、最後に Firefox のコンテナ機能に落ち着きました。今日はその紆余曲折を残しておきます。

複数アカウントを 3 つ、一日中同時に開きます。個人・開発・投資関連の 3 つで、朝から夜まで文字通り並べて眺めます。「見て、切り替えて、書く」を繰り返す作業なので、3 つの間の移動が 1 テンポでも重いとストレスが積みあがっていきます。この記事はその「1 テンポ」を軽くするための旅の話です。

![](/images/2026-07-02_x-multi-account-firefox-container_infographic.png)

上の 1 枚が今回の話の全体図です。Chrome プロファイルの重さ・自作拡張の敗因・Firefox Container の分離モデル・残った UX 課題までを 1 枚にまとめてあります。以降の本文はこの図を段ごとに掘っていく形で進みます。

## 「Chrome のプロファイル使えばいいじゃん」で 3 秒で詰む

一番最初の答えは、みんな同じことを言います。**Chrome のプロファイル機能を使えばよい**、と。

これは半分正解で、半分どうしようもなく実務に合いません。プロファイルを切り替えると、それぞれが**独立した Chrome のウィンドウとして立ち上がる**からです。2 つのアカウントを扱うだけで、画面には Chrome のウィンドウが 2 個並びます。3 つなら 3 個。もはやアカウント切り替えツールというよりウィンドウマネージャの練習問題です。

さらに、それぞれのウィンドウが自分の履歴とタブ列を持つので、「昨日開いたあのポスト、どっちの垢だっけ」を毎回思い出す必要があります。Chrome の拡張機能はプロファイルごとに独立にインストールしないといけないし、ブックマークもプロファイルごとに分かれます。3 プロファイル分の管理コストが素直に積みあがります。

![](/images/2026-07-02_x-multi-account-firefox-container_fig1.png)

上の図が今回の記事の出発点です。左の「プロファイルで分ける」が実際に取れる公式解、右が本当に欲しかった状態。「同じ Chrome の 1 ウィンドウの中で、タブごとに別のアカウントとしてログインしていてほしい」。この 1 行だけがしぶとく残り続けました。

## タブ単位で Cookie を分離する拡張を自作した

「無いなら作る」で拡張の設計を書きました。名前は `many-tab`。同一プロファイル・同一ウィンドウのまま、タブごとに Cookie と localStorage を分離するというコンセプトです。GitHub の [`ishizakahiroshi/many-tab`](https://github.com/ishizakahiroshi/many-tab) にコードは残しています。

設計は 2 段です。

まずネットワーク層に対して `declarativeNetRequest` で **タブ ID ごとに Cookie ヘッダを差し替える**。同じ URL への 2 つのタブから同時に飛んでいくリクエストでも、送信直前にヘッダの `Cookie` を書き換えれば、サーバ側から見ると別セッションに見えるはず、というアイデアです。

次にページ側の JS 実行環境に対しては、content script（MAIN world）で `document.cookie` と `localStorage` を **仮想化する**。SPA は起動時に `localStorage` を読んでログイン状態を復元するので、この復元先を per-tab に分けてやれば、同じサイトのタブでもタブごとに別のセッションで立ち上がってくれる、と考えました。

![](/images/2026-07-02_x-multi-account-firefox-container_fig2.png)

上の図が many-tab の分離モデルです。DNR で下から、content script で上から、Cookie と localStorage を挟み撃ちにする格好です。

軽量な自作 SaaS や、社内で建てている自前の管理画面のような「Cookie + localStorage で完結する」アプリでは、これはちゃんと動きました。admin 画面と一般ユーザー画面を同じウィンドウの隣り合ったタブで並べて操作できるようになって、「あ、動く」の手応えはあったのです。

## X で見事に負けた

意気揚々と X で試したところ、まったく分離できませんでした。片方のタブで別垢にログインしたはずが、もう片方のタブにも同じ垢のセッションが被ってくる。数分粘って、何度リロードしても、境界が引けない。

原因を追いかけて、現代の SaaS が保持している「識別できる状態」の在処が Cookie と localStorage だけではないことに、遅ればせながら気付きました。

![](/images/2026-07-02_x-multi-account-firefox-container_fig3.png)

上の図が現代 SaaS の状態管理の階層です。**Cookie と localStorage の上に、IndexedDB と Service Worker（Cache API 含む）が乗っていて、さらにその上に fingerprinting と反 multi-account 検知が乗っている**。X のように大規模で、かつアンチ multi-account の需要が明確にあるサービスは、これら全部を組み合わせて identity を確定させにきます。

many-tab がカバーしているのは、実質下 2 層だけです。IndexedDB は per-tab では上書きできず、Service Worker はブラウザプロセスにひとつ登録されると全タブで共有され、Cache API はサービスワーカ経由でクロスタブに漏れます。fingerprinting や UA・IP まで到達したら、もはや拡張の守備範囲を大きく超えています。

![](/images/2026-07-02_x-multi-account-firefox-container_fig4.png)

上の図が敗因のシンプルな要約です。下 2 層だけ分離しても、上 3 層が繋がっていたら SaaS 側から見た identity は繋がったままです。X ではまさにそれが起きていました。手を尽くしても、この壁は自作拡張では超えられそうにないと判断しました。

many-tab の README にはその後、動作対象外の一覧を明記してあります。X、Google 系全般、Instagram、Slack、Notion、Figma、Linear。**現代のちゃんとした SaaS はだいたい全部対象外**、というのが実態でした。強い教訓が残りました。

## Firefox の Multi-Account Containers に流れ着く

負けを認めて、Firefox の Multi-Account Containers を試しました。同じ「1 ウィンドウで複数アカウント」を、まったく違うアプローチで実現している機能です。

Multi-Account Containers はブラウザレベルで**コンテナごとにストレージパーティションを完全に分離**します。Cookie も localStorage も IndexedDB も Service Worker も Cache API も、コンテナが違えば違うディレクトリに書かれます。ページ側の JS からは同じサイトでも、下のストレージ層が物理的に別なので、SaaS 側から見ても完全に別 identity として扱われます。

![](/images/2026-07-02_x-multi-account-firefox-container_fig5.png)

上の図がコンテナ分離のモデルです。分離の位置が **下から上まで一貫している**ので、many-tab では超えられなかった上 3 層も同じ壁で仕切られています。X ですら、コンテナが違えば別垢としてログインを保持できました。

X で 3 コンテナを開くまでの流れは驚くほど平坦でした。コンテナを 3 つ作り、それぞれのコンテナで新しいタブを開き、それぞれで x.com にログインし直すだけ。10 分もかからず 3 垢環境が完成しました。ここに至るまでの半年ほどの自作は何だったのか、と一瞬遠い目になりました。

![](/images/2026-07-02_x-multi-account-firefox-container_fig6.png)

上の図が Chrome の profile と Firefox の Container の UX の差です。同じ目的を達成できていても、画面に何個ウィンドウが並ぶかで作業のしやすさは全然違います。3 プロファイルは 3 ウィンドウ、3 コンテナは 1 ウィンドウで済みます。この差は毎日効きます。

## それでも残った、地味な UX の不満

3 垢を 1 ウィンドウで運用できるようになって満足かというと、そうでもありませんでした。**タブがぜんぶ「ホーム / X」で見分けがつかない**という、地味だけど毎日効く問題が残りました。

Multi-Account Containers はタブの上部に細い色の線を引いてくれます。コンテナごとに色が違うので、その線でどのタブがどのコンテナかは分かるようにはなっています。ただ、視認性は正直かなり弱いです。カーソルを合わせてホバーすればコンテナ名がツールチップで出ますが、目線を動かして 1 秒待つ動作が挟まる時点で、素早い切り替えの敵です。

![](/images/2026-07-02_x-multi-account-firefox-container_fig7.png)

上の図が現状の見え方です。タブの表示上、色線を凝視しないと区別できません。3 本並ぶタブが全部「ホーム / X」と表示されていて、コンテナで完璧に分離されているという裏側の頑張りが、UI 上ではほとんど見えていない状態です。

## 次にほしいのは「タブタイトルにコンテナ名」

たどり着いた欲しいものは、拍子抜けするほどシンプルでした。**タブのタイトルの先頭にコンテナ名を差し込みたい**。それだけです。`[個人] ホーム / X` `[開発] ホーム / X` `[投資] ホーム / X` の 3 本に見えれば、色線に頼らず一発で見分けられます。

![](/images/2026-07-02_x-multi-account-firefox-container_fig8.png)

上の図が実現したい状態です。技術的には、Firefox の `contextualIdentities` API でコンテナ名を取得し、content script から `document.title` の先頭にプレフィックスを差し込むだけで実装できそう、と目星が付きました。SPA が title を書き換えてくるので `MutationObserver` で監視して都度上書きする必要はあります。手を動かせば、100 行前後で 1 本にまとまる規模です。

PoC としては、次の 20 行程度で骨格が組めます。

```js:background.js
// タブが属するコンテナ名を content script に返す
browser.runtime.onMessage.addListener(async (_msg, sender) => {
  const storeId = sender.tab?.cookieStoreId;
  if (!storeId || storeId === "firefox-default") return null;
  const id = await browser.contextualIdentities.get(storeId);
  return id?.name ?? null;
});
```

```js:content.js
(async () => {
  const name = await browser.runtime.sendMessage({ type: "getContainerName" });
  if (!name) return;
  const prefix = `[${name}] `;
  const apply = () => {
    if (!document.title.startsWith(prefix)) {
      document.title = prefix + document.title;
    }
  };
  apply();
  // SPA が route 遷移で title を書き換えるので MutationObserver で都度再適用
  const titleEl = document.querySelector("title");
  if (titleEl) {
    new MutationObserver(apply).observe(titleEl, {
      childList: true, subtree: true, characterData: true
    });
  }
})();
```

あとは manifest 側に `permissions: ["contextualIdentities", "cookies"]` と、`content_scripts` の `matches: ["<all_urls>"]` を足すだけで、Firefox の Multi-Account Containers 前提でタブのタイトルにコンテナ名が自動で差し込まれます。実運用ではデフォルトコンテナの扱い（プレフィックスなし）・コンテナ名変更のリアルタイム反映・SPA が `<title>` 要素そのものを差し替えてくるケースへの耐性、あたりを足していく形になります。

いま `tab-title-prefix` という名前で MVP を組んでいるところです。Firefox の Multi-Account Containers 前提でコンテナ名を自動プレフィックスする Phase 1 と、Chrome を含む「手動 URL ルール」で任意プレフィックスする Phase 2 の 2 段構えで設計しています。まとまったら別記事で紹介します。

## 学んだこと

紆余曲折を通して残ったことを、まとめます。

- **現代 SaaS の identity は Cookie と localStorage だけでは決まらない**。IndexedDB、Service Worker、Cache API、fingerprinting までを組み合わせて確定させにくる。拡張がカバーできるのはせいぜい下 2 層で、そこから上は「ブラウザレベルの分離」に頼るしかない
- **公式が用意した分離機能は、そこを一貫して分けている**。Firefox の Multi-Account Containers はまさに 5 層を下から上まで同じ壁で仕切る作りで、コンテナ間で identity が混ざる余地がない
- **自作で境界を突破しようとすると、負ける方向にコストがかかる**。半年遊んで見えた壁は、実装で越えられる壁ではなく、公式の分離機能がある側に引っ越すのが正解だった
- **落ち着いた先で見つけた地味な不満は、たいてい 100 行程度の拡張で解ける**。色線でしか区別できないタブ列に対して、コンテナ名をタイトルに差し込むだけで解決するはず

負けをちゃんと負けと認めて、勝ち筋の側に引っ越して、それでも残った小さな不満を次の一手にする。この順番を守れば、作業環境は少しずつ静かに整っていきます。

## 締めに

X の 3 垢並行運用は、今日ようやく毎日の摩擦がだいぶ減った状態になりました。あと 100 行、書きます。それで年内は落ち着けるはずです。ここまでの試行錯誤が、同じような要件を持つ人の参考になれば嬉しいです。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
田舎在宅のシステムエンジニアです。実務 18 年、バックエンドとインフラと AI 連携が主戦場。「現場の業務課題を、最小限の実装で、確実に動くものにする」を信条にしています。業務委託・フルリモートで受注できます。こんな相談、歓迎です。

- プロフィール: https://ishizakahiroshi.github.io/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi
