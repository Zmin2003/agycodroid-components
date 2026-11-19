# AGENTS.md —— agycodroid-components 市场仓库

社区组件配方仓库 + 市场 CI。App 的组件中心同时拉两份索引：官方索引
（主仓库 `components` Release，只有预置 chromium）和本仓库的社区索引
（`index` Release）。两份索引用同一把 Ed25519 钥匙签名。

## 布局

- `bases.json`：底座列表，第一行最新。**不要手改**——发新 devkit 时主仓库
  `publish-devkit.mjs` 的 market 步骤自动插 tag、推 main。
- `recipes/<id>/recipe.json`：一个组件一个目录，目录名 = id。规范见 `docs/recipe.md`。
- `verified.json`：真机验证记录（格式见 `docs/recipe.md`“真机验证”一节）。
  只有真在真机上装过、selftest 过了的构建才能写进去。
- `tools/`：市场 CI 的脚本（从上游 cccodroid-components 移植，仓库名和包名已改成 fork 的）。
  `lib.mjs`（共用）、`check-recipes.mjs`（配方校验）、`ci-build.mjs`（打包）、
  `build-index.mjs`（合成并签名社区索引）、`sign.mjs`（本地给索引签名）。
- `.github/workflows/build.yml`：市场 CI（工作流名 `build`，主仓库 market 步骤按名等它）。
  `check.yml`：PR 检查（配方格式 + `--dry` 解依赖）。
- `keys/index.pub.pem`：索引验签公钥（和 App 内嵌的是同一把，只读参考，验签不要用它当信任根）。

## 常用命令

```sh
node tools/check-recipes.mjs                        # 校验全部配方格式
node tools/ci-build.mjs --devkit ../agycodroid/devkit --dry   # 解一遍依赖，不打包
node tools/ci-build.mjs --devkit ../agycodroid/devkit --only <id> --no-upload  # 本地打一个组件
```

## 规则

- 仓库里**不放任何二进制**：配方只写“装哪些包”，包由 CI 从 Termux 官方仓库取。
  `fetch` 例外（官方仓库确实没有的上游构件），必须写 URL + 全 sha256 + why。
- 配方改了上游没变时，`revision` 加一才能重打（版本是 `<上游版本>-r<revision>`）。
- `id` 不许叫 `index`（索引的 Release 用它）。
- 签名私钥（`INDEX_SIGNING_KEY`）只放在仓库 secret 里，**绝不进仓库、绝不进 issue/PR/日志**。
  轮换钥匙要主仓库和本仓库一起换（App 内嵌公钥也要跟着换版本发版），不要只换一边。
- 不要手动改 `index` Release 的附件：索引由 CI 合成并签名，手动改的下次 CI 会覆盖，
  签名对不上的索引 App 直接拒收。
- 加配方、改底座、改工具后，推 `main` 前先在本地跑 `node tools/check-recipes.mjs`。
