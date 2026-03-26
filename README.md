# GitHub Repository Owner Analysis & Team Token Generation

这个项目包含两个主要功能：
1. 获取repo所有者及其所有拥有write权限的repo
2. 为GitHub Actions生成团队级别的installation token

## 🚀 快速导航

|  | 资源 | 说明 |
|---|------|------|
| 📖 | [QUICK_START.md](QUICK_START.md) | 快速开始指南（5分钟上手） |
| 🤖 | [COPILOT_COMMANDS_CN.md](COPILOT_COMMANDS_CN.md) | **GitHub Copilot 中文命令**（一键生成 action/workflow） ⭐ |
| 🤖 | [COPILOT_GUIDE.md](COPILOT_GUIDE.md) | GitHub Copilot 英文版使用指南 |
| 📁 | [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) | 项目结构说明 |
| 🧹 | [CLEANUP_SUMMARY.md](CLEANUP_SUMMARY.md) | 项目整理说明 |
| ⚙️ | [action.yml](action.yml) | 完整的 Action 定义 |

## 功能说明

### 1. get_repo.sh - 获取repo和write权限分析
获取当前repo的所有者及其所有拥有write权限的repo，并输出到CSV文件。

#### 前置要求
- bash shell
- `curl` 命令
- `jq` (可选)
- GitHub token

#### 使用方法
```bash
export GITHUB_TOKEN="your_github_token_here"
chmod +x get_repo.sh
./get_repo.sh
```

---

### 2. action.yml - GitHub Actions Token生成
为GitHub Actions工作流生成installation token，该token拥有对团队所有repo的write权限。

#### 输入参数

| 参数 | 说明 | 必需 |
|------|------|-----|
| `app-id` | GitHub App ID | ✓ |
| `private-key` | GitHub App私钥 | ✓ |
| `team-mapping-path` | team_mapping.yml文件路径 | ✗ (默认: team_mapping.yml) |

#### 输出

| 输出 | 说明 |
|------|------|
| `token` | 生成的installation token |
| `repositories` | token有权访问的repositories列表 |
| `team` | 当前repo对应的团队名称 |

#### 用法示例

```yaml
name: Generate Team Token

on: [push, pull_request]

jobs:
  generate-token:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate installation token
        id: token
        uses: ./ # 使用本action
        with:
          app-id: ${{ secrets.GITHUB_APP_ID }}
          private-key: ${{ secrets.GITHUB_APP_PRIVATE_KEY }}
          team-mapping-path: 'team_mapping.yml'
      
      # 使用生成的token
      - name: Use token in subsequent steps
        run: |
          echo "Token: ${{ steps.token.outputs.token }}"
          echo "Team: ${{ steps.token.outputs.team }}"
          echo "Repositories: ${{ steps.token.outputs.repositories }}"
      
      # 示例：使用token进行Git操作
      - name: Push with token
        env:
          GITHUB_TOKEN: ${{ steps.token.outputs.token }}
        run: |
          git config --global user.email "app@github.com"
          git config --global user.name "GitHub App"
          # 执行Git操作
          # git add .
          # git commit -m "Auto commit"
          # git push
      
      # 示例：调用GitHub API
      - name: Call GitHub API with token
        env:
          TOKEN: ${{ steps.token.outputs.token }}
        run: |
          curl -X POST \
            -H "Authorization: token $TOKEN" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/${{ github.repository }}/issues \
            -d '{"title":"Auto-generated issue","body":"This is an auto-generated issue"}'
```

#### 工作流程

1. **读取团队映射**: 从 `team_mapping.yml` 获取当前repo所对应的team
2. **获取团队repositories**: 调用GitHub API获取该team拥有的所有repo
3. **生成token**: 使用 `actions/create-github-app-token` 生成installation token，指定刚获取的repositories和`contents:write`权限

#### Team Mapping文件格式

`team_mapping.yml`：
```yaml
- repo: test-repo
  team: SOE-SRE
- repo: dev-repo
  team: DEV-TEAM
```

#### 权限要求

GitHub App需要具有以下权限：
- **Repository permissions**:
  - Contents: Read & write
  - Metadata: Read
- **Organization permissions**:
  - Members: Read

---

## 文件结构

```
dev-test/
├── .github/workflows/
│   └── test-token-generation.yml    # 完整的测试workflow
├── scripts/                          # 独立的 bash 脚本
│   ├── get-team.sh                  # 从 team_mapping.yml 获取 repo 对应的 team
│   ├── get-repos.sh                 # 使用 gh api 获取 team 的 repositories
│   └── output-token.sh              # 输出 token 信息
├── .gitignore                       # Git 忽略配置  
├── action.yml                       # GitHub Actions action 定义 (调用脚本)
├── get_repo.sh                      # 获取 repo 所有者和 write 权限
├── get-team-repos.sh                # 获取 team 的所有 repo
├── team_mapping.yml                 # repo 到 team 的映射配置
├── QUICK_START.md                   # 快速开始指南
└── README.md                        # 本文件
```

### scripts 目录中的脚本

#### get-team.sh
从 `team_mapping.yml` 获取当前 repo 对应的 team。

**参数：**
- `$1` - team_mapping.yml 文件路径 (默认: team_mapping.yml)
- `$2` - repo 名称

**输出：** team 名称

**用法：**
```bash
scripts/get-team.sh team_mapping.yml my-repo
```

#### get-repos.sh  
使用 `gh api` 命令获取 team 拥有的所有 repositories，**过滤出拥有 write 及以上权限的 repos**。

**参数：**
- `$1` - organization 名称
- `$2` - team 名称 (team-slug)

**输出：** JSON 数组格式的 repo 列表

**权限过滤逻辑：**
- ✓ `permissions.admin == true` - 管理员权限（包含）
- ✓ `permissions.push == true` - 写入权限（包含）
- ✗ 其他情况（不包含）

**依赖：**
- `gh` CLI 工具 (GitHub CLI)
- `jq` 工具（用于 JSON 处理）
- 有效的 GitHub token（通过 GH_TOKEN 环境变量或自动检测）

**特点：**
- 使用 GitHub REST API（offset-based 分页）
- 双重权限检查（admin 或 push）
- 自动处理分页（超过 100 个 repo）
- 自动去重

**用法：**
```bash
export GH_TOKEN="your_github_token"
scripts/get-repos.sh my-org data-team
# 输出: ["repo1","repo2","repo3"]
```

#### output-token.sh
输出 token 生成的摘要信息到日志。

**参数：**
- `$1` - team 名称
- `$2` - repositories JSON 列表
- `$3` - token 长度

**用法：**
```bash
scripts/output-token.sh "SOE-SRE" '["repo1","repo2"]' 30
```

---

## 测试Workflow - test-token-generation.yml

该workflow包含两个job：

### 1. generate-and-test-token
主要的token生成和验证job，包含以下步骤：

- **生成token** - 调用自定义action生成installation token
- **显示信息** - 输出生成的token信息和关联的team/repositories
- **验证有效性** - 调用GitHub API验证token是否有效
- **列出repo** - 展示token有权访问的所有repositories
- **克隆测试** - 尝试使用token克隆repository
- **写入操作** - 测试token的write权限（创建/关闭issue）
- **汇总报告** - 输出测试执行结果

### 2. git-operations
演示如何使用生成的token进行git操作的示例job：
- 配置git用户信息
- 演示token可用于push/pull/clone操作

### 触发条件

- Push to main或develop分支
- Pull Request到main分支
- 手动触发 (workflow_dispatch)

### 运行workflow

1. **自动运行** - 当你push或创建PR时
2. **手动运行** - 在GitHub Actions标签页，选择"Test Team Token Generation"，点击"Run workflow"

### 所需的GitHub Secrets

在仓库Settings → Secrets and variables中添加：
- `GITHUB_APP_ID` - GitHub App的ID
- `GITHUB_APP_PRIVATE_KEY` - GitHub App的私钥

### 查看运行结果

1. 进入仓库的Actions标签页
2. 找到"Test Team Token Generation"workflow
3. 点击最新的运行记录
4. 查看各个step的日志输出
