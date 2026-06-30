---
title: "AI CLI のセッションログ、誰も掃除しないので作ってます"
emoji: "🧹"
type: "tech"
topics: ["claudecode", "codex", "cli", "nodejs", "oss"]
published: true
---

![](/images/2026-06-30_ai-log-clean_hero.png)

![](/images/2026-06-30_ai-log-clean_infographic.png)

## 1.4 GB 貯まっているのに気づいたのが、つい先月

先月、ふと `~/.claude/projects/` のサイズを見たら 1.4 GB ありました。Claude Code を使い始めて半年、毎回の会話履歴を resume 用にぜんぶ書き残してくれていたみたいです。これ自体はありがたい機能で、過去の作業を `/resume` で引き直せます。

問題は、放置していると延々と貯まり続けることでした。本体に retention 設定があるのは Claude Code くらいで、Codex CLI も GitHub Copilot CLI も Cursor Agent も opencode も Grok Build も、入れた瞬間から書き残し始めて、消す仕掛けは持っていません。HDD の片隅で静かに膨張するタイプの太り方です。

## 作っているもの

`ai-log-clean` という小さな CLI です。各 AI コーディング CLI のセッションログを retention 60 日（既定）で日次自動掃除する、それだけのツール。

掃除といっても、いきなり削除はしません。デフォルトは「`~/.ai-log-clean/quarantine/<日付>/<provider>/...` にアーカイブ」で、30 日間そこに置いてから本削除します。`--delete` を明示すれば直接消す、`--dry-run` で何が消えるか確認できる、`--retention-days N` で日数を上書きできる、というよくある作り。配布は GitHub から `npx -y` で直接実行する形にしていて、npm registry には publish しません。main に push したらユーザーの次回起動で即反映、というシンプルな運用です。

:::message
**配布の方針**: 単体 exe を作らない・npm registry にも publish しない。理由は (1) Windows SmartScreen ブロックを構造的に回避できる (2) リリースタグ運用の手間を省ける (3) main push が即配布になる、の 3 点。bunx も使えますが GitHub spec のキャッシュが強めなので、`main` 即配布を成立させたいなら `npx -y` 推奨です。
:::

## 対応している AI CLI（2026-07-01 時点）

現状の対応 provider は 7 つです。それぞれのセッション保存先と本体 retention の有無を表にまとめます。「自分が使っている CLI が入っているか」をここで確認できます。

| AI CLI | 起動コマンド | セッション保存先 | 本体 retention |
|---|---|---|---|
| Claude Code | `claude` | `~/.claude/projects/<encoded-cwd>/*.jsonl` | あり（`cleanupPeriodDays`・既定 30 日） |
| Codex CLI | `codex` | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | なし（`openai/codex#6015` で要望中） |
| GitHub Copilot CLI | `gh copilot` | `~/.copilot/logs/process-*.log` / `~/.copilot/session-state/<uuid>/` | なし |
| Cursor Agent | `cursor-agent` | `~/.cursor/chats/<hash>/<uuid>/` | なし |
| opencode | `opencode` | `$XDG_DATA_HOME/opencode/log/*.log` / `.../storage/session_diff/*.json` | なし |
| Grok Build | `grok` | `~/.grok/sessions/<encoded>/<uuid>/` | なし（`logs/unified.jsonl` は append-only で対象外） |
| Antigravity CLI（旧 Gemini CLI 後継） | `agy` | `~/.gemini/antigravity-cli/brain/<id>/` / `.../conversations/<id>.db*` / `.../log/cli-*.log` | なし |

:::message
**Gemini CLI を使っていた方へ**: 2026-06-18 に Gemini CLI の個人プラン（Free / Google AI Pro / Ultra）が終了しました。後継の Antigravity CLI（`agy` コマンド）がリリースされており、ai-log-clean は Antigravity CLI 側に対応しています。旧 `~/.gemini/tmp/` の残骸は本ツールでは触らないので、必要なら手で削除してください。
:::

## 7 つの AI CLI のログがどれだけ貯まっているか

![](/images/2026-06-30_ai-log-clean_fig1.png)

上の図のとおり、今日の自分の環境で対応 provider の溜まり具合を測ってみました。Claude Code が 1.41 GB で頭ひとつ抜けています。続いて Codex CLI が 253.8 MB、Copilot CLI が 9.3 MB、Grok Build が 8.9 MB、Cursor Agent が 1.1 MB、opencode が 137 KB、Antigravity が 172 KB という並びです。

Claude Code が重いのは、毎日の主戦場として何時間も使っているからで、これは個人差が大きいと思います。Codex CLI も使い込んでいる人は数百 MB が当たり前で、`openai/codex#6015` の issue を眺めていると「半年で 2 GB 越えた」みたいな声が並んでいます。本体に掃除の仕掛けがない以上、放っておけば積もる一方です。

## Antigravity の調査でわかった、もっとひどい設計

Antigravity CLI（`agy` コマンド）は今日入れたばかりで、2026-06-18 の Gemini CLI 個人プラン終了に押し出されて触り始めた、いわば後継品です。これを ai-log-clean の対応 provider に追加するついでに、どこにログを書くのか調べました。

意外なことに、`~/.gemini/antigravity-cli/` を引き継いで使っていて、しかも書き残し方が 3 系統に分かれていました。

![](/images/2026-06-30_ai-log-clean_fig2.png)

図のとおり、`brain/<id>/` の下に会話ごとのファイル成果物（plan.md や scratch・JSONL ログ）、`conversations/<id>.db` の SQLite に会話本体、`log/cli-*.log` に CLI 本体のログ。1 会話書くたびに 3 箇所にまたがって書き残ります。

:::message alert
**問題はここ**: `settings.json` を覗いても retention・TTL・maxConversations のキー枠すら無いことです。公式 docs のどこにも自動掃除の設定は出てきません。`/switch` のピッカーで `Ctrl+Delete` を押して 1 件ずつ手で消す、それだけが用意された掃除手段でした。
:::

Google AI Developers Forum には「Antigravity がだんだん heavy になってきた、まとめて消したい」という未解決スレッドが立っています。前身の Gemini CLI（OSS）で 6000 件以上の PR を受け入れたあと closed-source の Antigravity に閉じて、個人プラン側の運用機能ごと削ぎ落とした経緯は The Register が叩いていました。

## resume を売りにする AI CLI は、構造的にログを捨てない

調べていて気づいたのは、これが Antigravity だけの問題じゃないということです。

`/resume` や `/switch` で過去の会話を引き直せる、を売りにしている AI CLI は、構造上ログを捨てる理由がない。捨てる仕組みを入れると resume と矛盾するし、捨てる粒度（何日? 件数? サイズ?）を決めて UI に出すと設定画面が膨らむ。だから多くは「無期限に残す・困ったら手で消して」になります。

Claude Code が `cleanupPeriodDays=30` を持っているのはむしろ例外で、他の業界標準は「永遠に貯める」が既定だった、という話でした。

## 最終局面に入っています

今日 Antigravity provider を追加して、これで対応 CLI が 7 つ揃いました。残りは README の最終確認と、Windows・macOS・Linux のそれぞれで `install` サブコマンドを通すリリーステストくらい。早ければ来週、正式リリースの予定です。

姉妹プロジェクトの `many-ai-cli`（複数 AI CLI を並列に走らせて承認を Web ダッシュボードで捌くやつ）と違って、こちらは導入も使い方も小さく完結する道具です。

- `npx -y github:ishizakahiroshi/ai-log-clean --dry-run` で何が掃除対象になるか見える
- `install --at 12:00 --retention-days 60` で各 OS のユーザー領域スケジューラに登録して放っておける
- 全 OS で管理者権限を要求しない（UAC や sudo を出さない）

ディスクの片隅で静かに太っていく AI CLI のログに、ちょっとだけ手綱を付けるための、小さい道具です。よかったら覗いてみてください。

リポジトリはこちら。
https://github.com/ishizakahiroshi/ai-log-clean

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

書いた人: ishizakahiroshi
群馬の北部で、保護猫2匹と暮らす、在宅エンジニア（何でも屋）
https://ishizakahiroshi.github.io/
https://github.com/ishizakahiroshi
X（業務委託・各種相談はこちら）：
https://x.com/ishizakahiroshi

バックエンド・インフラ・AI連携まわりで、業務委託のご相談を受け付けています。フルリモートです。スポットや週2〜3時間からでも歓迎で、いろんな案件に携われたらうれしいです。こんな相談、歓迎です。
