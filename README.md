# 自动通知应用升级

疑问：是如何来通知用户应用升级

## 官方文档了解情况

tauri 分发打包[官网](https://tauri.app/zh-cn/v1/guides/distribution/updater)

### 步骤1

首先生成私钥和秘钥

![image-20240122164243357](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122164243357.png)

我是`windows`系统

### 步骤2

在tauri.config.json中新增updater配置

重要的两个字段

`endpoints`：从地址中获取更新的内容，判断是否需要更新

`pubkey`:步骤一生成的公钥，带pub后缀的那个

![image-20240122164748610](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122164748610.png)

tips:

endpoints中通常用静态的json文件来配置，例如

![image-20240122165440214](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122165440214.png)

`url`:更新包的url地址

`signature`：.sig文件的内容,每次构建时可能都会改变

### 步骤3

在github中设置环境变量

TAURI_PRIVATE_KEY：步骤一生成的私钥

TAURI_KEY_PASSWORD：步骤一输入了两次的密码



![image-20240122165028410](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122165028410.png)

tips:

在本地打包时，需要配置环境变量

![image-20240122165220738](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122165220738.png)

### 步骤4

从[tauri-action文档](https://github.com/tauri-apps/tauri-action/tree/v0.3/)学习到了一个github打包的例子，如下

```
name: 'publish'

on: pull_request

jobs:
  create-release:
    permissions:
      contents: write
    runs-on: ubuntu-20.04
    outputs:
      release_id: ${{ steps.create-release.outputs.result }}

    steps:
      - uses: actions/checkout@v4
      - name: setup node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: get version
        run: echo "PACKAGE_VERSION=$(node -p "require('./package.json').version")" >> $GITHUB_ENV
      - name: create release
        id: create-release
        uses: actions/github-script@v6
        with:
          script: |
            const { data } = await github.rest.repos.createRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              tag_name: `v${process.env.PACKAGE_VERSION}`,
              name: `Desktop App v${process.env.PACKAGE_VERSION}`,
              body: 'Take a look at the assets to download and install this app.',
              draft: true,
              prerelease: false
            })
            return data.id

  build-tauri:
    needs: create-release
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        platform: [macos-latest, ubuntu-20.04, windows-latest]

    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4
      - name: setup node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: install Rust stable
        uses: dtolnay/rust-toolchain@stable
      - name: install dependencies (ubuntu only)
        if: matrix.platform == 'ubuntu-20.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libgtk-3-dev libwebkit2gtk-4.0-dev libappindicator3-dev librsvg2-dev patchelf
      - name: install frontend dependencies
        run: pnpm install # change this to npm or pnpm depending on which one you use
      - uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          releaseId: ${{ needs.create-release.outputs.release_id }}

  publish-release:
    permissions:
      contents: write
    runs-on: ubuntu-20.04
    needs: [create-release, build-tauri]

    steps:
      - name: publish release
        id: publish-release
        uses: actions/github-script@v6
        env:
          release_id: ${{ needs.create-release.outputs.release_id }}
        with:
          script: |
            github.rest.repos.updateRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              release_id: process.env.release_id,
              draft: false,
              prerelease: false
            })
```

拿到这个模板，github构建就成功了一半，然后呢，我们需要在构建完之后，来生成一个install.json文件来提示更新，这个install.json的地址就是上方的endpoints的地址，具体怎么做的，可以去下方仓库查看



### github页面构建

流水线会在这个分支生成一个install.json文件，从而可以通过这个文件来判断是否更新，如果可以通过例如`https://liboq.github.io/tauri-example/install.json`来访问到，则表示github页面构建成功(需要把我的仓库和用户名替换哦)

![image-20240122171404560](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122171404560.png)



### 最后

经过多次测试，最后终于成功了！自动提示更新成功

![image-20240122165945029](https://pikachu-2022-1305579406.cos.ap-nanjing.myqcloud.com/markdown/image-20240122165945029.png)

### 源码地址

[tauri-example](https://github.com/Liboq/tauri-example/tree/main)欢迎star!

### References

`https://mp.weixin.qq.com/s?__biz=MzIzNjE2NTI3NQ==&mid=2247485470&idx=1&sn=1bc6105add6614312db2b37784b8a3c4&chksm=e8dd49eadfaac0fc38610916c3430f43764eb6fd5e04771365ee277d329a647800561616a90d&scene=178&cur_album_id=2593843659863752704&poc_token=HD4jrmWjMnJHedUvWGODAD_UBWZ8d_Wah68hOd-M`

`https://github.com/tauri-apps/tauri-action/tree/v0.3/`
