---
title: "初めての Firefox 拡張を AMO に出したら、公開前に 2 回落ちた"
emoji: "📮"
type: "tech"
topics: ["firefox", "webextension", "amo", "manifestv3", "個人開発"]
published: true
---

![](/images/20260703-amo-first-submission-rejected-twice_hero.png)

[昨日の記事](https://zenn.dev/ishizakahiroshi/articles/20260702-x-multi-account-firefox-container)で、X の複数アカウント運用が Firefox の Multi-Account Containers に落ち着いた話を書きました。今日はその続きです。コンテナで開いた 3 つのタブが全部「ホーム / X」で見分けがつかない問題を、95 行の自作拡張で解決して、addons.mozilla.org（AMO）に初めて提出しました。

朝から始めて、リポジトリ作成、実装、動作確認まで順調。「あとはストアに上げるだけ」の状態で昼前。ここから検証に 2 回落ちました。

![記事の要約](/images/20260703-amo-first-submission-rejected-twice_infographic.png)

## 1 回目: 見たことのないプロパティを要求された

AMO の開発者センターで xpi をアップロードすると、自動検証が走ります。セキュリティテスト 0 エラー、拡張機能テスト 0 エラー、ローカライズ 0 エラー。よし、と思ったら一般テストで 1 エラー。

```
The "/browser_specific_settings/gecko/data_collection_permissions"
property is required for all new Firefox extensions
```

`data_collection_permissions`。ローカルの `about:debugging` では一度も見なかった名前です。

調べると、2026 年から新規の Firefox 拡張に必須になった宣言でした。拡張がどんなデータを収集するかを manifest に書き、Firefox がインストール時にユーザーへ組み込み表示する仕組みです。ポイントは、**データを一切収集しない拡張でも「収集しない」と宣言しなければいけない**こと。黙っていれば収集なし扱い、ではなく、無宣言はエラーです。

収集ゼロの場合はこう書きます。

```json:manifest.json
"browser_specific_settings": {
  "gecko": {
    "id": "tab-title-prefix@example.com",
    "data_collection_permissions": {
      "required": ["none"]
    }
  }
}
```

:::message
ローカル開発（`about:debugging` の一時読み込み）ではこのエラーは一切出ません。AMO の検証に投げて初めて発覚します。新規拡張を作るときは最初から manifest に入れておくのが正解です。
:::

manifest に 4 行足して、xpi を作り直して、再アップロード。

## 2 回目:「重複したアドオン ID」。まだ 1 本も公開してないのに

今度は別のエラーが出ました。

```
重複したアドオンの ID が見つかりました。
```

これは正直、意味が分かりませんでした。この拡張はまだ一度も公開されていません。ID は自分のドメイン風文字列で付けた一点もの。誰かに先を越されるはずもない。なんでだ。

真因は AMO 側の挙動でした。**検証に失敗したアップロードでも、アカウントには「下書き」が残ります**。1 回目の失敗で下書きが作られていて、そこにもう同じ ID が登録済み。その状態で「新しいアドオンを登録」からやり直したので、自分の下書きと衝突していたわけです。

対処は簡単で、開発者ハブの「自分のアドオン」一覧を開くと、さっき失敗したはずの拡張が下書き状態で居座っています。それを開いて、続きから修正版の xpi をアップロードすれば通ります。新規登録からやり直すのではなく、下書きの続きを開く。これだけでした。

![](/images/20260703-amo-first-submission-rejected-twice_fig1.png)

図にすると、つまずいた場所はこの 2 箇所です。1 回目は manifest の不足で正面から弾かれ、2 回目は「失敗しても下書きが残る」という仕様を知らずに入口を間違えました。どちらもエラーメッセージだけ読むと遠回りしそうになりますが、答えは近くにあります。

## 落ちなかったけど、知らなかったこと

提出フォームを進める中で、検証エラー以外にも「へえ」がいくつかありました。

まず、**WebExtension の manifest にはライセンスを書く欄がありません**。npm の package.json 感覚で探しても無い。オープンソースのライセンス表明は AMO の提出フォームで選ぶのが唯一の場所です。リポジトリの LICENSE と食い違わないよう、ここだけは手で合わせる必要があります。

それから権限の話。実装中は `tabs` 権限を付けていたのですが、リリース前の監査で「これ使ってないのでは」と気付きました。メッセージ送信元の `sender.tab` は権限なしで取れて、コンテナ判定に使う `cookieStoreId` は `cookies` 権限で露出します。`tabs` を削って実機確認しても全部動きました。権限が 1 つ減ると、インストール時にユーザーへ見せる警告も減ります。

![](/images/20260703-amo-first-submission-rejected-twice_fig2.png)

上の図が、今日の経験を「次に Firefox 拡張を作る自分」向けにまとめたチェックリストです。ちなみに一番上の `background.scripts` は開発初日に踏んだやつで、Firefox の MV3 は Chrome と違って `service_worker` 形式に対応していません。エラー文が「Add background.scripts.」と修正方法まで教えてくれるので、これは迷いませんでした。

## 学んだこと

- AMO の検証要件はローカル開発では見えない。新規拡張の manifest には最初から `data_collection_permissions` を入れる（収集ゼロでも `required: ["none"]` の宣言が必須）
- アップロードに失敗しても下書きは残る。「重複 ID」エラーが出たら、新規登録ではなく「自分のアドオン」一覧の下書きを開く
- ライセンスの表明場所は提出フォームだけ。manifest には書けない
- タグを打つのは AMO の検証を通過してから。先に打つと、manifest 修正のたびに打ち直しになる（なりました）

提出は無事に完了して、いまは審査待ちです。AMO は登録も公開も無料で、審査は最大 24 時間とのこと。メールを待ちながら、この記事を書いています。

公開されたら、コンテナ生活の仕上げとして使ってもらえる状態になります。コードは [GitHub](https://github.com/ishizakahiroshi/tab-title-prefix) に置いてあります。小さく、まず自分の 3 タブから。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
