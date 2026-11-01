# macOS 兼容移植说明（dsh-ops → DSH 0.2.1-rc.3）

本分支不是上游 dsh-ops 的正式发行版，而是在 macOS（Apple Silicon）上完成一次
可运行性验证后保留的最小改动集。上游代码与许可证不变，所有修改见下方 diff。

- 上游仓库：https://github.com/T-Auto/dsh-ops
- 上游提交：`2c0b0397ce4b48436ff76cf5c5b039054240a45c`（main）
- 上游版本：`dsh-ops@0.5.0-alpha3`（仓库中的未发布候选）
- 验证宿主：`@deepseek-ai/dsh-desktop-runtime 0.2.1-rc.3`
- 验证主机：macOS，Apple Silicon（`darwin-arm64`）

---

## 1. 修改内容

总共两个文件，无源码级改动。

### 1.1 `package.json`：移除 `dsh.client` 段

```diff
   "dsh": {
-    "bundle": { "patch": "./cordis.patch.yml" },
-    "client": {
-      "inject": [
-        "@deepseek-ai/dsh-api-gateway",
-        "@deepseek-ai/dsh-client-ui-renderer",
-        "@deepseek-ai/dsh-client-locale",
-        "@deepseek-ai/dsh-client-ui-plugin-manager",
-        "@deepseek-ai/dsh-client-ui-primitives"
-      ],
-      "platform": "web",
-      "immediately": true
-    }
+    "bundle": { "patch": "./cordis.patch.yml" }
   }
```

### 1.2 `cordis.patch.yml`：改为首轮保守配置

第一轮只开 Standard 预设的常驻文件工具，其余全部关闭（`bash`、`ptc`、
`cordis`、`discovery`、`panel`、`tui`），`hideOfficialTools` 保持 `false`。

---

## 2. 修改原因

### 2.1 为什么删 `dsh.client`

宿主 `@deepseek-ai/dsh-client-modules` 读取的浏览器半边声明形状与本包的写法不匹配：

| | 上游写法 | 宿主 0.2.1-rc.3 期望 |
| --- | --- | --- |
| 字段 | `dsh.client.inject`（模块名数组） | `dsh.client.external`（非基线模块请求） |
| 平台 | `platform: "web"` | 仅 `platform: 'web'` 满足 web carrier |
| 结果 | — | 校验失败，Client 半边不加载 |

同时宿主侧的两个依赖包在安装包内不存在（对 `app.asar` 中 562 个包做精确匹配，
命中 0 次）：

```
@deepseek-ai/dsh-client-ui-primitives   MISS
@deepseek-ai/dsh-client-ui-slots        MISS
```

本包的 `dsh.client.inject` 直接列了 `dsh-client-ui-primitives`。即使字段形状对得上，
解析也会失败。删掉 `dsh.client` 后浏览器半边不加载，**Host 侧工具完全不受影响**——
本分支只验证 Host 能力，因此这是最小改动。

`dsh-plugin.json`（v0.15 社区清单，含 `x-tool-namespaces` / `x-prompt-sections` /
`x-display`）宿主不读取，保留原样不影响加载；实际生效的是
`package.json` 的 `dsh.bundle.patch` 与 `lib/index.js` 的
`export async function apply(ctx, config)`。

### 2.2 为什么用保守配置

`hideOfficialTools: true` 会隐藏官方 `read` / `glob` / `grep`。首轮验证的目标是确认
`ops_*` 工具确实发布，此时若官方工具已被隐藏、而 OPS 工具因故未发布，模型会陷入
无文件工具可用的状态。先以 `false` 验证，确认 5 个 `ops_*` 全部可见后再开启。

（实测该风险已有兜底，见 3.4，但首轮仍按保守顺序推进。）

### 2.3 为什么不需要改源码

- `engines.dsh` 声明为 `>=0.2.0-rc.2 <0.2.2-0`，宿主 `0.2.1-rc.3` 落在区间内；
- 宿主 `@deepseek-ai/dsh-app-boot` 的 `evaluatePluginCompatibility()` 未拒绝该包；
- 本包用到的宿主服务在宿主内全部存在：`tools`、`systemPrompt`、`agents`、
  `ptcRuntime`、`subprocess`、`sandboxPolicy`、`approval`、`jobs`、`shellEnv`，
  以及 `dsh-tools`、`dsh-subprocess`、`dsh-atomic-write`、`dsh-typert-protocol`、
  `dsh-app-boot`、`cordis`。

---

## 3. 结果

### 3.1 构建

工具链：Node `24.22.1`、Rust `1.99.1`（`aarch64-apple-darwin`）。

```console
$ node scripts/build.mjs
build: bundled 11 browser modules (71052 bytes)
build: generated PTC declarations for 18 capabilities (113501 bytes)
build: compiled src/ to lib/ and copied 50 assets

$ cargo build --release --locked --manifest-path native/Cargo.toml
Finished `release` profile [optimized] target(s)

$ node bin/native-runtime.mjs stage --platform darwin-arm64
darwin-arm64: 4260336 bytes, sha256 ee4609db582a7c3b7f9c38190d010213d37acb9ed8946439ce321d00625222ca
Validated 1 native runtime(s); no install-time scripts or permission changes.
```

### 3.2 安装与启动

装进独立的 `web` profile（未触碰 `desktop` profile）：

```console
$ dsh plugin --profile web add ./dsh-ops-0.5.0-alpha3.tgz
+ dsh-ops 0.5.0-alpha3

$ dsh --profile web --dump-config | grep -A 12 'id: dsh-ops'
- id: dsh-ops
  name: dsh-ops
  config:
    global: { bash: false, hidePwsh: false, hideOfficialTools: false, dynamicTools: false }
    standard: { enabled: true, residentTools: true }
    ...
```

Loader 接受该行，组合配置正常输出。

### 3.3 工具发布（模型侧实测）

模型自述的可用工具列表：

```
ops_glob, ops_grep, ops_inspect_local_file, ops_read_files, ops_result
```

端到端调用验证：

| 调用 | 结果 |
| --- | --- |
| `ops_read_files` 读 `package.json` 第 1–10 行 | 正确返回第 5 行 `"access": "public"`，报告 `nextOffset=11`、文件共 250 行 |
| `ops_glob` `["**/*.ts"]` on `src` | 217 个文件，`matchedTotal=217`，`scanComplete=true` |
| `ops_grep` `publicToolName` on `src` | 72 行命中 / 22 个文件，主动报告 `scanComplete=false`、12 个文件因 `invalid-encoding` 跳过 |

三步任务在 `1 turns 2 steps / 5s` 内完成，浏览器端 `pageerror` 为 0。

### 3.4 降级保障（实测）

将 `runtime/darwin-arm64/dsh-ops-worker` 改名后重启，模型**仍能读文件**，走的是
宿主官方 `read`。

原因在 `src/publication/replacements.ts` 的 `plan()`：

```js
.filter(([official, counterpart]) => visible.has(counterpart) && (visible.has(official) || entry.deny.includes(official)))
```

该文件头注释写明：*"Missing counterparts always leave official tools available."*
只有 OPS 对应工具**真正可见**时才隐藏官方工具。worker 缺失 → `ops_read_files`
不存在 → `read` 自动回退。恢复 worker 后 5 个工具正常回来。

因此 `hideOfficialTools: true` 不存在"OPS 挂掉导致模型无工具"的单点风险。

### 3.5 测试套件

```console
$ DSH_OPS_REQUIRE_RUNTIME=1 node test/run.mjs
dsh-ops test: 7 of 162 suite(s) failed in 56s
```

其中 3 个真实失败，4 个是 `REQUIRE_RUNTIME=1` 下把 skip 计为失败。

| 失败套件 | 断言 | 归因 |
| --- | --- | --- |
| `bench-gates` | `git ls-tree -r -z <pinned-sha> -- lib native package.json` | 本验证发生在 `--depth 1` 的浅克隆 + 本地重建历史里，该固定提交不存在。**环境问题**，与改动无关 |
| `permissions` | `workspace-write` 下 `printf bad > denied.txt` 期望 `exitCode != 0` | macOS 上的 DSH 沙箱未返回 `EXECUTION_RESTRICTED`，实际返回 `exitCode 0`。**平台沙箱行为**，非本包改动引起 |
| `settings-client` | `browser package declares its client facet` | `TypeError: Cannot read properties of undefined (reading 'platform')` —— 正是被删除的 `dsh.client` 段。**预期内**，本分支主动放弃浏览器半边 |

失败套件中**没有任何一项**指向宿主 API 签名不兼容、服务缺失或运行时崩溃。

### 3.6 运行稳定性

web 实例连续运行、多轮浏览器交互，`pageerror` 计数为 0；
`Settings → Plugins → Installed plugins` 正常列出 `dsh-ops`。

---

## 4. Token 实测

工具 schema 是每请求都要付的固定成本；工具输出是一次性成本。两者分开计量。

### 4.1 Schema 固定开销

用 `js-tiktoken`（`cl100k_base`）计量，取宿主真实 schema 与 `renderNative` 的真实投影：

| 工具 | token |
| --- | --- |
| 官方 `read` | 125 |
| 官方 `write` | 124 |
| 官方 `edit` | 202 |
| 官方 `glob` | 194 |
| 官方 `grep` | 190 |
| **官方 5 项合计** | **835** |
| `ops_inspect_local_file` | 203 |
| `ops_glob` | 363 |
| `ops_grep` | 528 |
| `ops_read_files` | 148 |
| `ops_result` | 6 |
| **OPS 5 项合计** | **1248** |

差值 **+413 token（+49%）**，每请求支付。

### 4.2 单次调用输出

官方侧用与宿主一致的输出格式（`行号\t内容`、`路径:行号: 内容`、一行一路径），
OPS 侧用 `renderNative()` 的真实文本：

| 场景 | 官方 | OPS | 差 |
| --- | --- | --- | --- |
| `grep` 全目录 | 2939 | 3045 | **−3.6%**（更贵） |
| `glob` 全部 `.ts` | 3043 | 1600 | +47.4% |
| `read` 单文件 | 2332 | 2505 | **−7.4%**（更贵） |

### 4.3 净账随轮次翻转

```
轮次   官方       OPS       净差
1      9149       8398      +8.2%
2      9984       9646      +3.4%
3      10819      10894     -0.7%   ← 盈亏平衡
5      12489      13390     -7.2%
10     16664      19630     -17.8%
```

每次调用需在内容上省下超过 **413 token** 才能打平；只有大规模目录列举满足这个条件。

---

## 5. 结论

**功能上可用，稳定，有降级保障；"节省 token" 在当前实现下不成立。**

- 赚：大目录列举（`glob` 输出相对路径 + 只报计数）、极短任务（1–2 次调用）；
- 亏：单文件读取、目录级 `grep`、以及任何超过 3 次调用的会话；
- 真正的价值不在 token 数字：`scanComplete` / `skippedFiles` / `skipReasons` /
  `nextOffset` 这类诚实性元数据，能阻止模型把残缺结果当真，减少盲猜重试。实测中
  模型主动声明"72 是已观测下限，真实总数可能略高"，这是官方工具不给的信号。

基于上述数据，**不移植到日常使用的 profile**。本分支仅作为 macOS 兼容性的可复现
记录保留。

---

## 6. 复现

```console
# 依赖
npm ci --ignore-scripts

# 构建 TS + 原生 worker
node scripts/build.mjs
cargo build --release --locked --manifest-path native/Cargo.toml
node bin/native-runtime.mjs stage --platform darwin-arm64

# 打包并用宿主 CLI 装进一个独立 profile
npm pack
dsh plugin --profile web add ./dsh-ops-0.5.0-alpha3.tgz
dsh --profile web --dump-config

# 测试
DSH_OPS_REQUIRE_RUNTIME=1 node test/run.mjs
```

Rust 工具链若下载缓慢，可指定镜像：

```console
export RUSTUP_DIST_SERVER="https://mirrors.tuna.tsinghua.edu.cn/rustup"
```

---

## 7. 许可证

本分支与上游一致，以 **MIT** 分发，见 [LICENSE](./LICENSE) 与 [NOTICE](./NOTICE)。
上游 FastCtx 的历史归属说明见上游 `docs/PROVENANCE.md`，本分支未改动。
