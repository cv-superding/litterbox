<p align="center">
  <a href="./README_en.md">English</a> · <strong>简体中文</strong>
</p>

<p align="center">
  <img src="docs/assets/banner.svg" alt="litterbox — 你的 agent 到处拉屎，litterbox 给它一个固定的猫砂盆" width="100%">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-3fb950?style=flat-square" alt="license MIT">
  <img src="https://img.shields.io/badge/type-agent%20skill-2f81f7?style=flat-square" alt="type agent skill">
  <img src="https://img.shields.io/badge/core-one%20SKILL.md-f0883e?style=flat-square" alt="core: one SKILL.md">
  <img src="https://img.shields.io/badge/version-1.4.0-a371f7?style=flat-square" alt="version 1.4.0">
  <img src="https://img.shields.io/badge/PRs-welcome-3fb950?style=flat-square" alt="PRs welcome">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/install-npx%20skills%20add%20cv--superding%2Flitterbox-8957e5?style=flat-square" alt="install">
  <img src="https://img.shields.io/badge/works%20with-Claude%20Code%20%C2%B7%20Codex%20%C2%B7%20Cursor%20%C2%B7%20Gemini%20CLI%20%C2%B7%20OpenCode-111111?style=flat-square" alt="works with">
  <img src="https://img.shields.io/badge/deps-zero-3fb950?style=flat-square" alt="zero deps">
</p>

<p align="center"><em>你的 agent 到处拉屎。Litterbox 给它一个固定的猫砂盆。</em></p>

你肯定见过这个场景：让 AI 干完活，项目里多出一堆 `scratch_test.py`、`debug_dump.txt`、`output_final_v2.json`，散落在根目录、src、tmp 各处。想删？agent 经常**没有删除权限**；自己删？先得考古它今天到底干了什么。

litterbox 是一个工作区卫生技能，教 agent 一套固定流程：

1. **只搬不删** —— 移动几乎总有权限，删除未必。垃圾全部归拢到工作区根目录的 **`-Delete/`**，你扫一眼 manifest 再决定清空。
2. **覆盖前先备份** —— 凡是要改写已有文件，先把原版快照进 **`-Backup/时间戳/原相对路径/`**，旧版本永远找得回来。
3. **每次移动都记账** —— 两个桶里各有一份 `MANIFEST.md`：原路径、去向、原因、失败原因（权限不够的标记"需人工删除"）。
4. **依赖只进项目环境，运行时先问你** —— 装包前先建 `.venv`/用项目 `package.json`；机器缺 Python/Node 这类运行时时，先检测已有版本和冲突，再让你二选一：项目内（推荐）还是全局——检测到冲突就只能项目内。全局装过的，账本里登记卸载命令。
5. **泄密单独盯防** —— debug dump 归档前扫描标 `CONTAINS SECRETS`；每次要推送前再扫一遍暂存区，key/cookie/私钥永远不进仓库历史（已推出去的一律视为泄露，先吊销）。
6. **还原是一行命令** —— manifest 记了原路径，`cp` 回去就行，没有任何锁定。

`-Delete` 和 `-Backup` 开头的短横线让它们在文件管理器里**永远排在最上面**——你第一眼就能看到 agent 这次拉了多少。

## 它强制了什么

| 没有技能 | 装了技能 |
|---|---|
| `rm -rf scratch* 2>/dev/null` 然后宣称"清理完成"（权限失败被静默吞掉） | 只搬不删；搬不动的写进 manifest 标记"需人工删除" |
| 垃圾散落根目录，"以防万一"全留着 | 会话结束即归拢，`-Delete/` 可审查可还原 |
| 直接覆写 `config.yaml`，旧版本原地蒸发 | 改写前先快照到 `-Backup/时间戳/` |
| 看到 `legacy_export.py` 名字可疑就搬走，构建炸了 | 搬前先 grep 引用；被引用的一律不动并在报告里说明 |
| 垃圾桶本身变成第二个垃圾堆 | 每次移动必记 manifest，可审计可还原 |
| `pip install` 直接装进系统环境，无据可查 | 先建 `.venv`/项目清单再装；全局泄漏登记包名 + 卸载命令 |
| 机器缺 Python/Node，闷头装进全局，砸了别人的环境 | 先检测已有版本/版本管理器/冲突，再让你选：项目内（推荐）或全局；有冲突只能项目内 |
| 还原时直接 `cp` 回去，把备份之后的新修改盖掉了 | 还原前先查目标是否变过；变了先快照当前版再还原 |
| 干完活 dev server、测试容器还挂在后台 | 收尾报告"仍在运行"+ 停止命令，你说停才停 |
| move 一半失败，账本照样记"成功" | 移动后验证再写账本；桶永远 gitignore，不进仓库 |
| 开源时 key/cookie 跟着 `git push` 进了历史 | 推送边界扫描暂存区；`.env` 永远 gitignore，示例用 `.env.example` 占位 |
| README/示例里写死 `F:\Code\...` 绝对路径，别人跑不起来 | 可对外文件一律相对路径或占位符；绝对路径只允许存在于机器本地文件 |
| `.venv`/`node_modules` 被当垃圾乱搬乱删 | 可再生目录永不归档：确认 lockfile 存在、报大小、由你决定 |
| debug dump 里的 API key 跟着垃圾进桶 | 归档前扫描密钥，账本标记 `CONTAINS SECRETS` |

完整纪律（分类规则、永不触碰清单、五种反模式的 Before/After）见 [`SKILL.md`](SKILL.md)。

## 安装

```bash
npx skills add cv-superding/litterbox --global
```

Claude Code 2.1.142+ 也可以：

```text
/plugin marketplace add cv-superding/litterbox
/plugin install litterbox@litterbox
```

手动安装：把 `SKILL.md` 复制进 agent 的技能目录。

## 使用

不需要学任何新东西。只要 agent 完成了产生过临时文件的任务、你说"清理一下 / 太乱了"，它就会归拢。也可以显式调用：

```text
/litterbox
把这次任务产生的垃圾归拢一下
```

Shell 小提示：短横线开头的文件夹在命令里要用 `./` 前缀：`mv scratch_test.py ./-Delete/`。

## 还原

```bash
cat ./-Backup/MANIFEST.md          # 找到原路径
cp -r "./-Backup/2026-09-13_1542/src/config.yaml" src/config.yaml
```

## 许可

MIT
