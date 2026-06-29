# wsh-paper · 文献收割 Skill

飞书多维表格「文献与总结」的自动化文献收割器：读取 PDF 论文，按提示词自动填写各字段。

## 新环境一键安装

```bash
# 1. 本 Skill
git clone https://github.com/Evilcurls/wsh-paper-.git ~/.claude/skills/wsh-paper/

# 2. MinerU Skill（PDF 解析后端，必须）
git clone https://github.com/Nebutra/MinerU-Skill.git ~/.claude/skills/mineru/

# 3. lark-cli（飞书 CLI，含 lark-shared + lark-base）
#    安装方式参考飞书官方文档；首次使用需 lark-cli config init

# 4. Windows 编码修复（必须）
echo 'export PYTHONIOENCODING=utf-8' >> ~/.bashrc

# 5. MinerU Token（强烈推荐，否则 >20 页 PDF 无法解析）
#    免费获取：https://mineru.net/
export MINERU_TOKEN="你的token"
```

> **注意：** 本 Skill 依赖 `lark-cli`、`mineru`、`lark-shared`、`lark-base` 四个 skill，以及 `python3` CLI。缺失任何一个都会在运行前提示安装。

## 触发方式

在 Claude Code 中：

| 方式 | 示例 |
|------|------|
| 直接命令 | `/wsh-paper` |
| 表格名 | "帮我处理文献与总结"、"收割文献表格" |
| 动作描述 | "批量处理文献"、"填写文献字段"、"阅读论文并提取信息" |
| 提供链接 | "处理这个 base：https://xxx.larkoffice.com/base/xxx" |

## 工作流程

1. 定位飞书多维表格「文献与总结」→ 获取 base_token / table_id
2. 列出字段 & 定位待处理记录
3. 逐条：下载 PDF → MinerU 转 Markdown → 按提示词提取 → 更新 9 个字段

## 填写的 9 个字段

| 字段 | 说明 |
|------|------|
| 文章标题 | 完整提取论文标题 |
| 在哪下 | 代码/权重/数据集/Demo/基座模型链接 |
| 一句话总结 | 模板化核心问题概括（15-20字/槽位） |
| 结构 | 基础组件 + 核心创新 + 数据流 + 损失函数（`‖` 分隔） |
| 算力开销 | 硬件/时长/参数量/数据规模 |
| 关键指标 | 评测集/对标模型/核心指标（`‖` 分隔） |
| 局限性＆未解决问题 | 作者承认或 AI 推断的局限 |
| 文献日期 | YYYY-MM 或 YYYY 格式 |
| 供标签使用 | 媒体类型 → 内容性质 → 场景/任务 → 方法 → 范式 |

## 权限

飞书应用需开通以下 scope：

- `bitable:app` — 读取和写入多维表格
- `drive:drive` — 下载/上传附件文件

## License

MIT
