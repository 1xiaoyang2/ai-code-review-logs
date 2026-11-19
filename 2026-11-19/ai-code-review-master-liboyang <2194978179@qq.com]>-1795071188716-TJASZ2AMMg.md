# 代码评审：`GitCommand#diff()`

## 一、总体评价

本次改动质量较高，**修复了一个核心功能性 Bug**（用 `git log` 代替 `git diff`，导致 AI 拿到的是提交元信息而非代码变更，AI 根本无法评审），同时在**字符编码、资源管理、进程死锁防御、错误处理**上都有实质性改进。注释"必须使用 git diff，git log 只会返回提交元信息"写清了 Why，是好注释的典范。

但从 SDK 的健壮性角度看，仍有**超时控制、进程生命周期、边界场景（根提交/合并提交）**几处缺口，建议合入前处理 P1 问题。

---

## 二、本次改动做对了什么 ✅

| # | 改动 | 价值 |
|---|---|---|
| 1 | `git log` → `git diff A^ A` | **核心 Bug 修复**，AI 评审的前提是拿到 patch |
| 2 | 显式 `StandardCharsets.UTF_8` | 修复 Windows 默认 GBK 读取导致的中文注释乱码（这类项目 diff 里中文注释极常见） |
| 3 | try-with-resources | 修复原有 Reader 泄漏（原代码异常路径不会 close） |
| 4 | `redirectErrorStream(true)` | 防御经典死锁：stderr 缓冲区（约 64KB）写满后子进程阻塞、`waitFor()` 永不返回 |
| 5 | **先读尽输出、再 `waitFor()`** 的顺序 | 顺序正确。大 diff 场景若先 waitFor 后读流，stdout 缓冲区写满同样死锁 |
| 6 | 退出码校验 + 空结果告警 + 异常信息携带 output | 可观测性和排障效率明显提升 |
| 7 | `diffProgress` → `diffProcess`、`trim()` 哈希 | 细节到位 |

---

## 三、问题与风险

### 🔴 P1-1：外部进程调用无超时，git 挂起将阻塞整条链路

两处 `process.waitFor()` 均为无限等待。git 在凭证弹窗（credential helper）、hook 脚本卡死、大仓库慢 IO 等场景可能长时间不退出——在 GitHub Actions 中会占住 runner 直到 job 超时（默认可达 6 小时）。

**建议**：`waitFor(timeout, TimeUnit.SECONDS)` + 超时后 `destroyForcibly()`。

### 🔴 P1-2：异常/中断路径进程泄漏，中断状态未恢复

- 读流抛 `IOException` 或线程被中断时，两个 `Process` 均未销毁，产生僵尸进程；
- 作为 SDK 类库，捕获 `InterruptedException` 后应恢复中断标志（`Thread.currentThread().interrupt()`），否则上层无法感知中断。

### 🟡 P2-1：根提交（仓库首次提交）场景直接崩溃

`git diff <root>^ <root>` 中 `A^` 不存在，git 报 `unknown revision`、exit 128 → 抛异常。用户初始化仓库后的第一次 push 必然触发。

**建议**：一条命令同时兼容两种场景：

```bash
git diff-tree --root -p --no-commit-id <hash>
```

### 🟡 P2-2：合并提交场景的评审范围失真，需确认业务语义

`M^` 是第一父提交，`git diff M^ M` 会输出**所有被合并分支的累计变更**（可能横跨多个 PR）。若业务预期是"评审本次 push 的那一个提交"，此处语义不一致。建议结合 `git log --merges` 识别 merge commit 并单独决策。

### 🟡 P2-3：第一个 git 进程的 stderr 被丢弃，且哈希未校验格式

- `git log -1` 失败时（非 git 仓库/空仓库），错误细节在 stderr，当前异常只有 exit code，排障困难；
- 当前仅校验"非空"。注意：若日后给它也开 `redirectErrorStream(true)`，`fatal: not a git repository` 这类错误文本会被当作哈希继续下传。

**建议**：对结果做格式校验，一石二鸟：

```java
if (!latestCommitHash.matches("[0-9a-fA-F]{40}")) { // SHA-256 仓库为 64 位
    throw new GitCommandException("Invalid commit hash: " + latestCommitHash);
}
```

### 🟢 P3（低优先级）

1. **异常语义**：对外 SDK 抛裸 `RuntimeException` 不利于上游分类处理（重试/告警），建议定义 `GitCommandException`；
2. **大 diff 无上限**：全量载入内存，且下游 AI 接口有 token 上限，格式化提交/锁文件变更会直接撞限。建议采集侧截断（如 2000 行）并附加提示；
3. **stderr 警告混入正文**：`redirectErrorStream(true)` 后，成功场景下 `warning: LF will be replaced by CRLF` 等会混进发给模型的 diff。通常无害，但属于已知取舍，需团队知晓；
4. **重复样板**：两段"启动进程→读流→waitFor→校验"逻辑几乎相同，可提取 `execGit(String... cmd)`；`new File(".")` 硬编码工作目录，建议构造器注入；
5. 日志建议补充 commit hash，便于评审结果回写 GitHub Comment 时串联追溯。

---

## 四、参考重构

```java
private static final long GIT_TIMEOUT_SECONDS = 60;

public String diff() throws IOException, InterruptedException {
    // 1. 获取最新提交哈希
    String latestCommitHash = execGit("git", "log", "-1", "--pretty=format:%H").trim();
    if (!latestCommitHash.matches("[0-9a-fA-F]{40}")) {
        throw new GitCommandException("Invalid commit hash: " + latestCommitHash);
    }
    logger.info("reviewing commit: {}", latestCommitHash);

    // 2. 获取代码变更；--root 兼容仓库首次提交（无父提交）
    String diffContent = execGit("git", "diff-tree", "--root", "-p", "--no-commit-id", latestCommitHash);
    if (diffContent.isEmpty()) {
        logger.warn("git diff result is empty, please check whether the commit contains code changes");
    }
    logger.info("diff content length: {}", diffContent.length());
    return diffContent;
}

/** 执行 git 命令：合并 stderr、限时、异常路径销毁进程、失败抛出带上下文的异常 */
private String execGit(String... command) throws IOException, InterruptedException {
    ProcessBuilder pb = new ProcessBuilder(command);
    pb.directory(new File("."));
    pb.redirectErrorStream(true);
    Process process = pb.start();
    StringBuilder output = new StringBuilder();
    try {
        try (BufferedReader br = new BufferedReader(
                new InputStreamReader(process.getInputStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = br.readLine()) != null) {
                output.append(line).append('\n');
            }
        }
        if (!process.waitFor(GIT_TIMEOUT_SECONDS, TimeUnit.SECONDS)) {
            throw new GitCommandException("Git command timeout: " + String.join(" ", command));
        }
        if (process.exitValue() != 0) {
            throw new GitCommandException("Git command failed, exit code: " + process.exitValue()
                    + ", command: " + String.join(" ", command) + ", output: " + output);
        }
        return output.toString();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // 恢复中断标志
        throw e;
    } finally {
        process.destroyForcibly(); // 已退出的进程调用无害，异常路径防止泄漏
    }
}
```

---

## 五、结论

**核心修复正确且必要，代码质量较原版明显提升，可以合入**；但建议在合入前补齐 P1（超时 + 进程清理/中断处理），并跟进 P2（根提交兼容、merge commit 语义确认、哈希校验），避免 SDK 在边界场景下"首跑即崩"或"挂死 CI"。整体评分：**B+ → 补齐上述项后可达 A**。