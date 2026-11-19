# agycodroid-components —— agycodroid 社区组件市场

社区组件（非预置组件）的配方仓库 + 市场 CI。

App 的组件中心同时拉两份索引（`gateway/gw/components.mjs`）：
官方索引在主仓库 `Zmin2003/agycodroid` 的 `components` Release
（只有预置的 chromium），社区索引就在**本仓库**的 `index` Release。
两份索引用同一把 Ed25519 钥匙签名、同一把内嵌公钥验签；
id 撞了以官方为准。

## 目录结构

```text
bases.json                  ← 底座列表：{"bases": [{"tag": "devkit-2026.11.19"}, …]}，
                              第一行是最新的底座。由主仓库 publish-devkit.mjs
                              的 market 步骤在发 devkit 时自动加，不要手改。
recipes/                    ← 社区组件配方，一个组件一个目录（目录名 = 组件 id）
  README.md                 ← 怎么加配方
  <id>/
    recipe.json             ← 配方（规范见 docs/recipe.md）
    skill/                  ← 可选：SKILL.md，随组件安装
docs/
  recipe.md                 ← 配方编写完整规范（对着 devkit.mjs 的 loadRecipe 逐条写）
  index-flow.md             ← 官方索引与社区索引两条链路、签名钥匙、首个
                              components Release 的完整步骤
.github/workflows/build.yml ← 市场 CI（工作流名 build）：push 到 main 后为每个
                              底座重打每个配方，合成社区索引并签名，发到 index Release
.github/workflows/check.yml ← PR 检查：配方格式 + 解依赖（不打包）
tools/                      ← 市场 CI 的脚本：check-recipes（校验）、ci-build（打包）、
                              build-index（合成并签名社区索引）、sign.mjs（本地签名）
verified.json               ← 真机验证记录（格式见 docs/recipe.md）
keys/index.pub.pem          ← 索引验签公钥（参考用，和 App 内嵌的是同一把）
AGENTS.md                   ← 给 agent 的仓库说明
```

## 加一个组件

1. 在 `recipes/` 下新建目录，目录名就是组件 id
   （小写字母、数字、`-`，以字母或数字开头，最长 40 字符）。
2. 写 `recipe.json`（规范见 `docs/recipe.md`），可选加 `skill/` 目录。
3. 推到 `main`：CI 自动为 `bases.json` 里的每个底座重打、合成索引、
   签名、发到 `index` Release。不用手动打包、签名或上传。

本地先过一遍校验（不用等 CI）：

```sh
# 底座锁文件从主仓库 Release 取（tag 换成 bases.json 第一行的）：
gh release download devkit-2026.11.19 -R Zmin2003/agycodroid -p devkit.lock.json -D /tmp/base
node ../agycodroid/devkit/devkit.mjs component --recipe recipes/<id> \
  --target cc.agyroid --base /tmp/base/devkit.lock.json --dry
```

## Secret 设置（维护者做一次）

仓库 Settings → Secrets and variables → Actions → New repository secret：

| secret | 内容 |
| --- | --- |
| `INDEX_SIGNING_KEY` | Ed25519 索引签名私钥（PEM 全文，和主仓库 `devkit-release` 环境里的是**同一把**钥匙） |

- 没有这个 secret，CI 会在第一步直接失败（不会发无签名的索引）。
- 私钥绝不进仓库、绝不进 issue / PR / 日志。轮换钥匙要主仓库和本仓库
  一起换（App 内嵌的公钥也要跟着换版本发版），不要只换一边。

CI 发布 Release 用的 `GITHUB_TOKEN` 是 GitHub 自动给的，
工作流里已写 `permissions: contents: write`，不用另设。

## 和主仓库的关系

- 主仓库：`Zmin2003/agycodroid`——App、工具包、预置组件（chromium）、
  官方索引、发版脚本 `scripts/publish-devkit.mjs`。
- 本仓库：只放配方 + 底座列表 + 市场 CI，不放任何打包产物
  （`.tar.gz`、`component-*.json` 由 CI 生成，上传到 `index` Release，不进 git）。
- 发新版工具包时，主仓库 `publish-devkit.mjs` 的 market 步骤会把新 tag
  插进本仓库 `bases.json` 第一行，推 `main`，等本仓库 CI 成功后再对账；
  chromium 步骤再按新底座打浏览器组件、更新官方索引。
  完整链路见 `docs/index-flow.md`。

## 当前状态

- 配方：chromium（和预置组件同名同源的社区兜底；预置索引里有 chromium 时 App 用预置的那份）。
- 底座：无（`bases.json` 为空，等第一次发 devkit 时 market 步骤加上）。
- `index` Release：由第一次 CI 运行创建，内容是一份空的已签名索引
  `{"schema": 1, "components": {}}`——空着也要发，否则主仓库
  `publish-devkit.mjs` 的 `checkMarket` 下载社区索引会 404。
