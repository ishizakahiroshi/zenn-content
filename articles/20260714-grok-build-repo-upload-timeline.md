---
title: "gitignore は防御じゃない。Grok Build の repo アップロード事案から、秘密の置き場所を見直す"
emoji: "🔒"
type: "tech"
topics: ["セキュリティ", "ai", "llm", "gitignore", "devsecops"]
published: true
---

![](/images/2026-07-14_grok-build-repo-upload-timeline_hero.png)

![記事の要約](/images/2026-07-14_grok-build-repo-upload-timeline_infographic.png)

深夜、X のタイムラインを見ていたら「Grok Build がローカルのコードを Google Cloud Storage に送っている」という告発が流れてきました。半信半疑で自分の PC の `~/.grok/logs/unified.jsonl` を grep してみたら、245 件の送信記録が残っていました。数日使い倒しただけの環境で、それだけの回数です。

自分の環境で追試した結果と、X で起きたことの時系列、そして一番大事だと思っている「gitignore は防御じゃない」という設計原則を、まとめておきます。

## TL;DR

- Grok Build（xAI の AI コーディング支援 CLI）が cwd のリポジトリをスナップショット化して GCS に送っていた事案が、2026-07-12 頃に暴露されました
- 自環境で `~/.grok/logs/unified.jsonl` を確認したら、実際に送信履歴が 245 件記録されていました
- xAI は 7/13 夜にリモート設定で送信を停止し、7/14 に `/privacy opt-out` コマンドで停止＋過去分削除ができると発表しました。ただし削除の実効性は第三者検証できません
- 「gitignore しておけば安全」は今回の学びとは真逆です。gitignore は git が push する範囲の話であって、AI CLI・tar・rsync・Docker build context・IDE 同期・バックアップソフト等は一切尊重しません
- 対処法は「今すぐ opt-out」と「今週中に `.env` `secrets.*` をリポ配下から外に出す設計へ移行」の 2 段

![X 動向の時系列タイムライン](/images/2026-07-14_grok-build-repo-upload-timeline_fig1.png)

上の図は、下のセクションで説明する X の時系列を 1 枚にまとめたものです。7/9 のリリースから、暴露、告発の拡散、xAI の対応、opt-out 手順公開まで、5 日間で一気に動いています。

## X で何が起きたのか（一次ソースの時系列）

告発と xAI の反応、そしてタイムラインの反応を、順に追います。ソースは全部 X の一次ツイートです。

### 2026-07-09: Grok 4.5 リリース

xAI 公式 @SpaceXAI が Grok 4.5 を発表しました。「最初のモデル。コーディングとエージェント向けに特別に訓練された。Cursor を使用して訓練され、最高速度とコスト効率で最先端の知能を提供する」と説明されています。この Grok 4.5 が、後の Grok Build CLI の中身になります。

出典: [@SpaceXAI プロフィール](https://x.com/SpaceXAI) の 2026-07-09 投稿（個別ツイート URL は後日追記予定）

### 2026-07-11: Grok 4.5 が無料プランで開放

@SpaceXAI が @grok の RT で「Grok 4.5 が無料プランで試せるようになりました。X アカウントまたは SuperGrok アカウントをお持ちなら、Grok Build をご利用ください」と告知しました。ここから使用者が急拡大します。

出典: [@SpaceXAI プロフィール](https://x.com/SpaceXAI) の 2026-07-11 投稿（@grok の RT・個別ツイート URL は後日追記予定）

### 2026-07-12: 中国語 IT メディアが既に報道

日本語圏より先に、中国語 IT メディア 蓝点网 ([@landiantech](https://x.com/landiantech)) が既に取り上げていました。

> 🚨🚨🚨 SpaceXAI の人工知能コーディングツール #GrokBuild が、デフォルトで完全な Git リポジトリをアップロードしていることが暴露され、ツール自体が読み取っていないコードや呼び出しのコンテキスト、および Git の完全なコミット履歴が含まれています

つまり後述する @XBToshi の告発（7/13）より先に、7/12 の時点で暴露は始まっていました。@XBToshi は最初の告発者ではなく、英語圏での拡散者、という位置づけになります。

出典: [@landiantech プロフィール](https://x.com/landiantech) の 2026-07-12 投稿（個別ツイート URL は後日追記予定）

### 2026-07-13: @XBToshi の初回告発

Bitcoin & Monero Maxi の [@XBToshi](https://x.com/XBToshi/status/2076521420017045618)（フォロワー約 1 万）が英語圏で拡散します。1 本目のツイート:

> anyone ever used Grok Build, please check your grok logs. `cat ~/.grok/logs/unified.json | grep repo_state.upload` you will be pissed.

読み手が手を動かせる 1 行のコマンドが載っているのがポイントで、この時点で追試する人が一気に増えました。

### 2026-07-13: 告発本編（画像つき）

同日、告発の本編ツイートが来ます。

> AI 開発ツールの絶望的な現状。Grok のビルドが、バックグラウンドで 12GB の未使用リポジトリデータと完全な Git コミット履歴を GCP にこっそりダンプしている。ただスクリプトのオートコンプリートのためだけに。君を助けてビルドしたいんじゃない。ただローカル開発環境をトレーニングデータのオープンバッフェにしているだけだ。

貼られている画像には、Claude Code に追試させた結果が写っていました。`~/.grok/logs/unified.jsonl` の生ログ抜粋（`repo_state.upload.start` と `repo_state.upload.enqueued` の JSON）が、実際に手元で観測できていることを示す資料として使われていました。「AI が言うことだから、と思って別の AI に検証させた」というやり口も、時代を感じさせます。

出典: [@XBToshi 告発本編](https://x.com/XBToshi/status/2076683570509496599)

### 2026-07-13: 対処手順スレッド（scan mode 観察）

同じスレッドで @XBToshi は対処手順を公開しました（親スレッド: [@XBToshi 告発本編](https://x.com/XBToshi/status/2076683570509496599) の続き）。項目 4 が特に重要です。

> 2. hold upload queues
> 3. Uninstall to be safe: `rm -rf ~/.grok/downloads ~/.grok/logs ~/.grok/sessions ~/.grok/bin`
> 4. Rotate any credentials that live in `.env`, `.envrc`, `~/.config/**` inside repos you've opened with grok. The tarball scoop can vacuum those unless you had them .gitignored (and even then, uncommitted files are candidates depending on their scan mode)

読み解くと、彼の観察はこうです。

- gitignore していないファイルは、tarball が吸い上げる
- gitignore していても、コミットされていない（uncommitted）ファイルは、scan mode 次第で候補になる

「scan mode」というのは Grok Build 内部の設定のようですが、詳細は公開されていません。ここで初めて「gitignore していれば絶対安全」ではないことが、観察ベースで示されました。

### 2026-07-13 夜: xAI がリモート設定で送信を停止

告発を受けて xAI 側で対応が入ります。手元のログを見ると、この夜のどこかで送信フラグが false に切り替わりました。詳細は後述しますが、`trace.upload.decision` イベントの `uploads_enabled` が `true` から `false` に変わり、理由が `feature_off`、ソースが `remote` になっています。**xAI サーバー側から遠隔でオフにできる設計** でした。

### 2026-07-14 00:37 JST: xAI 公式声明

@SpaceXAI が公式声明を出しました。

> We care deeply about your privacy and respect customer choice. For teams using zero data retention, no trace and code data is ever retained. All API key use of Grok Build also respects ZDR. If ZDR is disabled, the /privacy command is available in the CLI to disable data retention.

いつでも /privacy を実行すれば設定を確認・変更できます、と続きます。「今使えます (available)」の現在形なのがポイントで、「これから作ります」ではなく「もうあります」の姿勢です。

出典: [@SpaceXAI 声明](https://x.com/SpaceXAI/status/2076692402442846289)

### 2026-07-14 00:47 頃 JST: 続報

同じスレッドで補足します（親スレッド: [@SpaceXAI 声明](https://x.com/SpaceXAI/status/2076692402442846289) の続き）。

> /プライバシーを実行して設定を変更すると、以前に同期されたすべてのデータが削除されます。

「以前分も削除される」がポイントで、ここが後で議論の焦点になります。

### 2026-07-14 01:00 頃 JST: 第三者からの質問

Farhan Ali Shah ([@FarhanWritess](https://x.com/FarhanWritess)) が [@SpaceXAI 声明スレッド](https://x.com/SpaceXAI/status/2076692402442846289) に返信で質問を投げていました。「知っておくのは良いね、明確にしてくれてありがとう。クイックな質問なんだけど、/privacy を実行して ZDR ...」他のユーザーも状況を把握しに動いていた、という時代の空気感が読めます。

### 2026-07-14 01:24 JST: @XBToshi の反論

@XBToshi は xAI の声明を素直には信じませんでした。

> @SpaceXAI we will never be able to verify if they truly removed our previously synced data, SAD.
>
> And with one caveat: Trust-only, not verifiable:
> - We can't audit xAI's GCS to confirm the past bundles were actually deleted.

「私たちの以前同期されたデータが本当に削除されたかどうかを検証することは決してできません、悲しいことに」「xAI の GCS を第三者が監査する手段はない = trust-only で信じるしかない」

これは技術的に妥当な指摘です。xAI 側のバケットを外部から覗く手段はないので、xAI が「削除しました」と言っても、それを検証する術がユーザー側には存在しません。

出典: [@XBToshi 反論](https://x.com/XBToshi/status/2076704308352163905)

### 2026-07-14 (後刻): @XBToshi の更新告知

@XBToshi は結局、xAI の対応を受けて手順を公開しました（親スレッド: [@XBToshi 告発本編](https://x.com/XBToshi/status/2076683570509496599) の続き）。

> Update: we can now opt out of code sync (更新: コード同期から今すぐオプトアウトできるようになりました)
>
> ```
> mkdir -p /tmp/grok-priv && cd /tmp/grok-priv
> grok
> /privacy opt-out
> ```

一時ディレクトリで grok を起動して opt-out する、という手順です。「センシティブなリポで opt-out 操作をするな」という配慮です。opt-out 操作のセッション自体でリポ情報を xAI に紐づけないため、意図的に無害な dir から叩く、というやり口です。ここは細かいけど大事なポイントで、追随する解説記事ではあまり触れられていない部分です。

## ローカルで実測してみた

自分の環境で追試してみました。Windows 11 の PowerShell 環境です。

### 送信履歴を確認する

```powershell
Select-String -Path "$env:USERPROFILE\.grok\logs\unified.jsonl" -Pattern "repo_state.upload" -SimpleMatch | Measure-Object
```

macOS / Linux ならこう。

```bash
cat ~/.grok/logs/unified.jsonl | grep repo_state.upload | wc -l
```

自分の環境では **245 件** の送信キューイングイベントが記録されていました。1 セッションあたり平均 3〜10 件の enqueue が発生していました。

### 送信先のパス構造

各イベントには `gcs_path` フィールドがあり、こんな構造でした。

```json
{
  "msg": "repo_state.upload.enqueued",
  "ctx": {
    "phase": "after_codebase",
    "turn_number": 0,
    "size_bytes": 6829,
    "gcs_path": "<sessionId>/turn_0/after_codebase.tar.gz",
    "blobs": 3
  }
}
```

- 送信先は Google Cloud Storage の xAI 管理バケット（推定）
- 各セッション（`sessionId`）ごとに、各ターンで before / after の 2 スナップショットが作られる
- `blobs` は「変更ファイル数」らしく、差分アップロード設計
- `size_bytes` は圧縮後の tar.gz サイズ

差分アップロードなので、告発主の言う「12GB のダンプ」は、セッションを長く回し続けた場合の理論値と考えるのが妥当です。自環境の 245 スナップショットの累計は数 MB でした。ただ **1 スナップショットの上限が `max_file_bytes: 1073741824`（1GB）** に設定されていたので、理論上は数百 GB 級までいけます。

### 対象になったリポの一覧

送信対象になっていたリポを重複除去して数えたら、17 個ありました。うち自分の管理する private リポも 4 個含まれていました。gitignore して git tracked ではない秘匿ファイル（`.env` 系）を持っているリポもあり、他人事ではなかったです。

### opt-out の状態を確認

TUI で `/privacy` を叩くと、状態が表示されます。初期状態はこう。

```
Privacy: share data
Usage and code data may be used by xAI to improve the product.
Use /privacy opt-out to enable privacy mode.
Learn more: https://x.ai/legal
```

「share data」が初期値でした。`/privacy opt-out` を叩くと切り替わります。

```
Privacy: privacy mode
Your code data will not be trained on or used to improve the product.
Use /privacy opt-in to share data and help improve the product.
```

明示的に「your code data will not be trained on」と言ってくれるようになるのは安心材料ですが、これは今後の話で、過去に送信されたデータは xAI 側にあるはずです（削除されたかは検証不能）。

## xAI の対応で気になったこと

xAI は 24 時間以内に動いていて、その速度自体は評価すべきだと思います。ただ、いくつかちぐはぐさもありました。

### 声明の現在形と実装のギャップ

「/privacy コマンドは available です」と現在形で発表されていましたが、追試したときの stable 版 (0.2.99) では、TUI で `/privacy` を叩いても最初は補完に出ませんでした。何度か試したら出てきたので、初期化中の遅延ロードが原因だったと分かりましたが、初見の人は「嘘だろ」と思うタイミングでした。

### alpha channel が 404

`grok update --alpha` で alpha channel に切り替えたら、次のバージョン (0.2.100) をダウンロードしようとして HTTP 404 で失敗しました。xAI 側の CDN 配置が追いついていない、という状態です。

```
Downloading grok v0.2.100 (windows-x86_64)...
Error: Auto-update failed: Download failed: HTTP 404 Not Found
```

「口先が先で実装配布が追いついていない」という状況証拠のひとつです。

### remote settings が config より優先される

送信を止めるフラグは、ローカルの `~/.grok/config.toml` にもありそうですが、実際には xAI サーバー側の user_preference（`remote settings`）が優先されます。ログの `trace.upload.decision` を見ると、`trace_upload_source: "remote"` と書かれていて、決定はサーバー側から来ています。

つまり、xAI が「戻します」と決めれば、ユーザーの意思とは関係なく戻る可能性はゼロではない、ということになります（ここは推測です）。

### 削除の実効性は trust-only

@XBToshi の指摘通り、xAI 側の GCS バケットを第三者が監査する術はありません。「削除されました」を信じるかどうか、しかありません。

## 本題: gitignore は防御じゃない

ここからが今回の一番大事な話です。

### なぜ「gitignore してあるから安全」が誤解なのか

gitignore は、`git add` `git commit` `git push` が対象にするファイルの範囲を決めるためのファイルです。つまり **git の話** です。以下は一切尊重してくれません。

- AI コーディング支援ツールの cwd スキャン（今回の Grok Build、Cursor、Copilot Workspace など）
- `tar` `zip` `rsync -a`
- Docker build context（`.dockerignore` を別途書かないと入る）
- IDE のワークスペース同期、クラウド保存（VS Code Settings Sync、IntelliJ Cloud 等）
- OneDrive / Time Machine 等のバックアップソフト（フォルダごとバックアップ対象になっているケース）
- 誰かが「このリポ zip で送って」と言ってきたときの共有

「.gitignore すれば秘密が守られる」は、防御の 1 層目でしかありません。しかも一番弱い層です。

![gitignore が守らない 6 領域](/images/2026-07-14_grok-build-repo-upload-timeline_fig2.png)

図に並べると、gitignore は中央で「私が守るのは git だけ」と主張していて、周りの 6 領域は完全に無防備、というのが分かりやすいと思います。AI CLI・tar・Docker・IDE 同期・バックアップ・zip 共有、どれも普通に使うツールばかりです。

### 正しい設計

秘密は「最初からリポ配下に置かない」というのが正しい設計です。

| ダメな設計 | 正しい設計 |
|---|---|
| `myrepo/.env` を作って gitignore | `myrepo/.env.example`（プレースホルダーのみ） |
| `myrepo/config/secrets.toml` を作って gitignore | `~/.config/myapp/secrets.toml`（リポ外） |
| 「gitignore してるから大丈夫」で運用 | 「そもそもリポに入れていない」で運用 |

![秘密の置き場所を、リポの外へ](/images/2026-07-14_grok-build-repo-upload-timeline_fig3.png)

上の表を絵にするとこうなります。左のダメな設計と右の正しい設計を並べたとき、「gitignore してるから大丈夫」から「そもそもリポに入れていない」への発想の転換が本質だと分かります。

置き場所の候補は、規模別に選べます。

- 個人: `~/.config/<app>/`、環境変数、1Password CLI (`op`)、`pass`
- 小規模チーム: sops + age、Doppler、1Password teams
- 中規模以上: HashiCorp Vault、AWS Secrets Manager、GCP Secret Manager

### 「そもそもリポに入れない」の実装パターン

Twelve-Factor App の Config で言われていることの実践です。

- リポ内は `.env.example`（プレースホルダー）だけ置く
- 実値は開発者ごとに `~/.config/<app>/.env` で持つ
- ランタイムでロード先を切り替える（例: `dotenv-cli` で `--path` オプションから外部ファイルを読める）
- 本番は環境変数、または external secret store（Vault 等）から注入

これができていれば、たとえ AI CLI がリポを scoop しても、そこに秘密は入っていない、という状態が作れます。

## 恥ずかしい話: 実は自分も同じ罠にはまっていた

正直に告白します。この事案を追いかけながらローカルに調査 HTML を書いていたのですが、その **HTML の初版で、クラウドサービスの account_id とサーバー IP をベタ書きしていました**。

理由は「置いた場所が `docs/local/` で、gitignore しているから大丈夫」でした。まさに今回の学びの逆をやっていたわけです。指摘されて気づき、伏字化しました。

`docs/local/` を gitignore しているから git push しても外には出ませんが、以下は全部リスクです。

- 次のセッションで AI CLI がこの HTML を context として読む可能性（Grok Build が再有効化されたら本当に送られる）
- スクショで Slack / 記事 / SNS に貼ったときの流出
- OneDrive / iCloud で自動同期される可能性
- ローカル PC が侵害されたら丸ごと流出
- zip で誰かに共有したときの流出

「gitignore = 安全」の誤解は、自分でも普通にやってしまうくらい根深いです。今回の事案は、その誤解を炙り出したという意味で、価値があったと思っています。

## 対処法チェックリスト

### 今すぐ（Grok Build を使ったことがある人）

1. `cat ~/.grok/logs/unified.jsonl | grep repo_state.upload | wc -l` で送信履歴の件数を確認
2. TUI で `grok` を起動し、`/privacy` で現在の状態を確認、`/privacy opt-out` を叩く
3. 表示が「Privacy: privacy mode」に切り替わることを確認
4. 一時ディレクトリから grok を起動して opt-out する @XBToshi の作法もおすすめです

### 今夜まで（金銭リスクのある秘密を持っている人）

5. `grok` を起動したリポ配下に `.env` `secrets.*` `credentials*` `*.pem` `*.key` `id_rsa*` `auth.json` があるかスキャン
6. gitignore してあっても、uncommitted なファイルは scan mode 次第で送られた可能性があります。金銭リスクの高いもの（クラウド課金・SaaS API）は保守的にローテート

### 今週まで（本題の設計見直し）

7. リポ内の `.env` 系を **外に出す**。`~/.config/<app>/` や外部シークレットストアへ
8. リポ内は `.env.example` などのプレースホルダーのみ
9. AI CLI 起動前に cwd の秘匿ファイルをチェックする仕組みを入れる（Grok Build 以外にも同種の話が今後も出うる）

## おまけ: many-ai-cli の話

自分は複数の AI CLI（Claude Code、Codex、Copilot、Cursor、opencode、Grok Build）を並行で使うときに、承認操作や進捗をブラウザで一元管理する [many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli) という Hub CLI を自作しています。今回の Grok Build も many-ai-cli 経由で叩いていたのですが、Grok の `repo_state.upload` は Hub 経由でも本体経由でも同じ挙動（Grok 本体側の設計なので）でした。Hub 側でできる保護には限界がある、という気づきもあります。今後、Hub 側で「起動前に cwd の秘匿スキャンをかけて確認する」ような仕組みを入れる余地はありそうです。

## おわりに

Grok Build 事案は、AI コーディング支援ツールが cwd を「読ませてもらう」のではなく「勝手にコピーして送る」という設計をしていた、というインシデントでした。xAI は迅速に対応しましたが、削除の実効性は検証できず、trust-only の状態です。

一方で、この事案が示した最も普遍的な学びは「gitignore は防御じゃない」でした。gitignore を第一の防御だと思ってリポ配下に秘密を置く設計は、AI CLI に限らず、tar・rsync・IDE 同期・バックアップソフトの前でも脆弱です。

秘密は最初からリポの外に置く。リポ内はプレースホルダーだけ。

シンプルですが、実践できていない人は多いです（自分も含めて）。この機会に、`~/.config/` へ移すところから、始めていこうと思っています。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
田舎在宅の SE、実務 18 年。バックエンド・インフラ・AI 連携が専門です。業務委託・受注受付中、フルリモート対応。

- Portfolio: https://ishizakahiroshi.github.io/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi

こんな相談、歓迎です。フィードバックや「うちも同じ現象出た」の共有もお待ちしています。
