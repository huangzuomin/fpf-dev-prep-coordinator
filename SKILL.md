---
name: fpf-dev-prep-coordinator
description: 快速完成 FPF 自举研发与新项目初始化开发准备：自主拉起并编排外部 ChatGPT DEV 与 QA 独立会话线程、计算并锁定工作令不可变哈希、构建本地账户级隔离模拟工位与物理/CI 执行环境、生成包含 Git Bundle 的不可变制品证据包与 Manifest、推动人类 SCOPE_GRANT 授权收敛并闭环独立 QA Admission 准入。当需要为新项目启动 FPF 规范治理开发、构建 DEV/QA 独立隔离工位、或进行跨线程跨环境初始化开发准备时使用。
---

# FPF 初始化开发准备与跨工位协调规程 (FPF Dev Prep Coordinator)

本规程指导 Codex 协调工位在新项目或自举研发任务启动时，高效、自驱、合规地完成 FPF 治理体系的初始化开发准备。核心目标是建立五权分立约束下的不可变基线、独立专业线程、受控隔离工位与可信准入闭环，杜绝任务在微观交互中停滞。

---

## 一、核心分工与自驱原则 (Anti-Stall & Role Isolation)

1. **协调工位（Codex 调度中枢）职责**：
   - 掌握项目地图、正式工作令与基线 Commit SHA；
   - **允许并应当自主拉起初始 ChatGPT 线程**，编排 DEV 设计与 QA 审查任务，不将跨线程搬运甩给人类；
   - 纳管本地离线模拟工位、物理专属备用机与云端 CI 沙箱；
   - 制作与校验不可变制品证据包；
   - 收集与收敛人类决策参数，推动人类签发正式 `SCOPE_GRANT`；
   - 坚守治理红线：在 QA Admission PASS 与 SCOPE_GRANT 就位前，**严禁启动 Candidate 候选代码编写，严禁创建分支，严禁修改正式项目地图**。
2. **DEV 总指挥（ChatGPT 独立线程 A）**：负责架构设计、工作令审查、切片与验收标准分解、生成 `DEV_HANDOFF`。
3. **QA 总指挥（ChatGPT 独立线程 B）**：负责独立 Admission 审查、反向测试策略制定、合规性独立裁决、签发 `QA_RESULT`。
4. **Scribe（ChatGPT 独立线程 C）**：负责在代码正式合入主干后，根据实际事实独立更新项目地图（`project-map.md`）。

---

## 二、六阶段标准化执行流程

```
[Phase 1: 工作令锁定与基线] ──> [Phase 2: 自主创建外部 ChatGPT 线程]
                                            │
                                            ▼
[Phase 4: 制品包与反脆弱验证] <── [Phase 3: 纳管本地/物理/CI 多层工位]
            │
            ▼
[Phase 5: 人类 SCOPE_GRANT 收敛] ──> [Phase 6: 终审 Admission PASS -> 启动开发]
```

---

### Phase 1: 工作令锁定与基线求真 (Ground Truth & Baseline Lock)

在开始任何调度前，必须对工作令正文与基线状态进行物理固定：

1. **不可变基线锁定**：
   - 读取主干最新 Commit SHA（作为 Controller N，如 `main @ <40-char-sha>`）；
   - 核验当前工作树状态，确认无未提交的脏改动污染基线。
2. **WORK_ORDER 精确 SHA-256 计算**：
   - 提取 Issue 正文中的完整 WORK_ORDER fenced YAML（从 ```yaml 到结束的 ```，包含围栏本身）；
   - 统一换行符为 `LF` (`\n`)，结束围栏后保留单一 `LF`；
   - 计算 UTF-8 字节序列的 SHA-256 哈希，作为后续所有交接、草案与授权令的不可变指纹。

```python
import hashlib, re

def compute_work_order_sha256(issue_body: str) -> str:
    match = re.search(r'(```yaml\s*\nkind:\s*WORK_ORDER[\s\S]*?```)', issue_body)
    if not match:
        raise ValueError("Missing WORK_ORDER fenced YAML")
    block = match.group(1).replace('\r\n', '\n').strip() + '\n'
    return hashlib.sha256(block.encode('utf-8')).hexdigest()
```

---

### Phase 2: 外部 ChatGPT 独立专业线程自主拉起与编排 (Autonomous Thread Setup)

协调工位无需等待人类手工打开网页或复制粘贴，应全景探测并自主拉起专用线程：

1. **通信驱动层选择优先级**：
   - 宿主机 Chrome 扩展 / 原生 CDP 调试端口 (`--remote-debugging-port`)；
   - Playwright 自动化驱动已登录会话；
   - 本地 C2C / Thread Bridge 服务；
   - 若自动化通道暂时中断，生成标准化交接卡片并如实标记 `MANUAL_HANDOFF`。
2. **必须保持三个独立无污染会话 (Clean Session Isolation)**：
   - **DEV 线程**：打开全新 ChatGPT 会话，投喂 DEV 启动令（包含目标、工作令 SHA-256、白名单路径、架构约束）；
   - **QA 线程**：打开另一全新 ChatGPT 会话，投喂 QA 准入指令与反向监察规范（要求其执行反脆弱审查，禁止使用 DEV 自评作为证据）；
   - **Scribe 线程**：待合并后使用，负责更新项目地图。
3. **输入输出卡片化与事件防重派**：
   - 每次跨线程交互生成唯一交接编号（如 `FPF-<TASK_ID>-ADMISSION-01`）；
   - 自动监听 ChatGPT 输出收敛，提取标准 YAML 卡片（`DEV_DESIGN`、`QA_RESULT`、`DEV_HANDOFF`）。

---

### Phase 3: 多层级执行工位与计算环境快速纳管 (Multi-Tier Station Bootstrap)

构建三层互补的执行计算环境，既支持敏捷离线演练，又具备云端最高隔离信任根：

#### 1. L1 本地离线模拟工位 (Local Simulation Staging)
在本地宿主机（如 WSL2 Ubuntu）构建账户级强隔离工位，快速验证单测与文件权限：
- 创建专用低权限账户：`fpf-dev` (UID 1000) 与 `fpf-qa-sim` (UID 1001)；
- 目录权限严格设为 `700`，`umask` 注入为 `0077`；
- 移除 `sudo` 权限并在 `/etc/shadow` 中锁定口令；
- 在所有文档中统一标记为 **`LOCAL_SIMULATION_STAGING`**（离线模拟工位，不作为正式独立 QA 信任根）。

```bash
# 本地工位快速配置命令
sudo useradd -m -u 1001 -s /bin/bash fpf-qa-sim
sudo chmod 700 /home/fpf-qa-sim
sudo usermod -L fpf-qa-sim
echo 'umask 0077' | sudo tee -a /home/fpf-qa-sim/.bashrc /home/fpf-qa-sim/.profile
```

#### 2. L2 物理专属备用环境 (Physical Standby VM / Runner)
当纳管局域网专属服务器（如 TrueNAS SCALE VM、本地专用备用机）：
- 检查真实可用物理内存（核对 `MemAvailable`、free 内存与 ZFS ARC 缓存，避免误判）；
- 确认当前主机无生产业务占用，确认虚拟化支持（VT-x/AMD-V 为 true）；
- 严格定性为“物理专属备用 QA 资源”，为主干 CI 提供物理级容灾。

#### 3. L3 云端 CI/CD 隔离执行沙箱 (GitHub Actions 3-Job Hardened Architecture - 黄金标准)
部署硬化版独立 QA 工作流（`.github/workflows/fpf-independent-qa.yml`），闭合三大经典漏洞：
- **Job 1 (Trusted Controller Admission)**：固定 Controller Commit SHA，校验输入格式，只读打包待测源码，导出受控 `trusted-qa-spec.json` 规约；
- **Job 2 (Hardened Sandbox Execution)**：剥离全部 GitHub Token，拉起 Docker 容器：
  `--network none`、`--user 1001:1001`、`--read-only`、`--cap-drop=ALL`、`--security-opt=no-new-privileges`、`--pids-limit=100`、Candidate 源码只读挂载 (`:ro`)；宿主监控器在容器外部独立记录退出码与日志；
- **Job 3 (Specification-Driven Evaluator)**：依据 Controller 规范独立扫描文件、排查符号链接与路径穿越、精确用正则提取用例执行数并强制校验 `test_count >= spec['expected_min_test_count']` 与 `allow_skipped: false`，自主构建受信清单并调用 Sigstore OIDC 签名。

---

### Phase 4: 不可变制品证据包规范与反脆弱验证 (Immutable Packaging & Real Probes)

QA 审查要求“证据优先、消除假阳性”，必须执行以下实证打包与测试规范：

1. **Git Bundle 固化历史，杜绝简单 Zip 压缩**：
   - 严禁直接打包工作目录（易导致 `.git` 缺失、换行符污染或未跟踪文件遗漏）；
   - 使用 `git bundle create synthetic-fixture.bundle HEAD main` 完整封装备份历史对象。
2. **自动化生成 Manifest 与哈希校验和**：
   - 遍历制品包内全部管理文件，输出 `manifest.json`，包含 `path`、`size_bytes` 与 `sha256`；
   - 独立生成 `manifest.sha256`，确保第三方独立解压后 100% 可复算校验。
3. **消除测试假阳性（真实门禁对抗与环境探测）**：
   - **门禁测试**：严禁使用内存硬编码字典，必须直接动态引用基线 `packages.fpf_policy.protected.is_protected` 与 `pr_gate.classify`，真实验证受保护文件拦截与 `GATE_PR` 分流；
   - **工作树干净检查**：严禁使用 `line.startswith('??')` 过滤未跟踪文件；必须进行对抗性金丝雀文件植入与清理测试，证明未跟踪文件会触发报错；
   - **OIDC Attestation 校验**：实现符合 in-toto Statement v1 与 SLSA Provenance v1 的完整解析器，并注入截断哈希与伪造类型的反向用例，确保拦截生效。

---

### Phase 5: 人类主权授权（SCOPE_GRANT）快速收敛 (Converged Human Scope Grant)

不要向人类抛出充满 `null` 的繁琐草案，协调工位应预先收敛参数，交付即复制即用的决策文本：

1. **参数预收敛建议**：
   - **有效期**：常规切片建议 30 天，自举制度建设建议 90 天 (`timeout_seconds: 7776000`)；
   - **动作集**：全量授予 A1 ~ A7（分支创建、Worktree 建立、受限白名单编辑、本地测试、提交 Commit、推送分支、创建 `GATE_PR`）；
   - **路径澄清**：如存在历史大小写问题（如 `PROJECT-MAP.md` vs `project-map.md`），在评论中一并附带声明澄清。
2. **Issue 评论格式模板**：
   向用户输出标准 Markdown 围栏块，由仓库 Owner 在 Issue 评论区一键发布：

```markdown
# SCOPE_GRANT 正式授权令与路径澄清

本授权令由仓库所有者签发，授予 FPF 研发任务（Issue #<ISSUE_NO>）执行权限。

```yaml
kind: SCOPE_GRANT
repository: <OWNER>/<REPO>
issue_number: <ISSUE_NO>
work_order_sha256: <CALCULATED_SHA256>
task_id: <TASK_ID>
authorizer: <OWNER_LOGIN>
authorized_at: "<ISO_UTC_TIME>"
expires_at: "<ISO_UTC_EXPIRES_TIME>"
baseline_sha: <CONTROLLER_SHA>
target_branch: fpf/<BRANCH_NAME>

allowed_paths:
  - <ALLOWED_PATH_1>
  - <ALLOWED_PATH_2>

forbidden_paths:
  - docs/coordination/project-map.md
  - .github/
  - schemas/

allowed_actions:
  - A1_CREATE_CANDIDATE_BRANCH
  - A2_CREATE_ISOLATED_WORKTREE
  - A3_RESTRICTED_EDIT_ALLOWED_PATHS
  - A4_EXECUTE_LOCAL_TESTS
  - A5_CREATE_DEV_COMMIT
  - A6_PUSH_CANDIDATE_BRANCH
  - A7_CREATE_GATE_PR

budget_limit:
  max_attempts: 10
  timeout_seconds: 7776000
```
```

3. **授权自动核验**：
   使用 `gh issue view <ISSUE_NO> --comments` 核验评论作者是否为仓库 Owner/CODEOWNER，校验其中的 SHA 与动作清单。

---

### Phase 6: 独立准入（Admission PASS）终审与开发工位启动交接

1. **组装终审材料送交 QA 线程**：
   - 已核验的 GitHub `SCOPE_GRANT` 评论链接与内容；
   - 16 项全绿的 `S01-S15-real-probe-results.json` 结果；
   - 最新 `manifest.json` 与 SHA-256 校验和；
   - 硬化版 GHA 3-Job 架构定义。
2. **确认 Admission PASS 裁决**：
   - 监听 QA 线程，获取明确的 `ADMISSION_VERDICT: PASS`；
   - 若返回 `CHANGES_REQUIRED`，执行最小闭环整改工作令并重新提交，严禁绕过 QA。
3. **合法启动阶段 C（开发执行工位）**：
   准入放行后，协调工位正式向 Codex DEV 执行工位下达开发令：
   - 依据动作 A1，基于基线 Controller SHA 创建候选分支 `fpf/...`；
   - 依据动作 A2，创建独立 Git Worktree；
   - 严格在 `allowed_paths` 内实施代码与文档修改。

---

## 三、快速检查清单 (Quick Checklist for New Projects)

- [ ] 基线 Commit SHA 是否固定且不可变？
- [ ] WORK_ORDER SHA-256 是否按 UTF-8/LF 标准算法精确计算？
- [ ] 是否为 DEV 与 QA 分别拉起了全新无污染的独立会话线程？
- [ ] 本地工位是否已配置 Linux 独立账户（700 / umask 0077 / 无 sudo）？
- [ ] 远端 GHA 流水线是否采用了固定 SHA 准入、Docker 无网络沙箱与独立 Job 3 规约裁决？
- [ ] 制品证据包是否采用了 Git Bundle 且附带完整的 `manifest.json`？
- [ ] 测试用例是否直接调用真实基线门禁代码，且无假阳性旁路过滤？
- [ ] 人类 `SCOPE_GRANT` 授权令是否已由 Owner 真实发布在 GitHub Issue 评论区？
- [ ] 协调账本 `ledger.md` 是否全程记录状态变迁与事件 ID？
- [ ] 阶段 C 启动前，是否确认取得了 QA 总指挥的正式 Admission PASS？

---

## 四、配套工程模版参考

详见配套模版库 [station-templates.md](references/station-templates.md)，包含自动化 Manifest 生成脚本与硬化版 3-Job GitHub Actions 完整工作流定义。
