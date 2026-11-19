# recipes/ —— 社区组件配方目录

这个目录目前有一个配方：chromium（和预置组件同名同源的社区兜底，见 `recipes/chromium/recipe.json` 里的 notes）。

## 加一个组件

1. 在这里新建一个目录，目录名就是组件 id，例如 `recipes/ffmpeg-full/`。
   id 只能用小写字母、数字和 `-`，以字母或数字开头，最长 40 个字符
   （`^[a-z0-9][a-z0-9-]{0,39}$`，见 `docs/recipe.md`）。
2. 在目录里放 `recipe.json`（必填），按 `docs/recipe.md` 的规范写。
3. 可选：放一个 `skill/` 目录，里面有 `SKILL.md` 时会自动装到
   `usr/share/agycodroid/components/<id>/skill/`。
4. 推到 `main`：CI 会为 `bases.json` 里的每个底座重打这个配方，
   合成社区索引并签名，发布到 `index` Release。
   配方本身不需要你手动打包、签名或上传。

## 目录结构

```text
recipes/
  README.md            ← 本文件（占位说明）
  <id>/                ← 一个组件一个目录，目录名 = 组件 id
    recipe.json        ← 必填：配方（见 docs/recipe.md）
    skill/             ← 可选：SKILL.md 等，自动随组件安装
      SKILL.md
    <其他文件>          ← 可选：recipe.json 里 files 指向的本地文件
```

## 注意

- 不要在这里放打包产物（`.tar.gz`、`component-*.json`）：产物由 CI 生成，
  上传到 `index` Release，不进 git。
- 不要改 `bases.json`：新底座由主仓库 `scripts/publish-devkit.mjs` 的
  market 步骤在发 devkit 时自动加到第一行。
- 配方校验规则见 `docs/recipe.md`；配方写错了 CI 会直接失败，
  不会把坏组件发出去。
