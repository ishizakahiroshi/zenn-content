---
title: "Opus 4.8 の thinking-only silent end_turn を生 jsonl から追った (manabi-map 西日本 v0.4 リリース判断中の実話)"
emoji: "🤖"
type: "tech"
topics: ["claude", "anthropic", "llm", "デバッグ", "opus"]
published: true
---

![Opus 4.8 が黙り込んだ夜 ヒーロー](/images/20260724-opus-4-8-thinking-only-hang/hero.png)

![記事の要約](/images/20260724-opus-4-8-thinking-only-hang/infographic.png)

## 何を作りながらこれが起きたか

自作で [manabi-map](https://manabi-map.app) という「親子で使う、学校選びの地図ノート」を作っています。住所を入れると通える高校が地図に出て、気になる学校を親子で保存・メモできる進路検討 Web サービスです (現状 東日本 20 都道県・2,591 校・v0.3.3・個人 OSS)。

- サイト (ブラウザで開くだけ・アプリ不要): https://manabi-map.app
- リポジトリ (Star をいただけると励みになります): https://github.com/ishizakahiroshi/manabi-map

今は西日本 27 府県 (追加約 5,000 校) を足す v0.4 リリース準備の最終段階で、Block 単位で品質ゲート (G4 承認) を通していく作業をしています。この判断作業を Claude Code (Opus 4.8 / 1M context) にやらせていたら、AI が visible な text も tool_use も返さずに `end_turn` する response を 45 分で 10 個吐いた。合計 `output_tokens` は約 64,000。画面には何も出ない。$4.9 溶かして進捗ゼロ。

生 jsonl を追いかけて調べたら、既に open な GitHub Issue #68515 (https://github.com/anthropics/claude-code/issues/68515) の再現だった。投稿するまでに 1 回誤診断して、自分自身に差し戻された過程を含めて書いておきます。

## 症状

![spinner が回り続ける画面を見つめて停止する開発者](/images/20260724-opus-4-8-thinking-only-hang/illustration-spinner.png)

夜の書斎で spinner だけが回り続け、こちらの手も止まってしまう感じ。画面には何も出ないので、次に何を打てばいいかも分からなくなります。

作業対象は西日本 v0.4 リリース判断。Block 6 (九州北部) と Block 7 (九州南部・沖縄) は 2026-07-23 に freeze 済み、残りの G4 承認と handoff メモ更新を進めていた段階でした。

Claude Code (Opus 4.8・1M context・effort=medium) に「西日本版と全国版はリリースできる状態か」を確認させて、「Block 6/5 の freeze 前 SHA 照合から着手します」と返してもらった。ここまでは正常。

そこから止まった。

- 「SHA 照合 (freeze 前チェック) のため、実ファイルのハッシュを実測します」の意図表明だけ 3 連続
- 編集対象は `docs/local/west-japan-v0.4-incremental/execution/handoff-current.md` (メンテナンス中の handoff メモ)
- Baked 12m 39s の thinking 時間表示だけ長く伸びる
- 「処理続けてください」を 4 回打っても実質進捗ゼロ
- 見ているだけで課金が積み上がる

多少長い thinking なら想像はつく。だが Baked 12m39s の間、何を Baked したのか一切見えない。

## 生ログはどこに置いてある

![画面いっぱいのデータの流れに顔を近づけて覗き込む開発者](/images/20260724-opus-4-8-thinking-only-hang/illustration-detective.png)

停止していた気配が消えて、探偵の顔つきに切り替わる瞬間。生ログを触り始めるとこういう集中に入ります。

Claude Code の session log は `~/.claude/projects/<cwd-slug>/<uuid>.jsonl` にある。Windows では `~` は `%USERPROFILE%` に展開されます。1 プロジェクト 1 subdir、UUID ごとに 1 セッションの jsonl が並ぶ。

manabi-map の cwd slug は `C--dev-github-public-manabi-map`。該当セッションは、ユーザー入力の一部 (「西日本版 全国版はReleaseできるんだっけ？」) を grep すれば一発で特定できた。file size と mtime で本命を絞る。

## jsonl は「1 line = 1 response」ではない

![正常 response は同じ msg_id で 3 line、異常 response は thinking-only の 1 line で stop=end_turn](/images/20260724-opus-4-8-thinking-only-hang/fig1.png)

同じ message.id が jsonl 上で何行に分かれて出現するかを数えると、正常と異常の分布は機械的に分かれます。この観測に至るまでの罠と検証手順を、以下で順に追います。

これが今回の落とし穴。Claude Code v2 の jsonl format は、**1 assistant response を content block ごとに別 line として書き出す**。thinking / text / tool_use の 3 block ある response なら jsonl 上は 3 line、同じ `message.id` を共有する。

正常な turn (例):

- Line 1: `uuid=A`, `msg_id=X`, `ctypes=["thinking"]`
- Line 2: `uuid=B`, `parent=A`, `msg_id=X`, `ctypes=["text"]`
- Line 3: `uuid=C`, `parent=B`, `msg_id=X`, `ctypes=["tool_use"]`, `tool=Read`

`message.id` で group すれば同じ response と分かる。逆に「1 line で 1 response 完結」の異常 response は、同じ `message.id` が他 line に出現しない。

## jq で assistant turn の骨格を切り出す

```bash
jq -c 'select(.type=="assistant") | {
  msg_id: .message.id,
  ts: .timestamp,
  stop: .message.stop_reason,
  out: .message.usage.output_tokens,
  ctypes: [.message.content[]?.type],
  tool_names: [.message.content[]? | select(.type=="tool_use") | .name]
}' <path>.jsonl
```

これで msg_id 別の line 分布と、各 line の content 型・tool 名が出せる。

## 1 回目の誤診断

上の jq を実行して、`out=17,615` なのに `ctypes=["thinking"]` だけ・`tool_names=[]` の assistant turn を 3 個発見した (13:44 / 13:54 / 13:57 UTC)。

即座に結論した。「thinking block だけで 17k tokens 焼いてる。extended thinking の budget 爆発だ」

Issue #63583 (silent stuck-turn: stop_reason=tool_use with no tool_use block・https://github.com/anthropics/claude-code/issues/63583) に comment 下書きを作った。

## 差し戻し (自分で自分に)

![机の反対側から半透明の自分に問い返される開発者](/images/20260724-opus-4-8-thinking-only-hang/illustration-mirror.png)

自分自身に「本当にそれ投稿する内容なの?」を問われる情景。1 人で書いていても、疑いを差し込む余白が要るというのがこの節の要点です。

ここで別セッションから同じテーマを追いかけていた自分自身が、Claude に 2 回疑いを入れた。「本当に投稿するべき内容なの？」

まず 1 回目で「投稿価値薄い」と Claude が後退した。thinking block の中身を dump したら、`thinking: ""` (空文字) + 長大な `signature` だった。Claude Code が local に thinking テキストを strip している可能性が濃厚。だとすると「thinking で 17k tokens 焼いた」と断定できない。しかも文言は #63583 じゃなくて #68515 の題名に一致する。

でも本当に stripped だけの話なのか。

## 決め手: message.id で group して line 数を数える

もし jsonl が streaming 途中で部分書きしているだけなら、「thinking-only の 1 line」は、同じ `message.id` の text/tool_use 追記 line が後にあるはず。

確認した。

- 正常 response の `message.id` は 3 line 揃う (thinking + text + tool_use)
- 異常 response の `message.id` は 1 line だけ (thinking のみ) で、同じ msg_id が他 line にどこにも無い

10 個の異常 response が本物と確定した。

| 時刻 (UTC) | output_tokens | stop_reason |
|------------|---------------|-------------|
| 13:21:02   | 1,466         | end_turn    |
| 13:21:27   | 1,143         | end_turn    |
| 13:44:45   | 17,615        | end_turn    |
| 13:45:08   | 1,145         | end_turn    |
| 13:54:20   | 21,880        | end_turn    |
| 13:57:02   | 7,836         | end_turn    |
| 14:00:50   | 3,114         | end_turn    |
| 14:01:50   | 2,770         | end_turn    |
| 14:06:05   | 4,515         | end_turn    |
| 14:06:59   | 2,634         | end_turn    |

合計 `output_tokens` は約 64,000。この間、text も tool_use も一切 emit されていない。

面白いのは、これらの合間に成功した tool 呼び出し (Read / PowerShell / Edit で `handoff-current.md` を実際に編集) が挟まっていること。モデル全体が壊れているわけではなく、**「tool result を受け取った次の action」で intermittently silent end_turn になる**。

![45 分間で 10 個の silent end_turn。output_tokens は焼かれ続けていて、最大 21,880 / 17,615 のピークが 10 分以内に連発する](/images/20260724-opus-4-8-thinking-only-hang/fig2.png)

上の timeline のとおり、45 分の中で silent end_turn が 10 個並び、ピークの 21,880 と 17,615 の間にも通常の tool 呼び出しは成功しています。「特定の action 直後にだけ発生」の像がひと目で見えます。

## caveat: thinking の中身は透明化されていない

言えるのは「text も tool_use も emit されていない」まで。thinking block に何が入っていたかは、`thinking` field が空だから transcript から読めない。API が実際は thinking content を返していて Claude Code が local strip したのか、そもそも API が空を返したのか、区別できない。

投稿する comment ではここを caveat として明示した。「言えるのは何が emit されていないか、だけです」と。

## 既存 issue との照合

- #63583 (https://github.com/anthropics/claude-code/issues/63583): `stop_reason=tool_use` with no tool_use block。私のは `stop=end_turn` なので不一致。primary 対象から外す
- #77728 (https://github.com/anthropics/claude-code/issues/77728): "continuing" と text を返すが nothing launched。私のは沈黙で「言葉すら無い」ので若干違う
- #68515 (https://github.com/anthropics/claude-code/issues/68515): Opus 4.8 が単純タスクで long thinking → action しない regression。**題名がほぼ一致**

primary target を #68515 に絞って投稿した。

## 診断メタデータのみの redacted 版を用意する

comment 内で「full jsonl or redacted excerpt を提供可能」と書いた以上、そのまま attach 可能な redacted 版を用意しておいた。

```bash
jq -c '{
  ts: .timestamp,
  uuid: .uuid,
  parent: .parentUuid,
  type: .type,
  isSidechain,
  role: (.message.role // null),
  model: (.message.model // null),
  msg_id: (.message.id // null),
  stop: (.message.stop_reason // null),
  usage: (.message.usage // null),
  content_types: [.message.content[]?.type],
  tool_names: [.message.content[]? | select(.type=="tool_use") | .name],
  tool_use_result_type: (.toolUseResult | type)
}' <raw>.jsonl > <redacted>.jsonl
```

残すもの: 診断に必要な API メタデータ (model / msg_id / stop_reason / token 数 / content 型 / tool 名の list)

落とすもの: user 入力 text、assistant text の中身、thinking の text と signature、tool 入力、tool 出力、system prompt

これで manabi-map の県別データ・plan md の中身などプロジェクト固有情報を全部落とせる。念のため禁則語 (県名・プロジェクト名等) を grep でゼロ確認した。

## 投稿

#68515 (https://github.com/anthropics/claude-code/issues/68515) に comment 投稿。数値表・「redacted 版提供可能」・caveat 全部込み。

https://github.com/anthropics/claude-code/issues/68515#issuecomment-5071079048

原本と redacted 版は manabi-map 側の private disk (`docs/local/incidents/` 配下) に、README (公開可否ガイド付き) と一緒に archive した。追加情報を求められたら redacted 版はそのまま attach 可能。

## 学び

- LLM の session log を触るときは、jsonl format の粒度を先に確認する。line = response と決め打つと誤診断する
- 1 回目の自分の解釈を疑う工程を挟む。「本当に投稿する内容なの?」の pushback に救われた
- 既存 issue と照合するとき、症状の骨格 (`stop_reason` と emit された block 型) で判定する。似ているからと安易に紐付けると混乱する
- 原本を保全すれば redact して即 attach できる。後から「あの時のログ欲しい」に対応できる

meta の話をひとつ。AI が便利になるほど、AI が出す一次分析を疑う工程が重要になる気がしている。自分自身が AI と一緒に作業している状況 (この記事も Claude と一緒に書いている) では特に。

## manabi-map はこんなときに刺さります

- 中 3 のお子さんの高校選びで、通える距離の学校を地図で一覧したい保護者
- 学校の文化祭や説明会で聞いた話を、親子で残しながら比較したい家族
- 塾サイトのランキングや広告に振り回されずに、進路を落ち着いて検討したい方
- 西日本の府県も対応予定 (v0.4 で 27 府県 / 追加約 5,000 校を準備中。今回の記事の hang はこの作業中に発生)

無料で使えて、ブラウザで開くだけです (アプリインストール不要・iPhone / Android どちらも): https://manabi-map.app

- リポジトリ (Issue / PR 歓迎): https://github.com/ishizakahiroshi/manabi-map
- ライセンス: コード AGPL-3.0-or-later / データ CC BY-SA 4.0

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば Issue でも X の DM でも大歓迎です。

## おわりに

小さく。AI に一次分析させても、投稿・公開の直前だけは 1 回だけ立ち止まる。それをこれからも習慣にしていきます。

---

書いた人: ishizakahiroshi

システムエンジニア (実務 18 年、バックエンド・インフラ・AI 連携)。業務委託・受注受付中、フルリモート対応です。「現場の業務課題を、最小限の実装で、確実に動くものにする」を軸にやっています。こんな相談、歓迎です。

- https://ishizakahiroshi.com/
- https://github.com/ishizakahiroshi
- https://x.com/ishizakahiroshi
