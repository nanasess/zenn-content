---
title: "GitHub のリリースノートを GitHub Actions で自動生成する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "Git"]
published: true
---
*Zenn の試し書きを兼ねているので後ほどもうちょっとちゃんと書きます*

GitHub のリリースノートを自動生成する機能がリリースされています。

https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes

以下のブログの記事にあるように、通常は **Auto-generated release notes** ボタンを使用します。
https://docs.github.com/ja/repositories/releasing-projects-on-github/automatically-generated-release-notes

この機能は GitHub API からでも利用可能です。
GitHub Actions から利用したい場合は、この API を使用します。

https://docs.github.com/en/rest/reference/repos#generate-release-notes-content-for-a-release

## サンプルコード

以下、 EC-CUBE2系の Weekly-build で自動生成している例です

```yaml
name: Auto-generated release note
on:
  schedule:
    - cron: '0 15 * * 2'
  release:
    types: [ published ]
jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-18.04
    steps:
      # 直前のリリースタグを取得する
      - name: PREVIOUS_TAG_NAME
        run: |
          echo "PREVIOUS_TAG_NAME=$(curl -H 'Accept: application/vnd.github.v3+json' https://api.github.com/repos/${{ github.repository }}/releases/latest | jq -r .tag_name)" >> $GITHUB_ENV
      # 直前のリリースタグを取得する
      - name: TAG_NAME for schedule
        if: github.event_name == 'schedule'
        run: echo "TAG_NAME=eccube2-weekly-$(date +%Y%m%d)" >> $GITHUB_ENV
      - name: TAG_NAME for release
        if: github.event_name == 'release'
        env:
          TAG_NAME: ${{ github.event.release.tag_name }}
        run: echo "TAG_NAME=${TAG_NAME}" >> $GITHUB_ENV
      # Weekly-build のリリースを生成する
      - name: Create Release
        if: github.event_name == 'schedule'
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # This token is provided by Actions, you do not need to create your own token
        with:
          tag_name: ${{ env.TAG_NAME }}
          release_name: ${{ env.TAG_NAME }}
          body: |
            EC-CUBE 2.17系の Weekly build🚀 です。毎週の改善内容を反映しております。
            常に安定して動作するよう努めていますが、思わぬ不具合を取り込んでしまっている場合もあります。十分に検証の上ご利用ください。
            <table>
            <thead><tr><th>File</th><th>Checksum(sha256)</th></tr></thead>
            <tbody>
            <tr><td><a href="https://github.com/${{ github.repository }}/releases/download/${{ env.TAG_NAME }}/${{ env.TAG_NAME }}.tar.gz">${{ env.TAG_NAME }}.tar.gz</a></td><td><a href="https://github.com/${{ github.repository }}/releases/download/${{ env.TAG_NAME }}/${{ env.TAG_NAME }}.tar.gz.checksum.sha256">${{ env.TAG_NAME }}.tar.gz.checksum.sha256</a></td></tr>
            <tr><td><a href="https://github.com/${{ github.repository }}/releases/download/${{ env.TAG_NAME }}/${{ env.TAG_NAME }}.zip">${{ env.TAG_NAME }}.zip</a></td><td><a href="https://github.com/${{ github.repository }}/releases/download/${{ env.TAG_NAME }}/${{ env.TAG_NAME }}.zip.checksum.sha256">${{ env.TAG_NAME }}.zip.checksum.sha256</a></td></tr>
            </tbody>
            </table>
          draft: false
          prerelease: true
      # 定型文のリリースノートを環境変数に保持しておく
      - name: RELEASE_BODY
        if: github.event_name == 'schedule'
        env:
          TAG_NAME: ${{ env.TAG_NAME }}
        run: |
          echo 'RELEASE_BODY<<EOF' >> $GITHUB_ENV
          echo $(curl -H 'Accept: application/vnd.github.v3+json'  https://api.github.com/repos/${{ github.repository }}/releases/tags/${{ env.TAG_NAME }} | jq -r .body | sed 's,",\\",g' | sed "s,',,g") >> $GITHUB_ENV
          echo 'EOF' >> $GITHUB_ENV
      # RELEASE_ID を環境変数に保持しておく
      - name: RELEASE_ID
        if: github.event_name == 'schedule'
        env:
          TAG_NAME: ${{ env.TAG_NAME }}
        run: |
          echo "RELEASE_ID=$(curl -H 'Accept: application/vnd.github.v3+json'  https://api.github.com/repos/${{ github.repository }}/releases/tags/${{ env.TAG_NAME }} | jq -r .id)" >> $GITHUB_ENV
      # リリースノートを自動生成する
      - name: GENERATED_NOTES
        if: github.event_name == 'schedule'
        run: |
          echo 'GENERATED_NOTES<<EOF' >> $GITHUB_ENV
          echo $(curl -X POST -H 'Accept: application/vnd.github.v3+json' -H 'authorization: Bearer ${{ secrets.GITHUB_TOKEN }}'  https://api.github.com/repos/${{ github.repository }}/releases/generate-notes -d '{"tag_name":"${{ env.TAG_NAME }}", "previous_tag_name":"${{ env.PREVIOUS_TAG_NAME }}"}' | jq .body | sed 's,",,g' | sed "s,',,g") >> $GITHUB_ENV
          echo 'EOF' >> $GITHUB_ENV
      # (ここから)デプロイ処理
      - name: Checkout code
        if: github.event_name == 'release' && (github.event.action == 'published' || github.event.action == 'prereleased' )
        uses: actions/checkout@v2
      - name: Checkout code
        if: github.event_name == 'schedule'
        uses: actions/checkout@v2
        with:
          ref: ${{ env.TAG_NAME }}

      - name: Install to Composer
        run: composer install --no-scripts --no-dev --no-interaction --optimize-autoloader

      - name: Packaging
        working-directory: ../
        env:
          TAG_NAME: ${{ env.TAG_NAME }}
          REPOSITORY_NAME: ${{ github.repository }}
        run: |
          echo $TAG_NAME
          echo "remove obsolete files..."
          rm -rf $GITHUB_WORKSPACE/.git
          rm -rf $GITHUB_WORKSPACE/.gitignore
          rm -rf $GITHUB_WORKSPACE/.github
          rm -rf $GITHUB_WORKSPACE/.editorconfig
          rm -rf $GITHUB_WORKSPACE/.php_cs.dist
          rm -rf $GITHUB_WORKSPACE/phpunit.xml.dist
          rm -rf $GITHUB_WORKSPACE/phpstan.neon.dist
          rm -rf $GITHUB_WORKSPACE/app.json
          rm -rf $GITHUB_WORKSPACE/Procfile
          rm -rf $GITHUB_WORKSPACE/build.xml
          rm -rf $GITHUB_WORKSPACE/README.md
          rm -rf $GITHUB_WORKSPACE/codeception.yml
          rm -rf $GITHUB_WORKSPACE/php.ini
          rm -rf $GITHUB_WORKSPACE/phpinicopy.sh
          rm -rf $GITHUB_WORKSPACE/phpinidel.sh
          rm -rf $GITHUB_WORKSPACE/*.phar
          rm -rf $GITHUB_WORKSPACE/setup.sh
          rm -rf $GITHUB_WORKSPACE/setup_heroku.php
          rm -rf $GITHUB_WORKSPACE/svn_propset.sh
          rm -rf $GITHUB_WORKSPACE/ctests
          rm -rf $GITHUB_WORKSPACE/tests
          rm -rf $GITHUB_WORKSPACE/templates
          rm -rf $GITHUB_WORKSPACE/patches
          rm -rf $GITHUB_WORKSPACE/docs
          rm -rf $GITHUB_WORKSPACE/html/test
          rm -rf $GITHUB_WORKSPACE/dockerbuild
          rm -rf $GITHUB_WORKSPACE/Dockerfile
          rm -rf $GITHUB_WORKSPACE/docker-compose*.yml
          rm -rf $GITHUB_WORKSPACE/zap
          find $GITHUB_WORKSPACE -name "dummy" -print0 | xargs -0 rm -rf
          find $GITHUB_WORKSPACE -name ".git*" -and ! -name ".gitkeep" -print0 | xargs -0 rm -rf
          find $GITHUB_WORKSPACE -name ".git*" -type d -print0 | xargs -0 rm -rf
          echo "set permissions..."
          chmod -R o+w $GITHUB_WORKSPACE/html/install/temp
          chmod -R o+w $GITHUB_WORKSPACE/html/user_data
          chmod -R o+w $GITHUB_WORKSPACE/html/upload
          chmod -R o+w $GITHUB_WORKSPACE/data/cache
          chmod -R o+w $GITHUB_WORKSPACE/data/downloads
          chmod -R o+w $GITHUB_WORKSPACE/data/Smarty
          chmod -R o+w $GITHUB_WORKSPACE/data/class
          chmod -R o+w $GITHUB_WORKSPACE/data/logs
          chmod -R o+w $GITHUB_WORKSPACE/data/upload
          chmod -R o+w $GITHUB_WORKSPACE/data/config
          chmod o+w $GITHUB_WORKSPACE/html
          echo "complession files..."
          pwd
          ls -al
          tar czfp $TAG_NAME.tar.gz ec-cube2
          zip -ry $TAG_NAME.zip ec-cube2 1> /dev/null
          md5sum $TAG_NAME.tar.gz | awk '{ print $1 }' > $TAG_NAME.tar.gz.checksum.md5
          md5sum $TAG_NAME.zip | awk '{ print $1 }' > $TAG_NAME.zip.checksum.md5
          sha1sum $TAG_NAME.tar.gz | awk '{ print $1 }' > $TAG_NAME.tar.gz.checksum.sha1
          sha1sum $TAG_NAME.zip | awk '{ print $1 }' > $TAG_NAME.zip.checksum.sha1
          sha256sum $TAG_NAME.tar.gz | awk '{ print $1 }' > $TAG_NAME.tar.gz.checksum.sha256
          sha256sum $TAG_NAME.zip | awk '{ print $1 }' > $TAG_NAME.zip.checksum.sha256
          ls -al
      - name: Upload binaries to release of TGZ
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.tar.gz
          asset_name: ${{ env.TAG_NAME }}.tar.gz
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of ZIP
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.zip
          asset_name: ${{ env.TAG_NAME }}.zip
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of TGZ md5 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.tar.gz.checksum.md5
          asset_name: ${{ env.TAG_NAME }}.tar.gz.checksum.md5
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of TGZ sha1 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.tar.gz.checksum.sha1
          asset_name: ${{ env.TAG_NAME }}.tar.gz.checksum.sha1
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of TGZ sha256 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.tar.gz.checksum.sha256
          asset_name: ${{ env.TAG_NAME }}.tar.gz.checksum.sha256
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of ZIP md5 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.zip.checksum.md5
          asset_name: ${{ env.TAG_NAME }}.zip.checksum.md5
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of ZIP sha1 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.zip.checksum.sha1
          asset_name: ${{ env.TAG_NAME }}.zip.checksum.sha1
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      - name: Upload binaries to release of ZIP sha256 checksum
        uses: svenstaro/upload-release-action@v1-release
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          file: ${{ runner.workspace }}/${{ env.TAG_NAME }}.zip.checksum.sha256
          asset_name: ${{ env.TAG_NAME }}.zip.checksum.sha256
          tag: ${{ env.TAG_NAME }}
          overwrite: true
      # (ここまで) デプロイ処理
      # リリースノートを更新する
      - name: Update Release notes
        if: github.event_name == 'schedule'
        run: |
          curl \
          -X PATCH \
          -H "Accept: application/vnd.github.v3+json" \
          -H "authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
          https://api.github.com/repos/${{ github.repository }}/releases/${{ env.RELEASE_ID }} \
          -d '{"draft":false, "body":"${{ env.RELEASE_BODY }}\n${{ env.GENERATED_NOTES }}"}'
```

## See Also

https://github.com/EC-CUBE/ec-cube2/pull/494
