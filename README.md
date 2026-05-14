# clean-c-drive

一个 [Claude Code](https://claude.ai/code) Skill，系统化清理 Windows C 盘空间。

## 触发词

在 Claude Code 中说以下任意内容即可激活：

> "清理C盘" "C盘满了" "C盘空间不足" "释放C盘空间" "C盘爆了" "C盘红了"  
> "clean c drive" "free up c drive" "disk full"

## 工作方式

Skill 按五个阶段顺序执行，**每步操作前都会列出候选项、说明用途与风险，等你明确确认才执行**，不会擅自删除任何内容。

| 阶段 | 内容 | 风险 |
|------|------|------|
| 0 | 现状评估：扫描 C 盘各目录大小 | 无 |
| 1 | 安全清理：临时文件、npm/Go/Chrome 缓存、回收站 | 低，缓存可自动重建 |
| 2 | 迁移到 D 盘：将开发工具缓存目录永久改写到 D 盘 | 低，不删数据只改路径 |
| 3 | Windows 自带工具：系统更新残留、Windows.old 等系统保护路径 | 低，引导使用 cleanmgr |
| 4 | 卸载程序：逐个确认，支持 MSI 静默卸载和提权脚本 | 中，操作不可逆 |
| 5 | 按时间清理大型应用数据：微信/钉钉/飞书等媒体文件 | 高，逐路径确认 |

## 安装

将 `SKILL.md` 放到 Claude Code 的 skills 目录（`~/.claude/skills/`），重启 Claude Code 即可。

```
~/.claude/skills/
└── clean-c-drive/
    └── SKILL.md
```

## 特性

- **先量再删**：每次清理前统计大小，让你看清楚再决定
- **分批确认**：先无风险项，再高风险项；明确说"清第1、3项"才执行对应操作
- **每轮核对**：报告本轮释放、累计释放、当前剩余空间
- **已知坑规避**：内置实战经验，如 `$WINDOWS.~BT` 不走 PowerShell、Chrome 只删缓存目录不碰书签/密码、微信数据可能在 D 盘等
