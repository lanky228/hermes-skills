---
name: apache-calcite-contribution
description: 当用户想向 Apache Calcite 贡献代码、修复 bug、改进文档或测试覆盖，或在中国网络环境下操作 Calcite 的 GitHub 仓库时使用。
---

# Apache Calcite 开源贡献指南

## Overview

本技能编码了向 Apache Calcite（SQL 解析/优化框架）贡献代码的完整流程：从寻找贡献机会、修改代码、构建验证到提交 PR。包含中国网络环境下的 GitHub 操作方案和从真实 PR 中总结的常见错误。

## 何时使用

- 用户想成为 Apache Calcite 贡献者
- 需要修复 Calcite 的 bug、改进文档或补充测试
- 遇到 Calcite 的 `@SuppressWarnings("JdkObsolete")` 或 TODO 注释想清理
- 在中国网络环境下需要 clone/push Calcite 仓库
- 需要了解 Apache 项目的 JIRA + PR 双轨提交流程

**不适用于**：StarRocks 等非 Calcite 项目；非 Java 的 Apache 项目（构建工具不同）；GitHub Issues 提交（Calcite 用 JIRA）。

## 项目概况

| 维度 | 值 |
|------|-----|
| 仓库 | https://github.com/apache/calcite |
| 语言 | 纯 Java |
| 规模 | ~5140 stars, ~1640 源文件, 测试覆盖率 ~14% |
| Issue 追踪 | **JIRA**（https://issues.apache.org/jira/projects/CALCITE），非 GitHub Issues |
| 默认分支 | `main` |
| 构建工具 | Gradle 8.x（wrapper 自带，无需预装） |
| 静态分析 | Google Error Prone（`JdkObsolete` 等规则）+ Spotless（import 排序） |
| CI | Jenkins (Apache) + SonarCloud + Validate Gradle Wrapper |

### 核心模块

| 模块 | 职责 |
|------|------|
| `core/` | SQL 解析、关系代数、优化器（Volcano/Cascades） |
| `linq4j/` | LINQ-for-Java 库 |
| 适配器模块 | Cassandra、Elasticsearch、Kafka、CSV 等 |

## 寻找贡献机会

### 1. JIRA newbie 标签（推荐入口）

Calcite 在 JIRA 有专门的 `newbie` 标签给新手：

```
https://issues.apache.org/jira/issues/?jql=labels%20%3D%20newbie%20%26%20project%20%3D%20Calcite%20%26%20status%20%3D%20Open
```

**注意**：ASF JIRA 需要账号才能查看详情。公开注册关闭，需通过 https://selfserve.apache.org/jira-account.html 申请。

### 2. TODO 注释扫描

```bash
grep -rn "TODO" core/src/main/java/ --include="*.java" | head -20
```

高价值 TODO 类型：
- `// TODO: replace with Deque` — 过时 API 替换（Error Prone `JdkObsolete` 标记）
- `// TODO: how to support timezones?` — 功能补全
- `// TODO:` + `@SuppressWarnings` — 被压制的技术债

> **背景**：`@SuppressWarnings("JdkObsolete")` + `// TODO: replace with Deque` 成对出现源于历史批量注解（CALCITE-4314），是预留的延迟重构入口，非随机遗漏。遇到这类成对标记即可安全清理。

### 3. 测试覆盖率提升

测试覆盖率仅 14%（1640 源文件 / 236 测试文件）。为工具类添加单元测试是高价值、低风险的贡献方向：

```bash
# 找没有对应测试的源文件
find core/src/main/java -name "*.java" | sed 's|/main/|/test/|;s|\.java|Test.java|' | xargs -I{} sh -c 'test -f {} || echo "MISSING: {}"'
```

## 代码修改规范

### Error Prone `JdkObsolete` 规则

Calcite 使用 Google Error Prone 静态分析。`JdkObsolete` 规则会标记过时的 JDK API。常见替换：

| 过时 API | 推荐替代 | 原因 |
|---------|---------|------|
| `java.util.Stack` | `java.util.ArrayDeque`（声明为 `Deque<E>`） | Stack 继承 Vector，所有方法 synchronized，单线程场景有多余锁开销 |
| `java.util.Vector` | `java.util.ArrayList` | 同上 |
| `java.util.Hashtable` | `java.util.HashMap` | 同上 |

**关键语义陷阱**：`Stack.add(e)` 和 `Stack.push(e)` 都添加到末尾（LIFO 从末尾操作）。但 `ArrayDeque.push(e)` 添加到头部，`ArrayDeque.add(e)` 添加到尾部。替换时必须将 `add()` 改为 `push()` 以保持 LIFO 语义等价。

```java
// ❌ 旧代码
@SuppressWarnings("JdkObsolete")
final Stack<Pattern> stack = new Stack<>();
stack.push(item);
E item = stack.pop();

// ✅ 新代码
final Deque<Pattern> stack = new ArrayDeque<>();
stack.push(item);   // push 到头部
E item = stack.pop(); // 从头部弹出
```

### Import 顺序（Spotless 强制）

Calcite 的 Spotless 插件强制 import 按字母序排列。新增 import 时必须插入到正确位置，否则 checkstyle 失败：

```java
// 正确顺序（字母序）
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;
import java.util.Set;
```

### 性能说明

`Stack` → `ArrayDeque` 的改动理由**不是性能**（JVM 逃逸分析会消除 Stack 的 synchronized 锁，实测微基准 ArrayDeque 可能反而慢 31%），而是：
1. 语义正确性（单线程用非同步容器）
2. Java 官方推荐
3. 消除 Error Prone 警告

> **注意**：`new Stack<>()` 调用 `new Vector()`，`capacityIncrement` 默认为 0，扩容时**翻倍**（与 ArrayDeque 相同），并非 +10 线性增长。两者扩容策略一致，仅初始容量不同（Stack 10 vs ArrayDeque 16），不应以"扩容策略更优"作为改动理由。

## 构建与验证

### 编译

```bash
cd calcite
./gradlew :core:compileJava
```

### Checkstyle / Spotless

```bash
./gradlew :core:spotlessCheck
```

### 运行测试

针对修改的文件选择对应的测试类：

```bash
# 示例：修改了 Pattern.java 和 TopDownRuleDriver.java
./gradlew :core:test --tests "org.apache.calcite.runtime.AutomatonTest"
./gradlew :core:test --tests "org.apache.calcite.runtime.DeterministicAutomatonTest"
./gradlew :core:test --tests "org.apache.calcite.runtime.EnumerablesTest"
./gradlew :core:test --tests "org.apache.calcite.test.JdbcAdapterTest"
```

**必须打印测试报告作为实证**，不能只说"应该通过"：

```
> Task :core:test
DeterministicAutomatonTest > 4 completed, 0 failed
AutomatonTest > 9 completed, 0 failed
EnumerablesTest > 53 completed, 0 failed, 1 skipped
BUILD SUCCESSFUL
```

**间接覆盖策略**：当目标类没有直接单元测试时，找通过功能开关间接调用它的集成测试。例如 `TopDownRuleDriver` 无直接单测，但 `JdbcAdapterTest` 中 `topDownOpt=true` 的用例 + MATCH_RECOGNIZE SQL 集成测试会间接覆盖它。

### Gradle 下载加速（中国网络）

Gradle wrapper 的 zip 下载在国内很慢。使用腾讯云镜像：

```bash
# 下载 gradle-8.x.x-bin.zip
curl -L -o /tmp/gradle.zip https://mirrors.cloud.tencent.com/gradle/gradle-8.14.4-bin.zip
# 解压到 wrapper dists 目录（路径根据实际版本调整）
unzip /tmp/gradle.zip -d ~/.gradle/wrapper/dists/gradle-8.14.4-bin/<hash>/
```

## 提交规范

### 三者标题一致性（强制）

Calcite 要求 **JIRA title = PR title = commit message**，格式：

```
[CALCITE-XXXX] Summary description
```

三者必须完全一致，否则 PR 会被要求修改。

### 创建 JIRA Issue

1. 需要 ASF JIRA 账号（通过 selfserve 申请）
2. Issue Type 选 `Improvement`（代码现代化）或 `Bug`
3. Summary 写英文简述，如：`Replace deprecated java.util.Stack with java.util.ArrayDeque in Pattern and TopDownRuleDriver`
4. 获取编号 `CALCITE-XXXX`

### Git 提交

```bash
git checkout -b CALCITE-XXXX-short-description main
git add -A
git commit -m "[CALCITE-XXXX] Summary description"
```

### PR 创建

PR 目标分支：`main`（默认分支）。不需要手动指定 reviewer — Apache 项目靠社区自发 review。

PR Body 模板：

```markdown
## Jira Link
CALCITE-XXXX (https://issues.apache.org/jira/browse/CALCITE-XXXX)

## Changes Proposed
- 具体改动列表

## Rationale
- 改动理由

## Verification
- Compilation: BUILD SUCCESSFUL
- Tests: XXX completed, 0 failed
```

## 中国网络环境下的 GitHub 操作

### Clone（通过镜像）

```bash
# ghfast.top 镜像（已验证可用）
git clone --depth 1 https://ghfast.top/https://github.com/apache/calcite.git
```

### Push（通过 GitHub API）

`ghfast.top` 镜像只支持读取，不支持带认证的 push。当 `git push` 超时时，使用 GitHub API：

```bash
# 1. 获取 fork 仓库的最新 commit SHA
# 2. 通过 API 更新分支引用（需要 PAT）
curl -X PATCH \
  -H "Authorization: Bearer <PAT>" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<username>/calcite/git/refs/heads/<branch> \
  -d '{"sha":"<commit-sha>","force":true}'
```

### 创建 PR（通过 GitHub API）

```bash
curl -X POST \
  -H "Authorization: Bearer <PAT>" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/apache/calcite/pulls \
  -d '{"title":"[CALCITE-XXXX] Summary","head":"<username>:<branch>","base":"main","body":"..."}'
```

## 自查清单（提交前必过）

- [ ] 读懂 CONTRIBUTING.md 和 PR 模板
- [ ] JIRA issue 已创建，编号正确
- [ ] Commit message 格式 `[CALCITE-XXXX] Summary`
- [ ] JIRA title = PR title = commit message（三者一致）
- [ ] Import 顺序通过 `spotlessCheck`
- [ ] 编译通过（打印 BUILD SUCCESSFUL）
- [ ] 相关测试全部通过（打印测试报告）
- [ ] 语义等价性验证（重构类改动需说明 `add()` → `push()` 之类的语义映射）
- [ ] PR Body 填全（Jira Link / Changes Proposed / Rationale / Verification）
- [ ] 无遗留 `@SuppressWarnings` 或 TODO（如果改动目的是清理它们）
- [ ] CI 全绿（Jenkins + SonarCloud + Validate Gradle Wrapper）

## 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 没跑测试就提交 PR | 被 reviewer 要求补测试，浪费时间 | 本地跑相关测试，打印报告 |
| Commit message 无 `[CALCITE-XXXX]` 前缀 | 不符合规范，被要求修改 | 先创建 JIRA issue，再 commit |
| 没创建 JIRA issue 就提 PR | reviewer 要求先创建 | JIRA 是 Apache 项目的 issue 跟踪系统，必须先有 issue |
| `Stack.add()` 直接改成 `ArrayDeque.add()` | LIFO 语义被破坏（add 加尾部，push 加头部） | 改成 `ArrayDeque.push()` |
| Import 顺序不对 | `spotlessCheck` 失败，CI 红灯 | 按字母序插入新 import |
| 用 GitHub Issues 而非 JIRA | Calcite 不用 GitHub Issues 跟踪 bug | 用 ASF JIRA |
| 直接 `git push` 超时后放弃 | 代码推不上去 | 改用 GitHub API 推送 |
| 改动太大或跨多个无关模块 | Review 困难，周期长 | 一个 PR 只做一个逻辑改动 |

## 社区渠道

- 邮件列表：`dev@calcite.apache.org`
- JIRA：https://issues.apache.org/jira/projects/CALCITE
- GitHub：https://github.com/apache/calcite
