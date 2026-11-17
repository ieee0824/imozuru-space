# imozuru-space 開発用サンドボックス (Izanagi + Apple Container / Docker)
#
# Debian 13 (trixie) をダイジェスト固定し、Izanagi v0.2.7 の izanagi-agent を載せる。
# glibc 向けにしか配布されていないツール (cc-sakura など) を動かすため、Alpine ではなく Debian にしている。
#
# agent とエントリポイントは `scripts/sandbox build-base` が Izanagi のソースから作る
# izanagi-vm:0.2.7 (Alpine) から取り出す。agent は静的リンクなので Debian でもそのまま動く。
# Izanagi 自体の Debian 版 (image/Dockerfile.debian) は Homebrew をバージョン固定せずに入れるため使わない。
# latest は使わずバージョンタグで固定する。
ARG IZANAGI_VERSION=0.2.7
FROM izanagi-vm:${IZANAGI_VERSION} AS izanagi

# 更新手順: docker pull debian:trixie-slim && docker inspect --format='{{index .RepoDigests 0}}' debian:trixie-slim
FROM debian:trixie-slim@sha256:a72b4433de1cf7d6bae0c4cee74d0547f58601a643b3375f070c51600d63b6b2

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV LANG=C.UTF-8

# 基本パッケージ
# - bash: Izanagi の対話シェル (izanagi shell は /bin/bash を起動する)
# - git: 差分確認・コミット操作
# - ripgrep: Claude Code の検索ツール (USE_BUILTIN_RIPGREP=0 でこちらを使う)
# - tini: PID 1 (末尾の ENTRYPOINT を参照)
# - procps / less: Alpine では busybox にあった ps・pkill と git のページャ
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        bash ca-certificates curl git less procps ripgrep tini \
    && rm -rf /var/lib/apt/lists/*

# Izanagi のイメージと同じユーザー・ワークスペース (コマンドは izanagi ユーザーで実行される)
RUN useradd -m -d /home/izanagi -s /bin/bash izanagi \
    && install -d -o izanagi -g izanagi /workspace

# izanagi-agent (root で実行) とエントリポイントは Izanagi のイメージのものをそのまま使う
COPY --from=izanagi /usr/local/bin/izanagi-agent /usr/local/bin/izanagi-agent
COPY --from=izanagi /entrypoint.sh /entrypoint.sh

# Node.js (Claude Code は Node 22 以上が必要。Debian の nodejs は 20 なので公式バイナリを入れる)
# ハッシュは https://nodejs.org/dist/v<版>/SHASUMS256.txt の値
ARG NODE_VERSION=24.22.3
ARG NODE_SHA256_ARM64=261af2af250dfd7e9f25cf151d1b2713a0adf8f10a9ad116c4947f68627affe8
ARG NODE_SHA256_AMD64=061ec3e3062115705b56b51fa9cc330ab50d34b6969387d4e4b1a50c6809acc2
RUN case "$(dpkg --print-architecture)" in \
        arm64) node_arch=arm64 sha="${NODE_SHA256_ARM64}" ;; \
        amd64) node_arch=x64 sha="${NODE_SHA256_AMD64}" ;; \
        *) echo "unsupported architecture: $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac \
    && curl -fsSL -o /tmp/node.tar.gz \
        "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${node_arch}.tar.gz" \
    && echo "${sha}  /tmp/node.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/node.tar.gz -C /usr/local --strip-components=1 --no-same-owner \
    && rm -f /tmp/node.tar.gz \
    && node --version && npm --version

ARG CLAUDE_CODE_VERSION=2.1.312

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
# ax (HTML→markdown) は入れていない (imozuru は標準ライブラリでの本文抽出に切り替える)
ARG IMOZURU_REF=v2.2.0
ARG IMOZURU_COMMIT=808407e501bc5a1e596655b0a6d62e1f3b20ba1b
# Debian の yt-dlp は古く YouTube の変更に追従できないため、公式リリースの zipapp をハッシュ固定で入れる
# ハッシュは https://github.com/yt-dlp/yt-dlp/releases/download/<版>/SHA2-256SUMS の yt-dlp の値
ARG YT_DLP_VERSION=2026.10.26
ARG YT_DLP_SHA256=e51e69e89489f199530f13d32a4342da3ddb2cf34842f364603a066db05afbf3

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 poppler-utils tesseract-ocr \
        tesseract-ocr-eng tesseract-ocr-jpn tesseract-ocr-jpn-vert \
    && rm -rf /var/lib/apt/lists/* \
    && tesseract --list-langs \
    && curl -fsSL -o /usr/local/bin/yt-dlp \
        "https://github.com/yt-dlp/yt-dlp/releases/download/${YT_DLP_VERSION}/yt-dlp" \
    && echo "${YT_DLP_SHA256}  /usr/local/bin/yt-dlp" | sha256sum -c - \
    && chmod 0755 /usr/local/bin/yt-dlp \
    && yt-dlp --version

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

# --- cc-sakura クライアント (https://github.com/ieee0824/cc-sakura-cloud-client) ---
# 配布バイナリ (glibc 向け) をリリースの SHA2-256 と同じハッシュで固定して入れる。
# ハッシュは https://github.com/ieee0824/cc-sakura-cloud-client/releases/download/v<版>/SHA256SUMS の値
# 参加トークン・鍵・profile はイメージに含めない (サンドボックス内で auth keygen / auth join する)
ARG CC_SAKURA_VERSION=0.4.0
ARG CC_SAKURA_SHA256_ARM64=97d82dfc9068054d883e9b08e5ae0c524db1c3116aefc17cb7517b29a5b3aba9
ARG CC_SAKURA_SHA256_AMD64=6f1841ea4f51f691f4f7f7fa8856fb4d7a79d8d379a8df075a7f7046f3a052be
RUN case "$(dpkg --print-architecture)" in \
        arm64) target=aarch64-unknown-linux-gnu sha="${CC_SAKURA_SHA256_ARM64}" ;; \
        amd64) target=x86_64-unknown-linux-gnu sha="${CC_SAKURA_SHA256_AMD64}" ;; \
        *) echo "unsupported architecture: $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac \
    && name="cc-sakura-cloud-client-v${CC_SAKURA_VERSION}-${target}" \
    && curl -fsSL -o /tmp/cc-sakura.tar.gz \
        "https://github.com/ieee0824/cc-sakura-cloud-client/releases/download/v${CC_SAKURA_VERSION}/${name}.tar.gz" \
    && echo "${sha}  /tmp/cc-sakura.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/cc-sakura.tar.gz -C /tmp --no-same-owner \
    && install -m 0755 "/tmp/${name}/cc-sakura" /usr/local/bin/cc-sakura \
    && rm -rf /tmp/cc-sakura.tar.gz "/tmp/${name}" \
    && cc-sakura --version

# Claude Code 用の使い方スキル (skills/cc-sakura) をバイナリと同じタグ (コミット固定) から入れる。
# skills/cc-sakura-watch は Codex CLI 用の返信アダプターなので入れない
ARG CC_SAKURA_COMMIT=3194bca693ddd6e718fb199b8a7b10fc8a90a183
RUN git clone --quiet --depth 1 --branch "v${CC_SAKURA_VERSION}" \
        https://github.com/ieee0824/cc-sakura-cloud-client.git /tmp/cc-sakura-src \
    && test "$(git -C /tmp/cc-sakura-src rev-parse HEAD)" = "${CC_SAKURA_COMMIT}" \
    && cp -R /tmp/cc-sakura-src/skills/cc-sakura /home/izanagi/.claude/skills/cc-sakura \
    && chown -R izanagi:izanagi /home/izanagi/.claude/skills/cc-sakura \
    && rm -rf /tmp/cc-sakura-src

EXPOSE 9001

# Izanagi のイメージ (v0.2.7 以降) と同じく、tini を PID 1 にして孤児プロセスを回収させる
# (agent は回収しないので、Claude Code が多数起動するコマンドがゾンビとして残るのを防ぐ)。
ENTRYPOINT ["/usr/bin/tini", "--", "/entrypoint.sh"]
