---
name: test-smoke-cycle
description: |
  通用冒烟测试循环：build → start → smoke test → stop → clean。
  自动检测项目的构建系统（Makefile、package.json、docker-compose 等），执行完整的冒烟测试流程并在结束后清理环境。
  Use when: 运行冒烟测试、验证项目端到端是否正常、跑 smoke test、测试 API 是否可用、
  验证部署前的基本功能、项目健康检查。即使用户只说"测一下"或"跑个测试看看"，
  只要上下文涉及启动服务并验证功能，就应该触发此技能。
---

# 冒烟测试循环（Smoke Test Cycle）

一个通用的端到端冒烟测试流程：构建项目 → 启动服务 → 执行测试 → 停止服务 → 清理环境。

这个技能的核心价值是**让你不用记住每个项目的测试命令** — 它会自动检测项目类型并走对应的流程。

## 执行流程

```
检测项目类型
  ↓
Phase 1: BUILD — 编译/构建项目
  ↓
Phase 2: START — 后台启动服务
  ↓
Phase 3: VERIFY — 等待服务就绪（health check）
  ↓
Phase 4: TEST — 执行冒烟测试脚本
  ↓
Phase 5: STOP — 停止服务
  ↓
Phase 6: CLEAN — 清理构建产物（可选）
  ↓
报告结果
```

## Phase 0: 检测项目类型

按优先级依次检查，命中第一个即停止：

| 优先级 | 检测文件 | 项目类型 | 构建系统 |
|---|---|---|---|
| 1 | `Makefile` + `go.mod` | Go 项目（Makefile 驱动） | make |
| 2 | `Makefile` | 通用 Makefile 项目 | make |
| 3 | `docker-compose.yml` | Docker Compose 项目 | docker compose |
| 4 | `package.json` | Node.js 项目 | npm/yarn/pnpm |
| 5 | `Cargo.toml` | Rust 项目 | cargo |
| 6 | `pom.xml` / `build.gradle` | Java 项目 | maven/gradle |

检测到项目类型后，在下方的**项目适配表**中查找对应的命令。

## 项目适配表

### Go 项目（Makefile 驱动）— 如 LinkStash

这是最常见的模式：Makefile 封装了所有操作。

```bash
# Phase 1: BUILD
make build

# Phase 2: START
make start

# Phase 3: VERIFY — 轮询 health endpoint，最多等 10 秒
for i in $(seq 1 10); do
  curl -sf http://localhost:8080/health && break
  sleep 1
done

# Phase 4: TEST
# 优先查找: scripts/smoke_test.sh > make smoke-test > make test
if [ -f scripts/smoke_test.sh ]; then
  bash scripts/smoke_test.sh
elif make -n smoke-test >/dev/null 2>&1; then
  make smoke-test
else
  make test
fi

# Phase 5: STOP
make stop

# Phase 6: CLEAN（可选，默认执行）
make clean
```

**端口检测**：优先从配置文件（`conf/*.yaml`、`.env`）中读取端口号，fallback 到 8080。

### Node.js 项目

```bash
# Phase 1: BUILD
npm run build        # 或 yarn build / pnpm build

# Phase 2: START
npm start &          # 后台启动
echo $! > /tmp/smoke-test.pid

# Phase 3: VERIFY
for i in $(seq 1 10); do
  curl -sf http://localhost:3000/health && break
  sleep 1
done

# Phase 4: TEST
npm test             # 或 scripts/smoke_test.sh

# Phase 5: STOP
kill $(cat /tmp/smoke-test.pid) 2>/dev/null
rm -f /tmp/smoke-test.pid

# Phase 6: CLEAN
rm -rf dist/ build/ node_modules/.cache
```

### Docker Compose 项目

```bash
# Phase 1+2: BUILD & START
docker compose up -d --build

# Phase 3: VERIFY
docker compose ps    # 确认所有服务 running
# + health check URL

# Phase 4: TEST
# 查找 scripts/smoke_test.sh 或 docker compose exec 方式

# Phase 5+6: STOP & CLEAN
docker compose down --volumes --remove-orphans
```

## 执行细节

### 错误处理

每个 Phase 都可能失败。处理原则：

- **BUILD 失败** → 直接终止，报告编译错误，不进入后续阶段
- **START 失败** → 检查端口占用（`lsof -i :PORT`），报告错误，跳到 CLEAN
- **VERIFY 超时** → 检查日志（`/tmp/*.log` 或 `docker compose logs`），报告后跳到 STOP
- **TEST 失败** → 记录失败数，继续执行 STOP 和 CLEAN，最后报告结果
- **STOP/CLEAN 失败** → 警告但不阻断，报告残留进程

关键：**无论测试是否成功，STOP 和 CLEAN 必须执行**。用 `trap` 或确保在 finally 块中清理。

### 输出格式

每个 Phase 开始和结束时打印状态：

```
🔨 [BUILD]   make build ... ✅ (3.2s)
🚀 [START]   make start ... ✅ (PID: 12345)
🏥 [VERIFY]  health check ... ✅ (2 retries)
🧪 [TEST]    smoke_test.sh ... ✅ 34 passed, 0 failed
🛑 [STOP]    make stop ... ✅
🧹 [CLEAN]   make clean ... ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Result: ALL PASSED ✅
  Duration: 28.5s
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 关于清理

默认行为：**执行 CLEAN**（清理构建产物），但**保留数据库和日志**。

这是因为冒烟测试通常使用唯一时间戳标记数据（如 LinkStash 的 `smoke-${UNIQUE}`），所以测试数据不影响后续操作。如果用户需要完全清理数据库，应该在 TEST 脚本中自行处理。

## 自定义扩展

如果项目不匹配任何已知模式，技能会提示用户提供以下信息：

1. 构建命令（如何编译）
2. 启动命令（如何启动服务）
3. 健康检查 URL（如何判断服务就绪）
4. 测试命令（如何执行冒烟测试）
5. 停止命令（如何停止服务）
6. 清理命令（如何清理环境）

收集后按相同的 6-Phase 流程执行。
