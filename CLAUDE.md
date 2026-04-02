# Claude Skills Collection

Personal Claude Code skills (slash commands) repository.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| logcli | `/logcli` | 查询 K8s 环境 Loki 日志，支持 test/prod/nlt 集群 |
| test-env | `/test-env` | 测试环境中间件配置速查 |
| test-nacos | `/test-nacos` | 连接测试环境 Nacos 查询配置 |
| test-pg | `/test-pg` | 连接测试环境 PostgreSQL |
| test-redis | `/test-redis` | 连接测试环境 Redis |

## Usage

Skills stored in `.claude/commands/` are **project-scoped** — only available when Claude Code runs in this directory.

To make a skill globally available, symlink or copy it to `~/.claude/commands/`:

```bash
# Symlink (recommended — changes auto-sync)
ln -sf ~/Work/claude-skills/.claude/commands/logcli.md ~/.claude/commands/logcli.md

# Or copy
cp .claude/commands/logcli.md ~/.claude/commands/
```

## Adding New Skills

1. Create `<skill-name>.md` in `.claude/commands/`
2. Use `/skill-name` in any Claude Code session (if global) or in this project directory
