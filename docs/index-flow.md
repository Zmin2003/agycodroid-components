# 索引链路：官方索引与社区索引

agycodroid 的组件中心同时拉两份索引（App 内 `gateway/gw/components.mjs`，
`compFetchIndex` + `compMerge`）。两份都必须有 Ed25519 签名（`.sig`），
验不过整份不用；id 撞了以官方（预置）为准。

## 1. 两份索引一览

| | 官方索引（预置） | 社区索引（市场） |
| --- | --- | --- |
| 仓库 | `Zmin2003/agycodroid`（主仓库） | `Zmin2003/agycodroid-components`（本仓库） |
| Release tag | `components` | `index` |
| 附件 | `components.json` + `components.json.sig` + chromium 包（`.tar.gz`） | `components.json` + `components.json.sig` + 各社区组件包（`.tar.gz`） |
| 索引 URL | `https://github.com/Zmin2003/agycodroid/releases/download/components/components.json` | `https://github.com/Zmin2003/agycodroid-components/releases/download/index/components.json` |
| 生成者 | 主仓库 `scripts/publish-devkit.mjs` 的 chromium 步骤 | 本仓库 CI（`.github/workflows/build.yml`） |
| 内容 | 只有 chromium（预置组件） | `recipes/` 下每个配方的构建 |
| 签名钥匙 | 同一把 Ed25519 索引签名钥匙（新钥匙，2026-11-19 换过） | 同一把 |

签名钥匙：私钥只在两处——主仓库发布机（`INDEX_SIGNING_KEY_FILE`，
`bldtools/index-signing-ed25519.pem`）和本仓库的 secret
`INDEX_SIGNING_KEY`（仓库 secret，由维护者添加，见 `README.md`）。
公钥内嵌在 App 里（`gateway/gw/components.mjs` 的
`COMP_INDEX_PUBKEY`，`MCowBQYDK2VwAyEAQJPFI78YJdSCFuHjAsDXBjpEPJYBMH4Ib+elSkcuxSQ=`），
两份索引用同一把公钥验。**私钥绝不进仓库、绝不进种子内容。**

## 2. 索引格式

```json
{
  "schema": 1,
  "components": {
    "<id>": {
      "name": { "zh": "…", "en": "…" },
      "desc": { "zh": "…", "en": "…" },
      "builds": [
        {
          "base": "<底座 gz 的 sha256，全 64 位>",
          "version": "<上游版本-r修订号>",
          "upstream": "<上游版本>",
          "url": "<Release 下载地址>",
          "asset": "<id>-<版本>-for-<底座 sha 前 12 位>.tar.gz",
          "sha256": "<包的 sha256>",
          "bytes": 267805495,
          "installed": 804036608,
          "bin": ["chromium-headless"],
          "skill": "usr/share/agycodroid/components/chromium/skill",
          "selftest": "chromium-headless --dump-dom '…'",
          "packages": 44,
          "baseDeps": ["abseil-cpp", "brotli", "…"]
        }
      ]
    }
  }
}
```

- 一个组件可以有多个 `builds`，按底座（工具包 gz 的 sha256）区分；
  用户装到的是匹配自己那版工具包的构建。
- 字段含义见主仓库 `devkit/components-index.mjs`（合成器）和
  `gateway/tests/fixtures/components-index/components.json`（例子）。
- `verified`（真机验证记录）：索引合成器不写这个字段；App 里
  `verified: e.source === "preset" || !!(b && b.verified)`——预置组件
  默认已验证，社区组件没写就是“待真机验”，不影响安装，
  只影响组件页上的可信度显示。`publish-devkit.mjs` 的 `checkMarket`
  会数有几家已有验证或继承，其余只提示，不报错。

**注意：** `components-index.mjs` 合成索引时，`name` / `desc`
只从主仓库 `devkit/devkit.json` 的 `components` 表取——这只影响**官方索引**
（预置 chromium 在那张表里登记过，显示正常）。社区索引不用它合成，
用本仓库的 `tools/build-index.mjs`，`name` / `desc` 直接取各配方的
`recipe.json`，所以社区配方显示正常，不需要主仓库那边改什么。

## 3. `publish-devkit.mjs` 各步骤怎么驱动这两条链

主仓库发一版工具包的完整步骤链（`scripts/publish-devkit.mjs`，
`preflight → tag → ci → fetch → pin → market → chromium → mirror → preview → testmerge → check`）：

- **market 步骤**（约 206–249 行）：把新 tag 插到本仓库 `bases.json`
  的第一行（正则 `/("bases": \[\n)/`，所以 `bases.json` 必须是多行写法），
  推到 `main`，然后等本仓库 CI：找一个 `workflowName` 含 `build`
  （不区分大小写）、`headSha` 等于这次提交的 run，等它成功；
  再跑 `checkMarket()`——下载社区索引验签，并核对 `recipes/` 下
  每个配方在新底座（`devkit.lock.json` 里 gz 的 sha256）上都有构建。
  **配方为空时也要发布一份空的已签名索引
  `{"schema": 1, "components": {}}`，否则 `checkMarket` 下载社区索引会 404，
  整个 market 步骤失败。** 这就是本仓库 CI 在配方为空时也要跑完
  “生成空索引 → 签名 → 发到 `index` Release”的原因。
- **chromium 步骤**：跑
  `devkit.mjs component chromium --target cc.agyroid --base <devkit.lock.json>`，
  再用 `devkit/components-index.mjs` 把 `component-chromium.json`
  合进线上的官方索引并重签。**合并前先验线上那份旧索引的签名**
  （验不过就不合并，防止有人改 Release 附件借发布机的钥匙签出“官方”组件）。
  然后先传包、后传索引（索引指向的包一定已经在线上），
  附件都传到主仓库的 `components` Release。
- **check 步骤**（全链对账，不信状态文件，重新从各处取来核）：
  Release 清单验签 + `devkit-pin.json` 对账；市场：`bases.json`
  第一行是新 tag，`checkMarket` 再跑一遍；预置浏览器：官方索引验签，
  且有新底座的 chromium 构建，包在线上且大小一致；镜像（fork 没有
  镜像站，跳过）；预览包（这次没打就不核）。

## 4. 首个 `components` Release 的完整步骤

背景：主仓库的 `components` Release 目前还不存在（官方索引、
chromium 包都还没发过）。第一次要按这个顺序走，**不能先打 chromium**：

1. **发 devkit**：`node scripts/publish-devkit.mjs devkit-2026.11.19`
   跑到 `pin` 为止（`preflight → tag → ci → fetch → pin`）。
   这一步产出新底座的 `devkit.lock.json`（`devkit-<tag>` Release）。
   chromium 包必须按新 devkit 底座打——底座的 gz sha256 就是
   构建的 `base`，旧底座打出来的包新工具包的用户装不上。
2. **market 步骤**：把新 tag 插进本仓库 `bases.json` 第一行并推 `main`。
   本仓库 CI 为新底座重打全部配方（现在只有一个 chromium 配方），
   发布/更新 `index` Release。`checkMarket` 下载社区索引验签通过。
3. **chromium 步骤**：按新底座打 chromium，
   `components-index.mjs` 合成官方索引并签名。
   这一步会先下载线上的旧官方索引来合并；`components` Release 还不存在
   时下载失败就**从空索引起步**，并在上传前自己先建一个空的 `components`
   Release（`publish-devkit.mjs` 里已经这样写了，不用手动建）。
4. **check 步骤**：`node scripts/publish-devkit.mjs devkit-2026.11.19 --check`
   全链对账：市场 bases 第一行、社区索引验签、官方索引验签 +
   新底座 chromium 构建在线，全部通过才算发完。
5. 之后每次发新 devkit，market → chromium → check 按顺序自动走，
   不需要再手动建 Release（`components` 和 `index` Release 都已存在，
   之后都是 `--clobber` 更新附件）。

顺序总表：

```text
发 devkit（tag/ci/fetch/pin）
  → market：bases.json 加 tag → 市场 CI 重打 → index Release（社区索引）
  → chromium：按新底座打 chromium → 合成官方索引 → components Release
  → check：全链对账
```

## 5. 本仓库 CI 的工作（`.github/workflows/build.yml`）

push 到 `main` 或手动触发时（工作流名 `build`，主仓库 market 步骤按名等它）：

1. checkout 本仓库 + checkout 主仓库 `Zmin2003/agycodroid`
  （sparse-checkout 只取 `devkit/`，里面有 `devkit.mjs`；Node 22）。
2. `node tools/check-recipes.mjs` 校验全部配方格式（不过就失败，不发坏索引）。
3. `node tools/ci-build.mjs --devkit app/devkit`：对 `bases.json` 里每个底座 ×
   每个配方：从主仓库 `devkit-<tag>` Release 取 `devkit.lock.json`，跑
   `devkit.mjs component --recipe recipes/<id> --target <锁里的 target> --base <锁文件>`；
   已发布过的（同名资产已在组件 Release 里）跳过；新打出来的包 + 构建清单
   上传到以组件 id 为 tag 的 Release（先传包、后传清单，半截上传不会进索引）。
4. `node tools/build-index.mjs --upload`（带 secret `INDEX_SIGNING_KEY`）：
   收集各组件 Release 里的构建清单，按底座只留最新版本，合并 `verified.json`
   （真机验证记录，含跨底座继承），合成社区索引并签名（本地先自验签），
   上传 `components.json` + `components.json.sig` 到 `index` Release
   （不存在就新建）。配方为空时也照样合成一份空索引签名上传，不会 404。
5. 远程核对不在市场 CI 里做：主仓库 `publish-devkit.mjs` 的 `checkMarket` /
   `check` 会从 Release 上重新下载社区索引验签，并核对每个配方在新底座上
   都有构建。

权限：`permissions: contents: write`（建/更新 Release、上传附件需要；
`GITHUB_TOKEN` 默认只有读）。
