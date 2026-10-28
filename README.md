# imozuru-space

[imozuru](https://github.com/MaryCache/imozuru) のスキルを入れた Claude Code を、
[Izanagi](https://github.com/ieee0824/izanagi) のサンドボックスの中で動かすための作業場所です。

Claude Code はホストから隔離したコンテナで動き、ホストの認証情報 (`~/.claude`・キーチェーン・`~/.ssh` など) には触れません。
imozuru で調べた結果は、このディレクトリの `research/` に残ります。

## 必要なもの

- macOS (Apple Silicon)
- [Izanagi](https://github.com/ieee0824/izanagi/releases) 0.2.6
- コンテナランタイム: Docker (Colima など) または Apple `container`
- git

## はじめかた

```bash
export SANDBOX_RUNTIME=docker     # Apple container を使うなら container

scripts/sandbox build-base        # Izanagi のベースイメージ (初回のみ)
scripts/sandbox build             # Claude Code + imozuru のイメージ
scripts/sandbox init              # Izanagi の設定を作成 (初回のみ)
scripts/sandbox up                # 起動
```

## 使いかた

```bash
scripts/sandbox shell             # サンドボックスに入る
```

サンドボックスの中で `claude` を起動し、初回は `/login` でログインします。
表示される URL をホストのブラウザで開き、出てきたコードを貼り付けてください。

あとは、次のように頼むと imozuru が調べ始めます。

```
芋づる式に調べて: 歌ってみたのMIXで、上手い人がどういう順番で処理しているか
```

結果は `research/<話題>/` に保存されます。サンドボックスの中で `~/research/<話題>/` と表示されるのと同じ場所です。
使い終わったら停止します。

```bash
scripts/sandbox down
```

停止するとコンテナは破棄され、Claude Code のログイン情報も消えます。次に起動したときは、もう一度 `/login` してください。

## コマンド一覧

| コマンド | 内容 |
|---|---|
| `scripts/sandbox build-base` | Izanagi v0.2.6 のソースからベースイメージを作る |
| `scripts/sandbox build` | Claude Code と imozuru を入れたイメージを作る |
| `scripts/sandbox init` | Izanagi の設定を作る (`--force` で作り直す) |
| `scripts/sandbox up` / `down` | 起動・停止 |
| `scripts/sandbox shell` | サンドボックス内のシェルに入る |
| `scripts/sandbox exec <cmd>` | サンドボックス内でコマンドを実行する |
| `scripts/sandbox check` | Claude Code とワークスペースへの書き込みを確認する |
| `scripts/sandbox logs` | Izanagi のログを表示する |

## 注意

- このディレクトリはサンドボックスから**書き込み可**でマウントされます。サンドボックス内での変更はホストのファイルにそのまま反映されます。
  `scripts/` や `.izanagi/` はホストで実行されるので、作業後に想定外の変更がないか確認してください。
- サンドボックスから外部への通信は制限されません。また、Izanagi による通信やファイルアクセスの監視も行われません。
- imozuru は問いの内容を検索語として外部に送ります。機密情報が関わる調べものには使わないでください。
- `research/` には集めた原文の写しが入るため、git の管理対象から外しています。再配布しないでください。

構成の詳細、ランタイムごとの違い、Izanagi の不具合への回避策、バージョンの更新方法は
[.izanagi/README.md](.izanagi/README.md) にまとめています。
