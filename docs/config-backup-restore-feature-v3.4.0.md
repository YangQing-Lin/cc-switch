# 配置备份、导入与导出功能说明

## 功能概述

本次更新为 CC Switch 添加了完整的**配置文件导入/导出**和**自动备份**功能，用户可以轻松导出当前配置到本地文件，或从外部文件导入配置，同时系统会自动创建备份以防止数据丢失。

---

## 主要变更文件

### 后端（Rust）

1. **src-tauri/src/import_export.rs** ✨ 新增
   - 核心模块，负责配置的导入、导出和备份管理

2. **src-tauri/src/lib.rs**
   - 注册了 4 个新的 Tauri 命令：
     - `export_config_to_file` - 导出配置
     - `import_config_from_file` - 导入配置
     - `save_file_dialog` - 保存文件对话框
     - `open_file_dialog` - 打开文件对话框

3. **src-tauri/Cargo.toml**
   - 新增依赖：`chrono = "0.4"` 用于生成备份时间戳

### 前端（React + TypeScript）

1. **src/components/ImportProgressModal.tsx** ✨ 新增
   - 导入进度提示弹窗组件
   - 显示导入状态（进行中/成功/失败）
   - 成功后 2 秒自动关闭并刷新数据

2. **src/components/SettingsModal.tsx**
   - 新增"导入导出配置"区域
   - 集成导出按钮和导入流程
   - 支持选择文件 → 执行导入 → 显示进度

3. **src/App.tsx**
   - 新增 `handleImportSuccess` 回调函数
   - 导入成功后自动刷新供应商列表和托盘菜单

4. **src/lib/tauri-api.ts**
   - 新增 4 个前端 API 方法，与 Rust 命令对应

5. **src/vite-env.d.ts**
   - 添加新 API 的 TypeScript 类型声明

### 国际化（i18n）

6. **src/i18n/locales/en.json** 和 **src/i18n/locales/zh.json**
   - 新增 13 条翻译键：
     - `settings.importExport` - "导入导出配置"
     - `settings.exportConfig` - "导出配置到文件"
     - `settings.selectConfigFile` - "选择配置文件"
     - `settings.import` - "导入"
     - `settings.importing` - "导入中..."
     - `settings.importSuccess` - "导入成功！"
     - `settings.importFailed` - "导入失败"
     - `settings.configExported` - "配置已导出到："
     - `settings.exportFailed` - "导出失败"
     - `settings.selectFileFailed` - "选择文件失败"
     - `settings.configCorrupted` - "配置文件可能已损坏或格式不正确"
     - `settings.backupId` - "备份ID"
     - `settings.autoReload` - "数据将在2秒后自动刷新..."

---

## 功能详解

### 1. 配置导出

**路径**: `设置 → 导入导出配置 → 导出配置到文件`

#### 以前的功能
- ❌ 不存在此功能，用户无法方便地备份配置

#### 新的功能
- ✅ 点击"导出配置到文件"按钮
- ✅ 系统弹出保存对话框，默认文件名：`cc-switch-config-YYYY-MM-DD.json`
- ✅ 选择保存位置后，当前完整配置（所有供应商）被导出为 JSON 文件
- ✅ 导出成功后弹窗提示文件路径

**实现代码**：
- 前端：[src/components/SettingsModal.tsx:359-374](src/components/SettingsModal.tsx)
- 后端：[src-tauri/src/import_export.rs:83-97](src-tauri/src/import_export.rs)

---

### 2. 配置导入

**路径**: `设置 → 导入导出配置 → 选择配置文件 → 导入`

#### 以前的功能
- ❌ 不存在此功能，用户无法从外部文件恢复配置

#### 新的功能
- ✅ 点击"选择配置文件"选择 JSON 配置文件
- ✅ 文件路径显示在按钮下方（仅显示文件名）
- ✅ 点击"导入"按钮执行导入操作
- ✅ 导入过程：
   1. **自动备份当前配置** 到 `~/.cc-switch/backups/backup_YYYYMMDD_HHMMSS.json`
   2. **验证导入文件格式** 是否为合法配置
   3. **写入新配置** 到 `~/.cc-switch/config.json`
   4. **更新内存状态** 和托盘菜单
- ✅ 导入成功后显示备份 ID，2 秒后自动刷新界面
- ✅ 导入失败时显示错误信息，可手动关闭

**实现代码**：
- 前端选择文件：[src/components/SettingsModal.tsx:377-389](src/components/SettingsModal.tsx)
- 前端执行导入：[src/components/SettingsModal.tsx:392-416](src/components/SettingsModal.tsx)
- 后端导入逻辑：[src-tauri/src/import_export.rs:99-134](src-tauri/src/import_export.rs)
- 进度弹窗：[src/components/ImportProgressModal.tsx](src/components/ImportProgressModal.tsx)

---

### 3. 自动备份机制

#### 以前的功能
- ❌ 无备份机制，配置损坏或误操作无法恢复

#### 新的功能
- ✅ **导入配置时自动备份**：每次导入前，当前配置被复制到 `~/.cc-switch/backups/`
- ✅ **备份命名规则**：`backup_YYYYMMDD_HHMMSS.json`（例如：`backup_20251006_230307.json`）
- ✅ **备份数量限制**：默认仅保留最近 **10 份备份**，旧备份自动清理
- ✅ **可配置清理策略**：在 [import_export.rs:6](src-tauri/src/import_export.rs) 修改 `MAX_BACKUPS` 常量

**实现代码**：
- 备份创建：[src-tauri/src/import_export.rs:9-35](src-tauri/src/import_export.rs)
- 旧备份清理：[src-tauri/src/import_export.rs:37-79](src-tauri/src/import_export.rs)

---

### 4. 用户体验优化

#### 导入进度提示弹窗

**以前**：无导入功能

**现在**：
- ⏳ **导入中**：显示旋转加载图标 + "导入中..." 文案
- ✅ **导入成功**：显示绿色对勾 + 备份 ID + "数据将在2秒后自动刷新..."
- ❌ **导入失败**：显示红色警告图标 + 错误原因 + "关闭"按钮

**实现代码**：[src/components/ImportProgressModal.tsx](src/components/ImportProgressModal.tsx)

#### 导入成功后自动刷新

- ✅ 导入成功 2 秒后，自动重新加载供应商列表
- ✅ 自动更新托盘菜单（显示最新供应商）
- ✅ 无需手动刷新页面或重启应用

**实现代码**：
- 前端刷新回调：[src/App.tsx:232-238](src/App.tsx)
- 弹窗倒计时逻辑：[src/components/ImportProgressModal.tsx:22-40](src/components/ImportProgressModal.tsx)

---

## 使用场景示例

### 场景 1：多设备同步配置
1. 在设备 A 上配置好所有供应商
2. 导出配置文件到云盘（如 OneDrive、iCloud）
3. 在设备 B 上打开 CC Switch，导入配置文件
4. 所有供应商配置瞬间同步完成

### 场景 2：配置备份与恢复
1. 在进行大量配置修改前，先导出当前配置作为快照
2. 实验性修改配置（如测试新的 API 端点）
3. 如果出现问题，直接导入之前的快照恢复

### 场景 3：团队配置分享
1. 团队管理员配置好标准的 API 供应商模板
2. 导出配置文件发给团队成员
3. 成员直接导入，无需手动逐个配置

---

## 技术亮点

1. **原子性写入**：导入过程中出现错误，配置不会损坏（Rust 的错误处理保证）
2. **格式验证**：导入前强制验证 JSON 结构，防止无效配置
3. **自动清理**：旧备份自动清理，避免磁盘空间无限占用
4. **跨平台**：文件对话框和路径处理在 Windows/macOS/Linux 上完全兼容
5. **用户体验**：全流程有明确的 UI 反馈（进度/成功/失败），无黑盒操作

---

## 相关文件索引

| 文件路径 | 作用 |
|---------|------|
| [src-tauri/src/import_export.rs](../src-tauri/src/import_export.rs) | 导入导出核心逻辑 |
| [src-tauri/src/lib.rs](../src-tauri/src/lib.rs) | 注册 Tauri 命令 |
| [src/components/SettingsModal.tsx](../src/components/SettingsModal.tsx) | 设置界面 UI |
| [src/components/ImportProgressModal.tsx](../src/components/ImportProgressModal.tsx) | 导入进度弹窗 |
| [src/App.tsx](../src/App.tsx) | 主应用逻辑 |
| [src/lib/tauri-api.ts](../src/lib/tauri-api.ts) | 前端 API 封装 |
| [src/i18n/locales/en.json](../src/i18n/locales/en.json) | 英文翻译 |
| [src/i18n/locales/zh.json](../src/i18n/locales/zh.json) | 中文翻译 |

---

## 总结

此功能极大提升了 CC Switch 的可用性和安全性：
- **导出**：随时备份配置，防止数据丢失
- **导入**：快速迁移/恢复配置，支持多设备同步
- **自动备份**：每次导入前自动保护现有配置
- **用户友好**：全流程有清晰的可视化反馈

未来可扩展方向：
- 云端配置同步（如 GitHub Gist）
- 配置版本管理（类似 Git 的时间线）
- 加密导出（保护 API Key）
