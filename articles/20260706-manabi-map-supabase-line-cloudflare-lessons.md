---
title: "個人開発の Web 進路サービスに Supabase + LINE + Cloudflare Pages を組み合わせて起きた落とし穴の記録"
emoji: "🗺️"
type: "tech"
topics: ["supabase", "cloudflare", "line", "react", "個人開発"]
published: true
---

群馬県内の高校を地図の上にピンで表示して、気になる学校をお気に入りに入れて、文化祭や説明会で見た感想を家族でメモしていく。そんな進路検討アプリを一人で作って、2026年7月に群馬県版として公開しました。Manabi Map という名前です。

専用の API サーバーは持たず、Vite + React 19 + Tailwind v4 + Leaflet のフロントエンドから Supabase に直接つなぎ、Cloudflare Pages でホスティングしています。LINE ログインを実装し、ドメインを取り、バックアップの仕組みを組んだところで、想定より派手な事故を一つ踏みました。今日はその記録です。

![](/images/20260706-manabi-map-supabase-line-cloudflare-lessons_hero.png)

## サーバーを持たない、という選択をどこまで貫くか

個人開発だと、バックエンドを自分で持つかどうかは毎回悩みます。今回は Supabase を Auth + Postgres + RLS のフルセットとして採用し、フロントから直接叩く構成にしました。専用の API サーバーは最後まで作っていません。

RLS でお気に入りやメモを本人限定にできる時点で、素朴な CRUD のためだけにサーバーを1枚挟む理由がほぼ無くなります。楽になった反面、Supabase の無料枠にどう収めるかという別の悩みが最初から付いてきました。これは後半で書きます。

![](/images/20260706-manabi-map-supabase-line-cloudflare-lessons_fig1.png)

上の図が実際の構成です。ブラウザから Supabase と GitHub Actions 経由の Cloudflare R2 へ矢印が伸びていて、その間に自前サーバーが一つも無いことが見て取れると思います。専用 API サーバーなしでも、認証・DB・バックアップまで一通り回っています。

## ドメインは Cloudflare Registrar で、年2,600円くらい

manabi-map.app というドメインは Cloudflare Registrar で契約しました。`.app` の TLD で、年額はだいたい2,600円くらいです。DNSSEC は自動で入りますし、Email Routing を設定しておけば hello@ や takedown@ 宛のメールをそのまま Gmail に転送できます。個人開発でメールサーバーを別に立てる気力はなかったので、これは素直にありがたい機能でした。

## LINE ログインで「素直に作ると必ず失敗する」設定がある

一番最初につまずいたのが LINE ログインです。Supabase の Custom OIDC Provider を、ドキュメント通りに素直に OIDC として作ると、LINE のウェブログインが返す ID トークンが HS256 署名で、Supabase 側は ES256 前提で検証しようとするため、必ず失敗します。

原因が分かるまでかなり時間を溶かしました。エラーメッセージだけ見ても署名アルゴリズムの不一致だとはすぐ気づけず、何度も設定を作り直しては同じ場所で落ちる、を繰り返していました。

最終的にたどり着いた回避手順はこうです。

- openid スコープを付けずに Provider を作成する
- 作成後に非 OIDC タイプへ変更する
- userinfo エンドポイントは `/oauth2/v2.1/userinfo` を指定し、JWKS は空欄のままにする
- ここまで済んでから openid を後付けで追加する

順番が肝で、最初から openid ありで作ると同じ壁にぶつかります。この手順は自分の中では「二度と忘れたくない知識」なので、リポジトリの開発ガイドに正典として残してあります。

![](/images/20260706-manabi-map-supabase-line-cloudflare-lessons_fig2.png)

左が素直な OIDC 設定で必ず失敗する経路、右が openid なしで作ってから非 OIDC 化する回避経路です。手順の順番を守らないと、右側の経路でも同じ壁に戻ってしまいます。

## Supabase の無料枠を守るために、学校データを静的ファイルにした

学校情報は住所検索のたびに全件フロントに返す必要があります。最初は素直に Supabase から直接 select していたのですが、対象校数が増えるほど egress が積み上がる設計です。無料枠は決して広くありません。

そこでビルド時に Supabase から学校データを取得して `schools.json` を書き出すスクリプトを用意し、実行時は Cloudflare Pages が配信する静的 JSON を fetch する形に切り替えました。フロント側の fetch 先を変えるだけなので実装自体は小さいのですが、ビルドコマンドが変わる関係でビルド時に環境変数が読めるかを Preview 環境で確認する一手間が増えます。地味ですが効きます。

## 一番の事故は、バックアップの workflow から起きた

Supabase の無料プランには自動バックアップがありません。なので GitHub Actions で毎晩 pg_dump して、gzip して、age で暗号化して、Cloudflare R2 に置く nightly-backup を組みました。ここまでは順調でした。

最初の設計は、接続情報を `SUPABASE_DB_URL` という1本の URL 形式の Secret にまとめて、そのまま `pg_dump "$SUPABASE_DB_URL"` に渡すというものです。よくある形だと思います。

初回の workflow_dispatch を実行した直後、ログにこんなエラーが出ました。

```
pg_dump: error: invalid integer value "xxxxxxxxxx" for connection option "port"
```

パスワードに URL エンコードが必要な特殊文字が含まれていたため、libpq が URL のパースに失敗し、パスワードの断片を port の値だと誤認して、そのままエラーメッセージに出力していました。GitHub Actions の Secret マスクは完全一致した文字列しか `***` に置き換えてくれないので、部分文字列は素通りします。しかもこのリポジトリは Public です。

夜中にログを見て「これ嘘だろ」と声が出ました。すぐにやったことは3つです。失敗した workflow の実行履歴を削除し、Supabase 側でパスワードをリセットし、それから設計を作り直しました。

## URL を捨てて、5分割の Secret にした

対応として、`SUPABASE_DB_URL` を捨てて `SUPABASE_DB_HOST` / `PORT` / `USER` / `NAME` / `PASSWORD` の5つに分割し、`PGPASSWORD` 環境変数でパスワードを直接渡す方式に書き換えました。

```yaml
env:
  PGPASSWORD: ${{ secrets.SUPABASE_DB_PASSWORD }}
  SUPABASE_DB_HOST: ${{ secrets.SUPABASE_DB_HOST }}
  SUPABASE_DB_PORT: ${{ secrets.SUPABASE_DB_PORT }}
  SUPABASE_DB_USER: ${{ secrets.SUPABASE_DB_USER }}
  SUPABASE_DB_NAME: ${{ secrets.SUPABASE_DB_NAME }}
run: |
  pg_dump -h "$SUPABASE_DB_HOST" -p "$SUPABASE_DB_PORT" \
    -U "$SUPABASE_DB_USER" -d "$SUPABASE_DB_NAME" \
    --format=custom --no-owner --no-acl -f backup.dump
```

これで URL のパースという工程自体が消えます。認証に失敗しても出るエラーは「password authentication failed」だけになり、断片が漏れる余地がありません。原因を抑え込むというより、原因が発生し得ない形に構造を変えた、という感覚に近いです。

このやり直しの途中で、もう2つ壁にぶつかりました。ひとつは Supabase Free の Direct connection が IPv6 only で、GitHub Actions のランナーから届かないという問題。これは Session Pooler 経由の接続に切り替えて解決しました。もうひとつは、Supabase 側の PostgreSQL が17系なのに Ubuntu 24.04 標準の pg_dump が16系で、meta-version が合わずに失敗するという問題。PostgreSQL の公式 apt リポジトリを追加して `postgresql-client-17` を入れることで揃えました。

```yaml
- name: Install pg_dump (PostgreSQL 17)
  run: |
    sudo apt-get update
    sudo apt-get install -y curl ca-certificates gnupg lsb-release age
    sudo curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
      -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
    echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
      https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
      | sudo tee /etc/apt/sources.list.d/pgdg.list
    sudo apt-get update
    sudo apt-get install -y postgresql-client-17
```

3つとも別々の問題なのに、同じ一晩でまとめて出てきました。参った。でも再実行したら全ステップ成功して、R2 に暗号化済みのダンプが1本置かれているのを見たときは、素直にほっとしました。

## Cloudflare Pages のヘッダは `_headers` を1枚置くだけ

Cloudflare Pages はレスポンスヘッダを `_headers` というファイルで宣言的に設定できます。今回はこの4つを入れました。

```
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), payment=(), usb=()
```

geolocation は現在地検索の機能で使うのであえて制限していません。CSP はフォント・地図タイル・Supabase・Nominatim の許可リストをきちんと洗い出してからでないと事故りやすいので、まだ入れていません。次の版に持ち越しです。

## お気に入りが「消えたことにならない」ようにする

もう一つ地味に大事な直しがあります。お気に入りやメモの取得に失敗したとき、以前は空データでそのまま state を上書きしていました。ネットワークが一瞬詰まっただけで、画面上はお気に入りが全部消えたように見えてしまう挙動です。

これを、取得が失敗したチャンネルは state を更新しない形に変えました。あわせて、お気に入りのトグルは楽観更新にして、DB 側の書き込みが失敗したら UI を元に戻すロールバックを入れています。

```ts
if (error) {
  // DB 失敗時は楽観更新を巻き戻す（UI と DB の乖離防止）
  setFavorites((cur) => ({ ...cur, [schoolId]: prev }))
  throw error
}
```

見た目は地味な差分ですが、進路の検討メモを預かるアプリとして、ここで嘘をつかないことは結構重要だと思っています。

## 重複データの掃除と、更新日時トリガ

個人偏差値の記録テーブルには、学科ごとの値とは別に「学校単位のメモ」を `department_id = NULL` の行として持たせています。この設計のせいで、ユニーク制約が NULL 同士を別物として扱ってしまい、同じユーザーが同じ学校に対して複数のメモ行を持てる状態になっていました。

マイグレーションで重複行を最新のものだけ残して削除し、制約を `UNIQUE NULLS NOT DISTINCT` に張り替えました。あわせて `moddatetime` の拡張機能を使って、更新のたびに `updated_at` が自動で書き換わるトリガも足しています。地味な整理ですが、後から集計や表示のロジックを書くときにこういう歪みがあると地味に面倒なので、早めに畳んでおきました。

## ライセンスはコードと分けて考えた

コードは AGPL-3.0-or-later にしています。自分がホストしなくても、誰かが改造版をサーバーとして動かして配布したら、そのソースも公開義務が生まれる形です。学校データは CC BY-SA 4.0。掲載している偏差値は商用サイトから転載したものではなく、公的資料を根拠にした Manabi Map 独自の推計値だと明記しています。

コードとデータでライセンスを分けるのは最初は面倒に感じましたが、進路情報という性質を考えると、データの出所と再配布条件は独立して説明できる形にしておきたいと思いました。

## 広告は「教育に関係あるものだけ」と、CLAUDE.md に書いた

未成年と保護者が使うサービスなので、広告の入れ方は最初からかなり絞っています。塾や予備校、私立高校や通信制高校、模試や参考書といった進路・教育に直接関係するものだけを、控えめに置く方針です。

Google AdSense のような無差別配信の広告ネットワークは使わないと決めています。教育カテゴリだけに絞って配信できる保証がない限り、内容を選べない広告枠は入れません。消費者金融やギャンブル、情報商材のような進路と無関係な広告は当然すべて対象外です。

この方針は口約束にせず、リポジトリの CLAUDE.md に非交渉の項目として書き込みました。AI に作業を頼むときも、収益化のために広告を増やす提案自体をそこで止められるようにしています。信頼が壊れたら直しようがない領域だと思っているので、ここだけは譲らないつもりです。

## まだ手を付けていないこと

CSP はまだ書いていませんし、復元試験もこれから本格化させる予定です。バックアップが取れていることと、実際に元に戻せることの間には結構な距離があるので、そこは次のマイルストーンで検証しようと思っています。

一人で作っていると、事故は大体こういう「まさかここで」という場所から来ます。今回は URL のパースというありふれた処理が、パスワードの断片をログに吐き出す入口になりました。次にどこで踏むのかは、正直まだ分かりません。

---

書いた人: ishizakahiroshi

田舎で猫と暮らしながらフルリモートで働く、実務18年のエンジニアです。バックエンド・インフラ・AI連携が専門。「現場の業務課題を、最小限の実装で、確実に動くものにする」を信条にしています。業務委託・受注受付中。こんな相談、歓迎です。

- プロフィール: https://ishizakahiroshi.github.io/
- note: https://note.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi
