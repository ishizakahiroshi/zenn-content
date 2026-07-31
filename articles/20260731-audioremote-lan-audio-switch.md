---
title: "隣の Windows 11 の音を、机の別 PC のブラウザから切り替える。Rust で作って公開しました"
emoji: "🎧"
type: "tech"
topics: ["rust", "windows", "oss", "webui", "audio"]
published: true
---

## Hyper-V ゲストで開発してると、ホストの音出力が触れない

物理 PC は 1 台。Windows 11 のホスト側は **Steam ゲーム機** (戦略シミュレーション。Hearts of Iron IV / Stellaris / Crusader Kings 3 / Timberborn / Cities: Skylines2 ) と、家族との映像視聴用。Nest Hub Max・スピーカー・有線ヘッドフォンが繋がっています。

普段の開発作業は、そのホスト上の **Hyper-V ゲスト (これも Windows 11)** の中でやっています。仕事とゲームを物理的に分けたいのと、環境を汚したくないのと、スナップショットが撮れる安心感で。

問題は会議のとき。ホスト側の Nest Hub Max で音が鳴ってるのを、有線ヘッドフォンに切り替えたい。**でも作業してるのは Hyper-V ゲストの中**。切り替えるためだけに Hyper-V ゲストから抜けて、ホスト側のデスクトップに戻って、通知トレイの音量アイコンをクリックして、出力デバイスを選び直す。会議が終わったらまた戻す。

1 日に何度もこれをやる。Web会議とかが頻繁にあると、切り替えが多いです。そもそも、業務中はYouTubeをラジオ間隔でながしているので、それを、有線ヘッドフォンに切り替えたり、その後、Nest Hub Max に戻したり結構手間だなと思っていました。

「これ、Hyper-V ゲストのブラウザからホスト側の音を切り替えられないか？」。思いついたら止まらなくて、Rust で書き始めました。名前は `audioremote`。この記事は、作りながら考えたことの記録と、そのまま v0.1.0 を公開するところまでです。

同じような構成の人、つまり「開発は仮想環境で・ゲームや配信はホストで」派に、たぶん刺さります。

![](/images/audioremote_hero.png)

---

## 自作で [audioremote](https://github.com/ishizakahiroshi/audioremote) という Windows 用ローカルサービスを作っています

Windows 11 ホストの既定音声出力デバイスを、同じ LAN 上のブラウザから 1 タップで切り替える単一 exe のローカルサービスです。ホストの前に戻らず、ゲスト Win11・スマホ・VM から出力先を変えられます。

同じ悩みを持っている方は、下記で入ります。

```powershell
npm i -g audioremote
# 一度だけ試すなら
npx audioremote
```

Rust のツールチェーンがあるなら `cargo install audioremote` でも入ります。Node も Rust も要らない人は GitHub Releases から zip を落として、同梱の `SHA256SUMS.txt` で照合してください。

リポジトリはこちらです（Star をいただけると励みになります）: https://github.com/ishizakahiroshi/audioremote

この記事は 2026-07-25 に「これ作ります」の段階で書き始めて、そのまま v0.1.0 を公開する日まで書き足しました。前半は壁打ちの記録、後半はリリース当日に起きたことです。

---

## TL;DR

- Windows 11 ホストの音声出力デバイスをブラウザから切替える単一 exe (Rust + axum + Web UI 内蔵)
- 想定シナリオ: **Hyper-V ゲスト内で開発**しながらホスト (Steam ゲーム機兼) の音を切り替えたい / 物理 2 台構成でゲーム PC + 配信 PC / 開発 + ホームラボ 等
- ゲスト側のブラウザで LAN URL を開くだけで使える (トークン埋込 URL・タップ 1 回でログイン済み)
- 非公開 COM `IPolicyConfig::SetDefaultEndpoint` で Console / Multimedia / Communications の 3 役割をまとめて切替
- v0.1 は開発者・ホームラボ層向け MVP。v0.2 で Tauri ゲストアプリ + トレイ + MS Store 参入
- フロントエンドは npm も bundler もフレームワークも使わない vanilla JS 完結
- 壁打ちで見えた設計判断 (LAN-first 転換 / loopback バイパス / URL fragment token / MS Store 見落とし発見 / audience 定義シャープ化) を全部この記事に残す
- **2026-07-31 に v0.1.0 を公開**。npm / crates.io / GitHub Releases の 3 経路。リリース直前の監査で 18 件の指摘が出て、そのうち 1 件は「任意の Web ページからホストの音声出力を切り替えられる」実際に動く CSRF だった

![](/images/audioremote_infographic.png)

---

## なぜ作った

### Hyper-V ゲストの中から Hyper-V ホストの音を触れない

私の PC 1 台の役割分担はこうなっています:

- **ホスト (物理 Win11)**: Steam でゲーム (Paradox の戦略シミュレーションが主戦)、家族との動画視聴、音楽再生。Nest Hub Max / スピーカー / 有線ヘッドフォンが物理的に繋がっている
- **ゲスト (Hyper-V 上の Win11)**: 開発全部。VS Code、Cursor、Claude Code、Slack、ブラウザ、AI CLI 群。仕事系ツールを詰め込んでスナップショット管理

会議アプリ (Google Meet / Discord / Slack Huddle) は Hyper-V ゲスト側で動きます (仕事ツールなので)。ここで音の出力先が問題になります。

- 会議中は自分の声がループしないよう **有線ヘッドフォン**にしたい
- 会議後は動画視聴用の **Nest Hub Max** に戻したい

ゲスト内の会議アプリは音を Hyper-V の「リモート オーディオ」経由でホストに流します。実際の物理音声デバイスの選択はホスト側の Windows Core Audio で決まる。**つまり音の切り替えは物理的にホスト側でしか行えません**。

なので毎回:
1. Hyper-V ゲストのフォーカスを外す
2. ホスト側のデスクトップに戻る
3. 通知トレイの音量アイコンをクリック
4. 出力デバイスを選ぶ
5. Hyper-V ゲストに戻る

この 5 ステップを 1 日何度もやる。**「この作業をゲスト内のブラウザから 1 タップでやりたい」**が動機です。

もちろん Hyper-V ゲストじゃなくて **本当に別 PC (LAN 越し)** で開発している人にも同じ構造で刺さります。私が Hyper-V にしてるのは家に PC を 2 台置く場所が無いから。

![](/images/audioremote_fig1-architecture.png)

上の図の通り、audioremote は「ホスト常駐サーバー」+ 「ゲスト側の薄いクライアント (現状はブラウザ)」の構成。ホスト側で Windows Core Audio (COM) を叩く。ゲストは HTTP で叩くだけの薄い層。

### 「Windows 標準でできないの？」の答え

できません。Windows の通知トレイの音量アイコンからは、そのマシンにログオンしてる人しか変更できない。RDP でリモート接続しても、Windows は RDP セッションに「リモート オーディオ」という仮想の 1 個だけを見せる仕様で、物理デバイスは触れない。

だから既存の解決策は無い。作るしかない、というのが動機です。

---

## 技術挑戦: Rust + Windows Core Audio + 非公開 COM

### なぜ Rust

Windows 専用ツールを書くなら普通は C# か C++ (WinUI / WPF / Win32) が定石です。でも Rust を選びました。理由:

- **単一 exe で配布したい** (rust-embed で Web UI 資産も焼き込む)
- **Rust 学習を兼ねたい** (実務では書いてない・素振りの機会が欲しい)
- **クロスプラットフォームの余地を残したい** (今は Windows only だが将来 macOS 対応の妄想余地)
- **サプライチェーンをシンプルに** (npm 依存ゼロ・crates 依存も最小限)

VB や C# へのピボットも壁打ちで検討しましたが、既に C1〜C4 の実装 (約 800 行) が Rust で書き終わっていて、書き直しコストが割に合わない結論。次の作品 (syncway 等) で C# を試すのが健全という着地です。

### 出力デバイスの列挙は `IMMDeviceEnumerator`

Rust の `windows` crate を経由して COM を叩きます。`EnumAudioEndpoints(eRender, ACTIVE|UNPLUGGED|DISABLED)` で再生エンドポイントを全部取り、`IMMDevice::GetId()` でデバイス ID、`OpenPropertyStore` + `PKEY_Device_FriendlyName` で表示名を取ります。

**PROPVARIANT の union 直参照は `windows` 0.58 で使えない**という地味な落とし穴があります (opaque 型になっている)。`prop.to_string()` の Display 経由が正解。

### 切替は非公開 COM `IPolicyConfig::SetDefaultEndpoint`

問題はここから。Windows Core Audio には「既定エンドポイントを変える」公開 API がありません。EarTrumpet や SoundVolumeView が使っている非公開 COM `IPolicyConfig` を Rust から手動で叩きます。

- CLSID: `{870af99c-171d-4f9e-af0d-e63df40c2bc9}` (`CPolicyConfigClient`)
- IID (Windows 10/11): `{f8679f50-850a-41cf-9c72-430f290290c8}`
- IID (Vista fallback): `{568b9108-44bf-40b4-9006-86afe5b5a620}`

vtable のスロット 13 に `SetDefaultEndpoint(LPCWSTR device_id, ERole role)` が座っています。他のスロットは opaque ポインタで場所だけ確保、SetDefaultEndpoint だけ型付けする形で `#[repr(C)]` の vtable を Rust で手書きしました。

```rust
#[repr(C)]
struct IPolicyConfigVtbl {
    query_interface: unsafe extern "system" fn(*mut c_void, *const GUID, *mut *mut c_void) -> HRESULT,
    add_ref: unsafe extern "system" fn(*mut c_void) -> u32,
    release: unsafe extern "system" fn(*mut c_void) -> u32,
    _reserved_3_to_12: [*const c_void; 10],
    set_default_endpoint: unsafe extern "system" fn(*mut c_void, PCWSTR, ERole) -> HRESULT,
    _reserved_14: *const c_void,
}
```

3 役割 (`eConsole` / `eMultimedia` / `eCommunications`) を順に呼びます。**まとめて切り替えないと会議アプリの Communications 既定が取り残される**ため、常に 3 つ同時が仕様。

### 検証は RDP 内では不可

`cargo run -- list` を最初に叩いたら「リモート オーディオ」1 個だけが返ってきて焦りました。原因は RDP セッション内で走らせていたから。Windows は RDP セッションに対して物理デバイスを見せない設計です。

**物理コンソールで動かさないと本物の検証にならない**という制約が判明。plan と README に「開発時は物理コンソール必須」を書き足しました。

---

## UI/UX 壁打ち 第一幕: LAN-first 転換

### 「安全な既定 = 127.0.0.1」が製品目的を殺していた

最初の HTTP サーバー実装は `bind = "127.0.0.1"` を既定にしていました。「LAN 露出はユーザーが明示的に選ぶべき」という発想です。

でも壁打ちしていて気付きました。**「別 PC のブラウザから隣のホストを切替える」が製品の core value なのに、既定で LAN に応答しないのは目的と矛盾している**。

初回ユーザーが exe を叩く → 何も応答しない → 「LAN で使うには config.toml 編集」→ 玄人操作要求 → 大半のユーザーが脱落。

**転換**: `bind = "0.0.0.0"` を既定へ。Bearer トークン認証で守るのは変わらず、loopback (127.0.0.1) だけ認証を素通しにする (host 自身は自分の音を触れる)。

### ゲスト側で「トークン手打ち」の摩擦

LAN 越しにアクセスするゲストブラウザはトークン必須。でもトークンをホストのコンソールからコピーしてゲスト側で手打ちするのは玄人動線。

解決策: **URL フラグメントにトークンを埋め込んで印字**。

```
Open on your other machine (guest Win11 / phone / VM) - token embedded:
  http://203.0.113.5:17650/#t=ar_live_xxxxxxxxxxxx   (Ethernet)
  http://203.0.113.9:17650/#t=ar_live_xxxxxxxxxxxx   (vEthernet / 仮想スイッチ)
```

フラグメント (`#t=...`) の値は**サーバーに送信されず**、ブラウザ内で完結します。サーバーログや Referer には残らない。Web UI 側は `location.hash` から抜き取り、`localStorage` に保存後 `history.replaceState` で URL からクリア。ゲストは 2 回目以降トークン抜き URL をブックマークして使えます。

### NIC 列挙で「LAN IP どこ？」を消す

さらに `local-ip-address` crate で全 NIC を自動列挙。物理 LAN と仮想スイッチ (Hyper-V vEthernet 等) をラベル付きで両方出す。**ユーザーが `ipconfig` を叩かなくて済む**動線です。

![](/images/audioremote_fig2-flow.png)

上の図で示したように、初回動線は「ホストで exe → 自動ブラウザ起動 → 他マシン URL コピー → ゲストで開く → 使える」の 5 ステップ。2 回目以降はゲスト側でブックマークタップ → タップ切替の 2 ステップ。

---

## UI/UX 壁打ち 第二幕: loopback バイパスと成功フィードバック

### ホスト自身にもトークン強制するな

ゲスト側のトークン導線を整えたら、次はホスト側の摩擦。**ホスト自身のブラウザで `127.0.0.1:17650` を開いたときにもトークン画面が出る**のは冗長。だって自分自身の音を触るだけなのに。

そこで **loopback (127.0.0.1 / ::1) から来た HTTP 要求は Bearer 検証をスキップ**するミドルウェアを追加。実装はシンプル、axum の `ConnectInfo<SocketAddr>` で peer IP を取り、loopback なら next.run。

```rust
if is_loopback(peer.ip()) {
    return Ok(next.run(request).await);
}
```

セキュリティ的に問題ないか。Windows OS が保証する loopback インターフェースは「ローカルからのみ」の経路なので、認証を落としても外部露出はゼロ。LAN からのアクセスはトークン必須のまま。

と、この時は思っていました。ここに穴があったことはリリース直前の監査で分かります。後半で書きます。

追加で **DNS rebinding guard** (許可 Host ヘッダを起動時に snapshot して外部由来を弾く) と **CIDR allowlist** (v0.2 で allowed_networks に対応) も入れました。

### 成功トーストは要らない・失敗だけ強めに

デバイス切替に成功したときにトーストを出していたのを、途中で削除しました。

理由:
- タップ即座に**カードの左端 ○ (未選択) → ● (選択・緑塗り)** が動く
- この視覚変化そのものがフィードバック
- そこに「切替えました」トーストを重ねると redundancy

代わりに**失敗時だけ上中央に赤トースト**。失敗はカードの状態が戻るだけだと気付きにくいので、視認性を確保。

「コピー」ボタンも同じ思想:
- 押した瞬間、ボタンが**緑にフラッシュ + bounce アニメーション**
- 同時に URL 入力欄が**薄緑にフラッシュ** (何がコピーされたか示す)
- 1.6 秒で元に戻る

トーストは出さない。「押した感」と「対象が分かる感」の両方をインラインで完結させる。

---

## audience を再定義する: 「非エンジニアが 2 台の Win11 持ってる」の嘘

### 「MS Store で非エンジニアに」の見落とし

配布経路の話をしていて、壁打ちの相手 (私自身) がふとこう言いました。「MS Store で出せばダブルクリックで完結する非エンジニアにも届くじゃん」。

一瞬「その通り」と思いました。SmartScreen 完全回避 + auto-update + winget にも自動連携。しかも登録料は 2025-09 の新オンボードで個人・法人ともに撤廃されていて、無料で入れます（https://learn.microsoft.com/ja-jp/windows/apps/publish/whats-new-individual-developer）。なぜこれを見落としていたんだろうと思いました。

ただ後で調べ直して分かったのは、無料になったのは入口だけということ。実際に効いてくるコストは金銭ではなく手間です。本人確認、MSIX パッケージング、認定審査、掲載素材。1 作品で通しておけば残りに流用できる、という性質のもの。

### でも「そもそも非エンジニアはこれ使わない」

さらに壁打ちしていて次の指摘が出ました。

**「非エンジニアが Windows PC を 2 台同じ LAN に置いて、ホスト・ゲスト構成で運用してるのが怪しくない？」**

これは真理でした。

一般的な人は Windows PC 1 台 + スマホの構成。「机の隣にホスト機を置いて、そこで音を鳴らして、別 PC から操作する」ライフスタイル自体が非エンジニア想定外。**MS Store で見つけた非エンジニアが**:

- 「これ何ができるの？」
- 「私の PC 1 台しか無いけど」
- 「音を鳴らしっぱなしにする用事無いよ」

というリアクションになる。**リーチしても刺さらない**。

### 対象を絞ると製品が尖る

audience 定義を絞り込んだ結果:

- ☓ 音ゲー勢 (ホスト完結の話 → Windows 標準で十分)
- ☓ 1 台 PC ユーザー (Hyper-V も使ってない)
- ☓ スマホしか無い層
- ○ **Hyper-V ゲストで開発している人** (私はここ・ホストは Steam ゲーム機兼家庭用)
- ○ **物理的に 2 台の Windows** を同じ LAN に置いている人 (開発 + 配信、開発 + ホームラボ、開発 + HTPC 等)
- ○ 配信勢 (ゲーム PC + 配信 PC の 2 台構成)
- ○ 小オフィスのターミナル運用
- ○ ホストにスピーカー + Nest Hub Max 等の複数出力を繋いでる人

**共通項は「音を鳴らす場所と、操作する場所が違う」**。物理的に離れているか、仮想化で離れているか、の違いだけ。

audience 数は少ないが濃い層。この層に fit する製品を作る方が、Store で薄く広く撒くより価値がありそう。

---

## MS Store の位置付けを再々修正

audience 定義を反映して、MS Store の戦略的優先度を下げました。

| | audioremote | many-ai-cli (別プロダクト) |
|---|---|---|
| Store で非エンジニア到達価値 | 低 (audience 技術寄り) | 高 (AI 触りたい層は広い) |
| Store の auto-update 価値 | 低 (更新頻度少) | 高 (vendor 追随で頻繁) |
| SmartScreen 回避価値 | 低 (技術寄りは自力で解決) | 高 (新規参入者に優しい) |

**audioremote は Store 戦略の主軸に据えない**。ただし v0.2 で Tauri ゲストアプリを作るとき MSIX 化のコストがほぼゼロで済むので、そのタイミングで opportunistic に参入する。

MS Store の学習は、私が別に持っている GUI 完成品 `offline-md-editor-viewer` で先に消化してから、audioremote に還元する順序。

### MS Store 参入順序 (確定)

1. **offline-md-editor-viewer**: Store 提出学習の教材 (Tauri 完成品)
2. **many-ai-cli**: Store 価値最大化のプロダクト
3. **audioremote v0.2**: 上記で学習済みのタイミングで opportunistic
4. **PlainSheet**: 完成後 (時期未定)

![](/images/audioremote_fig3-msstore-strategy.png)

---

## v0.1 のスコープを確定する

壁打ちの結果、v0.1 のスコープが最初の想定より広がりました。

### 追加された機能: マスター音量 + ミュート

出力デバイス切替だけでは「不完全なリモコン」でした。ホストで動画を流していて、**音量が合わないときにゲストから調整できない**。またホストに戻る羽目になる。

- `IAudioEndpointVolume` 経由でマスター音量スライダ (0-100)
- ミュート / アンミュート toggle
- Per-app volume (アプリごとの音量ミキサ) は v0.2 送り

これで audioremote は「デバイス切替 + 音量調整 + ミュート」の**ちゃんとしたオーディオリモコン**になります。

### 追加された配布経路の検討: Scoop と winget

`winget` と別軸で dev-oriented 層に刺さります。manifest 追加のみで low-cost:

```powershell
scoop install audioremote
```

この時点では v0.1 の配布を 4 系統（npm + winget + Scoop + GitHub Releases 直 DL）で考えていました。実際に v0.1.0 で出したのは **npm + crates.io + GitHub Releases の 3 系統**で、winget と Scoop は v0.2 送りになりました。crates.io が入れ替わりで増えたのは、Rust 製なので `cargo install` の口を先に用意した方が筋が通ると判断したからです。

![](/images/audioremote_fig4-distribution.png)

### v0.1 の含む・含まない

**含む** (v0.1):
- Windows Core Audio デバイス列挙 + 3 役割まとめ切替 (実装済)
- Bearer トークン認証 + loopback バイパス + DNS rebinding guard (実装済)
- 内蔵 Web UI (vanilla JS・言語切替・About モーダル・共有パネル・SVG アイコン) (実装済)
- ホスト起動時 splash art + NIC 自動列挙 + LAN URL 印字 (実装済)
- マスター音量スライダ + ミュートボタン (**新規追加**)
- 対話ウィザード `audioremote setup` (実装済)
- `--install-autostart` (HKCU Run への登録・解除)
- 配布: npm + crates.io + GitHub Releases 直 DL（winget と Scoop は v0.2 送り）

**含まない** (v0.2 送り):
- タスクトレイアイコン
- Tauri ゲストアプリ
- Per-app volume / EQ / メディアキー
- MS Store 提出
- コード署名

![](/images/audioremote_fig5-scope-comparison.png)

---

## v0.2 は Tauri ゲストアプリ + トレイ + Store の統合版

### 「サーバー起動しっぱなし」の心理的違和感

壁打ちで「常駐サーバー概念って UX 的にどうなの？」の議論が出ました。ホストが起きてる = サーバー動いてるという設計は core value なので変えられない。でも「ずっと動いてる何か」に対する心理的違和感は残る。

解決策は 2 つ:

1. **タスクトレイアイコン** (ホスト側 Rust に `tray-icon` crate 追加)
2. **Tauri ゲストアプリ** (ゲスト側を native ウィンドウ化)

トレイアイコンがあれば「起動しっぱなし」が可視化される。Discord みたいに「そこに居る」感が出せる。Tauri アプリにすればゲスト側でも「アプリを開く・閉じる」の自然な感覚になる。

### Tauri の選択理由

- 既存 Web UI コード (vanilla JS + HTML + CSS) を**そのまま流用可能**
- native ウィンドウ・Windows ネイティブ通知使える
- MSIX bundler が Tauri v2 に統合済み・Store 参入コストほぼゼロ
- Rust ベースなので既存の Rust 学習投資がそのまま生きる
- macOS / Linux 対応が副次的についてくる

### Web UI は残す

Tauri アプリを主推奨にしても、Web UI は残します:

- インストール摩擦ゼロが強み (「ブラウザで開くだけ」の即時性)
- スマホ / Linux / 追加インストール不要ケースのフォールバック
- Tauri アプリと同じ HTTP API を叩くので実装は 1 セットで済む

**「native アプリらしく使いたい人」→ Tauri**、**「1 回だけ触りたい人」→ Web UI**、両方 offer する構造。

---

## フロントエンド 0 依存の意義

audioremote の Web UI は npm も bundler もフレームワークも使っていません。`web/` 配下は `index.html` + `style.css` + `app.js` + `icons/` の 4 種のみ。

- **React / Vue / Svelte 等**: 無し
- **Tailwind / Bootstrap**: 無し
- **Font Awesome / Material Icons**: 無し (アイコンは全部自作 SVG)
- **Google Fonts**: 無し (system-ui + ヒラギノ / 游 / Segoe UI)
- **npm / pnpm / bun**: 無し (`package.json` すら無し)
- **jQuery**: 無し
- **webpack / vite / esbuild**: 無し

JS は **vanilla** (`h()` ヘルパー 12 行だけ自作、あとは `fetch` / DOM API のみ) で約 350 行。

意義は何か。特筆すべきは **JS サプライチェーン攻撃の経路が構造的に存在しない**こと。npm ecosystem の脆弱性 (xz-utils 級の事件がフロント側から入ってくる可能性) が audioremote では**原理的に発生しない**。

「軽い」ことよりも、「攻撃対象面が小さい」ことの方が価値です。個人 OSS で作る自作ツールなら、この選択は理にかなっています。

![](/images/audioremote_illustration-ui.png)

---

## そして v0.1.0 を公開しました (2026-07-31)

ここから先は、上の壁打ちから 6 日後の話です。

### リリース直前に監査を回した

タグを打つ前に、セキュリティと品質の監査を一度通しました。未署名の exe を LAN に晒す製品で、しかも既定が `0.0.0.0` bind です。「動いたから出す」で済ませたくなかった。

出てきた指摘は 18 件。うち 16 件を直して、6 件は理由を書いて保留にしました。保留のほうも「やらない」ではなく「今やる価値がこれだけ薄い」を明記して残す形にしています。

正直、この監査を回さずにタグを打っていたら、と思うと少し寒くなりました。以下がその理由です。

### 一番効いたのは「動く CSRF」だった

上の「loopback バイパス」の節で、こう書きました。「loopback インターフェースはローカルからのみの経路なので、認証を落としても外部露出はゼロ」。

これが甘かった。

`POST /api/devices/<id>/default` は body を取りません。body が無いということは、ブラウザから叩くときに preflight が発生しないということです。そして URL に `127.0.0.1` という IP リテラルを直接書けば、DNS も使いません。つまり:

- 送信元の TCP peer は loopback に見える。だからトークン検査を素通りする
- `Host` ヘッダは `127.0.0.1:17650` になる。だから DNS rebinding guard も通る
- preflight が無いので CORS でも止まらない

結論として、**利用者がホストのブラウザで悪意あるページを 1 枚開いただけで、そのページから音声出力デバイスを切り替えられる**状態でした。実害は「音が変な所から出る」程度とはいえ、他人のページが自分のホストの状態を書き換えられるのは普通に穴です。

対処は Fetch Metadata です。状態を変える要求には `Sec-Fetch-Site` が `same-origin` か `none` であることを求め、`Origin` があれば `Host` と突き合わせます。この 2 つはページのスクリプトからは設定できない forbidden header なので、偽装できません。curl やスクリプトはどちらも送ってこないので、そのまま通します。ブラウザ経由の第三者起点だけを弾ける形です。

「OS が保証する境界」を信じたのが間違いだったのではなくて、**ブラウザという踏み台がその境界の内側にいる**ことを見落としていました。

### 3 秒ごとのポーリングが、静かにメモリを食っていた

`IMMDevice::GetId` は `PWSTR` を返します。この文字列は Windows 側が確保するので、呼び出した側が `CoTaskMemFree` で解放する契約です。windows-rs の `PWSTR` は所有権を持たないので、放っておくと解放されません。

Web UI は 3 秒ごとにデバイス一覧を取りに来ます。デバイスが N 台なら、ざっくり `2*(N+3)+1` 個の確保が 3 秒ごとに積まれていく計算になります。常駐サービスなので、これは効いてきます。

対処は、ポインタを受け取った瞬間に RAII で包むだけ。取得直後に包めば、途中の変換失敗や早期 return でも漏れません。

こういうのは「動作確認」では絶対に出てきません。動くので。

### 宣言していた MSRV が嘘だった

`Cargo.toml` に `rust-version = "1.75"` と書いていました。書いた記憶があるだけで、検証はしていませんでした。

監査は「rust-embed が 1.80 を要求しているので 1.75 では通らない」と指摘してきました。ただこれは直接依存しか見ていない指摘でした。lock されている 136 crate の `rust-version` を実測すると、`toml 1.1` や `sha2 0.11` や `indexmap 2.14` あたりが **1.85** を要求していました。

なので 1.80 でもまだ嘘で、正しくは 1.85。宣言し直して、その版を固定した CI ジョブを足しました。これで次に誰かが依存を上げたとき、宣言と実態がずれた瞬間に赤くなります。

MSRV は「公開している契約」なので、検証していない数字を書くのは README に嘘を書くのと同じでした。

### 公開当日に 2 回転んだ

準備を終えてタグを push したあと、2 回止まりました。

1 回目は自分の CI です。`npm test` に書いた `node --test "test/*.test.mjs"` が Node 20 で落ちました。`--test` のグロブ展開は Node 22 以降で、Node 20 はこれをただのパスとして扱います。手元が Node 24 で、そちらではディレクトリ指定が通らなかったのでグロブに逃げていました。CI を Node 20 に固定したのは自分なのに、そこで突き合わせていなかった。引数なしの `node --test` が両方で動くので、それに直しました。

2 回目は crates.io です。`403 Forbidden: authentication failed`。パッケージングも、展開したパッケージからのビルド検証も通って、アップロードの認証だけで落ちました。トークンを publish-update スコープで再発行して、失敗したジョブだけ再実行したら通りました。

このとき、ついでに自分のワークフローの穴も 2 つ見つかりました。crates.io だけを出し直す経路が用意されていなかったこと。それと、crates.io の API は User-Agent の無いリクエストに 403 を返すので、「この版は公開済みか」を見る判定が**常に未公開側に倒れていた**こと。どちらも直して、実際に手動実行して動くところまで確認しました。復旧経路は、作った時点では動きません。叩いて初めて分かります。

### 公開した瞬間、README が嘘になる

3 経路に出し終わって、リポジトリを見たら README の先頭にこう書いてありました。

```
> Status: v0.1 in development. Not yet released.
## Install (planned - not yet published)
```

npm にも crates.io にも出ているのに、リポジトリの顔が「まだ出していません」と言っている状態。公開作業のチェックリストに「README を実態に合わせる」を入れていませんでした。

リリースは、タグを push した瞬間には終わりません。

### 出したもの

- **GitHub Releases**: zip + `SHA256SUMS.txt`
- **npm**: `audioremote` と、exe を同梱する platform package の 2 本
- **crates.io**: `audioremote`

npm の platform package は、公開直後に数分だけ 404 を返しました。反映待ちです。ここで「公開できていない」と早合点しかけたので、公開直後の確認は遅延を織り込んだほうがいいです。

## 残っているのは v0.2

### v0.2 (次バージョン)

- Tauri ゲストアプリ (Web UI コード流用)
- ホスト側にタスクトレイアイコン + グローバルメニュー
- `--install-autostart` の完成 (firewall rule 自動追加含む)
- Per-app volume (session mixer)
- MSIX 化 + MS Store 提出 (Tauri 側)
- Azure Trusted Signing 検討
- リモート再起動と「落ちたら起こす」(トレイ常駐が本体を子プロセスとして管理する形。起動は物理コンソールセッションでやる必要があるので、常駐プロセスから spawn してセッションを継承させるのが唯一素直な解になります)
- winget と Scoop の manifest (v0.1 で見送った分)

---

## ここまでで学んだこと

![](/images/audioremote_illustration-desk.png)

- **audience を先に絞る**と設計判断が楽になる。「非エンジニアにも」は聞こえがいいけど、audience の 20% 未満なら切り捨てて 100% を尖らせた方が製品として強い
- **既定値は目的と一致させる**。「安全側 default」の慣習を、目的と衝突するときは躊躇なく反転させる (LAN-first 転換)
- **成功のフィードバックは最小に、失敗のフィードバックは最大に**。redundancy は UX の敵
- **設計判断は壁打ちで固まる**。自分ひとりで考えるより、対話形式で「これで良いのか？」を問い続ける方が漏れが少ない (今回 Q1〜Q4 の 3 段階で判断が全部変わった)
- **Store 参入は audience とセット**で考える。Store があるから届くわけではない。Store の audience とプロダクトの audience が一致してるかを見る
- **書きながら実装しながら壁打ち**が最強。文書化 (plan / recap) と実装が同じ日にある方が、後から見て意図が読める

リリース当日に足されたぶん:

- **「OS が保証する境界」を信じるときは、その内側に何がいるかを数える**。loopback は確かに外部から届かない。でもブラウザは内側にいて、外部のページの指示で動く
- **動作確認で見つからない欠陥がある**。メモリリークも MSRV の嘘も、動くので気づけない。監査を別の目として一度通す価値はここにあります
- **復旧経路は、作った時点では動かない**。実際に叩いて初めて分かる。今回は crates.io だけを出し直す経路が用意されていなくて、それに気づいたのが本番で詰まった後でした
- **公開直後の検証は反映遅延を織り込む**。npm の新規パッケージは数分 404 を返します。慌てて「失敗した」と判断しかけました
- **リリースはタグを push した瞬間には終わらない**。README が「まだ出していません」と言ったままでした

---

## 作者の関連プロダクトも紹介させてください

本文中で名前を出したので、興味があれば覗いてもらえると嬉しいです。どれも Windows 特化の個人 OSS で、同じ「開発しながら日常運用を楽にする」文脈で作っています。

- **[many-ai-cli](https://github.com/ishizakahiroshi/many-ai-cli)**: 複数の AI コーディング CLI (Claude Code / Codex / Copilot / Cursor / Grok 等) を並列で走らせて、承認をブラウザ 1 タブに集約するローカル Web ダッシュボード。ターミナルの往復が消えます。**MS Store 参入の第一候補**として学習を先行させる予定
- **[offline-md-editor-viewer](https://github.com/ishizakahiroshi/offline-md-editor-viewer)**: Markdown ファイルをオフラインで開いてすぐ表示する Tauri デスクトップアプリ。すでに Windows Releases 配布あり。この記事に出てくる **audioremote v0.2 の Tauri ゲストアプリ**は、この offline-md で得た MSIX 化知見を移植する予定です

どれも「開発中に自分で使いたくて作ったツール」です。同じ棚で開発してるので、フィードバックはまとめて受け止めます。

## audioremote はこんなときに刺さります

- **Hyper-V ゲストで開発**しているが、ホスト側の音声デバイスを触りたい人 (私と同じ構成)
- Windows PC を **2 台同じ LAN** に置いて、片方から片方の音声出力を操作したい人
- **ゲーム PC + 配信 PC** の 2 台構成で、ゲーム側の音を配信側から切り替えたい人
- ホストにスピーカー + Nest Hub Max + 有線イヤホン等**複数の出力デバイス**を繋いでいる人
- 会議アプリで **Console / Multimedia / Communications** の 3 役割を毎回まとめて切り替えたい人
- ホストの前に戻らずに **音量 + ミュート**もゲストから調整したい人 (v0.1 で対応)

いずれかに心当たりがあれば、`npm i -g audioremote` で 1 分で試せます。設定ゼロで動きます。フロントエンド 0 依存の単一 exe なので、Windows 11 ホストに置いて起動するだけです。

- リポジトリ (Issue / PR 歓迎): https://github.com/ishizakahiroshi/audioremote
- npm: https://www.npmjs.com/package/audioremote
- crates.io: https://crates.io/crates/audioremote

Star をいただけると開発の励みになります。使ってみて「ここが不便」があれば、Issue でも X の DM でも大歓迎です。

---

## おわりに

作りながら考えたことを、記録として残しました。壁打ちの過程は、後から見返すと**製品の意図がどう変わったか**が分かって価値があります。

6 日前に「これ作ります」と書き始めて、v0.1.0 が 3 経路に出ました。ただ手応えとして残っているのは、公開できたことより、公開前に穴が 1 つ見つかったことのほうです。あの CSRF に気づかないまま出していたら、この記事は「作りました」だけの記事になっていました。

個人 OSS なので「作って自分で使うため」がまず第一で、他人に届けば嬉しい、くらいの温度感で続けます。v0.2 は Tauri とトレイ常駐です。

小さく。次の 1 タップを、ホストの前まで歩かずに済ませていきます。

---

※ ヘッダー画像とインフォグラフィックは AI（画像生成）で作成しています。

※ 本文の挿絵も AI（画像生成）で作成しています。

書いた人: ishizakahiroshi (システムエンジニア・実務 18 年・田舎在宅・バックエンド / インフラ / AI 連携)

- ポートフォリオ: https://ishizakahiroshi.com/
- GitHub: https://github.com/ishizakahiroshi
- X: https://x.com/ishizakahiroshi

業務委託・フルリモートで受注しています。バックエンド / インフラ / AI 連携が専門。「現場の業務課題を、最小限の実装で、確実に動くものに」を大事にしています。相談歓迎です。
