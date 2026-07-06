---
title: "初めての Firefox 拡張、2 回落ちてから 3 日で公開された"
emoji: "🎉"
type: "tech"
topics: ["firefox", "webextension", "amo", "manifestv3", "個人開発"]
published: true
---

![](/images/20260706-amo-approved-after-two-rejections_hero.png)

[前回の記事](https://zenn.dev/ishizakahiroshi/articles/20260703-amo-first-submission-rejected-twice)で、初めての Firefox 拡張提出が 2 回落ちて、直してアップロードし直したところまで書きました。あれから 3 日。今朝 9 時ごろ、メールが届いていました。承認でした。

朝のコーヒーを淹れながらスマホを見て、件名だけで分かりました。`tentatively approved` という書き方に一瞬引っかかりつつも、公開 URL が発行されていて、実際にアクセスすると自分の拡張のページが表示されました。今日はこの 3 日間と、実装の最終形の話です。

![](/images/20260706-amo-approved-after-two-rejections_infographic.png)

## そもそもの発端を 1 行で

X の複数アカウントを同時に開くと、タブが全部「ホーム / X」で見分けがつかない。この地味な不満から始まった話は、[Chrome 拡張の自作に挫折して Firefox の Multi-Account Containers にたどり着いた記録](https://zenn.dev/ishizakahiroshi/articles/20260702-x-multi-account-firefox-container)に詳しく書きました。半年ほどの遠回りの末、コンテナ機能そのものは公式に任せて、自分が作るのは「タブタイトルの先頭にコンテナ名を差し込むだけ」の薄い拡張にする、というところまで方針が固まった話です。

その拡張を実際に組んで AMO に出したら、`data_collection_permissions` の宣言漏れと、失敗アップロードの下書きが残っていたことによる ID 衝突で 2 回落ちた、というのが前回の記事でした。

3 本目のこの記事は、承認された今の話と、実装が最終的にどう固まったかの話です。

## 「暫定承認」の文面が、少しだけ気になった

メールにはこう書いてありました。要約すると「自動審査で通過し、暫定的に承認しました」。自動審査を抜けても、後から人力レビューに回って修正依頼や取り下げが来る可能性は残っている、というニュアンスです。

正直、ここで気持ちが半分だけ晴れました。ストアに並んだこと自体は嬉しいのですが、まだ完全に「終わった」とは言い切れない状態です。実際に自分の Firefox に AMO 経由でインストールし直して、3 つのコンテナで x.com を開いて、タブタイトルの先頭に `[個人]` `[開発]` `[投資]` が出るのを確認しました。動いています。あとは数日、追加のレビュー通知が来ないかを見るだけです。

## 実装は最終的に 3 ファイル、100 行に収まった

前回の記事で載せた PoC は 20 行程度でしたが、実運用に載せるにあたって、設定の保存・国際化・権限の絞り込みを足しました。それでも全体で 100 行程度です。

サービスワーカー側はコンテナ名を返すだけです。

```js:service_worker.js
async function getContainerName(tab) {
  if (!tab || !tab.cookieStoreId || tab.cookieStoreId === "firefox-default") {
    return null;
  }
  try {
    const identity = await browser.contextualIdentities.get(tab.cookieStoreId);
    return identity ? identity.name : null;
  } catch (e) {
    return null;
  }
}

browser.runtime.onMessage.addListener((message, sender) => {
  if (!message || message.type !== "getContainerName") return;
  return getContainerName(sender.tab);
});
```

content script 側は、設定を読んでプレフィックスを組み立て、`MutationObserver` で SPA の title 書き換えに追従する部分です。

```js:content.js
async function init() {
  const settings = await browser.storage.local.get(["enabled", "format"]);
  const enabled = settings.enabled !== false;
  if (!enabled) return;

  const containerName = await browser.runtime.sendMessage({ type: "getContainerName" });
  const prefix = computePrefix(containerName, settings.format || DEFAULT_FORMAT);
  if (!prefix) return;

  applyPrefix();
  const titleEl = document.querySelector("title");
  if (titleEl) {
    new MutationObserver(applyPrefix).observe(titleEl, { childList: true });
  }
}
```

![](/images/20260706-amo-approved-after-two-rejections_fig1.png)

上の図が最終的な権限まわりです。実装中は `tabs` 権限も付けていたのですが、リリース前の見直しで実は使っていないことに気付いて削りました。メッセージの送信元は `sender.tab` で権限なしに取れて、コンテナの判別に要るのは `cookies` だけで足りたからです。権限が 1 つ減ると、インストール時にユーザーへ見せる警告も 1 つ減ります。地味ですが、公開前に見直しておいてよかった点です。

## タブの見え方は、結局こう変わった

![](/images/20260706-amo-approved-after-two-rejections_fig2.png)

上の図が半年越しで欲しかった状態です。色線を目で追わなくても、タブの文字を読むだけでどの垢か分かります。地味な変化ですが、日に何十回も繰り返す動作なので、積み重なる負担の減り方は思ったより大きいです。

ストアの説明ページに載せるスクリーンショットも一応 1 枚用意したのですが、最初に撮ったものは自分の実アカウントの投稿がそのまま写り込んでいました。タブタイトルはページの DOM の一部なので、訪問先のサイトから見えるだけでなく、こういう「見せるための画像」にも実際の投稿内容がそのまま乗ってしまいます。結局、ダミーのアカウント名と架空の投稿で撮り直しました。地味に忘れがちな落とし穴でした。

## 学んだこと

- 「暫定承認」は公開されたその日がゴールではない。数日は追加レビューの可能性を見込んでおく
- 実装は最初の PoC から大きく増えなくていい。今回は結局 3 ファイル 100 行程度で収まった。むしろ権限を削る作業のほうが後から効いてくる
- 公開用のスクリーンショットにも、タブタイトル経由で実データが写り込む。載せる前に一呼吸置いて確認する価値がある
- 半年遠回りしても、最後に残る「本当に作るべきもの」は驚くほど小さい。今回で言えば 100 行の拡張だった

## 締めに

X の複数アカウント運用は、Chrome のプロファイル分離に挫折し、自作の Cookie 分離拡張で負けて、Firefox のコンテナ機能に流れ着き、最後に「タブタイトルにコンテナ名を差し込むだけ」の小さな拡張で一段落しました。3 本の記事を通して振り返ると、遠回りした分だけ、最後の一手がどれだけ小さくて済むかがはっきり見えた気がします。

:::message
この拡張は Firefox の Multi-Account Containers（コンテナ機能）が前提です。持っていない場合は先にそちらをインストールしてコンテナを作っておく必要があります。コンテナが 1 つもない状態だと、プレフィックスは何も差し込まれません（デフォルトコンテナの挙動をそのまま維持する設計のためです）。
:::

コードは [GitHub](https://github.com/ishizakahiroshi/tab-title-prefix) に、拡張本体は [Firefox Add-ons](https://addons.mozilla.org/addon/tab-title-prefix/) にあります。Chrome 対応の手動 URL ルール機能は、また気が向いたら次の話にします。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
