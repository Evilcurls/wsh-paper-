# wsh-paper · 文献收割 Skill

飞书多维表格「文献与总结」的自动化文献收割器：读取 PDF 论文，按提示词自动填写各字段。

## 依赖

| 依赖 | 用途 | 安装 |
|------|------|------|
| `lark-cli` | 飞书 Base API 操作 | 飞书官方安装 |
| [MinerU Skill](https://github.com/Nebutra/MinerU-Skill) | PDF → Markdown 转换 | `git clone` 到 `~/.claude/skills/mineru/` |
| `python3` | MinerU 运行环境 | 系统自带或安装 |

## 安装

```bash
git clone https://github.com/<你的GitHub用户名>/wsh-paper-skill.git ~/.claude/skills/wsh-paper/
```

> **注意：** 需要同时安装 MinerU Skill 作为 PDF 解析后端。

## 环境要求（Windows）

在 `~/.bashrc` 中添加：

```bash
export PYTHONIOENCODING=utf-8
```

## 触发方式

在 Claude Code 中说：

- "帮我收割文献表格"
- "处理文献与总结里的论文"
- `/wsh-paper`

或者直接提供 Base 链接：

- "处理这个 base 里的文献：https://xxx.larkoffice.com/base/xxx"

## 工作流程

1. 定位飞书多维表格「文献与总结」→ 获取 base_token / table_id
2. 列出字段 & 待处理记录
3. 逐条处理：下载 PDF → MinerU 转 Markdown → 按提示词提取 → 更新字段

## 填写的字段

| 字段 | 说明 |
|------|------|
| 文章标题 | 完整提取论文标题 |
| 在哪下 | 代码/权重/数据集/Demo/基座模型链接 |
| 一句话总结 | 模板化一句话核心问题概括 |
| 结构 | 模型架构、数据流向、损失函数 |
| 算力开销 | 硬件/时长/参数量/数据规模 |
| 关键指标 | 评测集/对标模型/提升指标 |
| 局限性＆未解决问题 | 作者承认或 AI 推断的局限 |
| 文献日期 | YYYY-MM 或 YYYY 格式 |

## 权限

需要飞书应用开通以下 scope：

- `bitable:app` — 读取和写入多维表格
- `drive:drive` — 下载附件文件

## License

MIT
