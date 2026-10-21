---
name: markitdown
description: Use the markitdown CLI to convert document files into plain Markdown text. Use in scenarios that call for text-based reading of document files (文本化读取文档文件): quickly extracting readable text from local PDF, Word (.docx), PowerPoint (.pptx), Excel (.xlsx/.xls), HTML, CSV/TSV, JSON, XML, EPUB, images (EXIF metadata), audio (metadata), or ZIP archives; piping document content into the agent's context; or batch-converting files to .md. Prefer a dedicated OCR-capable document parsing tool for scanned PDFs or images that require OCR, long documents needing page/block-level locators and citations, or high-fidelity table/formula extraction.
---

# markitdown

`markitdown` 是微软 MarkItDown 项目的命令行工具，把各种文档文件一次性转换为 Markdown 纯文本。它的定位是**轻量、快速、纯本地的"文档文本化"工具**：把文档内容变成模型可直接阅读的文本，用于阅读、摘要、问答、提取信息或接入 shell 管道。

## 何时使用本 skill

当任务需要**以文本方式读取文档文件内容**时使用，例如：

- 读取 / 查看 / 总结本地 PDF、Word、PowerPoint、Excel、HTML、EPUB、CSV 等文档的内容。
- 把文档内容快速注入对话上下文（转换结果直接输出到 stdout）。
- 批量把一批文档文件转换成 `.md` 文本，供后续检索或处理。
- 在 shell 管道中处理文档（stdin 输入、stdout 输出）。

## 何时改用其他工具

- **扫描版 PDF / 需要 OCR 的图片**：markitdown 离线模式不做 OCR，扫描件会输出空白或乱码 → 改用支持 OCR 的文档解析工具（如 MinerU、GROBID，或 markitdown 自带的 `-d` Azure Document Intelligence 云端模式）。
- **长文档需要按页/块续读、稳定定位引用、复杂表格公式高保真解析** → 改用具备分页/块级定位与结构化输出能力的文档解析工具。
- **创建或编辑 Office 文档**（而非读取文本）：markitdown 只做只读转换，不能写回 Office 格式 → 改用文档创建/编辑类工具。
- **高质量语音转写**：markitdown 仅提取音频元数据（可选转写依赖外部服务）→ 用专用 ASR 工具。

## 环境检查与安装

```bash
markitdown --version    # 确认已安装
```

未安装时（需要用户批准后执行）：

```bash
pipx install 'markitdown[all]'   # 推荐，隔离环境 + 全部格式支持
# 或
pip install 'markitdown[all]'
```

可选 extras：`[pdf]`、`[docx]`、`[pptx]`、`[xlsx]`、`[xls]`、`[outlook]`、`[az-doc-intel]`、`[all]`。只装了最小版时，对应格式会报转换失败，按错误提示补装 extras 即可。

## 基本用法

```bash
markitdown 文档.pdf                  # 转换结果输出到 stdout
markitdown 文档.docx -o 输出.md      # 写入文件
markitdown 文档.pdf > 输出.md        # 重定向写入文件
cat 文档.pdf | markitdown -x .pdf    # 从 stdin 读取，-x 给出扩展名提示
markitdown < 文档.pdf                # stdin 重定向（此时无法探测扩展名，建议加 -x）
```

## 命令行选项

| 选项 | 说明 |
|---|---|
| `-o, --output FILE` | 输出到文件；缺省输出到 stdout |
| `-x, --extension EXT` | 扩展名提示（如 `.pdf`），stdin 输入或文件无扩展名时使用 |
| `-m, --mime-type TYPE` | MIME 类型提示 |
| `-c, --charset CS` | 字符集提示（如 `UTF-8`），纯文本类文件乱码时使用 |
| `-d, --use-docintel` | 改用 Azure Document Intelligence 云端提取（需配合 `-e` 或环境变量 `MARKITDOWN_DOCINTEL_ENDPOINT`） |
| `--use-cu` | 改用 Azure Content Understanding（需 `--cu-endpoint` 或环境变量 `MARKITDOWN_CU_ENDPOINT`） |
| `-p, --use-plugins` | 启用第三方插件；`--list-plugins` 查看已安装插件 |
| `--keep-data-uris` | 保留 base64 等 data URI（默认截断，减小输出体积） |
| `-v, --version` | 显示版本 |

## 支持的输入格式

| 类别 | 格式 | 说明 |
|---|---|---|
| PDF | `.pdf` | 基于 pdfminer 提取文本层；**无 OCR**，扫描件输出为空 |
| Office | `.docx` `.pptx` `.xlsx` `.xls` | docx→标题/段落/表格；pptx→逐页幻灯片文本；xlsx→逐 sheet 的 Markdown 表格 |
| 网页 | `.html` `.htm` | 转 Markdown 正文 |
| 数据/文本 | `.csv` `.tsv` `.json` `.xml` `.txt` `.md` | CSV/TSV 转为 Markdown 表格 |
| 电子书 | `.epub` | 章节文本 |
| 图片 | `.jpg` `.png` 等 | 仅提取 EXIF 元数据，**不做 OCR** |
| 音频 | `.mp3` `.wav` 等 | 提取元数据；转写需外部语音识别服务 |
| 压缩包 | `.zip` | 逐个转换包内文件并拼接输出 |
| 其他 | YouTube URL（字幕）、`.msg` 邮件、RSS | 视安装的 extras 而定 |

## Agent 实战要点

1. **大文件先落盘再分段读**：长文档直接输出到 stdout 会一次性塞满上下文。用 `markitdown file.pdf -o /tmp/out.md` 写盘后，再分段读取。
2. **以退出码判断成败**：stderr 中可能出现无害的警告输出（如 macOS 上的 onnxruntime telemetry 提示），不代表失败。转换成功退出码为 0；文件不存在、格式不支持、缺 extras 时退出码为 1 并打印 Python traceback。
3. **输出为空要警觉**：PDF 转换结果为空或极少，大概率是扫描件（无文本层）→ 停止重试，改用支持 OCR 的解析方案。
4. **路径含空格/中文要加引号**：`markitdown "~/Downloads/我的 报告.pdf"`。
5. **stdin 必须给类型提示**：管道输入时没有文件名可探测，用 `-x .pdf`（或 `-m application/pdf`）指明。
6. **批量转换示例**：

   ```bash
   for f in *.pdf; do markitdown "$f" -o "${f%.pdf}.md" || echo "FAILED: $f"; done
   ```

7. **默认截断 data URI**：输出中的内嵌图片/base64 默认被截断以控制体积；确需保留时加 `--keep-data-uris`（注意输出体积可能暴增）。
8. **文本乱码**：对 `.txt`/`.csv` 等纯文本类输入，用 `-c GB18030` 等指定正确字符集。

## 与重型文档解析工具的分工

| | markitdown（本 skill） | OCR 文档解析工具（如 MinerU 等） |
|---|---|---|
| 定位 | 轻量通用文本化，一步到位 | 重型文档解析管线 |
| OCR | 无（或借助云端 `-d` 模式） | 通常内置（扫描件、图片） |
| 输出 | 整篇 Markdown 文本流 | 常带页/块定位符的结构化结果，可续读、可引用 |
| 适用 | 快速读取、批量转换、管道处理 | 长文档、扫描件、复杂版面、需要引用出处 |

两者互补：先用 markitdown 快速试探，发现是扫描件、版面复杂或需要精确引用出处时，再切换到重型解析工具。
