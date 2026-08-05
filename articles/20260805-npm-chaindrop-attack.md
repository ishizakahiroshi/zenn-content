---
title: "npmとは何か。ChainDrop攻撃を依存関係・preinstall・provenanceから理解する"
emoji: "🛡️"
type: "tech"
topics: ["npm", "javascript", "security", "githubactions", "サプライチェーン"]
published: true
---

## npmの仕組みから、ChainDropサプライチェーン攻撃を読み解く

![npmとChainDrop攻撃のヒーロー](/images/2026-08-05_npm-chaindrop_hero.png)

![記事の要約](/images/2026-08-05_npm-chaindrop_infographic.png)

2026年8月4日、`keyv@6.0.0`を起点とする大規模なnpmサプライチェーン攻撃が見つかりました。攻撃は「ChainDrop」と呼ばれ、開発者端末やCI/CDにある認証情報を盗み、盗んだ権限で別のnpmパッケージを汚染するワームとして広がりました。

この記事は2026年8月5日23時56分JST時点の公開情報を基にしています。調査中の事案なので、影響パッケージと悪性バージョンの一覧は更新されます。固定の件数や、この記事に載せた11個の初期パッケージだけで安全判定しないでください。

## 急いでいるなら、先にここだけ確認してください

調査中の端末で、新たに`npm install`や`npx`を実行するのは止めます。確認用ツールを追加インストールしたことで、別のコードを動かしてしまっては本末転倒です。

最初の11個のフルペイロード保有パッケージは、次のコマンドで依存ツリーを確認できます。

```bash
npm ls keyv flat-cache file-entry-cache cacheable-request cacheable @cacheable/memory cache-manager @cacheable/node-cache @cacheable/utils @cacheable/net ecto --all
```

ただし、これは入口の確認にすぎません。ChainDropは盗んだ資格情報を使い、別組織が管理する数百のパッケージへ自己伝播しました。完全な確認には、Wiz Researchが公開している更新型CSVなどと、実際のlockfileを突き合わせます。

- 影響パッケージ一覧: https://github.com/wiz-sec-public/wiz-research-iocs/blob/main/reports/keyv-packages.csv
- StepSecurityの技術解析と検索可能な一覧: https://www.stepsecurity.io/blog/chaindrop-npm-worm

すでに`node_modules`がある場合は、今回の特徴的なファイルも読み取り専用で探します。

```bash
rg --files node_modules | rg '(^|/)(setup\.mjs|Math_Symbol\.js|math_init\.js)$'
```

PowerShellなら次でも確認できます。

```powershell
Get-ChildItem -LiteralPath .\node_modules -Recurse -File -ErrorAction SilentlyContinue |
  Where-Object { $_.Name -in @('setup.mjs', 'Math_Symbol.js', 'math_init.js') }
```

同名の正規ファイルが存在する可能性はあるので、名前が一致しただけで断定はしません。反対に、ファイルが消えていても安全とは限りません。インストール時に実行され、資格情報を送信した後で痕跡の一部が消えている可能性があるからです。

**悪性バージョンを開発端末またはCI/CDランナーでインストールした証拠がある場合、その環境から参照できた認証情報は漏えいした前提で扱います。** パッケージを安全版へ上げただけでは、すでに盗まれたトークンは戻ってきません。

## そもそもnpmは、何をしているのでしょうか

npmは、Node.jsのパッケージマネージャーです。同時に、次の3つを指す名前でもあります。

- パッケージを探したり管理したりするWebサイト
- ターミナルから`npm install`や`npm publish`を実行するCLI
- JavaScriptのパッケージとメタデータを保管するレジストリ

npm公式ドキュメントも、この3要素で説明しています。

https://docs.npmjs.com/about-npm/index.html

パッケージは、ざっくり言えば再利用できるソフトウェア部品です。npmへ公開するパッケージには`package.json`があり、名前、バージョン、依存する別パッケージ、実行スクリプトなどが記録されています。

```json
{
  "name": "example-package",
  "version": "1.2.3",
  "dependencies": {
    "another-package": "^4.0.0"
  }
}
```

`npm install`を実行すると、CLIはレジストリから必要なパッケージを取得し、通常は`node_modules`へ展開します。直接指定したパッケージが別のパッケージを必要とすれば、それも自動でたどります。

```text
自分のアプリ
├─ 直接依存A
│  └─ 推移的依存C
└─ 直接依存B
   └─ 推移的依存C
```

ここで開発者が自分で選んだのはAとBだけです。CはAやBの内部事情で入ってきます。これを推移的依存と呼びます。

便利です。本当に便利です。日付処理も、HTTP通信も、キャッシュも、入力検証も、全部をゼロから書かずに済みます。

同時に、信頼する相手が増えます。

![npmが取得・固定・展開・実行する流れ](/images/2026-08-05_npm-chaindrop_fig1.png)

図にすると、npmは単なるダウンローダーではありません。依存関係を解決し、版を固定し、ファイルを展開し、条件によってはインストール用コードまで実行する仕組みです。

## package.jsonとlockfileは、役割が違います

`package.json`に`"^4.0.0"`と書かれている場合、一定の範囲にある新しい版を許可します。毎回のインストールで結果が変わると困るので、実際に解決した正確な依存ツリーを`package-lock.json`へ記録します。

npm公式ドキュメントでは、lockfileは同じ依存ツリーを再現し、チーム、デプロイ、CIで同じ版を使うためのものと説明されています。

https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json/

CIでは、`npm install`より`npm ci`が向いています。`npm ci`は`package.json`とlockfileが一致しなければ失敗し、lockfileを書き換えません。

https://docs.npmjs.com/cli/v8/commands/npm-ci/

ただし、lockfileは万能のお守りではありません。

- 悪性バージョンを固定していれば、毎回同じ悪性バージョンを再現します
- リポジトリ外の`npx`やグローバルインストールは、プロジェクトのlockfileで固定されません
- テスト中に新規プロジェクトを生成する処理が、別の場所で最新版を解決することがあります
- 「いつ、どの端末で、そのlockfileを使ってインストールしたか」は通常のlockfileに残りません

StepSecurityは、BackstageのE2Eテストが新規アプリを生成して最新依存を取得したため、リポジトリのlockfileが変更されていないのに悪性コードが実行された事例を報告しています。

https://www.stepsecurity.io/blog/chaindrop-npm-worm

この話、かなり嫌です。`git diff`が空だから依存は変わっていない、と言い切れない経路があるわけです。

## 「インストールしただけで実行」は、なぜ起きるのでしょうか

npmにはライフサイクルスクリプトがあります。パッケージのビルドやネイティブモジュールの準備など、正当な用途で使う仕組みです。

```json
{
  "scripts": {
    "preinstall": "node setup.mjs",
    "postinstall": "node finish.js"
  }
}
```

`preinstall`、`install`、`postinstall`などは、インストール処理の途中で実行されます。npm公式の実行順序はこちらです。

https://docs.npmjs.com/cli/using-npm/scripts/

ChainDropの初期パッケージには、次の組み合わせが追加されていました。

```json
{
  "scripts": {
    "preinstall": "node setup.mjs"
  }
}
```

利用者がそのライブラリのAPIを呼ぶ必要はありません。アプリを起動する必要すらありません。依存関係の解決結果に悪性バージョンが入り、インストール処理が走れば、`setup.mjs`が先に動きます。

ここが「脆弱性のあるライブラリ」と「悪意あるパッケージ」の大きな違いです。脆弱性は特定の機能や入力を通したときに成立することがあります。今回は、インストールという開発の日常動作そのものが実行トリガーでした。

![正規の部品箱に実行装置が紛れ込んだ倉庫](/images/2026-08-05_npm-chaindrop_illustration-warehouse.png)

絵では「見た目が同じ正規の箱」を主役にしています。利用者が怪しいサイトから何かを拾ったのではなく、いつもの経路で正規名の部品を受け取ったことが、この攻撃のやっかいさです。

## サプライチェーン攻撃は、信頼の途中を狙います

サプライチェーン攻撃は、最終標的の企業や利用者を正面から破る代わりに、その相手が信頼して取り込む部品、更新経路、ビルド環境、取引先などを侵害します。

宅配便で考えると分かりやすいです。玄関をこじ開けるのではなく、正規の送り主、正規の伝票、いつもの配送経路を使う箱の中身を入れ替えます。受け取る側は、自分で扉を開けます。

npmでは、次の信頼が連なっています。

```text
利用者
  ↓ パッケージ名と版を信頼
npmレジストリ
  ↓ 発行者と公開権限を信頼
メンテナーのnpm / GitHubアカウント
  ↓ 自動化を信頼
GitHub Actionsなどの公開ワークフロー
  ↓ ソースとビルド手順を信頼
リポジトリのコミット、タグ、設定
```

全部を壊す必要はありません。どこか1か所で、後段が信頼するものを書き換えられればいい。

今回、最初に狙われたのは利用者のアプリではありませんでした。Keyv系パッケージを管理するメンテナーのGitHubアカウントと、そこから正規リリースを作る経路だったと複数社が報告しています。

- Aikido Security: https://www.aikido.dev/blog/keyv-and-friends-compromised-in-npm-supply-chain-attack
- StepSecurity: https://www.stepsecurity.io/blog/chaindrop-npm-worm
- Wiz Research: https://www.wiz.io/blog/keyv-and-cacheable-npm-supply-chain-attack
- Socket: https://socket.dev/blog/popular-npm-packages-in-the-keyv-and-cacheable-namespaces-compromised-in-active-supply-chain

## 2026年8月4日、何が起きたのでしょうか

StepSecurityが2026年8月4日18時10分UTC時点でまとめたタイムラインでは、最初の主な流れは次のようになっています。

- 09:02 UTCごろ、`keyv`のmainブランチへ悪性ファイルを含むコミットが入る
- 09:04 UTCごろ、`.claude/settings.json`や`.vscode/tasks.json`を使う別の実行経路も追加される
- 09:35 UTCごろ、`keyv@6.0.0`が正規のGitHub Actions OIDC Trusted Publishingで公開される
- 09:38 UTC以降、盗んだ資格情報を使った第2波が別組織のパッケージへ広がる
- 10:09から10:14 UTCごろ、`cacheable`系の悪性バージョンが相次いで公開される
- 10:39 UTCごろから、npm側のunpublishなどの対応が始まる

タイムラインと調査時点はStepSecurityの解析を参照しています。

https://www.stepsecurity.io/blog/chaindrop-npm-worm

最初の11個のフルワーム保有パッケージは、次のとおりです。

| パッケージ | 確認された悪性バージョン |
|---|---:|
| `keyv` | `6.0.0` |
| `flat-cache` | `6.1.24` |
| `file-entry-cache` | `11.1.6` |
| `cacheable-request` | `13.0.20` |
| `@cacheable/utils` | `2.5.1` |
| `cacheable` | `2.5.1` |
| `@cacheable/memory` | `2.2.1` |
| `cache-manager` | `7.2.10` |
| `@cacheable/node-cache` | `3.1.2` |
| `ecto` | `5.0.1` |
| `@cacheable/net` | `2.1.1` |

この表だけをgrepして終わってはいけません。これらは初期のフルペイロード保有者です。ワームはその後、別のパッケージ名と多数の悪性バージョンへ広がりました。

## ChainDropは、どうやって次のパッケージへ移ったのでしょうか

攻撃の流れを分解すると、次のループになります。

1. 被害者が悪性バージョンを含む依存関係をインストールする
2. `preinstall`が`setup.mjs`を実行する
3. `setup.mjs`が正規のBunランタイムを取得する
4. Bunが難読化された第2段階のJavaScriptを実行する
5. npm、GitHub、クラウド、CI/CDなどの資格情報を探す
6. 盗んだnpm公開権限で、被害者が管理する別パッケージへ同じ仕組みを注入する
7. 次の利用者がインストールし、ループが続く

StepSecurityのサンドボックス解析では、第2段階は727,680バイトのBunバンドルで、初期波では`Math_Symbol.js`、第2波では`math_init.js`という名前が使われました。Aikidoも、`setup.mjs`が公式GitHub ReleasesからBun 1.3.13を取得して第2段階を動かす構造を報告しています。

- StepSecurity: https://www.stepsecurity.io/blog/chaindrop-npm-worm
- Aikido Security: https://www.aikido.dev/blog/keyv-and-friends-compromised-in-npm-supply-chain-attack

正規のBunを使うのは、なかなか嫌らしい設計です。ネットワーク上では、いきなり無名の実行ファイルを怪しいサーバーから取るのではなく、まず`github.com`にある本物のランタイムへアクセスします。ドメインの評判だけを見る防御をすり抜けやすくなります。

![ChainDropが資格情報を盗み次のパッケージへ自己伝播する循環](/images/2026-08-05_npm-chaindrop_fig2.png)

上の図では、1回のインストールが「その端末の被害」で終わらず、公開権限を通じて次の配布物を作るところまでを1周にしています。被害者が次の配布者へ変わるので、手作業の攻撃より速く広がります。

## 盗まれたのは、npmトークンだけではありません

複数の解析によると、ペイロードは開発端末とCI/CDランナーにある広い範囲の情報を探しました。

- npmの公開権限
- GitHubのトークンとリポジトリアクセス
- GitHub Actionsランナーのメモリにあるシークレット
- AWSの認証情報、Secrets Manager、SSM Parameter Store
- KubernetesのサービスアカウントやSecret
- HashiCorp Vaultのトークンと保存値
- SSH鍵、Git認証、各種CLI設定
- AI開発ツールの認証情報や設定
- 環境変数、`.env`、シェル履歴など

「CIのログに出していないから安全」とは限りません。StepSecurityとAikidoは、GitHub Actionsの`Runner.Worker`プロセスのメモリを読み、ジョブへ注入されたシークレットを探す処理を報告しています。

さらに、GitHub APIを使って`.claude/settings.json`や`.vscode/tasks.json`などをリポジトリへ追加し、VS Codeでフォルダを開いたときやClaude Codeのセッション開始時に再実行する経路も解析されています。npmを消しても、リポジトリ側に残ったフックから再感染する可能性があります。

https://www.stepsecurity.io/blog/chaindrop-npm-worm

![静かなCIランナーの周囲へ認証情報の経路が伸びる情景](/images/2026-08-05_npm-chaindrop_illustration-ci-secrets.png)

この挿絵は、CI/CDを単なるビルド機ではなく、公開権限やクラウド権限が集まる交差点として描いています。攻撃者にとって価値が高いのは、ソースコード1個より、その先へ進める資格情報です。

## Ethereumを使うC2は、何が厄介なのでしょうか

StepSecurityの解析では、ペイロードはEthereumメインネット上のコントラクトから現在のC2ドメイン一覧を取得する、EtherHidingと呼ばれる方法を使っていました。観測された送信先の1つは`npm-cache[.]com`です。

ドメインをプログラムへ固定すると、そのドメインを停止または遮断された時点で攻撃基盤が死にます。ブロックチェーン上の値を参照する方式なら、ペイロード本体を再配布しなくても参照先を更新できます。

しかも、問い合わせ先として多数の正規Ethereum RPCサービスを試します。正規GitHubからBunを取り、正規RPCでC2を解決する。単純な「知らないドメインを全部止める」だけでは見つけにくい構造です。

ここは、遮断対象と監視対象を分けて考えます。観測された悪性C2は遮断候補ですが、正規RPCサービスを一括で悪性扱いすると業務影響が出ます。プロセスの親子関係、`npm install`直後のBun取得、RPC問い合わせ、C2通信という文脈で検知する必要があります。

https://www.stepsecurity.io/blog/chaindrop-npm-worm

## provenanceが付いていたのに、なぜ防げなかったのでしょうか

npm provenanceは、パッケージがどのソースコミットとビルド環境から公開されたかを検証可能にする仕組みです。公開経路のなりすましや、レジストリ上での不正な差し替えを見抜くうえで価値があります。

一方、npm公式ドキュメントは、provenanceが悪意あるコードを含まないことまで保証するものではないと明記しています。証明するのは、ソースとビルドのつながりです。

https://docs.npmjs.com/generating-provenance-statements/

今回、攻撃者は正規の公開ワークフローを偽装したのではありません。悪性ファイルをソース側へ入れ、そのソースを正規ワークフローにビルドさせました。

```text
安全なソース + 正規ワークフロー = provenance付きの安全な成果物
悪性ソース + 正規ワークフロー = provenance付きの悪性成果物
```

provenanceは嘘をついていません。「このコミットから、この正規ワークフローで作られた」と正しく証明しました。問題は、そのコミット自体がすでに汚染されていたことです。

![正式な検品印が押されたまま中身だけ汚染された荷物](/images/2026-08-05_npm-chaindrop_illustration-provenance.png)

検品印が無意味という話ではありません。印が証明する範囲を取り違えない、という話です。送り元と製造経路を証明できても、中身の安全性にはコードレビュー、変更監視、実行時制御が別に要ります。

`npm audit signatures`で署名やprovenanceを検証することには意味があります。ただし、今回のように正規経路が悪性ソースを公開したケースを、それだけで安全判定することはできません。

https://docs.npmjs.com/viewing-package-provenance/

## 「444」「1,381」「2,212」「1,300超」は、なぜ食い違うのでしょうか

最初にニュースを読んだとき、ここでかなり混乱しました。数字が違いすぎます。

StepSecurityは2026年8月4日18時10分UTC時点で、444パッケージ名、2,212の悪性バージョンとしています。内訳は、初期の11パッケージと、ワームが広げた433パッケージ、2,201バージョンです。

https://www.stepsecurity.io/blog/chaindrop-npm-worm

Aikidoは2026年8月5日13時15分CESTの更新で、少なくとも444パッケージ、1,381バージョンとしています。

https://www.aikido.dev/blog/keyv-and-friends-compromised-in-npm-supply-chain-attack

BleepingComputerは見出しで1,300超のパッケージと報じていますが、本文には約1,300個の公開GitHub投下リポジトリという別の数字も登場します。パッケージ名、悪性バージョン、投下用リポジトリを同じ「件数」として読むと崩れます。

https://www.bleepingcomputer.com/news/security/massive-chaindrop-npm-supply-chain-attack-infects-hundreds-of-packages/

加えて、次の違いがあります。

- 集計した時刻
- パッケージ名を数えたか、`name@version`を数えたか
- 同じパッケージへ多数の過去バージョンを再公開したものをどう数えたか
- npmから削除済みの版を残したか
- 初期感染と第2波をどこまで確認済みにしたか
- 重複、再公開、誤検知をどう整理したか

だから、ニュースの最大値を競っても、手元の安全確認にはあまり役立ちません。必要なのは、更新される`package, malicious versions`の組と、自分のlockfile、インストール履歴を突き合わせることです。

## lockfileに名前があったら、感染確定なのでしょうか

ここは分けて考えます。

lockfileに悪性`name@version`があることは、**その依存ツリーが悪性バージョンを選んだ証拠**です。ただし、その端末で実際にインストール処理が走ったかまでは、lockfile単体では分かりません。

反対に、現在のlockfileに名前がなくても、過去のCI実行や別ブランチ、一時生成ディレクトリ、`npx`、グローバルインストールで実行していた可能性があります。

判断に使いたい材料は次です。

- 当時の`package-lock.json`、`pnpm-lock.yaml`、`yarn.lock`
- CI/CDの実行時刻、キャッシュ、依存解決ログ
- `node_modules`内の版と特徴的ファイル
- npmキャッシュに残るtarballやメタデータ
- EDRやプロセス監視の`node -> setup.mjs -> bun`という親子関係
- Bun取得、Ethereum RPC、既知C2への通信記録
- `.claude`、`.vscode`、GitHub Actionsワークフローの不審な変更
- npmとGitHubの公開履歴、監査ログ

![lockfile一致から実行有無と資格情報対応を分ける判断表](/images/2026-08-05_npm-chaindrop_fig3.png)

図のように、「依存に含まれる」と「コードが実行された」と「資格情報が悪用された」は別の段階です。ただし実行証拠がある段階では、漏えいの確証を待ってローテーションを遅らせない方が安全です。

## 被弾した可能性があるとき、何をどの順で行うのでしょうか

### 1. まず新しい実行を止める

該当端末やランナーで依存インストールを止めます。自動更新、定期ビルド、リリースジョブも一時停止します。自己伝播するので、npm公開権限を持つ環境を動かし続けるほど影響が増えます。

### 2. ネットワークから隔離し、証拠を残す

感染が疑わしい端末は隔離します。いきなり`node_modules`やキャッシュを削除すると、実行有無を判断する材料も消えます。組織のインシデント対応手順があるなら、先にディスク、ログ、CI実行記録、監査ログを保全します。

### 3. 清潔な端末から資格情報を失効させる

感染した可能性がある端末で、新しいトークンを発行しません。新しい値まで盗まれる可能性があるからです。

対象はnpmトークンだけではありません。その端末またはCIから参照できたGitHub、クラウド、Vault、Kubernetes、SSH、APIキー、データベース認証などを洗い出します。

ローテーションは「パスワードを変える」だけでは足りません。既存セッション、PAT、アプリトークン、デプロイキー、CIシークレット、OIDCの信頼設定、クラウドの一時認証経路も確認します。

### 4. 公開履歴とリポジトリ変更を調べる

- 自分または組織のnpmパッケージに身に覚えのない版がないか
- GitHubのコミット、タグ、Release、Actions実行に不審なものがないか
- `.claude/settings.json`や`.vscode/tasks.json`へ実行フックが増えていないか
- GitHub Actionsワークフローやartifactに不審な変更がないか
- 新しい公開リポジトリや、説明文が不審なリポジトリが作られていないか

### 5. 安全な基点から再構築する

悪性パッケージだけ削除して端末を使い続けるより、信頼できるイメージやバックアップから再構築する方が確実です。ソースリポジトリ側のフックも確認し、クリーンなコミットへ戻します。

その後で、安全版と更新済みIOC一覧を確認し、lockfileを作り直すか修正します。復旧時も、最初はネットワークと権限を絞った環境で依存を取得します。

![汚染環境を横に置き清潔な端末で鍵を交換して再構築する情景](/images/2026-08-05_npm-chaindrop_illustration-rebuild.png)

削除、ローテーション、再構築は同じ作業に見えて役割が違います。悪性コードを消すこと、盗まれた権限を無効にすること、再感染しない基点へ戻ることを分けて進めます。

## npm auditだけでは、今回を判定しきれません

`npm audit`は、依存関係を既知の脆弱性情報と照合するための大事な道具です。ただし、公開直後の悪性パッケージが即座に脆弱性データベースへ載るとは限りません。

脆弱性スキャナーが得意なのは「この名前と版には、この既知の問題がある」という照合です。今回必要なのは、それに加えて次の観測です。

- マルウェア調査会社が更新する悪性`name@version`一覧
- パッケージtarballの差分とハッシュ
- インストールスクリプトの追加
- 実行プロセスと外向き通信
- 公開権限やリポジトリの不審な操作

「`npm audit`が0件だから、今回のChainDropにも無関係」は成立しません。0件が意味する範囲を狭く読む必要があります。

## 次から被害を小さくするには、何を変えればよいのでしょうか

### lockfileをコミットし、CIはnpm ciを使う

これは基本ですが効きます。依存解決を毎回やり直す範囲を減らし、意図しない最新版の流入を抑えます。

ただし、すでに書いたとおり、悪性版を固定する可能性は残ります。lockfileの差分レビューと、悪性版一覧との照合を組み合わせます。

### 依存更新にクールダウン時間を置く

公開直後の版を即時に本番へ取り込まず、一定時間待つ方法です。今回の初期封じ込めは公開から数時間で始まりました。すべての攻撃に効くわけではありませんが、最新版を即座に拾うより調査会社とレジストリが検出する時間を稼げます。

### インストールスクリプトを既定で信用しない

調査時に一時的にスクリプトを止めるなら、次があります。

```bash
npm ci --ignore-scripts
```

ただし、正規のビルドやネイティブモジュール準備も止まります。動作確認なしで本番手順へ入れるのは危険です。

npm 12では依存パッケージのインストールスクリプトを既定でブロックし、`npm approve-scripts`で版を固定して許可する仕組みが公式ドキュメントに掲載されています。利用中のnpm版で挙動が違うため、チームの版を先に確認してください。

https://docs.npmjs.com/cli/v12/commands/npm-approve-scripts/

### CI/CDの資格情報を短命・最小権限にする

- 通常のテストジョブにnpm公開権限を渡さない
- `GITHUB_TOKEN`の既定権限をreadへ寄せる
- クラウドの長期キーを置かず、短命なOIDC認証を使う
- リリース環境と通常CIを分離する
- セルフホストランナーを使い回さず、ジョブごとに破棄できる構成へ寄せる
- 外向き通信を必要な宛先へ絞る

OIDCやprovenanceをやめる話ではありません。長期トークンを減らし、公開経路を追跡可能にする価値は残ります。そのうえで、ソース側の侵害と実行時のふるまいも監視します。

### 自分が公開できるパッケージを棚卸しする

盗まれた公開権限が何個のパッケージへ届くかで、次の被害範囲が決まります。

- 使っていないnpmトークンを失効する
- 古い自動化トークンを残さない
- メンテナー権限を定期的に見直す
- 2FAとTrusted Publishingを使う
- 1つの資格情報へ多数の組織・パッケージを集約しすぎない
- 身に覚えのないpublishを検知する

### SBOMとlockfileに、取得履歴を足す

SBOMは、どのソフトウェア部品が含まれるかを知るために役立ちます。lockfileは、正確な依存ツリーを再現するために役立ちます。

それでも、インシデントの最中に欲しくなったのは別の問いでした。

- 何時に入れたか
- どの端末またはCIで入れたか
- どのURLとハッシュから取ったか
- その時点でスクリプトが実行されたか
- 誰が、なぜ導入したか

今回、自分のリポジトリを調べたときも、最後は`node_modules`の更新時刻から推測するしかありませんでした。正直、心もとない。

この穴を埋めるために、取得イベントをappend-onlyで記録する[intakelog](https://github.com/ishizakahiroshi/intakelog)を作り始めています。まだearly developmentですが、lockfileやSBOMを置き換えるのではなく、「いつ、どこから入ったか」を隣に残す道具です。

設計の経緯は別の記事にまとめています。

https://qiita.com/ishizakahiroshi/items/333278757e8e93c64b9b

## オープンソースを使わない、という結論にはしません

npmもオープンソースも、現代の開発には欠かせません。全部を自前で書けば安全になるわけでもありません。自前コードにも脆弱性は入りますし、少人数で抱えた暗号や認証処理の方が危ないこともあります。

今回見えたのは、オープンソースの是非ではなく、信頼を二択にしていた弱さだと思っています。

「有名だから安全」「provenanceがあるから安全」「lockfileがあるから安全」「auditが0だから安全」。どれも、それぞれが証明する範囲では役に立ちます。範囲を越えて安全証明として使った瞬間に、穴ができます。

小さく始めるなら、lockfileを固定する。CIを`npm ci`にする。公開権限を通常ジョブから外す。インストールした時刻を残す。まずはその4つからでいいと思っています。

まだ事件の最終件数は確定していません。だからこそ、最大件数のニュースを追うより、手元の`name@version`と実行時刻を確認します。地味ですが、今夜やるならそこです。

## 参照資料

- npm公式 About npm: https://docs.npmjs.com/about-npm/index.html
- npm公式 Scripts: https://docs.npmjs.com/cli/using-npm/scripts/
- npm公式 package-lock.json: https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json/
- npm公式 provenanceの制約: https://docs.npmjs.com/generating-provenance-statements/
- StepSecurity ChainDrop技術解析: https://www.stepsecurity.io/blog/chaindrop-npm-worm
- Aikido Security Keyv侵害解析: https://www.aikido.dev/blog/keyv-and-friends-compromised-in-npm-supply-chain-attack
- Wiz Research Keyv / Cacheable解析: https://www.wiz.io/blog/keyv-and-cacheable-npm-supply-chain-attack
- Wiz Research IOC一覧: https://github.com/wiz-sec-public/wiz-research-iocs/blob/main/reports/keyv-packages.csv
- Socket Keyv / Cacheable解析: https://socket.dev/blog/popular-npm-packages-in-the-keyv-and-cacheable-namespaces-compromised-in-active-supply-chain
- BleepingComputer概況: https://www.bleepingcomputer.com/news/security/massive-chaindrop-npm-supply-chain-attack-infects-hundreds-of-packages/

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.com/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
