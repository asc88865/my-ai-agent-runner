# GitHub 自托管 Runner 安装指南 (Windows)

## 前置条件

- [ ] Windows 机器（当前：E 盘）
- [ ] GitHub 账号 + 仓库（需管理员权限）
- [ ] 本机管理员权限

---

## 1. 安装 GitHub CLI (`gh`)

### 1.1 下载安装
```powershell
# 用 winget 安装（推荐）
winget install --id GitHub.cli
```

### 1.2 登录
```powershell
gh auth login
# 选择 GitHub.com → HTTPS → 用浏览器登录
```

---

## 2. 下载并配置 Runner

### 2.1 获取注册令牌
```powershell
# 替换 <owner/repo> 为你的仓库
gh api repos/<owner/repo>/actions/runners/registration-token > token.json
# 或使用 PAT（Personal Access Token）
```

### 2.2 创建 Runner 目录
```powershell
mkdir C:\actions-runner
cd C:\actions-runner
```

### 2.3 下载 Runner 包
```powershell
# 下载最新 Windows x64 Runner
Invoke-WebRequest -Uri "https://github.com/actions/runner/releases/latest/download/actions-runner-win-x64-2.317.0.zip" -OutFile "actions-runner-win-x64-2.317.0.zip"
# 验证 SHA256（可选）
# 解压
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("actions-runner-win-x64-2.317.0.zip", ".")
```

### 2.4 注册 Runner
```powershell
# 交互式注册
.\config.cmd --url https://github.com/<owner/repo> --token <REGISTRATION_TOKEN>
```

配置参数说明：
- `--name`：Runner 名称（默认主机名）
- `--labels`：自定义标签（如 `windows,gpu,local-agent`）
- `--work`：工作目录（默认 `_work`）
- `--runnergroup`：Runner 组
- `--replace`：替换已存在的同名 Runner

### 2.5 启动 Runner
```powershell
# 前台运行（先测试）
.\run.cmd

# 后台运行（安装为 Windows 服务）
.\svc.cmd install
# 启动服务
.\svc.cmd start
# 查看状态
.\svc.cmd status
```

---

## 3. 验证 Runner 状态

在浏览器打开仓库：
```
Settings → Actions → Runners
```
应看到你的 Runner 显示为 **Idle** 状态。

---

## 4. 创建测试工作流

在仓库中创建 `.github/workflows/test-runner.yml`：

```yaml
name: Test Self-Hosted Runner
on: [push, workflow_dispatch]

jobs:
  test:
    runs-on: self-hosted  # 使用自托管 Runner
    
    steps:
      - uses: actions/checkout@v4
      
      - name: 系统信息
        run: |
          systeminfo | findstr /B /C:"OS"
          wmic cpu get name
          wmic memorychip get capacity
      
      - name: 测试 AI Agent 能力
        run: |
          # 检查 Python
          python --version
          # 检查 Node.js
          node --version
          # 检查 Docker
          docker --version
```

---

## 5. AI Agent 自动化场景配置

### 5.1 安装 Python + AI 工具包（Runner 本机）
```powershell
# 安装 Python
winget install Python.Python.3.12

# 安装常用 AI 包
pip install openai langchain langchain-community chromadb
pip install selenium playwright         # 浏览器自动化
pip install pandas numpy                # 数据处理
```

### 5.2 创建 AI Agent 工作流模板

```yaml
name: AI Agent Workflow
on:
  workflow_dispatch:
    inputs:
      task:
        description: 'AI 任务描述'
        required: true
        default: '分析最近的 issue'

jobs:
  ai-agent:
    runs-on: self-hosted
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: 运行 AI Agent
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python .github/agents/run_agent.py --task "${{ github.event.inputs.task }}"
      
      - name: Agent 自动创建 PR
        if: success()
        run: |
          gh pr create --title "AI Agent 自动提交" --body "由 AI Agent 自动生成" --base main
```

### 5.3 环境变量 / Secrets 设置

| Secret | 用途 | 获取方式 |
|---|---|---|
| `OPENAI_API_KEY` | 调用 OpenAI API | OpenAI 控制台 |
| `GITHUB_TOKEN` | 操作仓库 API | 自动提供，无需手动设置 |
| `AZURE_OPENAI_KEY` | 调用 Azure OpenAI | Azure 门户 |

在仓库设置中添加：
```
Settings → Secrets and variables → Actions → New repository secret
```

---

## 6. 高级配置

### 6.1 标签策略
```powershell
.\config.cmd --labels windows,gpu,local,ai-agent
```
然后在工作流中可精确匹配：
```yaml
runs-on: [self-hosted, windows, gpu]
```

### 6.2 多 Runner 并行
可在多台机器上注册同一个仓库的 Runner，GitHub 会自动调度。

### 6.3 Runner 更新
```powershell
.\svc.cmd stop
.\run.cmd  # 让 Runner 自己更新
.\svc.cmd start
```

---

## 7. 故障排查

| 问题 | 解决方案 |
|---|---|
| Runner 显示 offline | 检查本机是否运行了 `run.cmd` 或服务 |
| 工作流卡住 pending | Runner 没有匹配的标签；检查 `runs-on:` 配置 |
| 权限不足 | 检查 GitHub Token 是否有 `admin:org` 或 `repo` 权限 |
| Windows 防火墙阻止 | 确认 outbound HTTPS (443) 允许 |
| 解压失败 | 下载完整 ZIP 文件，可能文件损坏 |

---

## 8. 下一步：Reasonix + 自托管 Runner 联动

安装完成后，可以：
1. 将 Reasonix 本身注册为一个 AI Agent 工具
2. 在 Actions 工作流中调用 Reasonix 的命令行接口
3. 通过 webhook 触发 Reasonix 执行自动化任务

详见后续配置。
