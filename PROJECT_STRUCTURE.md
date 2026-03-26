# 项目结构

## 核心文件夹和文件

```
dev-test/
├── .github/
│   └── workflows/
│       └── test-token-generation.yml        # GitHub Actions 测试 workflow
├── scripts/                                  # GitHub App Token 生成脚本
│   ├── get-team.sh                          # 从 team_mapping.yml 获取 team
│   ├── get-repos.sh                         # 使用 gh api 获取 team 的 repositories
│   └── output-token.sh                      # 输出 token 信息
├── action.yml                               # GitHub Actions composite action 定义
├── team_mapping.yml                         # repo 到 team 的映射配置
├── README.md                                # 项目主文档
├── QUICK_START.md                           # 快速开始指南
├── COPILOT_GUIDE.md                         # GitHub Copilot 使用指南
└── .gitignore                               # Git 忽略配置
```

## 文件说明

### 主要配置文件
- **action.yml** - GitHub Actions 复合action，协调所有生成token的步骤
- **team_mapping.yml** - 将 repository 映射到负责team，格式为 YAML

### 脚本文件 (scripts/)
- **get-team.sh** - 解析 YAML 配置，返回 repo 对应的 team
- **get-repos.sh** - 通过 GitHub API 获取 team 拥有的 write 及以上权限的 repositories
- **output-token.sh** - 格式化输出 token 和相关信息

### 文档文件
- **README.md** - 完整的项目文档和使用说明
- **QUICK_START.md** - 快速上手指南，包括配置步骤
- **COPILOT_GUIDE.md** - 使用 GitHub Copilot 快速生成 workflow 和 action 的指南

## 文件清理说明

已移除以下冗余文件：
- `get-team-repos.sh` ❌ (与 scripts/ 中的脚本重复)
- `GRAPHQL_UPDATE.md` ❌ (文档已合并至 README.md)
- `REFACTORING.md` ❌ (历史文档，不再需要)
- `get_repo.sh` ❌ (旧的独立脚本，功能已整合至 action)

## 快速导航

- 🚀 **快速开始**: 查看 [QUICK_START.md](QUICK_START.md)
- 📖 **完整文档**: 查看 [README.md](README.md)
- 🤖 **Copilot 指南**: 查看 [COPILOT_GUIDE.md](COPILOT_GUIDE.md)
- ⚙️ **Action 定义**: 查看 [action.yml](action.yml)
- 🔧 **脚本实现**: 查看 [scripts/](scripts/)
