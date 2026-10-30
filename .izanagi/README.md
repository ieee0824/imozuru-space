# 開発サンドボックス (Izanagi + Claude Code + imozuru)

[Izanagi](https://github.com/ieee0824/izanagi) v0.2.6 で隔離したコンテナの中で、
Claude Code 2.1.312 と [imozuru](https://github.com/MaryCache/imozuru) v2.2.0 のスキルを使う環境です。
コンテナのランタイムは Apple `container` と Docker のどちらでも使えます。

## 構成

| ファイル | 役割 |
|---|---|
| `.izanagi/Dockerfile` | `izanagi-vm:0.2.6` + Claude Code + imozuru スキルとその依存 (Python・poppler・tesseract・yt-dlp) |
| `.izanagi/izanagi.toml.example` | Izanagi 設定の雛形 (このファイル自体は読み込まれない) |
| `scripts/sandbox` | ビルド・起動・停止などの操作コマンド |
| `research/` | imozuru の調査結果 (サンドボックス内の `~/research`)。`up` 時に作成 |

## 前提条件

- macOS (Apple Silicon)
- Izanagi **0.2.6** (`izanagi --version`)。ホストと guest agent の版をそろえるため、`scripts/sandbox` は他の版では動きません
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
bash scripts/sandbox build-base   # Izanagi v0.2.6 のソース (コミット固定) から izanagi-vm:0.2.6
bash scripts/sandbox build        # imozuru-space-sandbox:claude-2.1.312 (+ :latest)

# 2. 設定を作る (初回とランタイムの切り替え時)。保存先は ~/.izanagi/settings/<hash>/izanagi.toml (共有範囲外)
bash scripts/sandbox init         # 切り替え時は init --force

# 3. 起動して確認
bash scripts/sandbox up
bash scripts/sandbox check        # claude --version と /workspace の書き込み確認

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
  `jpn_vert` は Alpine にパッケージが無いので、`jpn` と同じ tessdata 4.1.0 の版を SHA256 で固定して取得しています。
- [ax](https://github.com/yusukebe/ax) は glibc 向けの Bun バイナリしか配布されていないため入れていません。
  HTML の本文は imozuru が標準ライブラリで取り出します (見出しや表の形は ax より崩れやすい)。
- imozuru の注意のとおり、問いの内容は検索語として外部に送られます。

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

## Izanagi 0.2.6 の不具合への回避策

起票済みの Issue ごとに、次の回避策を入れています。Izanagi 側で修正されたら外せます。

- [#115](https://github.com/ieee0824/izanagi/issues/115) `image/build.sh` が Apple `container` で agent の取り出しに失敗する
  → `build-base` は `<runtime> build --output type=local` で agent をビルドし、そのバイナリを `build.sh` に渡しています。
- [#116](https://github.com/ieee0824/izanagi/issues/116) タグ付きのイメージ名 (`name:tag`) を検出できない
  → 設定はタグなしの `imozuru-space-sandbox` (= `:latest`) を指定しています。`build` は
  `imozuru-space-sandbox:claude-<版>` を作ってから `:latest` を付け直し、`up` は両者が同じイメージか確認してから起動します。
- [#117](https://github.com/ieee0824/izanagi/issues/117) PID 1 の agent が孤児プロセスを回収せず、ゾンビがたまる
  → イメージの ENTRYPOINT で `tini` を PID 1 にしています。

## バージョンの更新

| 対象 | 変更する場所 |
|---|---|
| Izanagi | `scripts/sandbox` の `IZANAGI_VERSION`・`IZANAGI_COMMIT`、`Dockerfile` の `IZANAGI_VERSION`、ホストの `izanagi` |
| Claude Code | `scripts/sandbox` の `CLAUDE_CODE_VERSION`、`Dockerfile` の `CLAUDE_CODE_VERSION` |
| imozuru | `Dockerfile` の `IMOZURU_REF`・`IMOZURU_COMMIT` |

変更後に `build-base` (Izanagi のとき) → `build` → `init --force` を実行してください。
Claude Code はイメージ内で自動更新しません (`DISABLE_AUTOUPDATER=1`)。
