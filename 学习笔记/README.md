# 学习笔记

> 本目录是个人学习 [github/spec-kit](https://github.com/github/spec-kit) 的笔记与心得。
> 所有笔记仅在 `study-note` 分支上维护，`main` 永远与上游保持一致。

## 关于 spec-kit

GitHub 官方的 **规范驱动开发（Spec-Driven Development）** 工具包。
核心理念：先写规范（spec），再让 AI 写代码，代码永远服从规范。

## 仓库结构速览

```
spec-kit/
├── .specify/        # spec-kit 自身的配置与模板
├── bundles/         # 发布包
├── docs/            # 文档（学习入口）
├── examples/        # 示例项目
├── extensions/      # 扩展
├── integrations/    # 与其他工具的集成
├── presets/         # 预设
├── AGENTS.md        # 给 AI Agent 的指令
├── README.md / README.zh-CN.md
└── pyproject.toml   # Python 项目配置
```

## 当前分支

```bash
git branch --show-current   # 应该是 study-note
```

## 常用工作流

### 写完笔记后提交并推送

```bash
git add .
git commit -m "docs: 笔记标题"
git push origin study-note
```

### 拉取原作者更新（先切回 main 操作）

```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

# 把上游的更新带回到 study-note
git checkout study-note
git merge main
```

### 切换回 study-note 继续写笔记

```bash
git checkout study-note
```

## 笔记组织建议

按仓库目录对应建子目录：

```
学习笔记/
├── docs/                  # 对应原仓库 docs/
│   ├── overview.md        # 整体理解
│   ├── workflow.md        # 工作流梳理
│   └── commands.md        # 命令参考
├── examples/              # 对应原仓库 examples/
│   └── walkthrough.md     # 示例演练
├── integrations/          # 对应原仓库 integrations/
│   └── copilot-cli.md     # 集成方式记录
└── README.md              # 本文件
```

每个文件可以放：
- 概念梳理、命令清单、最佳实践
- 自己的练习记录、疑问与解答

## 远程仓库说明

| 名称 | 地址 | 用途 |
|---|---|---|
| `origin` | `https://github.com/asasas-bit/spec-kit.git` | 自己的 fork，推送笔记到这里 |
| `upstream` | `https://github.com/github/spec-kit.git` | 原仓库（GitHub 官方），只读，用于同步更新 |

```bash
git remote -v   # 查看远程配置
```
