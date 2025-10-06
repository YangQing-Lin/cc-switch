# CC-Switch Rust后端架构与功能分析

> **文档版本**: v1.0
> **项目版本**: 3.3.1
> **最后更新**: 2025-10-01

## 项目概述

**CC-Switch** 是一个基于 Tauri 2.0 的跨平台桌面应用，用于管理和切换 Claude Code 与 Codex CLI 工具的 API 供应商配置。项目采用 **单一数据源（SSOT）** 架构，配置集中存储于 `~/.cc-switch/config.json`，切换时直接写入目标工具的实际配置文件。

**技术栈**: Rust + Tauri 2.0 (后端) + React + TypeScript (前端)

---

## 核心架构设计

### 1. 模块划分 (src-tauri/src/)

| 模块 | 文件 | 职责 |
|------|------|------|
| **应用入口** | `main.rs` / `lib.rs` | 应用启动、托盘菜单初始化、事件循环管理、窗口生命周期 |
| **命令层** | `commands.rs` | Tauri 命令接口（22个命令），前端调用的所有后端功能 |
| **Claude配置** | `config.rs` | Claude Code 配置文件读写、通用文件操作工具（原子写入、归档） |
| **Codex配置** | `codex_config.rs` | Codex 配置文件读写（双文件原子操作 + 回滚机制） |
| **应用配置** | `app_config.rs` | 应用级配置结构（`MultiAppConfig`），支持多应用管理、版本迁移 |
| **供应商管理** | `provider.rs` | 供应商数据结构（`Provider`、`ProviderManager`） |
| **全局状态** | `store.rs` | 全局应用状态（`AppState`），封装配置的互斥访问 |
| **应用设置** | `settings.rs` | 用户设置管理（托盘、语言、自定义配置目录） |
| **VS Code集成** | `vscode.rs` | VS Code 用户 settings.json 路径自动检测 |
| **配置迁移** | `migration.rs` | 副本文件合并、版本升级（v1→v2）、去重算法 |

---

## 关键功能逻辑

### 2. SSOT 供应商切换机制

**核心理念**: 配置仅存储在 `config.json` 中，切换时通过三步确保数据一致性。

#### 实现位置
- `commands.rs:296-400` - `switch_provider()` 函数
- `lib.rs:206-245` - 托盘菜单调用的内部切换函数

#### 步骤流程
```rust
// 1. 回填（Backfill）：将当前 live 配置写回内存中的当前供应商
if !manager.current.is_empty() {
    let live_config = read_from_live_config_files();
    manager.providers.get_mut(&manager.current).settings_config = live_config;
}

// 2. 切换（Switch）：从目标供应商的 settings_config 写入 live 配置文件
let target_config = target_provider.settings_config;
write_to_live_config_files(&target_config)?;

// 3. 持久化（Persist）：更新当前供应商ID，保存 config.json
manager.current = target_id;
state.save()?;
```

#### 关键特性
- **Claude**: 直接写入 `~/.claude/settings.json`（或兼容旧版 `claude.json`）
- **Codex**: 原子写入 `auth.json` + `config.toml`，失败时自动回滚
- **无副本文件**: 不再为每个供应商生成独立配置文件，减少文件碎片

---

### 3. Codex 原子写入与回滚机制

Codex 需要同时写入两个文件（`auth.json` 必需，`config.toml` 可选），通过事务机制确保一致性。

#### 实现位置
- `codex_config.rs:57-103` - `write_codex_live_atomic()` 函数

#### 核心代码
```rust
pub fn write_codex_live_atomic(auth: &Value, config_text_opt: Option<&str>) -> Result<(), String> {
    let auth_path = get_codex_auth_path();
    let config_path = get_codex_config_path();

    // 1. 备份旧文件内容（用于回滚）
    let old_auth = if auth_path.exists() {
        Some(fs::read(&auth_path)?)
    } else {
        None
    };

    // 2. 第一步：写入 auth.json
    write_json_file(&auth_path, auth)?;

    // 3. 第二步：写入 config.toml（失败则回滚）
    let cfg_text = config_text_opt.unwrap_or("");
    if !cfg_text.trim().is_empty() {
        // TOML 语法校验
        toml::from_str::<toml::Table>(&cfg_text)?;
    }

    if let Err(e) = write_text_file(&config_path, &cfg_text) {
        // 回滚 auth.json 到旧内容
        if let Some(bytes) = old_auth {
            let _ = atomic_write(&auth_path, &bytes);
        } else {
            let _ = delete_file(&auth_path);
        }
        return Err(e);
    }

    Ok(())
}
```

**设计亮点**:
- 使用临时文件 + `rename` 实现原子写入（`config.rs:148-196`）
- 跨平台兼容：Windows 上先删除再 rename，Unix 系统直接覆盖
- TOML 语法校验：写入前验证配置有效性（`validate_config_toml`）
- 失败回滚：第二步失败时恢复第一步的旧状态

---

### 4. 配置文件备份、恢复与归档机制

#### 4.1 备份（Backup）触发条件

**常规备份 (config.json.bak)**
- **触发时机**: 每次保存配置文件前自动触发（`app_config.rs:111-116`）
- **文件位置**: `~/.cc-switch/config.json.bak`
- **备份策略**: 覆盖式备份，仅保留最新一次
- **用途**: 防止配置文件损坏，支持快速恢复最近版本

```rust
// 保存配置时自动备份
if config_path.exists() {
    let backup_path = get_app_config_dir().join("config.json.bak");
    if let Err(e) = copy_file(&config_path, &backup_path) {
        log::warn!("备份 config.json 到 .bak 失败: {}", e);
    }
}
```

#### 4.2 归档（Archive）触发条件

**场景 1: v1→v2 版本升级归档**
- **触发时机**: 检测到 v1 格式配置时自动触发（`app_config.rs:82-96`）
- **文件位置**: `~/.cc-switch/config.v1.backup.<timestamp>.json`
- **归档内容**: 升级前的完整 v1 配置
- **目的**: 保留升级前的配置快照，支持降级恢复

```rust
// v1→v2 升级前备份
let ts = SystemTime::now().duration_since(UNIX_EPOCH).unwrap_or_default().as_secs();
let backup_path = backup_dir.join(format!("config.v1.backup.{}.json", ts));
match copy_file(&config_path, &backup_path) {
    Ok(()) => log::info!("已备份旧版配置文件: {}", backup_path.display()),
    Err(e) => log::warn!("备份旧版配置文件失败: {}", e),
}
```

**场景 2: 副本文件迁移归档**
- **触发时机**: 首次启动扫描到副本文件时触发（`migration.rs:353-381`）
- **文件位置**: `~/.cc-switch/archive/<timestamp>/<category>/`
- **归档内容**:
  - Claude 副本: `settings-*.json`
  - Codex 副本: `auth-*.json` + `config-*.toml`
  - 主配置: 迁移前的 `config.json`
- **目的**: 安全保存旧版副本文件，避免数据丢失

```rust
// 归档副本文件
for (_, p, _) in claude_items.into_iter() {
    match archive_file(ts, "claude", &p) {
        Ok(Some(_)) => {
            let _ = delete_file(&p);  // 归档成功后删除原文件
        }
        _ => {
            // 归档失败则不删除原文件，保守处理
        }
    }
}
```

**场景 3: 迁移前主配置归档**
- **触发时机**: 副本文件迁移前（`migration.rs:165-170`）
- **文件位置**: `~/.cc-switch/archive/<timestamp>/cc-switch/config.json`
- **归档内容**: 迁移前的主配置文件
- **目的**: 保留迁移前的配置状态

#### 4.3 恢复（Restore）触发条件

**自动回滚机制（Codex 双文件写入失败）**
- **触发时机**: Codex 第二步写入失败时自动触发（`codex_config.rs:98-106`）
- **恢复范围**: 第一步已写入的 `auth.json`
- **恢复方式**:
  - 若原文件存在 → 恢复旧内容
  - 若原文件不存在 → 删除新写入的文件
- **目的**: 保证双文件事务一致性

```rust
// Codex 写入失败自动回滚
if let Err(e) = write_text_file(&config_path, &cfg_text) {
    // 回滚 auth.json 到旧内容
    if let Some(bytes) = old_auth {
        let _ = atomic_write(&auth_path, &bytes);
    } else {
        let _ = delete_file(&auth_path);
    }
    return Err(e);
}
```

**手动恢复（用户操作）**
- **触发方式**: GUI 中无原生恢复功能，需手动操作
- **恢复步骤**:
  1. 从 `.bak` 或归档目录找到备份文件
  2. 手动复制并重命名为 `config.json`
  3. 重启应用加载恢复的配置

#### 4.4 归档目录结构

```
~/.cc-switch/
├── config.json              # 主配置文件
├── config.json.bak          # 最近一次备份
├── settings.json            # 应用设置
├── migrated.copies.v1       # 迁移标记文件
└── archive/                 # 归档根目录
    ├── <timestamp1>/
    │   ├── cc-switch/
    │   │   └── config.json  # 迁移前主配置归档
    │   ├── claude/
    │   │   ├── settings-provider1.json
    │   │   └── settings-provider2.json
    │   └── codex/
    │       ├── auth-provider1.json
    │       └── config-provider1.toml
    ├── <timestamp2>/
    │   └── ...
    └── config.v1.backup.<timestamp>.json  # v1→v2 升级归档
```

#### 4.5 归档文件管理

**归档文件创建**（`config.rs:74-93`）
```rust
pub fn archive_file(ts: u64, category: &str, src: &Path) -> Result<Option<PathBuf>, String> {
    if !src.exists() {
        return Ok(None);
    }
    let mut dest_dir = get_archive_root();
    dest_dir.push(ts.to_string());
    dest_dir.push(category);
    fs::create_dir_all(&dest_dir)?;

    let file_name = src.file_name().unwrap_or_default().to_string_lossy().to_string();
    let mut dest = dest_dir.join(file_name);
    dest = ensure_unique_path(dest);  // 避免文件名冲突

    copy_file(src, &dest)?;
    Ok(Some(dest))
}
```

**归档清理策略**
- 当前版本：无自动清理机制
- 建议改进：保留最近 N 个归档，自动清理旧归档（参见潜在改进建议）

---

### 5. 原子文件写入实现

所有配置文件写入都使用原子操作，避免半写状态导致配置损坏。

#### 实现位置
- `config.rs:148-196` - `atomic_write()` 函数

#### 核心代码
```rust
pub fn atomic_write(path: &Path, data: &[u8]) -> Result<(), String> {
    let parent = path.parent().ok_or("无效的路径")?;

    // 1. 生成唯一临时文件名（使用纳秒时间戳）
    let file_name = path.file_name().ok_or("无效的文件名")?;
    let ts = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_nanos();
    let tmp_path = parent.join(format!("{}.tmp.{}", file_name.to_string_lossy(), ts));

    // 2. 写入临时文件
    {
        let mut f = fs::File::create(&tmp_path)?;
        f.write_all(data)?;
        f.flush()?; // 确保数据写入磁盘
    }

    // 3. 保留原文件权限（Unix系统）
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        if let Ok(meta) = fs::metadata(path) {
            let perm = meta.permissions().mode();
            let _ = fs::set_permissions(&tmp_path, fs::Permissions::from_mode(perm));
        }
    }

    // 4. 跨平台原子替换
    #[cfg(windows)]
    {
        // Windows 上 rename 目标存在会失败，先移除再重命名
        if path.exists() {
            let _ = fs::remove_file(path);
        }
        fs::rename(&tmp_path, path)?;
    }

    #[cfg(not(windows))]
    {
        // Unix 系统直接覆盖（原子操作）
        fs::rename(&tmp_path, path)?;
    }

    Ok(())
}
```

**技术要点**:
1. **临时文件命名**: 使用纳秒时间戳确保唯一性
2. **显式刷新**: 调用 `flush()` 确保数据落盘
3. **权限保留**: Unix 系统复制原文件权限
4. **平台差异**: Windows 需先删除目标文件，Unix 直接覆盖

---

### 6. 配置迁移系统

#### 迁移场景

**1. 副本文件合并（首次启动）**

旧版本为每个供应商生成独立副本文件，新版本统一存储在 `config.json` 中。

- **Claude**: 扫描 `~/.claude/settings-*.json`
- **Codex**: 扫描 `~/.codex/auth-*.json` 和 `config-*.toml`
- **去重策略**: 按 `name + API Key` 去重（忽略大小写）
- **优先级**: live 配置 > 副本文件
- **归档**: 迁移后将原文件移至 `~/.cc-switch/archive/<timestamp>/`

**2. 版本升级（v1 → v2）**

- **v1**: 仅支持 Claude 单应用
- **v2**: 支持 Claude + Codex 多应用
- **自动转换**: 检测旧版格式自动升级
- **备份**: 升级前创建 `config.v1.backup.<timestamp>.json`

#### 实现位置
- `migration.rs:147-392` - `migrate_copies_into_config()` 主迁移函数
- `migration.rs:394-437` - `dedupe_config()` 去重算法
- `app_config.rs:57-105` - v1→v2 版本升级逻辑

#### 去重算法核心代码
```rust
fn dedupe_one(
    mgr: &mut ProviderManager,
    extract_key: &dyn Fn(&Value) -> Option<String>,
) -> usize {
    let mut keep: HashMap<String, String> = HashMap::new(); // key -> id
    let mut remove: Vec<String> = Vec::new();

    for (id, p) in mgr.providers.iter() {
        // 组合键: 名称（小写）+ API Key
        let k = format!(
            "{}|{}",
            p.name.trim().to_lowercase(),
            extract_key(&p.settings_config).unwrap_or_default()
        );

        if let Some(exist_id) = keep.get(&k) {
            // 如果当前项是激活供应商，替换旧的保留项
            if *id == mgr.current {
                remove.push(exist_id.clone());
                keep.insert(k, id.clone());
            } else {
                remove.push(id.clone());
            }
        } else {
            keep.insert(k, id.clone());
        }
    }

    // 删除重复项
    for id in remove.iter() {
        mgr.providers.remove(id);
    }
    remove.len()
}
```

**去重键提取**:
- **Claude**: `env.ANTHROPIC_AUTH_TOKEN` (migration.rs:39-45)
- **Codex**: `auth.OPENAI_API_KEY` 或 `auth.openai_api_key` (migration.rs:47-56)

---

### 7. 系统托盘菜单

#### 动态菜单生成

实现位置: `lib.rs:20-117` - `create_tray_menu()` 函数

**菜单结构**:
```
┌─────────────────────┐
│ 打开主界面           │
├─────────────────────┤
│ ─── Claude ───      │  ← 禁用状态标题
│ ☑ Provider A        │  ← 当前激活
│ ☐ Provider B        │
├─────────────────────┤
│ ─── Codex ───       │
│ ☐ Provider C        │
│ ☑ Provider D        │  ← 当前激活
├─────────────────────┤
│ 退出                │
└─────────────────────┘
```

#### 关键代码
```rust
fn create_tray_menu(app: &AppHandle, app_state: &AppState) -> Result<Menu, String> {
    let config = app_state.config.lock()?;
    let mut menu_builder = MenuBuilder::new(app);

    // 1. 顶部菜单项
    let show_main = MenuItem::with_id(app, "show_main", "打开主界面", true, None)?;
    menu_builder = menu_builder.item(&show_main).separator();

    // 2. Claude 供应商列表
    if let Some(claude_manager) = config.get_manager(&AppType::Claude) {
        let header = MenuItem::with_id(app, "claude_header", "─── Claude ───", false, None)?;
        menu_builder = menu_builder.item(&header);

        for (id, provider) in &claude_manager.providers {
            let is_current = claude_manager.current == *id;
            let item = CheckMenuItem::with_id(
                app,
                format!("claude_{}", id),
                &provider.name,
                true,        // enabled
                is_current,  // checked
                None
            )?;
            menu_builder = menu_builder.item(&item);
        }
    }

    // 3. Codex 供应商列表（同理）
    // ...

    // 4. 退出菜单
    let quit = MenuItem::with_id(app, "quit", "退出", true, None)?;
    menu_builder = menu_builder.separator().item(&quit);

    menu_builder.build()
}
```

#### 托盘事件处理

实现位置: `lib.rs:136-202` - `handle_tray_menu_event()` 函数

```rust
fn handle_tray_menu_event(app: &AppHandle, event_id: &str) {
    match event_id {
        "show_main" => {
            // 显示主窗口
            if let Some(window) = app.get_webview_window("main") {
                let _ = window.unminimize();
                let _ = window.show();
                let _ = window.set_focus();
            }
        }
        "quit" => {
            app.exit(0);
        }
        id if id.starts_with("claude_") => {
            let provider_id = id.strip_prefix("claude_").unwrap();
            // 异步执行切换
            tauri::async_runtime::spawn(async move {
                switch_provider_internal(app, AppType::Claude, provider_id).await
            });
        }
        id if id.starts_with("codex_") => {
            // Codex 切换逻辑同理
        }
        _ => {
            log::warn!("未处理的菜单事件: {}", event_id);
        }
    }
}
```

#### 菜单更新机制

前端通过调用 `update_tray_menu()` 命令触发菜单刷新 (lib.rs:248-261):

```rust
#[tauri::command]
async fn update_tray_menu(
    app: tauri::AppHandle,
    state: tauri::State<'_, AppState>,
) -> Result<bool, String> {
    if let Ok(new_menu) = create_tray_menu(&app, state.inner()) {
        if let Some(tray) = app.tray_by_id("main") {
            tray.set_menu(Some(new_menu))?;
            return Ok(true);
        }
    }
    Ok(false)
}
```

---

### 8. 配置文件路径管理

#### 路径解析策略

**优先级**: 自定义目录 > 默认目录

```rust
// Claude 配置目录
pub fn get_claude_config_dir() -> PathBuf {
    if let Some(custom) = crate::settings::get_claude_override_dir() {
        return custom; // 用户设置的自定义目录
    }
    dirs::home_dir().unwrap().join(".claude") // 默认目录
}

// Codex 配置目录（同理）
pub fn get_codex_config_dir() -> PathBuf {
    if let Some(custom) = crate::settings::get_codex_override_dir() {
        return custom;
    }
    dirs::home_dir().unwrap().join(".codex")
}
```

#### 波浪号展开

实现位置: `settings.rs:113-129` - `resolve_override_path()` 函数

```rust
fn resolve_override_path(raw: &str) -> PathBuf {
    if raw == "~" {
        return dirs::home_dir().unwrap();
    }

    // 支持 ~/path 和 ~\path 格式
    if let Some(stripped) = raw.strip_prefix("~/")
        .or_else(|| raw.strip_prefix("~\\"))
    {
        return dirs::home_dir().unwrap().join(stripped);
    }

    PathBuf::from(raw) // 绝对路径直接使用
}
```

#### 兼容性处理

**Claude**:
```rust
pub fn get_claude_settings_path() -> PathBuf {
    let dir = get_claude_config_dir();
    let settings = dir.join("settings.json");

    if settings.exists() {
        return settings; // 优先使用新版文件名
    }

    let legacy = dir.join("claude.json");
    if legacy.exists() {
        return legacy; // 兼容旧版文件名
    }

    settings // 默认新建使用 settings.json
}
```

**Codex**:
- `auth.json`: 必需文件
- `config.toml`: 可选文件（允许为空）

---

### 9. VS Code 集成

#### 自动检测逻辑

实现位置: `vscode.rs:14-65`

```rust
/// 获取 VS Code 用户 settings.json 的候选路径列表（按优先级排序）
pub fn candidate_settings_paths() -> Vec<PathBuf> {
    let mut paths = Vec::new();
    let products = vec![
        "Code",            // VS Code Stable
        "Code - Insiders", // VS Code Insiders
        "VSCodium",        // VSCodium
        "Code - OSS",      // OSS 发行版
    ];

    #[cfg(target_os = "macos")]
    {
        if let Some(home) = dirs::home_dir() {
            for prod in products {
                paths.push(
                    home.join("Library")
                        .join("Application Support")
                        .join(prod)
                        .join("User")
                        .join("settings.json")
                );
            }
        }
    }

    #[cfg(target_os = "windows")]
    {
        if let Some(roaming) = dirs::config_dir() {
            for prod in products {
                paths.push(roaming.join(prod).join("User").join("settings.json"));
            }
        }
    }

    #[cfg(all(unix, not(target_os = "macos")))]
    {
        if let Some(config) = dirs::config_dir() {
            for prod in products {
                paths.push(config.join(prod).join("User").join("settings.json"));
            }
        }
    }

    paths
}

/// 返回第一个存在的 settings.json 路径
pub fn find_existing_settings() -> Option<PathBuf> {
    candidate_settings_paths()
        .into_iter()
        .find(|p| p.is_file())
}
```

#### Tauri 命令接口

实现位置: `commands.rs:695-732`

```rust
/// VS Code: 获取用户 settings.json 状态
#[tauri::command]
pub async fn get_vscode_settings_status() -> Result<ConfigStatus, String> {
    if let Some(p) = vscode::find_existing_settings() {
        Ok(ConfigStatus {
            exists: true,
            path: p.to_string_lossy().to_string(),
        })
    } else {
        // 返回首选路径（标记不存在）
        let preferred = vscode::candidate_settings_paths().into_iter().next();
        Ok(ConfigStatus {
            exists: false,
            path: preferred.unwrap_or_default().to_string_lossy().to_string(),
        })
    }
}

/// VS Code: 读取 settings.json 文本（仅当文件存在）
#[tauri::command]
pub async fn read_vscode_settings() -> Result<String, String> {
    if let Some(p) = vscode::find_existing_settings() {
        std::fs::read_to_string(&p)
            .map_err(|e| format!("读取 VS Code 设置失败: {}", e))
    } else {
        Err("未找到 VS Code 用户设置文件".to_string())
    }
}

/// VS Code: 写入 settings.json 文本（仅当文件存在；不自动创建）
#[tauri::command]
pub async fn write_vscode_settings(content: String) -> Result<bool, String> {
    if let Some(p) = vscode::find_existing_settings() {
        config::write_text_file(&p, &content)?;
        Ok(true)
    } else {
        Err("未找到 VS Code 用户设置文件".to_string())
    }
}
```

**前端集成**: 前端使用 `jsonc-parser` 库安全修改 `chatgpt.apiBase` 等字段，保留注释和格式。

---

### 10. Tauri 命令层

#### 供应商 CRUD 操作

实现位置: `commands.rs`

| 命令 | 行号 | 功能 | 关键逻辑 |
|------|------|------|---------|
| `get_providers` | 47-69 | 获取所有供应商 | 支持 `app_type`/`app`/`appType` 参数兼容 |
| `get_current_provider` | 72-94 | 获取当前供应商ID | 返回激活的供应商ID |
| `add_provider` | 97-161 | 添加供应商 | 若为当前供应商，先写 live 再保存配置 |
| `update_provider` | 164-234 | 更新供应商 | 同上，确保 live 配置同步 |
| `delete_provider` | 237-294 | 删除供应商 | 检查非当前供应商，删除副本文件 |
| `switch_provider` | 297-400 | 切换供应商 | SSOT 三步流程（回填→切换→持久化） |

#### 配置验证

实现位置: `commands.rs:15-44`

```rust
fn validate_provider_settings(app_type: &AppType, provider: &Provider) -> Result<(), String> {
    match app_type {
        AppType::Claude => {
            if !provider.settings_config.is_object() {
                return Err("Claude 配置必须是 JSON 对象".to_string());
            }
        }
        AppType::Codex => {
            let settings = provider.settings_config
                .as_object()
                .ok_or("Codex 配置必须是 JSON 对象")?;

            // 验证 auth 字段
            let auth = settings
                .get("auth")
                .ok_or("Codex 配置缺少 auth 字段")?;
            if !auth.is_object() {
                return Err("Codex auth 配置必须是 JSON 对象".to_string());
            }

            // 验证 config 字段（可选，但必须是字符串）
            if let Some(config_value) = settings.get("config") {
                if !(config_value.is_string() || config_value.is_null()) {
                    return Err("Codex config 字段必须是字符串".to_string());
                }
                // TOML 语法校验
                if let Some(cfg_text) = config_value.as_str() {
                    codex_config::validate_config_toml(cfg_text)?;
                }
            }
        }
    }
    Ok(())
}
```

#### 其他重要命令

| 命令 | 行号 | 功能说明 |
|------|------|----------|
| `import_default_config` | 403-481 | 从 live 配置导入首个 "default" 供应商（首次启动） |
| `get_config_status` | 492-515 | 检测配置文件是否存在 |
| `get_config_dir` | 524-541 | 获取当前生效的配置目录路径 |
| `open_config_folder` | 546-574 | 使用系统文件管理器打开配置目录 |
| `pick_directory` | 577-606 | 弹出目录选择器（用于自定义配置目录） |
| `open_external` | 609-624 | 打开外部链接（自动补全 https://） |
| `get_app_config_path` | 627-633 | 获取应用配置文件路径 |
| `open_app_config_folder` | 636-654 | 打开应用配置文件夹 |
| `get_settings` | 657-660 | 获取应用设置 |
| `save_settings` | 663-667 | 保存应用设置 |
| `check_for_updates` | 670-682 | 打开 GitHub Releases 页面 |
| `is_portable_mode` | 685-693 | 判断是否为便携版（检测 portable.ini） |

---

### 11. 应用设置管理

#### 设置项结构

实现位置: `settings.rs:7-40`

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AppSettings {
    #[serde(default = "default_show_in_tray")]
    pub show_in_tray: bool,                    // 显示系统托盘图标

    #[serde(default = "default_minimize_to_tray_on_close")]
    pub minimize_to_tray_on_close: bool,       // 关闭时最小化到托盘

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub claude_config_dir: Option<String>,     // 自定义 Claude 配置目录

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub codex_config_dir: Option<String>,      // 自定义 Codex 配置目录

    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub language: Option<String>,              // 界面语言（en/zh）
}

impl Default for AppSettings {
    fn default() -> Self {
        Self {
            show_in_tray: true,
            minimize_to_tray_on_close: true,
            claude_config_dir: None,
            codex_config_dir: None,
            language: None,
        }
    }
}
```

#### 全局状态管理

实现位置: `settings.rs:108-142`

```rust
fn settings_store() -> &'static RwLock<AppSettings> {
    static STORE: OnceLock<RwLock<AppSettings>> = OnceLock::new();
    STORE.get_or_init(|| RwLock::new(AppSettings::load()))
}

pub fn get_settings() -> AppSettings {
    settings_store().read().expect("读取设置锁失败").clone()
}

pub fn update_settings(mut new_settings: AppSettings) -> Result<(), String> {
    new_settings.normalize_paths(); // 清理空白路径
    new_settings.save()?; // 持久化到 ~/.cc-switch/settings.json

    // 更新内存
    let mut guard = settings_store().write().expect("写入设置锁失败");
    *guard = new_settings;
    Ok(())
}
```

**路径规范化** (settings.rs:47-68):
```rust
fn normalize_paths(&mut self) {
    // 去除空白字符串
    self.claude_config_dir = self.claude_config_dir
        .as_ref()
        .map(|s| s.trim())
        .filter(|s| !s.is_empty())
        .map(|s| s.to_string());

    // 语言校验（仅保留 en/zh）
    self.language = self.language
        .as_ref()
        .map(|s| s.trim())
        .filter(|s| matches!(*s, "en" | "zh"))
        .map(|s| s.to_string());
}
```

---

### 12. 平台特定功能

#### macOS Dock 可见性管理

实现位置: `lib.rs:119-134`

```rust
#[cfg(target_os = "macos")]
fn apply_tray_policy(app: &AppHandle, dock_visible: bool) {
    let desired_policy = if dock_visible {
        ActivationPolicy::Regular    // 显示 Dock 图标
    } else {
        ActivationPolicy::Accessory  // 隐藏 Dock 图标，仅托盘模式
    };

    if let Err(err) = app.set_dock_visibility(dock_visible) {
        log::warn!("设置 Dock 显示状态失败: {}", err);
    }

    if let Err(err) = app.set_activation_policy(desired_policy) {
        log::warn!("设置激活策略失败: {}", err);
    }
}
```

**使用场景**:
- 窗口关闭最小化到托盘时：隐藏 Dock 图标
- 从托盘或 Dock 恢复窗口时：显示 Dock 图标

#### macOS 标题栏着色

实现位置: `lib.rs:316-343`

```rust
#[cfg(target_os = "macos")]
{
    if let Some(window) = app.get_webview_window("main") {
        use objc2::rc::Retained;
        use objc2::runtime::AnyObject;
        use objc2_app_kit::NSColor;

        let ns_window_ptr = window.ns_window().unwrap();
        let ns_window: Retained<AnyObject> =
            unsafe { Retained::retain(ns_window_ptr as *mut AnyObject).unwrap() };

        // 使用与主界面 banner 相同的蓝色 #3498db = RGB(52, 152, 219)
        let bg_color = unsafe {
            NSColor::colorWithRed_green_blue_alpha(
                52.0 / 255.0,  // R: 52
                152.0 / 255.0, // G: 152
                219.0 / 255.0, // B: 219
                1.0,           // Alpha: 1.0
            )
        };

        unsafe {
            use objc2::msg_send;
            let _: () = msg_send![&*ns_window, setBackgroundColor: &*bg_color];
        }
    }
}
```

#### Windows 任务栏管理

实现位置: `lib.rs:143-146`

```rust
#[cfg(target_os = "windows")]
{
    // 窗口隐藏时从任务栏移除
    let _ = window.set_skip_taskbar(true);
}
```

#### 窗口关闭拦截

实现位置: `lib.rs:280-299`

```rust
.on_window_event(|window, event| match event {
    tauri::WindowEvent::CloseRequested { api, .. } => {
        let settings = crate::settings::get_settings();

        if settings.minimize_to_tray_on_close {
            api.prevent_close(); // 阻止窗口关闭
            let _ = window.hide();

            #[cfg(target_os = "windows")]
            {
                let _ = window.set_skip_taskbar(true);
            }

            #[cfg(target_os = "macos")]
            {
                apply_tray_policy(&window.app_handle(), false);
            }
        } else {
            window.app_handle().exit(0); // 直接退出应用
        }
    }
    _ => {}
})
```

---

## 数据流图

```
┌──────────────────────────────────────────────────────────┐
│                    前端 (React)                          │
│  - 供应商列表渲染                                         │
│  - 切换按钮事件                                          │
│  - VS Code 同步控制                                      │
└─────────────────┬────────────────────────────────────────┘
                  │ Tauri IPC (invoke)
                  ↓
┌──────────────────────────────────────────────────────────┐
│             commands.rs (Tauri 命令层)                    │
│  - switch_provider(app_type, id)                         │
│  - add_provider(provider)                                │
│  - update_provider(provider)                             │
│  - validate_provider_settings()                          │
└─────────────────┬────────────────────────────────────────┘
                  │
                  ↓
┌──────────────────────────────────────────────────────────┐
│         store.rs (AppState + Mutex锁)                     │
│  - 全局配置管理 (MultiAppConfig)                          │
│  - 线程安全访问                                          │
└─────────────────┬────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
┌──────────────┐     ┌──────────────┐
│  config.rs   │     │ codex_config │
│  (Claude)    │     │   (Codex)    │
│  - 读写JSON  │     │  - 双文件    │
│  - 原子写入  │     │  - 事务回滚  │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ↓                    ↓
┌─────────────────────────────────────────────────────────┐
│     文件系统                                             │
│  ~/.claude/settings.json        (Claude live 配置)       │
│  ~/.codex/auth.json             (Codex 认证，必需)       │
│  ~/.codex/config.toml           (Codex 配置，可选)       │
│  ~/.cc-switch/config.json       (应用主配置，SSOT)       │
│  ~/.cc-switch/settings.json     (应用设置)               │
└─────────────────────────────────────────────────────────┘
```

---

## 核心数据结构

### Provider (provider.rs:8-41)

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Provider {
    pub id: String,
    pub name: String,

    #[serde(rename = "settingsConfig")]
    pub settings_config: Value, // 完整的配置 JSON

    #[serde(skip_serializing_if = "Option::is_none", rename = "websiteUrl")]
    pub website_url: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub category: Option<String>, // "official" | "custom"

    #[serde(skip_serializing_if = "Option::is_none", rename = "createdAt")]
    pub created_at: Option<i64>,
}
```

**Claude settings_config 示例**:
```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-ant-..."
  },
  "models": {
    "opus": "claude-opus-4-20250514"
  }
}
```

**Codex settings_config 示例**:
```json
{
  "auth": {
    "OPENAI_API_KEY": "sk-...",
    "OPENAI_API_BASE": "https://api.openai.com/v1"
  },
  "config": "[model]\npreferred_model = \"gpt-4\""
}
```

### ProviderManager (provider.rs:43-64)

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProviderManager {
    pub providers: HashMap<String, Provider>, // id -> Provider
    pub current: String,                      // 当前激活的供应商 ID
}

impl Default for ProviderManager {
    fn default() -> Self {
        Self {
            providers: HashMap::new(),
            current: String::new(),
        }
    }
}
```

### MultiAppConfig (app_config.rs:34-54)

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MultiAppConfig {
    #[serde(default = "default_version")]
    pub version: u32, // 配置版本（当前为 2）

    #[serde(flatten)]
    pub apps: HashMap<String, ProviderManager>, // "claude" / "codex" -> ProviderManager
}

impl Default for MultiAppConfig {
    fn default() -> Self {
        let mut apps = HashMap::new();
        apps.insert("claude".to_string(), ProviderManager::default());
        apps.insert("codex".to_string(), ProviderManager::default());

        Self { version: 2, apps }
    }
}
```

### AppState (store.rs:4-31)

```rust
pub struct AppState {
    pub config: Mutex<MultiAppConfig>,
}

impl AppState {
    pub fn new() -> Self {
        let config = MultiAppConfig::load().unwrap_or_else(|e| {
            log::warn!("加载配置失败: {}, 使用默认配置", e);
            MultiAppConfig::default()
        });

        Self {
            config: Mutex::new(config),
        }
    }

    pub fn save(&self) -> Result<(), String> {
        let config = self.config.lock().map_err(|e| format!("获取锁失败: {}", e))?;
        config.save()
    }
}
```

---

## 配置文件位置总结

| 文件 | 路径 | 用途 |
|------|------|------|
| **应用配置** | `~/.cc-switch/config.json` | 存储所有供应商的配置（SSOT） |
| **应用设置** | `~/.cc-switch/settings.json` | 托盘、语言、自定义目录等 |
| **迁移标记** | `~/.cc-switch/migrated.copies.v1` | 防止重复迁移副本文件 |
| **配置备份** | `~/.cc-switch/config.json.bak` | 每次保存前自动备份 |
| **归档目录** | `~/.cc-switch/archive/<timestamp>/` | 旧配置文件归档 |
| **Claude 配置** | `~/.claude/settings.json` | Claude Code 实际使用的配置 |
| **Claude 旧版** | `~/.claude/claude.json` | 旧版文件名（兼容支持） |
| **Codex 认证** | `~/.codex/auth.json` | Codex 认证信息（必需） |
| **Codex 配置** | `~/.codex/config.toml` | Codex 自定义配置（可选） |
| **VS Code 设置** | 平台相关路径 | VS Code 用户 settings.json |

---

## 依赖项分析 (Cargo.toml)

### 核心依赖

```toml
[dependencies]
tauri = { version = "2.8.2", features = ["tray-icon"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.8"
dirs = "5.0"
log = "0.4"
```

**说明**:
- `tauri`: 主框架，启用 `tray-icon` 特性支持系统托盘
- `serde` + `serde_json`: JSON 序列化/反序列化
- `toml`: TOML 配置解析和校验
- `dirs`: 跨平台用户目录路径（home/config）
- `log`: 日志记录

### Tauri 插件

```toml
tauri-plugin-single-instance = "2"  # 单实例检测
tauri-plugin-opener = "2"           # 打开外部链接/文件夹
tauri-plugin-dialog = "2"           # 系统对话框（目录选择器）
tauri-plugin-log = "2"              # 日志输出
tauri-plugin-updater = "2"          # 自动更新（桌面端）
tauri-plugin-process = "2"          # 进程管理
```

### 平台特定依赖

```toml
[target.'cfg(target_os = "macos")'.dependencies]
objc2 = "0.5"
objc2-app-kit = { version = "0.2", features = ["NSColor"] }
```

**用途**: macOS 原生 API 调用（Dock 管理、标题栏着色）

### 发布优化配置

```toml
[profile.release]
codegen-units = 1      # 单编译单元（更好的优化）
lto = "thin"           # 链接时优化（减小体积）
opt-level = "s"        # 优化二进制大小
panic = "abort"        # panic 时直接终止（减小体积）
strip = "symbols"      # 移除调试符号
```

**效果**: 显著减小最终可执行文件大小（对 Linux AppImage 尤其重要）

---

## 技术亮点总结

### 1. SSOT 架构
- **彻底消除配置同步问题**: 单一配置源避免数据不一致
- **回填机制**: 切换前自动保存当前 live 配置
- **原子切换**: 先写文件，成功后再更新内存状态

### 2. 事务式写入
- **Codex 双文件原子操作**: 第二步失败自动回滚第一步
- **临时文件 + rename**: 避免半写状态导致配置损坏
- **跨平台兼容**: Windows 先删除目标文件，Unix 直接覆盖

### 3. 智能迁移
- **自动检测副本文件**: 首次启动扫描并合并
- **按名称+密钥去重**: 避免重复供应商
- **优先级控制**: live 配置 > 副本文件 > 当前激活状态
- **安全归档**: 迁移后将原文件移至归档目录，而非直接删除

### 4. 跨平台适配
- **macOS**: Dock 隐藏 + 标题栏着色 + Reopen 事件处理
- **Windows**: 任务栏管理 + 文件删除再重命名
- **Linux**: 标准文件操作
- **WSL 支持**: 自动检测 WSL 环境

### 5. 细粒度错误处理
- **所有文件操作返回 `Result<T, String>`**: 清晰的错误信息
- **配置验证**: 写入前校验 JSON 结构和 TOML 语法
- **失败回滚**: 操作失败自动恢复旧状态

### 6. 线程安全
- **`Mutex<MultiAppConfig>`**: 保护配置并发访问
- **`RwLock<AppSettings>`**: 读多写少的设置管理
- **`OnceLock`**: 延迟初始化全局状态

### 7. 灵活配置
- **自定义配置目录**: 支持覆盖默认路径
- **波浪号展开**: 支持 `~/path` 格式
- **便携版模式**: 检测 `portable.ini` 启用绿色版
- **多语言支持**: 英文/中文切换

### 8. 用户体验优化
- **托盘菜单快速切换**: 无需打开主界面
- **VS Code 一键同步**: 自动检测并应用配置
- **目录选择器**: 可视化选择自定义配置目录
- **自动备份**: 每次保存前备份旧配置

---

## 潜在改进建议

### 1. 错误处理增强
**当前**: 使用 `String` 类型表示错误
**建议**: 使用 `thiserror` 定义类型化错误

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("文件不存在: {0}")]
    FileNotFound(String),

    #[error("JSON 解析失败: {0}")]
    JsonParse(#[from] serde_json::Error),

    #[error("TOML 解析失败: {0}")]
    TomlParse(#[from] toml::de::Error),

    #[error("配置验证失败: {0}")]
    ValidationError(String),
}
```

### 2. 异步文件操作
**当前**: 使用同步 `std::fs`
**建议**: 使用 `tokio::fs` 避免阻塞主线程

```rust
use tokio::fs;

pub async fn read_json_file<T: DeserializeOwned>(path: &Path) -> Result<T, AppError> {
    let content = fs::read_to_string(path).await?;
    Ok(serde_json::from_str(&content)?)
}
```

### 3. 配置验证增强
**当前**: 手动检查字段类型
**建议**: 使用 JSON Schema 验证供应商配置结构

```rust
use jsonschema::JSONSchema;

fn validate_claude_config(config: &Value) -> Result<(), String> {
    let schema = serde_json::json!({
        "type": "object",
        "required": ["env"],
        "properties": {
            "env": {
                "type": "object",
                "required": ["ANTHROPIC_AUTH_TOKEN"]
            }
        }
    });

    let compiled = JSONSchema::compile(&schema)?;
    compiled.validate(config)?;
    Ok(())
}
```

### 4. 日志分级
**当前**: 开发环境使用 `Info` 级别
**建议**: 生产环境降低日志级别，添加文件输出

```rust
if cfg!(debug_assertions) {
    app.plugin(
        tauri_plugin_log::Builder::default()
            .level(log::LevelFilter::Info)
            .build()
    )?;
} else {
    app.plugin(
        tauri_plugin_log::Builder::default()
            .level(log::LevelFilter::Warn)
            .target(tauri_plugin_log::Target::new(
                tauri_plugin_log::TargetKind::LogDir { file_name: Some("app.log".into()) }
            ))
            .build()
    )?;
}
```

### 5. 单元测试
**当前**: 缺少测试覆盖
**建议**: 为核心逻辑添加测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_atomic_write() {
        let tmp = std::env::temp_dir().join("test.json");
        let data = b"test data";

        atomic_write(&tmp, data).unwrap();
        assert_eq!(std::fs::read(&tmp).unwrap(), data);
    }

    #[tokio::test]
    async fn test_switch_provider_rollback() {
        // 测试切换失败时的回滚逻辑
    }
}
```

### 6. 配置备份管理
**当前**: 仅保留最新一次备份（.bak）
**建议**: 保留最近 N 次备份，支持一键回滚

```rust
pub fn create_backup(path: &Path, max_backups: usize) -> Result<(), String> {
    let backups_dir = path.parent().unwrap().join("backups");
    fs::create_dir_all(&backups_dir)?;

    let timestamp = now_ts();
    let filename = path.file_name().unwrap().to_string_lossy();
    let backup_path = backups_dir.join(format!("{}.{}.bak", filename, timestamp));

    copy_file(path, &backup_path)?;

    // 清理旧备份
    cleanup_old_backups(&backups_dir, max_backups)?;
    Ok(())
}
```

### 7. 性能监控
**建议**: 添加关键操作的性能日志

```rust
use std::time::Instant;

pub async fn switch_provider(...) -> Result<bool, String> {
    let start = Instant::now();

    // 执行切换逻辑
    let result = do_switch().await?;

    log::info!("供应商切换耗时: {:?}", start.elapsed());
    Ok(result)
}
```

### 8. 配置模式验证
**建议**: 在开发环境下启用严格模式校验所有配置

```rust
#[cfg(debug_assertions)]
fn validate_strict(provider: &Provider) -> Result<(), String> {
    // 严格校验所有字段
    validate_provider_settings(&app_type, provider)?;

    // 检查配置完整性
    if provider.settings_config.is_null() {
        return Err("配置不能为空".into());
    }

    // 检查 URL 格式
    if let Some(url) = &provider.website_url {
        if !url.starts_with("http") {
            return Err("网站 URL 格式错误".into());
        }
    }

    Ok(())
}
```

---

## 相关文档

- [CLAUDE.md](../CLAUDE.md) - Claude Code 使用指南（针对 AI 助手）
- [README.md](../README.md) - 项目概述和用户手册
- [CHANGELOG.md](../CHANGELOG.md) - 版本更新历史
- [roadmap.md](./roadmap.md) - 功能路线图

---

## 维护者

- **作者**: Jason Young
- **仓库**: [farion1231/cc-switch](https://github.com/farion1231/cc-switch)
- **许可**: MIT License

---

**文档结束**
