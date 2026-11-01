# FPF Dev Prep Coordinator (`fpf-dev-prep-coordinator`)

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![FPF Compliance](https://img.shields.io/badge/FPF-Separation%20of%20Powers-green.svg)](https://github.com/huangzuomin/five-powers-flow)

**FPF 初始化开发准备与跨工位协调技能 (Agent Skill)**，用于在任何新项目或子系统中快速建立符合 **FPF（Five-Powers-Flow 五权分立）** 治理规范的开发与独立审查工位。

---

## 核心能力

1. **工作令指纹求真**：自动提取并计算 Issue WORK_ORDER 的标准化不可变 SHA-256，绑定固定基线 Commit SHA。
2. **外部 ChatGPT 独立专业线程编排**：全景探测宿主机通信通道，自主拉起隔离的 DEV（架构总指挥）与 QA（独立审查总指挥）无污染会话。
3. **三层执行环境纳管**：
   - **L1 本地离线模拟工位**：基于 Linux/WSL2 账户隔离（UID 1001, 700 权限, umask 0077, 无 sudo）。
   - **L2 物理专属备用环境**：快速勘测与复核局域网独立虚拟机（如 TrueNAS VM），作为物理级容灾。
   - **L3 远端 CI/CD 硬化沙箱**：GitHub Actions 3-Job 硬化架构（不可变 Controller 校验、零 Token 零网络只读 Docker 沙箱、规约驱动裁决器与 Sigstore OIDC 签名）。
4. **不可变制品证据包与 Git Bundle 固化**：避免简单 Zip 压缩带来的历史丢失，自动化生成权威 `manifest.json` 与校验和文件。
5. **人类主权授权令（SCOPE_GRANT）收敛**：参数预先收敛，生成即复制即用的 GitHub Issue 评论块，实现合规准入门禁。

---

## 目录结构

```text
fpf-dev-prep-coordinator/
├── SKILL.md                          # 技能规程主文档与操作流程
├── references/
│   └── station-templates.md          # Manifest 生成脚本与硬化版 3-Job GHA 工作流模版
├── LICENSE                           # Apache 2.0 许可证
└── README.md                         # 本说明文件
```

---

## 安装与使用

### 1. Codex 环境安装
将本项目克隆或复制到 Codex 技能目录即可自动发现：
```bash
git clone https://github.com/huangzuomin/fpf-dev-prep-coordinator.git ~/.codex/skills/fpf-dev-prep-coordinator
```

### 2. 触发与执行
当需要为新项目启动 FPF 规范治理开发时，在对话中提及任务背景或直接唤起：
> "为当前项目初始化 FPF 开发与审查工位，拉起 DEV 和 QA 准备。"

Codex 将自动遵循 [SKILL.md](SKILL.md) 定义的六阶段流程推进闭环。

---

## 许可证

本项目基于 [Apache License 2.0](LICENSE) 开源。
