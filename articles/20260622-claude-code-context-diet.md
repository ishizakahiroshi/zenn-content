---
title: "Claude Code の常駐 context を 14% 削った話。disabledTools は存在しなかった"
emoji: "🪶"
type: "tech"
topics: ["claudecode", "anthropic", "ai", "context", "cli"]
published: false
---

![](/images/2026-06-22_claude-code-context-diet_hero.png)

Claude Code をしばらく使っていて、ふと `/context` を開いたら、何もしていないのに 80.7k tokens が常駐していました。1M context のモデルだから割合では 8%、誤差みたいなものです。ただ、200k context のモデルに切り替えた瞬間にこれは 40%。前提条件で半分近く食う計算になります。

正直、見なかったことにしようかと思いました。でも、ここから何が無駄に載っているのか棚卸しを始めて、最終的に **常駐 context を 11.2k tokens / 約 14%** 削れたので、そこに至るまでの試行錯誤を残しておきます。途中で「公式ドキュメントに `disabledTools` というキーは存在しなかった」という地味に大きな発見もあったので、合わせて。

検索用に置いておくと: Claude Code, `/context`, settings.json, `disableWorkflows`, `disableArtifact`, prompt caching, CLAUDE.md 分割。

## `/context` を開いたら、何もしていないのに 80k 食われていた

Claude Code には `/context` というスラッシュコマンドがあって、現在のセッションが何にトークンを使っているか分解して見せてくれます。私の手元での内訳はこんな感じでした。

| カテゴリ | 消費 |
|---|---|
| System prompt | 9.4k tokens |
| System tools | 43.1k tokens |
| Memory files | 24.6k tokens |
| Skills | 3.5k tokens |
| Messages（会話本体） | 80 tokens |
| **合計** | **80.7k / 1M tokens (8.1%)** |

Messages が 80 tokens なのに合計 80.7k。つまり「私がまだ何もしていない時点で 80.6k は固定で載っている」状態です。

特に Memory files の 24.6k が引っかかりました。これは `~/.claude/CLAUDE.md` などの設定ファイル群で、AI に「こう振る舞ってね」を伝えるための地の文です。長年運用しているうちに、私の場合は 600 行を超える分量になっていました。

## 本当に毎ターン全部要るの? という疑いから棚卸し

CLAUDE.md の中身を改めて分類してみたら、こんな構成でした。

| 章 | サイズ | 性質 |
|---|---|---|
| 略記ルール、自動生成ショートカット、出力フォーマット規約など | 残す必要あり | 毎ターン判定材料として要る |
| コーディング規範（根本原因優先・憶測断定の禁止・UX 最優先 など 6 章） | 約 70 行 | コードに触る時にしか発火しない |
| 家族情報の参照ルール、KB 統合参照、CLI 配布方針、npm トークン場所 | 各 10〜30 行 | 特定の話題が出た時にだけ必要 |
| pending メモの簡易ルール | 14 行 | pending を作る時にだけ要る |

ここで「毎ターン判定材料として要るもの」と「トリガーされた時にだけ Read すればいいもの」を分けられることに気づきました。後者は別ファイルに切り出して、CLAUDE.md からは「コードに触る前に必ず `rule_coding-stance.md` を Read」のように 3〜5 行のトリガー文だけ残せば、本文の数 k 分が常駐から消えます。

実際にやった分割:

- コーディング規範 6 章 → `~/.claude/guides/rule_coding-stance.md`
- 家族情報の連携 → `~/.claude/guides/reference_family-references.md`
- pending メモのルール → `~/.claude/guides/pending_rules.md`
- npm トークン復旧手順 → 既存の参照ガイドに追記
- VPS デプロイ手順 / リリース配布インフラ（プロジェクト側の `CLAUDE.local.md`） → プロジェクトの `docs/local/` 配下に別ファイル

これで `~/.claude/CLAUDE.md` は 648 行 → 283 行（約 -23%）に。プロジェクト側の `CLAUDE.local.md` は 77 行 → 45 行（-43%）になりました。**「分割」「常駐減」って言葉だけだとカリカリな話に聞こえますが、実態は普通の文書整理です。トリガー文をきれいに書けるかどうかが全部。**

:::message
トリガー文を雑に書くと「読むべき場面なのに AI が参照しに行かない」事故が起きます。私のところでも、似たトリガー方式で 1 回ありました。grep 可能で明示的な語彙（「コードに触る前に」「家族の話題が出たら」など）にするのが安全です。
:::

## 「`disabledTools` で要らないツール切ればいいでしょ」が落とし穴だった

ここまでで Memory が 24.6k → 21.2k に減って、3.4k 浮いた状態。「ここから先は System tools の 43.1k に手を入れるしかないな」と思って、settings.json に `disabledTools` を書こうとしました。

ところが公式 docs を当たったら **そんなキーは無い**。`disabledTools` / `enabledTools` / `skillOverrides` / `disabledSkills` のどれも未定義です。

混乱しました。「`permissions.deny` でブロックすれば実質同じでは?」とも思ったんですが、これは **呼び出し時にブロックするだけで、ツール定義そのものは context に載り続ける** 方式。つまり実行は止められても、トークンは食ったままです。

ここで一度立ち止まって、ちゃんと公式 docs を読み直しました（一次情報を当たる、これ大事）。

![](/images/2026-06-22_claude-code-context-diet_fig1.png)

実在するキーはこっちでした。

| 設定キー | 効果 | リスク |
|---|---|---|
| `disableArtifact` | Artifact ツール定義を削除（claude.ai へのセッション出力公開機能） | 未使用なら無リスク |
| `disableWorkflows` | Workflow ツール定義を削除 + bundled workflow コマンド無効化 | ローカル Workflow が使えなくなる。ultracode モードや一部 skill が影響 |
| `disableBundledSkills` | bundled skill 群（`/loop` `/schedule` `/run` `/verify` `/code-review` 等）を context から削除 | これらの slash command が全部使えなくなる。過激 |
| `maxSkillDescriptionChars` | 各 skill の description 表示文字数を切る（既定 1536） | skill 起動の感度が下がる |
| `includeGitInstructions` | システムプロンプトから git status snapshot と commit/PR 用 instructions を除去 | 「直近のコミット何だっけ」を毎回自分で `git log` する必要が出る |
| `claudeMdExcludes` | 特定 CLAUDE.md を loading 対象外に | 誤って必要な CLAUDE.md を除外する事故 |

`permissions.deny` は context 削減じゃない、と公式 docs に書いてあるのを読んで、ようやく「ああ、自分は別物を混同していた」と腑に落ちました。実行制御と読み込み制御は別レイヤー。

## 効くやつだけ慎重に当てる

リスクと効果のバランスで、私は **`disableArtifact: true` + `disableWorkflows: true`** の 2 行だけ追加しました。Artifact は使った記憶がゼロ、Workflow は多 agent オーケストレーションが必要になったら 1 行消せば即復活できるので。

```json:settings.json（抜粋）
{
  "disableArtifact": true,
  "disableWorkflows": true
}
```

`disableBundledSkills` は手元のスキル運用（`/r*****` `/s*****` `/c***-r*****` など）が全滅するので候補から外しました。`maxSkillDescriptionChars` は skill 起動の感度が落ちるので保留。`includeGitInstructions: false` は git ステータスが毎セッション欲しいので入れず。

## 実測 Before / After

新しいセッションを起こして `/context` を再実行。

| カテゴリ | Before | After | 削減 |
|---|---|---|---|
| Total | 80.7k (8.1%) | **69.5k (7.0%)** | **-11.2k（-13.9%）** |
| System prompt | 9.4k | 9.4k | 0 |
| System tools | 43.1k | **35.6k** | **-7.5k**（disableWorkflows の効果） |
| Memory files | 24.6k | **21.2k** | **-3.4k** |
| Skills | 3.5k | 3.3k | -0.2k |

System tools 側で **-7.5k tokens**。Workflow ツールの JSON schema が context から丸ごと消えた分です。1M context モデルだと「8.1% → 7.0%」で体感は微妙ですが、200k context モデル運用時には **40.4% → 34.8%（-5.6pt）** で結構違います。

ちなみに副作用として `deep-research` skill が消えました。Workflow に依存している skill なので連動して落ちた模様。私は使っていなかったので問題なしですが、deep research に依存している人は `disableWorkflows` は入れない判断もアリです。

![](/images/2026-06-22_claude-code-context-diet_fig2.png)

## で、結局いくら安くなったの?

11.2k tokens / ターン削減の per-turn 価値を、主要モデルで Medium effort 相当の単価で計算してみました（input token 換算・cache miss 時）。

| モデル | input $/1M | per turn 節約 | 1 日 50 ターン × cache miss 30% 想定 |
|---|---|---|---|
| Claude Opus 4.8 | $5.00 | $0.056（約 8 円） | **約 0.84 ドル / 日（約 126 円）** |
| Claude Sonnet 4.6（4.5 と同価） | $3.00 | $0.034（約 5 円） | 約 0.50 ドル / 日（約 76 円） |
| GPT-5.5 standard | $5.00 | $0.056（約 8 円） | 約 0.84 ドル / 日（約 126 円） |
| GPT-5.4 | $2.50 | $0.028（約 4 円） | 約 0.42 ドル / 日（約 63 円） |

「月 100 万円浮きました!」みたいな話ではないです。ヘビーに使っても **月 25〜38 円 × Opus** くらい。これだけ見るとしょぼいんですが、本当に効くのは別のところで:

- **キャッシュミス時の re-prime コスト** が累積で効く。5 分以上のアイドル後の再開時に、毎回 11.2k tokens 分のコストが浮く
- **長時間セッションでの context 圧縮タイミング** が遅れる。古い文脈が残りやすい
- **200k context モデル運用時の作業可能トークン** が 5.6pt 増える
- AI 側の認知負荷が下がる（ルール本文を毎回スキャンしなくて済む）

数字を狙うより、**「ルールの整理整頓・トリガー化による振る舞いの安定化」が本来の主目的**で、コスト削減はおまけです。これは正直に書いておきたい。

## ハマったポイントと学び

時系列でまとめると:

- AI スキル名のマスク（本記事では `s****-h***` 形式で表記している部分） — 公開記事だと個別 skill 設計に踏み込むと本筋からズレるので伏せました
- 公式 docs を読まずに「ありそうなキー名」で settings.json を書こうとすると、`disabledTools` のような実在しないキーを生やしてしまう（実際 AI に聞いても最初は「存在する」と返ってきた）。**一次情報を当たるの、本当に大事**
- `permissions.deny` を「context 削減」と混同しない。あれは実行制御
- `disableWorkflows` は強力だが、依存している skill（私の場合は deep-research）も連動で消える。**入れる前に「ワークフロー使う場面あるか」を一度自問**してから
- 削減幅は環境に強く依存する。私の手元では Memory が膨張していたので効きましたが、すでにスリムな人は誤差で終わる可能性大

:::message alert
グローバルな `~/.claude/CLAUDE.md` を編集する前に必ずバックアップを取ってください。`*.bak.YYYY-MM-DD` のような命名で、数週間置いておけば事故ったとき即戻せます。
:::

## 同じことを試したい人向け

この記事で使った「コンテキスト棚卸し → トリガー化分割 → settings.json チューニング」の手順を、他の人が自分の環境に適用できるように簡易の md スキルとして配布します。

https://github.com/ishizakahiroshi/claude-code-context-diet

リポには README + SKILL.md + 詳細手順（`docs/playbook.md`）+ 公式キーリファレンス（`docs/settings-keys-reference.md`）+ CLAUDE.md 分割パターン例 が入っています。MIT ライセンス。

図解と表でしっかり読みたい方は github.io ホストの詳細版・図解版もあります。

https://ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/

中身は本記事の手順を再現性のある形に整えただけのものなので、もし「自分も `/context` 開いたら 80k 食ってた」って人がいれば、参考にしてもらえれば。

私の環境（Windows 11 / PowerShell / Claude Code ヘビーユーザー）に最適化された手順なので、Mac/Linux や他の AI CLI（Codex CLI / Cursor Agent / opencode 等）を併用している方は適宜読み替えてください。

## 締め

設定ファイルが膨らんでいくのを、放置しがちな性格でした。`/context` を 1 回開くだけで状況が見えるので、月 1 回くらい眺める習慣にしようと思っています。

11.2k 削れて、**月 100 円浮く**くらいの地味な話ですが、こういう棚卸しは年に何回かやると気分もいいです。同じ環境の人がいれば、よければ自分の `/context` も見てみてください。

---

書いた人: ishizakahiroshi
田舎在宅エンジニア（バックエンド・インフラ・AI連携）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X : https://x.com/ishizakahiroshi
