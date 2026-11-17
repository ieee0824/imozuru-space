# 開発サンドボックス (Izanagi + Claude Code + imozuru)

[Izanagi](https://github.com/ieee0824/izanagi) v0.2.7 で隔離したコンテナの中で、
Claude Code 2.1.312 と [imozuru](https://github.com/MaryCache/imozuru) v2.2.0 のスキル、
[cc-sakura クライアント](https://github.com/ieee0824/cc-sakura-cloud-client) v0.4.0 を使う環境です。
コンテナのランタイムは Apple `container` と Docker のどちらでも使えます。

イメージは Debian 13 (trixie) ベースです。cc-sakura のように glibc 向けにしか配布されていないバイナリを動かすため、
Izanagi 標準の Alpine (musl) ではなく Debian にしています。

## 構成

| ファイル | 役割 |
|---|---|
| `.izanagi/Dockerfile` | Debian 13 + `izanagi-vm:0.2.7` の agent + Node.js + Claude Code + imozuru スキルとその依存 (Python・poppler・tesseract・yt-dlp) + cc-sakura |
| `.izanagi/izanagi.toml.example` | Izanagi 設定の雛形 (このファイル自体は読み込まれない) |
| `scripts/sandbox` | ビルド・起動・停止などの操作コマンド |
| `research/` | imozuru の調査結果 (サンドボックス内の `~/research`)。`up` 時に作成 |

## 前提条件

- macOS (Apple Silicon)
- Izanagi **0.2.7** (`izanagi --version`)。ホストと guest agent の版をそろえるため、`scripts/sandbox` は他の版では動きません
  - インストール手順: <https://github.com/ieee0824/izanagi#配布バイナリ>
- コンテナランタイム (どちらか)
  - Apple `container` CLI (`container system start` で起動しておく)
  - Docker (Colima など。`docker info` が通ること)
- git (ベースイメージのビルド時に Izanagi のソースを取得)

## 使い方

ランタイムは `SANDBOX_RUNTIME=container|docker` で選びます。
`init` で設定に書き込んだ後は、指定しなくても設定の値が使われます。
イメージはランタイムごとに別々に保存されるので、切り替えるときはそのランタイムでもう一度 `build-base`・`build` を実行してください。

```bash
export SANDBOX_RUNTIME=docker     # Apple container なら container (既定)

# 1. イメージを作る (初回とバージョン更新時のみ)
bash scripts/sandbox build-base   # Izanagi v0.2.7 のソース (コミット固定) から izanagi-vm:0.2.7 (agent の取り出し元)
bash scripts/sandbox build        # imozuru-space-sandbox:claude-2.1.312

# 2. 設定を作る (初回とランタイムの切り替え時)。保存先は ~/.izanagi/settings/<hash>/izanagi.toml (共有範囲外)
bash scripts/sandbox init         # 切り替え時は init --force

# 3. 起動して確認
bash scripts/sandbox up
bash scripts/sandbox check        # claude・cc-sakura の --version と /workspace の書き込み確認

# 4. サンドボックスの中で Claude Code を使う
bash scripts/sandbox shell
#   (サンドボックス内) claude
#   初回は /login で OAuth ログイン (表示される URL をホストのブラウザで開き、コードを貼り付ける)
#   例: 「芋づる式に調べて: ○○」 → 結果は /workspace/research/<話題>/ (ホストの research/) に保存される

# 5. 停止
bash scripts/sandbox down
```

## imozuru

- README の手動インストール (`skills/imozuru/` を `~/.claude/skills/` へコピー) を、タグ `v2.2.0` (コミット固定) で
  イメージに組み込んでいます。`/plugin install` で入れると、コンテナ内に入るので `down` のたびに消えます。
- 調査結果の既定の置き場 `~/research` は `/workspace/research` へのシンボリックリンクにしてあり、
  ホストの `research/` に残ります。
- 任意の外部コマンドのうち `pdftotext`・`pdftoppm`・`tesseract` (eng・jpn・jpn_vert)・`yt-dlp` を入れています。
  - tesseract の言語データは Debian のパッケージ (tessdata_fast 系の 4.1.0) です。
    Alpine 版のイメージで使っていた tessdata より小さく速い代わりに、OCR の精度はやや落ちます。
  - Debian の `yt-dlp` は古く YouTube の変更に追従できないため、公式リリースの zipapp を SHA256 で固定して入れています。
- [ax](https://github.com/yusukebe/ax) は入れていません。
  HTML の本文は imozuru が標準ライブラリで取り出します (見出しや表の形は ax より崩れやすい)。
- imozuru の注意のとおり、問いの内容は検索語として外部に送られます。

## cc-sakura クライアント

- [cc-sakura-cloud-client](https://github.com/ieee0824/cc-sakura-cloud-client) v0.4.0 の配布バイナリ (Linux glibc 版) を
  `/usr/local/bin/cc-sakura` に入れています。リリースの `SHA256SUMS` と同じ値を Dockerfile に固定して検証しています。
- Claude Code 用の使い方スキル `skills/cc-sakura` を、バイナリと同じタグ (コミット固定) から `~/.claude/skills/cc-sakura` に入れています。
  `skills/cc-sakura-watch` は Codex CLI 用の返信アダプターなので入れていません。
- 参加トークン・署名鍵・profile はイメージに入れていません。サンドボックス内で、クライアントの README の手順
  (`auth keygen` → 招待 JSON を保存 → `auth join`) で参加します。
- **鍵・profile (`~/.config/cc-sakura-client/`) と送信台帳 (`~/.local/state/cc-sakura/`) はコンテナの中にあり、
  `scripts/sandbox down` で消えます。** ホストには意図的に保存していません。次に起動したときは新しい招待で参加し直す必要があり、room には別の参加者として登録されます。
  結果が不明な投稿の `auth retry` も、down した後はできません。
- Claude Code から MCP として使う場合は、サンドボックス内で登録します (`~/.claude.json` に保存されるので、これも down で消えます)。

  ```bash
  claude mcp add sakura -- cc-sakura --config /home/izanagi/.config/cc-sakura-client/room.toml mcp
  ```

- v0.4.0 の共有 daemon は、コンテナ内の同じユーザー (`izanagi`) のプロセスだけで共有されます。
  サンドボックス内の他のプロセス (npm の postinstall など) からも、鍵や profile を読めます。

## セキュリティ上の注意

### ワークスペースは書き込み可

`[share] read_only = false` で、プロジェクトのディレクトリ全体を `/workspace` に書き込み可でマウントします。
サンドボックス内の Claude Code やツールの変更は **ホストのファイルにそのまま反映され、取り消せません**。

- `scripts/sandbox` と `.izanagi/` も共有範囲に含まれ、ホストで実行されます。
  サンドボックスでの作業後は、これらや Git フック・IDE 設定などに想定外の変更がないか確認してから使ってください。
- 信頼できないコードを動かす場合は、使い捨てのコピーを別ディレクトリに作り、そこでサンドボックスを使ってください。

### 認証情報

- イメージにも設定ファイルにも認証情報は入れていません。ホストの `~/.claude` やキーチェーンもマウントしません。
- Claude Code の認証はサンドボックス内で `/login` します。認証情報はコンテナ内の
  `/home/izanagi/.claude` にだけ保存され、`scripts/sandbox down` でコンテナごと破棄されます。
  そのため、起動するたびにログインし直す必要があります。
- 起動中は、コンテナ内で同じユーザー (`izanagi`) として動く他のプロセス (npm の postinstall など) からも、
  この認証情報を読めます。

### ネットワークと監視

- どちらのランタイムでもコンテナのネットワークは標準の bridge です。**外への通信は制限されません**
  (Claude Code が Anthropic API に、imozuru が Web に接続するために必要)。
- Izanagi はこの構成 (`apple-container` バックエンド) ではホスト側の syscall トレーサーを使えないため、`tracer = "none"` にしています。
  **ファイルアクセスや通信の監視・検知は行われません**。`izanagi logs` にも syscall イベントは記録されません。

### ランタイムによる違い

| | Apple `container` | Docker (Colima) |
|---|---|---|
| 隔離 | コンテナごとに軽量 VM (Virtualization.framework) | Colima の VM 上で、他の Docker コンテナとカーネルを共有 |
| メモリ | `container` の既定 (1GB)。Izanagi からは変更不可 | Colima の VM 全体 (この環境では約 8GB) |
| 届く範囲 | 外部ネットワーク | 外部ネットワーク + 同じ VM で公開されている他のコンテナのポート |

Colima の VM にはホームディレクトリがマウントされているため、コンテナから VM に抜けられた場合の影響は
Apple `container` より大きくなります。

## Izanagi 0.2.7 への移行

v0.2.7 では [#115](https://github.com/ieee0824/izanagi/issues/115)・
[#116](https://github.com/ieee0824/izanagi/issues/116)・
[#117](https://github.com/ieee0824/izanagi/issues/117) が修正されています。
ベースイメージ (agent の取り出し元) は上流の `image/build.sh` でビルドします。
上流のイメージは #117 への対処として `tini` を PID 1 にしているので、このリポジトリの Debian のイメージでも同じく `tini` を PID 1 にしています。
設定はタグ付きイメージを直接参照します。

既存環境では、稼働中のサンドボックスを旧版で停止してからホストの Izanagi を 0.2.7 に更新し、
`build-base` → `build` → `init --force` → `up` を実行してください。
`init --force` は既存の設定を雛形で上書きするため、独自の設定は事前に控えてください。

## バージョンの更新

| 対象 | 変更する場所 |
|---|---|
| Izanagi | `scripts/sandbox` の `IZANAGI_VERSION`・`IZANAGI_COMMIT`、`Dockerfile` の `IZANAGI_VERSION`、ホストの `izanagi` |
| Debian | `Dockerfile` の `FROM debian:trixie-slim@sha256:...` (更新手順はコメントを参照) |
| Node.js | `Dockerfile` の `NODE_VERSION`・`NODE_SHA256_ARM64`・`NODE_SHA256_AMD64` |
| Claude Code | `scripts/sandbox` の `CLAUDE_CODE_VERSION`、`Dockerfile` の `CLAUDE_CODE_VERSION`、`izanagi.toml.example` の `image` |
| imozuru | `Dockerfile` の `IMOZURU_REF`・`IMOZURU_COMMIT` |
| yt-dlp | `Dockerfile` の `YT_DLP_VERSION`・`YT_DLP_SHA256` |
| cc-sakura | `Dockerfile` の `CC_SAKURA_VERSION`・`CC_SAKURA_SHA256_ARM64`・`CC_SAKURA_SHA256_AMD64`・`CC_SAKURA_COMMIT` (スキル) |

SHA256 の取得元は `Dockerfile` のコメントに書いてあります。

変更後に `build-base` (Izanagi のとき) → `build` → `init --force` を実行してください。
Claude Code はイメージ内で自動更新しません (`DISABLE_AUTOUPDATER=1`)。
