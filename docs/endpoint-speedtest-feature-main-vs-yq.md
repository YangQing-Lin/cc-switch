# 端点测速功能说明（main vs yq 分支差异分析）

## 功能概述

main 分支相比 yq 分支新增了**完整的 API 端点测速与管理系统**，允许用户：
- 批量测试多个 API 端点的网络延迟
- 自动选择最快的端点
- 动态添加/删除自定义端点
- 持久化保存自定义端点列表
- 查看彩色编码的延迟指标

此功能显著改善了第三方/自定义供应商的使用体验，特别适合有多个代理节点或需要优化 API 响应速度的用户。

---

## 核心判断

✅ **值得做**

**理由**：
1. **真实需求**：第三方 API 供应商常提供多个接入点（如不同区域节点），用户需要测速来选择最优路径
2. **简洁设计**：测速逻辑仅用 HTTP GET 请求测延迟，没有过度设计
3. **零破坏性**：纯新增功能，不影响现有供应商管理流程
4. **数据结构清晰**：端点信息独立存储在 `Provider.meta.custom_endpoints`，不污染核心配置

---

## 主要变更文件

### 后端（Rust）

1. **src-tauri/src/speedtest.rs** ✨ 新增
   - 核心测速模块：并发测试多个端点的网络延迟
   - 使用 `reqwest` 发送 HTTP GET 请求
   - 热身机制：首次请求用于建立连接，第二次请求才计时
   - 错误分类：超时、连接失败、无效 URL 等

2. **src-tauri/src/commands.rs**
   - 新增 5 个 Tauri 命令：
     - `test_api_endpoints` - 批量测试端点延迟
     - `get_custom_endpoints` - 获取自定义端点列表
     - `add_custom_endpoint` - 添加自定义端点
     - `remove_custom_endpoint` - 删除自定义端点
     - `update_endpoint_last_used` - 更新端点最后使用时间

3. **src-tauri/src/lib.rs**
   - 注册 `speedtest` 模块
   - 注册 5 个新命令到 Tauri 运行时

4. **src-tauri/src/provider.rs**
   - 新增 `ProviderMeta` 结构体
   - 为 `Provider` 添加 `meta` 字段（可选）
   - `meta.custom_endpoints` 存储自定义端点（以 URL 为键去重）

5. **src-tauri/src/settings.rs**
   - 新增 `CustomEndpoint` 结构体：
     - `url`: 端点地址
     - `added_at`: 添加时间戳
     - `last_used`: 最后使用时间（可选）

6. **src-tauri/Cargo.toml**
   - 新增依赖：
     - `reqwest` - HTTP 客户端
     - `futures` - 异步并发处理
     - `chrono` - 时间戳生成

### 前端（React + TypeScript）

1. **src/components/ProviderForm/EndpointSpeedTest.tsx** ✨ 新增
   - 完整的端点管理 Modal 组件（620 行）
   - 功能：
     - 显示端点列表（带延迟指标和彩色编码）
     - 运行测速（并发测试所有端点）
     - 添加自定义端点（支持 URL 校验）
     - 删除端点（悬停显示删除按钮）
     - 自动选择最快端点（可选）
     - ESC 键关闭弹窗
   - 延迟颜色编码：
     - 绿色 < 300ms
     - 黄色 300-500ms
     - 橙色 500-800ms
     - 红色 ≥ 800ms

2. **src/components/ProviderForm.tsx**
   - 为 Claude 和 Codex 供应商表单添加"管理与测速"按钮（带 Zap 图标）
   - 集成 `EndpointSpeedTest` Modal
   - 从 preset 的 `endpointCandidates` 加载候选端点
   - 创建供应商时合并自定义端点到 `meta.custom_endpoints`
   - 编辑供应商时即时同步自定义端点到后端

3. **src/lib/tauri-api.ts**
   - 新增 5 个前端 API 方法，对应后端 Tauri 命令
   - 新增 `EndpointLatencyResult` 接口定义

4. **src/types.ts**
   - 新增 `CustomEndpoint` 接口
   - 新增 `ProviderMeta` 接口
   - 为 `Provider` 添加 `meta?: ProviderMeta` 字段
   - 为 `Settings` 添加 `customEndpointsClaude` 和 `customEndpointsCodex` 字段

5. **src/utils/providerConfigUtils.ts**
   - 新增 `extractCodexBaseUrl()` - 从 TOML 配置提取 base_url
   - 新增 `setCodexBaseUrl()` - 设置 TOML 配置中的 base_url

6. **src/vite-env.d.ts**
   - 扩展全局 `window.api` 类型声明，包含测速和端点管理 API

### 配置预设（Presets）

7. **src/config/providerPresets.ts**
   - 为 `ProviderPreset` 接口添加 `endpointCandidates?: string[]` 字段
   - 为 PackyCode（Claude）添加 5 个候选端点：
     - `https://api.packycode.com`（主站）
     - `https://api-hk-cn2.packycode.com`（香港 CN2）
     - `https://api-hk-g.packycode.com`（香港 PCCW Global）
     - `https://api-us-cn2.packycode.com`（美国 CN2）
     - `https://api-cf-pro.packycode.com`（Cloudflare）

8. **src/config/codexProviderPresets.ts**
   - 为 PackyCode（Codex）添加相同的 5 个候选端点

---

## 功能详解

### 1. 端点延迟测速

**路径**: `供应商表单 → base_url 输入框旁 → 管理与测速 → 测速按钮`

#### 以前的功能（yq 分支）
- ❌ 无测速功能，用户需要手动试错来确定最快端点
- ❌ 无法直观对比多个端点的速度

#### 新的功能（main 分支）
- ✅ 点击"测速"按钮并发测试所有端点
- ✅ 实时显示测速进度（loading 动画）
- ✅ 清空旧的延迟数据，确保显示准确
- ✅ 显示延迟结果（毫秒）及彩色编码：
  - 🟢 < 300ms：绿色（优秀）
  - 🟡 300-500ms：黄色（良好）
  - 🟠 500-800ms：橙色（一般）
  - 🔴 ≥ 800ms：红色（较慢）
- ✅ 自动按延迟从低到高排序
- ✅ 可选"自动选择最快端点"（测速后自动切换）
- ✅ Codex 测速超时时间 12 秒，Claude 8 秒

**实现代码**：
- 前端测速逻辑：[src/components/ProviderForm/EndpointSpeedTest.tsx:312-382](src/components/ProviderForm/EndpointSpeedTest.tsx)
- 后端测速引擎：[src-tauri/src/speedtest.rs:32-106](src-tauri/src/speedtest.rs)

**技术细节**：
- **热身请求**：第一次请求用于建立连接/DNS 解析，第二次才计时（避免首包惩罚）
- **并发测试**：使用 `join_all` 并行测试所有端点
- **超时控制**：可配置超时时间（2-30 秒），默认 8 秒
- **错误处理**：区分超时、连接失败、无效 URL 等错误类型

---

### 2. 自定义端点管理

**路径**: `供应商表单 → 管理与测速 → 添加/删除端点`

#### 以前的功能（yq 分支）
- ❌ 无法保存多个候选端点
- ❌ 每次切换端点需要手动复制粘贴 URL

#### 新的功能（main 分支）
- ✅ 在输入框输入 URL，点击 `+` 按钮添加
- ✅ 支持 Enter 键快捷添加
- ✅ URL 校验：
  - 格式校验（必须是有效 URL）
  - 协议检查（仅支持 HTTP/HTTPS）
  - 去重检查（不允许重复 URL）
- ✅ 自动归一化 URL（去除尾部斜杠）
- ✅ 悬停端点条目显示删除按钮（统一 UI）
- ✅ 删除端点时自动回退到第一个可用端点
- ✅ 持久化到 `~/.cc-switch/config.json` 的 `provider.meta.custom_endpoints`

**实现代码**：
- 前端添加端点：[src/components/ProviderForm/EndpointSpeedTest.tsx:211-285](src/components/ProviderForm/EndpointSpeedTest.tsx)
- 前端删除端点：[src/components/ProviderForm/EndpointSpeedTest.tsx:287-310](src/components/ProviderForm/EndpointSpeedTest.tsx)
- 后端添加逻辑：[src-tauri/src/commands.rs:780-832](src-tauri/src/commands.rs)
- 后端删除逻辑：[src-tauri/src/commands.rs:835-878](src-tauri/src/commands.rs)

**存储结构**：
```json
{
  "providers": {
    "provider-123": {
      "name": "PackyCode",
      "meta": {
        "custom_endpoints": {
          "https://api.example.com": {
            "url": "https://api.example.com",
            "addedAt": 1728307200000,
            "lastUsed": 1728310800000
          }
        }
      }
    }
  }
}
```

---

### 3. 端点候选预设

**位置**: `src/config/providerPresets.ts` 和 `src/config/codexProviderPresets.ts`

#### 以前的功能（yq 分支）
- ❌ 预设配置不包含多端点信息
- ❌ 用户需要自行寻找供应商的备用端点

#### 新的功能（main 分支）
- ✅ 预设配置新增 `endpointCandidates` 字段
- ✅ PackyCode 预设包含 5 个区域节点
- ✅ 打开测速 Modal 时自动加载候选端点
- ✅ 与用户自定义端点合并显示（去重）

**实现代码**：
- 预设定义：[src/config/providerPresets.ts:117-125](src/config/providerPresets.ts)
- 加载逻辑：[src/components/ProviderForm.tsx:1104-1117](src/components/ProviderForm.tsx)

---

### 4. UI/UX 优化

#### Modal 设计
- **响应式**：最大宽度 2xl，高度 80vh，内容超长自动滚动
- **键盘支持**：ESC 键关闭弹窗
- **背景遮罩**：点击背景关闭（避免误操作）
- **Linear 设计风格**：圆角 xl、极简样式、清晰层级

#### 端点列表
- **卡片式布局**：每个端点为独立卡片（space-y-2）
- **选中状态**：蓝色边框 + 蓝色背景 + 蓝色指示点
- **悬停效果**：边框变色 + 显示删除按钮
- **延迟显示**：
  - 有数据：彩色编码的毫秒数
  - 测速中：旋转 loading 图标
  - 失败：显示"失败"文本
  - 未测试：显示 "—"

#### 按钮状态
- **测速按钮**：
  - 固定宽度（w-20）防止布局抖动
  - 测速中显示 loading + "测速中"
  - 禁用状态：无端点时不可点击
- **自动选择**：checkbox + 文案，默认勾选

**实现代码**：
- Modal 结构：[src/components/ProviderForm/EndpointSpeedTest.tsx:415-616](src/components/ProviderForm/EndpointSpeedTest.tsx)
- 延迟颜色编码：[src/components/ProviderForm/EndpointSpeedTest.tsx:552-573](src/components/ProviderForm/EndpointSpeedTest.tsx)

---

### 5. 与 base_url 双向同步

#### Claude 供应商（JSON 配置）
- **读取**：从 `settingsConfig.env.ANTHROPIC_BASE_URL` 提取 base_url
- **写入**：用户在 Modal 中选择端点后，自动更新 JSON 配置中的 `ANTHROPIC_BASE_URL`
- **同步时机**：
  - Modal 打开时：从配置读取当前 base_url
  - 用户选择端点时：更新配置和表单状态

**实现代码**：[src/components/ProviderForm.tsx:464-482](src/components/ProviderForm.tsx)

#### Codex 供应商（TOML 配置）
- **读取**：调用 `extractCodexBaseUrl()` 从 TOML 字符串解析 `base_url`
- **写入**：调用 `setCodexBaseUrl()` 更新 TOML 配置中的 `base_url`
- **同步时机**：
  - 初始化时：从配置提取 base_url
  - 用户选择端点时：更新 TOML 配置

**实现代码**：
- 同步逻辑：[src/components/ProviderForm.tsx:484-501](src/components/ProviderForm.tsx)
- TOML 工具：[src/utils/providerConfigUtils.ts](src/utils/providerConfigUtils.ts)

---

## 使用场景示例

### 场景 1：多区域节点选择
1. 用户选择 PackyCode 作为 Claude 供应商
2. 点击 base_url 旁的"管理与测速"按钮
3. Modal 自动显示 5 个预设节点（香港、美国、Cloudflare 等）
4. 点击"测速"，系统并发测试所有节点
5. 结果显示：香港 CN2 延迟 120ms（绿色）、美国 CN2 延迟 450ms（黄色）
6. 系统自动选择香港节点并更新配置
7. 点击"完成"，base_url 已更新为最优节点

### 场景 2：添加自定义代理
1. 用户有自己的代理服务器 `https://my-proxy.example.com`
2. 在端点管理 Modal 中输入该 URL，点击 `+` 添加
3. 系统验证 URL 格式，保存到 `provider.meta.custom_endpoints`
4. 运行测速发现自定义代理延迟 80ms，优于官方节点
5. 选择该端点，配置自动更新
6. 下次编辑此供应商时，自定义端点仍然保存

### 场景 3：动态网络环境适配
1. 用户在公司网络时，国外节点延迟高（800ms+）
2. 添加国内镜像节点，测速后延迟仅 50ms
3. 切换到家庭网络后，原本慢的国外节点变快（200ms）
4. 再次测速，系统自动切换到最优节点
5. 所有端点历史记录保留，无需重新添加

---

## 技术亮点

### 1. 数据结构设计：消除特殊情况
- **问题**：自定义端点需要存储额外元数据（添加时间、使用时间），但不应污染核心 `Provider` 配置
- **方案**：独立 `ProviderMeta` 结构，仅保存到 `~/.cc-switch/config.json`，不写入 live 配置（如 Claude 的 `~/.config/claude_code_ext/settings.json`）
- **好处**：
  - 核心配置保持纯净
  - 元数据变更不触发 live 配置重写
  - 删除供应商时元数据自动清理

**代码证明**：
```rust
// src-tauri/src/provider.rs:22-25
#[serde(skip_serializing_if = "Option::is_none")]
pub meta: Option<ProviderMeta>,
```
`skip_serializing_if` 确保 `meta` 不会写入 live 配置文件。

### 2. 测速准确性：热身机制
- **问题**：首次 HTTP 请求包含 DNS 解析、TCP 握手、TLS 握手，延迟偏高
- **方案**：先发送一次"热身"请求（忽略结果），第二次请求才计时
- **效果**：测得的延迟更接近真实 API 调用场景

**代码证明**：
```rust
// src-tauri/src/speedtest.rs:68-72
// 先进行一次"热身"请求，忽略其结果，仅用于复用连接/绕过首包惩罚
let _ = client.get(parsed_url.clone()).send().await;

// 第二次请求开始计时，并将其作为结果返回
let start = Instant::now();
```

### 3. 并发性能：join_all
- **问题**：顺序测试 N 个端点需要 N × 超时时间
- **方案**：使用 `futures::join_all` 并发测试
- **效果**：5 个端点并发测速仅需 ~1 次超时时间

**代码证明**：
```rust
// src-tauri/src/speedtest.rs:43-104
let tasks = urls.into_iter().map(|raw_url| { ... });
let results = join_all(tasks).await;
```

### 4. UI 一致性：统一删除交互
- **问题**：旧设计中部分端点可删除、部分不可删除，造成认知负担
- **方案**：所有端点悬停时统一显示删除按钮
- **效果**：用户操作一致，不需要记忆"哪些端点可以删除"

**代码证明**：
```tsx
// src/components/ProviderForm/EndpointSpeedTest.tsx:575-584
<button
  onClick={(e) => { handleRemoveEndpoint(entry); }}
  className="opacity-0 transition hover:text-red-600 group-hover:opacity-100"
>
  <X className="h-4 w-4" />
</button>
```
所有端点条目都有删除按钮，仅通过 `group-hover:opacity-100` 控制显示。

### 5. 向后兼容：渐进增强
- **兼容性**：`Provider.meta` 是可选字段（`Option<ProviderMeta>`）
- **旧配置**：现有供应商没有 `meta` 字段，功能正常工作
- **新配置**：创建供应商时自动初始化 `meta`
- **零破坏**：不影响现有用户，不需要迁移脚本

---

## 关键洞察

### 数据结构：端点元数据独立存储
- **核心关系**：`Provider` 包含 `meta: Option<ProviderMeta>`，`ProviderMeta` 包含 `custom_endpoints: HashMap<String, CustomEndpoint>`
- **存储位置**：
  - `~/.cc-switch/config.json`：保存完整 `Provider`（含 `meta`）
  - `~/.config/claude_code_ext/settings.json`（live 配置）：仅写入 `settingsConfig`，不包含 `meta`
- **生命周期**：
  - 添加端点：写入 `meta.custom_endpoints` → 保存 config.json
  - 切换端点：更新 `settingsConfig` → 保存 config.json + 写入 live 配置
  - 删除端点：从 `meta.custom_endpoints` 移除 → 保存 config.json

### 复杂度：消除的特殊情况
- **去除前**：需要判断"当前端点"、"预设端点"、"自定义端点"的不同存储位置
- **去除后**：统一为 `EndpointEntry[]` 列表，仅通过 `isCustom` 标记区分（影响持久化行为）
- **证据**：`buildInitialEntries()` 函数将所有端点归一化为统一结构，消除边界条件

### 风险点：双向同步的循环依赖
- **潜在问题**：
  - 用户修改 base_url → 触发 useEffect 同步到 codexConfig
  - codexConfig 变化 → 触发 useEffect 提取 base_url
  - base_url 变化 → 死循环
- **解决方案**：`isUpdatingBaseUrlRef` 和 `isUpdatingCodexBaseUrlRef` 标志位阻断循环
- **代码位置**：[src/components/ProviderForm.tsx:464-501](src/components/ProviderForm.tsx)

---

## Linus 式方案总结

### 如果从头设计这个功能：

1. **第一步：简化数据结构**
   - 端点就是 URL + 延迟 + 时间戳，不需要"标签"、"分组"等臆想出来的概念
   - 存储用 HashMap（URL 为键），自动去重，无需维护 ID 映射

2. **第二步：消除特殊情况**
   - 所有端点统一接口：可选择、可删除、可测速
   - 不区分"当前端点"和"候选端点"，用户选择哪个就是哪个

3. **第三步：用最笨但最清晰的方式实现**
   - 测速：简单的 HTTP GET + 计时，不搞 TCP ping、traceroute 等复杂操作
   - 并发：直接 `join_all`，不手动管理线程池
   - UI：Rust 返回结构化数据，前端只负责渲染，不做业务逻辑

4. **第四步：确保零破坏性**
   - `meta` 是可选字段，老配置不受影响
   - live 配置写入逻辑不变，仅扩展数据源

---

## 相关文件索引

| 文件路径 | 作用 | 代码行数 |
|---------|------|---------|
| [src-tauri/src/speedtest.rs](../src-tauri/src/speedtest.rs) | 测速核心引擎 | 107 |
| [src-tauri/src/commands.rs](../src-tauri/src/commands.rs) | 端点管理命令 | +191 |
| [src-tauri/src/provider.rs](../src-tauri/src/provider.rs) | Provider 数据结构扩展 | +12 |
| [src-tauri/src/settings.rs](../src-tauri/src/settings.rs) | CustomEndpoint 定义 | +19 |
| [src/components/ProviderForm/EndpointSpeedTest.tsx](../src/components/ProviderForm/EndpointSpeedTest.tsx) | 端点管理 Modal | 620 |
| [src/components/ProviderForm.tsx](../src/components/ProviderForm.tsx) | 供应商表单集成 | +480 (重构) |
| [src/lib/tauri-api.ts](../src/lib/tauri-api.ts) | 前端 API 封装 | +199 |
| [src/types.ts](../src/types.ts) | TypeScript 类型定义 | +19 |
| [src/utils/providerConfigUtils.ts](../src/utils/providerConfigUtils.ts) | TOML 配置工具 | +22 |
| [src/config/providerPresets.ts](../src/config/providerPresets.ts) | 预设端点候选 | +13 |
| [src/config/codexProviderPresets.ts](../src/config/codexProviderPresets.ts) | Codex 预设端点 | +8 |

---

## 代码统计

### 新增代码
- **Rust**：
  - 新增文件：1 个（speedtest.rs，107 行）
  - 修改文件：4 个，净增 ~250 行
- **TypeScript/React**：
  - 新增文件：1 个（EndpointSpeedTest.tsx，620 行）
  - 修改文件：6 个，净增 ~750 行
- **总计**：+1730 行，-285 行（重构），净增 ~1445 行

### 文件变更统计
```
15 files changed, 1730 insertions(+), 285 deletions(-)

src-tauri/Cargo.lock                              | 284 +++++-----
src-tauri/Cargo.toml                              |   3 +
src-tauri/src/commands.rs                         | 191 ++++++-
src-tauri/src/lib.rs                              |   8 +
src-tauri/src/provider.rs                         |  12 +
src-tauri/src/settings.rs                         |  19 +
src-tauri/src/speedtest.rs                        | 106 ++++
src/components/ProviderForm.tsx                   | 480 重构
src/components/ProviderForm/EndpointSpeedTest.tsx | 620 ++++++++++++++++++++++
src/config/codexProviderPresets.ts                |   8 +
src/config/providerPresets.ts                     |  13 +
src/lib/tauri-api.ts                              | 199 +++++--
src/types.ts                                      |  19 +
src/utils/providerConfigUtils.ts                  |  22 +
src/vite-env.d.ts                                 |  31 +-
```

---

## Git 提交历史

### 两个关键提交

1. **ca488cf** - "feat: Implement Speed Test Function" (2025-10-07)
   - 作者：YoVinchen
   - 实现完整的测速系统（后端 + 前端）
   - 添加端点持久化到 settings.json
   - 集成 Modal UI 和交互逻辑
   - 代码量：+1708 行，-285 行

2. **420a423** - "feat: improve endpoint speed test UI and accuracy" (2025-10-07)
   - 作者：Jason
   - 添加延迟颜色编码（绿/黄/橙/红）
   - 修复测速按钮宽度（防止布局抖动）
   - 添加热身请求机制（提高测速准确性）
   - 代码量：+26 行，-2 行

### 合并过程
- 两个提交来自不同分支，在 main 分支合并
- **c17fc94** - "Merge branch 'yq'" 将功能合并到 main 分支
- 无冲突，功能完全独立于配置导入/导出功能

---

## 总结

### 功能价值
此功能极大提升了第三方/自定义供应商的用户体验：
- **测速**：客观数据支持决策，不再依赖主观判断
- **管理**：持久化端点列表，避免重复输入
- **自动化**：一键测速 + 自动选择，零学习成本
- **可视化**：彩色延迟指标，一目了然

### 设计优点
1. **数据结构清晰**：端点元数据独立存储，不污染核心配置
2. **零破坏性**：完全向后兼容，老用户无感知
3. **简洁实现**：测速逻辑直接明了，无过度设计
4. **用户友好**：全流程可视化，无黑盒操作

### 未来扩展方向
- **定期自动测速**：每 N 小时后台自动测速并切换最优节点
- **延迟历史图表**：记录端点延迟趋势，辅助判断稳定性
- **Ping vs HTTP 测速**：提供 ICMP ping 选项（更低开销）
- **全局端点库**：跨供应商共享端点（如多个供应商用同一个代理）
- **智能推荐**：基于地理位置推荐最优节点

---

## 与 yq 分支的核心差异

| 维度 | yq 分支 | main 分支 |
|-----|---------|----------|
| **测速功能** | ❌ 不存在 | ✅ 完整实现 |
| **端点管理** | ❌ 仅支持单个 base_url | ✅ 支持多端点列表 |
| **持久化** | ❌ 无端点历史记录 | ✅ 保存到 provider.meta |
| **UI 组件** | ❌ 无专用 Modal | ✅ EndpointSpeedTest.tsx（620 行）|
| **预设端点** | ❌ 无候选列表 | ✅ providerPresets.endpointCandidates |
| **延迟可视化** | ❌ 无 | ✅ 彩色编码 + 自动排序 |
| **并发测速** | ❌ 无 | ✅ join_all 并发 |
| **热身机制** | ❌ 无 | ✅ 首次请求不计时 |
| **TOML 支持** | ⚠️ 基础支持 | ✅ 提取/设置 base_url 工具 |
| **代码行数** | 基线 | +1445 行净增长 |

---

## 推荐操作

对于 yq 分支的用户：
1. **合并 main 分支**：`git merge origin/main` 获取测速功能
2. **无需迁移**：现有配置自动兼容，无需手动修改
3. **体验测速**：编辑任意第三方供应商 → 点击"管理与测速"按钮
4. **添加端点**：输入自定义代理 URL → 测速 → 自动选择最快

对于 main 分支的维护者：
1. **保持简洁**：拒绝添加"端点标签"、"端点分组"等不必要的复杂性
2. **优化性能**：考虑缓存测速结果（如 5 分钟内不重复测试同一端点）
3. **完善文档**：在用户手册中说明测速功能的使用场景
4. **监控反馈**：收集用户对测速准确性的反馈，调整超时参数

---

**文档版本**：1.0
**生成时间**：2025-10-07
**对应版本**：v3.5.0+（main 分支）
**分支对比**：main vs yq（基于 commit c17fc94）
