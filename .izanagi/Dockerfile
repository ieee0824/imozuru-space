# imozuru-space 開発用サンドボックス (Izanagi + Apple Container / Docker)
#
# ベースは Izanagi v0.2.7 のソースからビルドした izanagi-vm イメージ
# (Alpine 3.23 をダイジェスト固定 + 同じ版の izanagi-agent)。
# `scripts/sandbox build-base` で作成する。latest は使わずバージョンタグで固定する。
ARG IZANAGI_VERSION=0.2.7
FROM izanagi-vm:${IZANAGI_VERSION}

ARG CLAUDE_CODE_VERSION=2.1.312

# Claude Code の実行に必要なもの
# - libgcc / libstdc++: musl 版ネイティブバイナリの依存
# - ripgrep: Claude Code の検索ツール (USE_BUILTIN_RIPGREP=0 でこちらを使う)
# - git: 差分確認・コミット操作
RUN apk add --no-cache git ripgrep libgcc libstdc++ ca-certificates

# Claude Code をバージョン固定でインストール (認証情報はイメージに含めない)
RUN npm install -g "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
    && npm cache clean --force \
    && claude --version

# Claude Code の設定ディレクトリ。コンテナ破棄 (izanagi down) で消える。
RUN install -d -o izanagi -g izanagi -m 0700 /home/izanagi/.claude

# --- imozuru スキル (https://github.com/MaryCache/imozuru) ---
# スクリプトは Python 3.10+ の標準ライブラリで動く。次のコマンドは入っていれば使われる:
# - pdftotext / pdftoppm (poppler) と tesseract (英・日、縦書き): PDF の本文と OCR
# - yt-dlp: YouTube の字幕
# ax (HTML→markdown) は glibc 向けの Bun バイナリしか配布されていないため入れない
# (imozuru は標準ライブラリでの本文抽出に切り替える)
ARG IMOZURU_REF=v2.2.0
ARG IMOZURU_COMMIT=808407e501bc5a1e596655b0a6d62e1f3b20ba1b
# Alpine に jpn_vert のパッケージが無いので、jpn (= tessdata 4.1.0) と同じ版をハッシュ固定で取る
ARG TESSDATA_JPN_VERT_SHA256=36879c1aa07634302c9dcb2c1ed4f8fb347c705ad7522dd693ee2e9d68028a79

RUN apk add --no-cache python3 poppler-utils tesseract-ocr \
        tesseract-ocr-data-eng tesseract-ocr-data-jpn yt-dlp \
    && curl -fsSL -o /usr/share/tessdata/jpn_vert.traineddata \
        https://github.com/tesseract-ocr/tessdata/raw/4.1.0/jpn_vert.traineddata \
    && echo "${TESSDATA_JPN_VERT_SHA256}  /usr/share/tessdata/jpn_vert.traineddata" | sha256sum -c - \
    && tesseract --list-langs

# README の手動インストール (skills/imozuru/ を ~/.claude/skills/ へコピー) をタグ固定で行う。
# /plugin install だとコンテナ内の ~/.claude に入り、down のたびに消えるためイメージに含める
RUN git clone --quiet --depth 1 --branch "${IMOZURU_REF}" https://github.com/MaryCache/imozuru.git /tmp/imozuru \
    && test "$(git -C /tmp/imozuru rev-parse HEAD)" = "${IMOZURU_COMMIT}" \
    && install -d -o izanagi -g izanagi -m 0700 /home/izanagi/.claude/skills \
    && cp -R /tmp/imozuru/skills/imozuru /home/izanagi/.claude/skills/imozuru \
    && chown -R izanagi:izanagi /home/izanagi/.claude/skills \
    && rm -rf /tmp/imozuru

# imozuru の調査結果の既定の置き場 (~/research) を共有ワークスペースへ向け、down 後も残す
RUN ln -s /workspace/research /home/izanagi/research
