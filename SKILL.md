---
name: wsh-paper
version: 1.0.0
description: "飞书多维表格「文献与总结」文献自动收割：读取PDF论文，按提示词自动填写文章标题、开源资源、一句话总结、模型结构、算力开销、关键指标、局限性、文献日期、分类标签等字段。当用户需要批量处理文献表格、收割论文信息、填写文献字段、处理文献与总结、阅读论文并提取信息、文献条目填写时使用。"
metadata:
  requires:
    bins: ["lark-cli", "python3"]
    skills: ["mineru", "lark-shared", "lark-base"]
  setup:
    win32: "export PYTHONIOENCODING=utf-8"
    all: "export MINERU_TOKEN='...'  # https://mineru.net/"
---

# wsh-paper · 文献收割

> ## 🔧 新环境安装
>
> 在一个全新的环境里使用本 Skill，需要依次安装：
>
> ```bash
> # 1. 本 Skill
> git clone <本skill仓库URL> ~/.claude/skills/wsh-paper/
>
> # 2. MinerU Skill（PDF 解析后端，本 Skill 核心依赖）
> git clone https://github.com/Nebutra/MinerU-Skill.git ~/.claude/skills/mineru/
>
> # 3. lark-cli（飞书 CLI，通常已包含 lark-shared 和 lark-base 两个 skill）
> #     安装方式参考飞书官方文档；首次使用需 lark-cli config init
>
> # 4. （Windows 必须）修复编码问题
> echo 'export PYTHONIOENCODING=utf-8' >> ~/.bashrc
>
> # 5. （强烈推荐）设置 MinerU Token，否则 >20 页 PDF 无法解析
> export MINERU_TOKEN="你的token"   # 免费获取：https://mineru.net/
> ```
>
> ### 运行前自检
>
> ```bash
> which lark-cli          # 飞书 CLI 必须可用
> which python3           # Python 3 必须可用（Windows 上可能需要 ln -sf $(which python) ~/bin/python3）
> echo $MINERU_TOKEN      # 可选，但无 Token 时 >20 页 PDF 会失败
> ls ~/.claude/skills/mineru/scripts/mineru.py   # MinerU 必须存在
> ```
>
> 如果任意依赖缺失，**先引导用户安装**再继续，不要静默跳过。
>
> ### 触发关键词
>
> 当用户提到以下任意表达时，应自动触发本 Skill：
> - **表格名：** 文献与总结、文献表格
> - **动作：** 收割论文、批量处理文献、填写文献字段、阅读论文并提取信息
> - **字段名：** 文章标题、在哪下、一句话总结、结构、算力开销、关键指标、局限性、文献日期、供标签使用
> - **直接命令：** `/wsh-paper`

## 概述

本 Skill 自动化处理飞书多维表格「文献与总结」中的文献条目：读取论文 PDF（从附件或链接下载），使用 MinerU 转换为 Markdown，然后按每个字段的提示词约束填写表格。

## 环境要求

- **Windows 用户必须**在 `~/.bashrc` 中设置 `export PYTHONIOENCODING=utf-8`，否则 MinerU 输出 emoji 时触发 GBK 编码错误
- 如果 `python3` 命令不可用（`which python3` 无结果或指向 Microsoft Store 存根），创建 symlink：`ln -sf $(which python) ~/bin/python3`（需确保 `~/bin` 在 PATH 中）
- MinerU 脚本路径：`~/.claude/skills/mineru/scripts/mineru.py`
- 所有 Python 调用使用 `python3`（已通过 `~/bin/python3 -> python` 链接）

### ⚠️ MinerU Token 须知

MinerU 的**免费 Agent API 限制为 ≤10MB 且 ≤20 页**。超过此限制的 PDF 必须设置 Token：

```bash
export MINERU_TOKEN="你的token"
```

Token 免费获取地址：https://mineru.net/

> **强烈建议所有用户提前设置 `MINERU_TOKEN`**。学术论文 PDF 很多超过 20 页（如 CVPR 论文通常 8-10 页尚可，但学位论文、综述、长文经常超限）。没有 Token 时 MinerU 会报错，导致整条记录无法处理。

## 核心流程

### 第一步：定位 Base 和表格

```bash
# 如果用户提供了 Base URL
lark-cli base +url-resolve --url "<base_url>" --as user

# 如果用户提供了标题关键词
lark-cli base +title-resolve --title "文献与总结" --as user
```

从返回结果中提取：
- `base_token` — 后续所有命令的 `--base-token`
- `table_id` — 定位到「文献」表

### 第二步：确认字段结构

```bash
lark-cli base +field-list --base-token <base_token> --table-id <table_id> --as user
```

确认以下目标字段是否存在（字段名以实际返回为准）：

| 字段用途 | 常见名称（模糊匹配） | 类型 |
|---------|-------------------|------|
| PDF附件 | `文献pdf`、`文献PDF`、`pdf` | attachment |
| 文献链接 | `文献链接`、`链接`、`url` | url / text |
| 文章标题 | `文章标题`、`标题` | text |
| 开源资源 | `在哪下`、`开源`、`代码链接` | text |
| 一句话总结 | `一句话总结`、`总结` | text |
| 模型结构 | `结构`、`模型结构` | text |
| 算力开销 | `算力开销`、`算力` | text |
| 关键指标 | `关键指标`、`指标` | text |
| 局限性 | `局限性＆未解决问题`、`局限性` | text |
| 文献日期 | `文献日期`、`日期` | text |
| 分类标签 | `供标签使用`、`标签`、`分类` | text |

如果某个字段不存在，报告用户并跳过该字段。

### 第三步：定位待处理记录

使用「表格」视图筛选未填写的记录：

```bash
# 列出记录，使用视图筛选（视图名通常为「表格」或默认视图）
lark-cli base +record-list \
  --base-token <base_token> \
  --table-id <table_id> \
  --view-id "表格" \
  --as user \
  --format json
```

**重要：** 逐条处理记录，不要批量处理。每条处理完毕立即更新。

判断一条记录是否需要处理的条件（满足任一）：
- `文章标题` 字段为空
- `一句话总结` 字段为空
- 所有文本字段都为空

如果所有记录都已填写，告知用户并结束。

### 第四步：获取论文 PDF

对于每一条待处理记录：

#### 4a. 优先从附件字段下载

```bash
lark-cli base +record-download-attachment \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --output ./paper_cache/ \
  --as user
```

如果 `文献pdf` 附件字段已有文件，下载到 `./paper_cache/` 目录。

#### 4b. 如果无附件，从链接下载

如果 `文献pdf` 为空但 `文献链接` 字段有值，使用 curl 下载：

```bash
curl -L -o "./paper_cache/paper_<record_id>.pdf" "<文献链接>"
```

> ⚠️ **【严禁跳过】下载完成后，必须立即上传 PDF 到 [文献pdf] 附件字段。**
>
> **这条规则优先级最高。即使后续字段填写失败，上传也不能跳过。**
> 原因：不上传会导致下次仍需重复下载，且浪费附件存储空间。

```bash
lark-cli base +record-upload-attachment \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --field-id "文献pdf" \
  --file "./paper_cache/paper_<record_id>.pdf" \
  --as user
```

**上传后验证：** 检查命令输出中是否返回了 `file_token`，若有则表示上传成功。若上传失败，重试一次；仍失败则报告用户并继续处理后续字段。

> ⚠️ **重要提醒：** 4b 下载 + 上传完成后，该记录的 PDF 已存入附件字段。下次处理同一记录时走 4a 直接从附件下载即可，无需再次从链接下载。

#### 4c. 如果既无附件也无链接

跳过该记录，报告用户。

### 第五步：使用 MinerU 转换 PDF 为 Markdown

**先检查 PDF 页数：**

```bash
PYTHONIOENCODING=utf-8 python3 -c "
import fitz; doc = fitz.open('./paper_cache/<pdf_file>')
print(f'页数: {doc.page_count}')
"
```

- **≤20 页：** 直接使用免费 Agent API
- **>20 页：** 必须检查 `MINERU_TOKEN` 是否已设置

```bash
# 检查 Token 是否存在
echo $MINERU_TOKEN
```

**如果 Token 未设置且 PDF 超过 20 页，立即停止并告知用户：**

> ⚠️ 这篇 PDF 有 XX 页，超过 MinerU 免费 Agent API 的 20 页限制。
> 请到 https://mineru.net/ 免费获取 Token，然后设置：
> `export MINERU_TOKEN="你的token"`
> 设置后告诉我继续。

**转换命令：**

```bash
# 关键：设置 UTF-8 编码
PYTHONIOENCODING=utf-8 python3 ~/.claude/skills/mineru/scripts/mineru.py \
  "./paper_cache/<pdf_file>" \
  --stdout \
  --lang en
```

- `--stdout` 将 Markdown 输出到标准输出，直接读取
- `--lang en` 针对英文论文设置语言（中文论文用 `--lang ch`）
- 对于扫描版 PDF，添加 `--ocr` 参数
- 设置 `MINERU_TOKEN` 后，MinerU 会自动路由到 Standard API（支持 ≤200MB / ≤200 页）

**如果转换失败且报 Token 相关错误：**

> ⚠️ MinerU 转换失败。可能原因：PDF 超过 20 页但未设置 Token，或 Token 无效。
> 请确认 `MINERU_TOKEN` 已正确设置（获取地址：https://mineru.net/）。

**输出即为完整的结构化 Markdown 论文内容**，包含标题、摘要、章节、公式、表格等。

### 第六步：按提示词填写每个字段

阅读 MinerU 输出的 Markdown 内容，严格按照以下每个字段的提示词要求提取信息，并通过 `lark-cli base +record-upsert` 逐字段更新。

**关键规则：**
1. 每条记录逐字段更新，不要一次性写所有字段（避免超长 JSON 转义问题）
2. 每次更新后验证返回结果
3. 严格按照提示词模板输出，不要添加引导语或 Markdown 语法
4. 如果论文中确实未提及某项信息，填"未提及"或"无"

#### 字段 1：文章标题

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"文章标题": "<提取的标题>"}' \
  --as user
```

**提取规则：**
1. 保留标题原文的全部内容，包括主标题、副标题、冒号/破折号后的所有描述
2. 只去除：作者名、期刊名、年份、DOI、卷号页码等元数据
3. 不截断、不缩写、不概括
4. 仅输出标题本身

#### 字段 2：在哪下（开源资源链接）

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"在哪下": "<格式化的开源信息>"}' \
  --as user
```

**严格按以下纯文本模板输出，未提及则填"无"：**

```
代码: [URL或'未开源']
权重: [URL或'未开源']
数据集: [URL或'未开源']
Demo: [URL或'无']
基座模型: [如LLaMA-3-8B，若有链接附上，用分号隔开；若无可填'无']
```

禁止使用 Markdown 语法（如**、##、-等），禁止输出引导语，禁止换行解释。

#### 字段 3：一句话总结

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"一句话总结": "<模板化总结>"}' \
  --as user
```

**要求：每个【】槽位填入 15-20 字以内的短语，只保留最核心的信息，不要堆砌细节。**

**严格使用以下模板填空输出，禁止修改模板外文字，不使用 Markdown，不换行：**

在【现有场景/背景】下，现有的【方法/模型】存在【具体缺陷/痛点】，本文旨在解决【核心问题】以实现【最终目标】。

**示范输出（仅供格式参考，内容与你的论文无关）：**
在自然语言处理模型依赖大量标注数据的背景下，现有的有监督分类模型存在跨域泛化能力差、标注成本高的缺陷，本文旨在解决如何利用无标注文本进行预训练以实现少样本场景下的高效迁移。

#### 字段 4：结构（模型架构）

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"结构": "<架构描述>"}' \
  --as user
```

**要求：**
- 简述论文中的模型架构，剥离数学推导，聚焦数据流向
- **每项严格控制在 20 字以内**
- 按以下格式输出，用 `‖` 分隔各项，**严禁换行，严禁任何 Markdown 符号**
- 不要输出任何前缀引导语

**格式：**
基础组件：【基座/编码器等，≤20字】‖ 核心创新：【最关键的结构设计，≤20字】‖ 数据流：【用"→"串联特征流向，≤20字】‖ 损失函数：【损失类型与计算范围，≤15字；若论文无训练损失则填"N/A"】

**示范输出（仅供格式参考）：**
基础组件：ViT编码器 + 轻量Transformer解码器 ‖ 核心创新：非对称编解码，编码器仅处理25%可见patch ‖ 数据流：图像→切patch→掩码75%→编码→拼接掩码token→解码重建 ‖ 损失函数：仅对掩码patch计算MSE

#### 字段 5：算力开销

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"算力开销": "<算力信息>"}' \
  --as user
```

**严格按以下纯文本格式输出，未提及填"未提及"：**

```
硬件配置: [如8张A100]
训练时长: [如3天]
模型参数量: [如7B或70B]
训练数据规模: [如1T Tokens或10万张图]
```

禁止 Markdown，禁止引导语，禁止换行解释。

#### 字段 6：关键指标

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"关键指标": "<指标总结>"}' \
  --as user
```

**要求：**
- 总结论文的实验结论，提取核心评测信息
- **每项严格控制在 20 字以内，指标项每条不超过 15 字**
- 按以下格式输出，用 `‖` 分隔各项，指标项内部用 `/` 分隔不同任务
- **严禁换行，严禁任何 Markdown 符号**，不要输出任何前缀引导语

**格式：**
核心评测集：【数据集名称，≤20字】‖ 对标模型：【主要基线名称，≤20字】‖ 核心指标：【任务名+指标+结果，每条≤15字，多条用"/"分隔】

**示范输出（仅供格式参考）：**
核心评测集：ImageNet-1K、COCO、ADE20K ‖ 对标模型：MoCo v3、DINO、BEiT ‖ 核心指标：ImageNet微调87.8% top-1 / COCO检测超有监督基线 / ADE20K mIoU 53.6%

#### 字段 7：局限性＆未解决问题

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"局限性＆未解决问题": "<局限性描述>"}' \
  --as user
```

**要求：**
- 提取论文中作者承认的局限性及未来工作
- 若无明确写出，推断1-2个潜在缺陷并加 `[AI推断]` 前缀
- **严格输出格式（仅1-2行，不要引导语，禁止 Markdown 符号）**

#### 字段 8：文献日期

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"文献日期": "<日期>"}' \
  --as user
```

**提取规则（按优先级从高到低执行）：**

1. **最高优先：** 从论文首页或标题下方，查找明确标注的 "Published in"、"Published"、"录用日期"、"Received/Accepted" 等字段，提取年份和月份（如有）
2. **次高优先：** 查找页眉页脚中期刊的卷号、期号、年份信息
3. **若以上都没有：** 从会议信息中提取（如 "Proceedings of AAAI 2025"、"ICCV 2024" 等）
4. **若仅有年份：** 直接输出年份
5. **若完全无法确定：** 从 arXiv 编号中推断年份（如 arXiv:2405.12345 → 2024年）

**输出格式：**
- 如果有完整年月：`YYYY-MM`
- 如果只有年份：`YYYY`
- 如果有多个日期（如 Submitted/Accepted），取 Accepted 日期，若无则取 Published 日期，若无则取 Submitted 日期
- **仅输出日期，不要加任何前缀、后缀、解释**

#### 字段 9：供标签使用（分类标签）

```bash
lark-cli base +record-upsert \
  --base-token <base_token> \
  --table-id <table_id> \
  --record-id <record_id> \
  --json '{"供标签使用": "<标签>"}' \
  --as user
```

**打标规则：**
请严格从以下标签库中按【媒体类型 → 内容性质 → 场景/任务 → 方法 → 范式】的顺序进行选择。

**标签库：**

**【1. 媒体类型标签】（必选1个）**
`视频`, `图像`, `音频/语音`, `三维/点云`, `通用/多媒体`

**【2. 内容性质标签】（可选1个，有明显倾向时选）**
`AIGC生成内容`, `美学`, `用户生成内容（UGC）`, `专业内容（PGC）`

**【3. 场景/任务标签】**
- 失真类型：`压缩失真`, `传输失真`, `采集噪声`, `模糊`, `过曝/欠曝`, `色彩失真`, `伪影`, `丢包`, `混响`, `背景噪声`, `削波`, `编码失真`, `超分辨率伪影`, `几何失真`, `拼接失真`, `生成失真/AIGC伪影`, `闪烁`, `抖动`
- 评价范式：`全参考（FR）`, `部分参考（RR）`, `无参考（NR）`, `主观质量评价`, `客观质量评价`, `实时质量评价`, `个性化质量评价`
- 应用场景：`流媒体`, `视频会议`, `高清/超高清`, `VR/360视频`, `语音增强`, `音频增强`, `盲测`, `直播`, `短视频`, `低延时`, `高动态范围（HDR）`, `数字人/虚拟人/说话人头`, `云游戏/游戏画质`

**【4. 方法标签】**
- 架构类：`CNN`, `Transformer`, `ViT`, `3D-CNN`, `图神经网络`, `RNN/LSTM`, `自编码器`, `生成对抗网络（GAN）`, `状态空间模型（SSM/Mamba）`, `知识蒸馏`, `混合专家模型（MoE）`
- 大模型/生成式：`大语言模型（LLM）`, `多模态大模型（LMM）`, `视觉语言模型（VLM）`, `Video-LLM`, `Diffusion模型`, `CLIP`, `思维链（CoT）`, `Agent`
- 训练/优化范式：`预训练`, `微调`, `提示学习`, `指令微调`, `RLHF`, `强化学习（RL）`, `深度强化学习（DRL）`, `奖励建模`, `交互式优化`, `自监督学习`, `对比学习`, `域适应/跨域泛化`, `零样本学习`, `少样本学习`
- 三维相关：`点云`, `网格`, `三维重建`, `三维渲染`, `NeRF`, `Gaussian Splatting`, `双目/立体视觉`, `深度估计`, `虚拟现实（VR）`, `增强现实（AR）`, `空间感知`
- 传统/信号：`传统信号处理`, `传统机器学习`, `手工特征`

**【5. 范式标签】（可选1个）**
`理论`, `算法`, `系统`, `应用`, `综述`, `数据集/基准`

**输出要求：**
1. 输出 3-6 个标签，按【媒体类型 → 内容性质 → 场景/任务 → 方法 → 范式】的顺序排列
2. 必须包含 1 个【媒体类型标签】和至少 1 个【方法标签】
3. 【内容性质标签】仅在论文明显聚焦于 AIGC、美学、UGC 或 PGC 时添加；若论文不涉及质量评价（如纯生成、编解码、感知类论文），【场景/任务标签】可省略
4. 【范式标签】仅在论文明显偏重某类研究性质时添加
5. 如无完全匹配标签，用 `[新增]` 前缀自行生成
6. **仅输出标签，英文逗号分隔。不要序号、不要解释、不要换行**

### 第七步：处理完一条记录后

**按顺序执行以下检查，每一项都不能跳过：**

1. ⚠️ **PDF 上传检查（最高优先级）：** 如果本次是从 `文献链接` 下载的 PDF，**必须确认**已成功上传到 `[文献pdf]` 附件字段。检查 `+record-upload-attachment` 的返回结果中是否有 `file_token`。若未执行上传，**现在立即补上**。
2. 确认所有 9 个字段都已成功更新
3. 清理临时 PDF 文件（`./paper_cache/` 目录下的对应文件）
4. 继续处理下一条记录
5. 处理完毕或遇到无法处理的记录时，向用户报告进度

## 批量处理建议

- 每次处理 3-5 条记录，避免上下文过长导致质量下降
- 每批处理完毕后主动暂停，向用户报告进度，询问是否继续
- 处理过程中如遇到无法下载的链接或损坏的 PDF，记录并跳过，最后统一报告

## 权限

| 操作 | 所需 scope |
|------|-----------|
| 读取 Base 结构 | `bitable:app` |
| 读取记录 | `bitable:app` |
| 更新记录 | `bitable:app` |
| 下载附件 | `drive:drive` |

所有操作使用 `--as user` 身份。

## 注意事项

- **写入前预览：** 对第一次更新操作先使用 `--dry-run` 确认目标字段和 Base 正确
- **字段名匹配：** 第二步返回的真实字段名可能与模板名称不同，务必使用真实字段名
- **长文本转义：** 字段值包含特殊字符（引号、换行）时，使用 `@file` 方式传入 JSON 或使用 stdin
- **限流处理：** 如遇到 `1254291`（并发写冲突），等待 3 秒后重试
- **MinerU 超时：** PDF 转换约需 10-15 秒，设置足够的超时时间
- **编码问题：** Windows 上务必设置 `PYTHONIOENCODING=utf-8`
