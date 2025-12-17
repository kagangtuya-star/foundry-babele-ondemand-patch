# Babele Compendium On-Demand Patch  按需翻译补丁

这是一个针对 Foundry VTT + Babele 的性能优化补丁。它通过**“启动期轻量索引 + 读取期按需加载”**的策略，显著降低了进入世界时的等待时间以及网络和内存的峰值压力。

本方案特别适用于像 **PF2e** 这样拥有海量 Compendium 条目的大型翻译合集包。

## 📋 环境要求

| 组件            | 要求版本      |
| :-------------- | :------------ |
| **Foundry VTT** | v13           |
| **Babele**      | `2.7.5`       |
| **依赖模块**    | `lib-wrapper` |

---

## 💡 核心原理与优势

### 🚫 传统模式的痛点
原版 Babele 在世界启动（`ready` 阶段）时，会扫描翻译目录，并对每一个 Pack 的翻译文件执行全量加载（`fetch(...).json()`）。
当翻译包包含成千上万个条目时（例如几万个法术或物品），这会导致：
1. **世界启动极慢**：需要等待所有巨大的 JSON 文件下载并解析完毕。
2. **内存占用高**：所有翻译数据一次性载入内存。

### ✅ 本方案：两阶段翻译策略
本补丁将翻译加载过程拆分为两个阶段：

1.  **启动期（快速索引）**
    只加载极小的索引文件（`labels.json` 和 `titles.json`），仅用于处理：
    *   侧边栏的包名（Compendium Label）。
    *   索引条目的标题（用于搜索和列表显示）。
    *   包内的文件夹名（Folders）。

2.  **使用期（按需加载）**
    只有当你真正点击并打开某个具体的 Item/Actor 文档时，才会触发网络请求，加载该 Pack 对应的完整翻译文件并应用详细内容的翻译。

### 🚀 你将获得的效果
*   **秒进世界**：不再需要在启动时下载/解析数兆字节的 `<collection>.json`。
*   **体验无损**：Compendium 列表、搜索功能、侧边栏名称依然是完全翻译过的。
*   **资源节约**：仅在需要时消耗内存和带宽。

---

## 🔧 轻量索引生成器

为了实现“启动期轻量索引”，你需要从完整的翻译文件中提取数据并生成 `labels.json` 和 `titles.json`。本项目提供了自动化脚本 `generate-light-index.mjs`。

### 脚本功能

扫描翻译目录，自动生成：

*   **labels.json**：包名映射。
*   **titles.json**：包含 `titles` (条目标题) 和 `folders` (文件夹名) 的轻量映射。

### CLI 使用方法

```bash
node tools/generate-light-index.mjs --input <翻译目录> --include-folders
```

| 参数                | 简写 | 说明                                             |
| :------------------ | :--- | :----------------------------------------------- |
| `--input`           | `-i` | 指定包含 JSON 翻译文件的目录路径                 |
| `--include-folders` |      | 在输出中包含文件夹（Folders）映射                |
| `--compact`         |      | 输出压缩格式的 JSON（无空格换行）                |
| `--no-recursive`    |      | 仅扫描当前目录，不递归子目录                     |
| `--deep`            |      | 递归抽取深层嵌套的 `name` 字段（默认不建议开启） |

## 🛠️ 安装与使用

你可以根据需求选择以下两种方式之一来使用本补丁。

### 方式 A：作为独立模块安装（普通用户/GM）

如果你只是想优化现有的翻译包，可以单独安装此补丁模块。

1.  **下载/安装**
    将 `foundry-babele-ondemand-patch/` 文件夹复制到你的 Foundry 数据目录：
    `Data/modules/foundry-babele-ondemand-patch/`
2.  **启用模块**
    在世界设置中启用以下模块：
    *   `Babele`
    *   `libWrapper`
    *   `Babele On-Demand Patch`
3.  **配置**
    在模块设置中找到 `Babele`（补丁会接管或复用此设置）：
    *   将 **Loading Mode** 设置为 `ondemand`（按需加载），其实默认就是。
    *   保存并刷新页面即可生效。

### 方式 B：集成在翻译包中（翻译包作者）

如果你是翻译包的维护者，建议直接将补丁集成到你的模块中，让用户开箱即用，无需额外安装补丁模块。

1.  **复制脚本**
    将 `foundry-babele-ondemand-patch.js` 放入你的模块目录，例如：
    `your-translation-module/scripts/foundry-babele-ondemand-patch.js`
2.  **更新 `module.json`**
    在你的模块配置中声明依赖并引入脚本：
    ```json
    {
      "relationships": {
        "requires": [
          { "id": "babele", "type": "module" },
          { "id": "lib-wrapper", "type": "module" }
        ]
      },
      "esmodules": [
        "scripts/foundry-babele-ondemand-patch.js",
        "scripts/main.js" 
      ]
    }
    ```
    *(或者，你也可以在你的 `main.js` 中使用 `import "./foundry-babele-ondemand-patch.js";`)*

---

## 🤖 自动化：GitHub Actions

推荐使用 GitHub Actions 在每次推送代码时自动生成索引，免去手动运行脚本的麻烦。

### 1. 准备工作
确保你的仓库中有以下文件结构：
*   `tools/generate-light-index.mjs` (生成脚本)
*   `.github/workflows/generate-light-index.yml` (工作流文件)

### 2. 权限设置
进入仓库 Settings -> Actions -> General -> Workflow permissions，选中 **Read and write permissions**，以便 Actions 可以提交生成的 JSON 文件回仓库。

### 3. 工作流配置范例
将以下内容保存为 `.github/workflows/generate-light-index.yml`。请根据实际情况修改 `INPUT_DIRS` 路径。

```yaml
name: Generate Babele light index

on:
  push:
    branches: [ main ]
  workflow_dispatch: {}

permissions:
  contents: write

concurrency:
  group: generate-light-index-${{ github.ref }}
  cancel-in-progress: true

jobs:
  generate:
    # 避免死循环，不响应机器人自己的提交
    if: github.actor != 'github-actions[bot]'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Generate labels.json / titles.json
        env:
          SCRIPT_PATH: tools/generate-light-index.mjs
          # 修改此处：支持多行，填入你的翻译文件所在目录
          INPUT_DIRS: |
            translations/zh-cn
        run: |
          set -euo pipefail
          if [ ! -f "$SCRIPT_PATH" ]; then
            echo "Error: Script not found at $SCRIPT_PATH"
            exit 1
          fi
          
          # 逐行读取目录并运行生成脚本
          while IFS= read -r dir; do
            [ -z "$dir" ] && continue
            if [ ! -d "$dir" ]; then
              echo "Warning: Input dir not found: $dir"
              continue
            fi
            echo "Processing: $dir"
            node "$SCRIPT_PATH" --input "$dir" --include-folders
          done <<< "$INPUT_DIRS"

      - name: Commit changes
        run: |
          set -euo pipefail
          # 检查是否有文件变动
          if git diff --quiet; then
            echo "No changes detected."
            exit 0
          fi
          
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add -A
          git commit -m "chore: regenerate light index [skip ci]"
          git push
```

---

## 📂 完整集成示例结构

以下是一个集成了补丁和自动化的翻译包（以 `pf2e_compendium_chn` 为例）的推荐目录结构：

```text
pf2e_compendium_chn/
├── module.json                      # 声明依赖 babele, lib-wrapper 和脚本
├── scripts/
│   ├── main.js
│   └── foundry-babele-ondemand-patch.js  # 核心补丁脚本
├── tools/
│   └── generate-light-index.mjs     # 索引生成工具
├── translations/
│   └── zh-cn/                       # 翻译文件目录
│       ├── labels.json              # [自动生成] 包名索引
│       ├── titles.json              # [自动生成] 标题与文件夹索引
│       ├── pf2e.feats-srd.json      # 完整翻译文件
│       └── pf2e.monsters-srd.json   # 完整翻译文件
└── .github/
    └── workflows/
        └── generate-light-index.yml # 自动化工作流
```

### 示例文件内容参考

**translations/zh-cn/titles.json (生成结果示例):**
```json
{
  "pf2e.monsters-srd": {
    "titles": {
      "Brimstone Rat": "硫磺鼠 Brimstone Rat"
    },
    "folders": {
      "Animals": "动物"
    }
  }
}
```
