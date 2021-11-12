# see https://qiita.com/CopyAndPaste/items/6c04950d9fe57c6cfe76
FROM node:14

ENV NODE_PATH /opt/node_modules

WORKDIR /workspace

RUN apt-get -y update && apt-get install -y --no-install-recommends \
        git \
    && apt-get -y clean

RUN npm init --yes \
    && npm install -g zenn-cli@latest \
    && npx zenn init
