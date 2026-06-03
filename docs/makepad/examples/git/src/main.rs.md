# git/src/main.rs

演示 Makepad 的 Git 协议实现 — 通过 HTTP(S) 协议进行远程仓库的引用发现和深度为 1 的检出（checkout）。

## 整体结构

### UI 定义（第21-233行）

- **第24-67行**：`GitLogList` — 包装 PortalList，用于显示操作日志
- **第69-233行**：主 UI — 标题栏、远程/目标状态卡片、Poll Now/Checkout (HTTP) 按钮、状态标签、日志列表

### GitLogList widget（第277-315行）

从全局 `EVENT_LOGS` 读取日志条目，在 PortalList 中渲染，条目数过多时截断到 20000 条

### Git 同步引擎（第244-654行）

异步状态机（`SyncPhase`）：
- `Idle` → `AwaitLsRefs`（Poll 流程）
- `Idle` → `AwaitInfoRefs`（Checkout 流程）
- `AwaitInfoRefs` → `AwaitUploadPack`（获取 pack 数据）

`App` 的方法：
- `poll_remote`：请求远程 HEAD 引用哈希
- `checkout_http`：启动完整 checkout（info/refs → upload-pack）
- `send_git_request`：通过 `cx.http_request` 发送 HTTP 请求
- `process_ls_refs_response`：处理 ls-refs 响应，比对本地 HEAD
- `process_info_refs_response`：处理 info/refs 响应并构建 upload-pack 请求
- `process_upload_pack_response`：处理 pack 响应并执行 checkout

### 事件处理（第676-766行）

- `handle_startup`：设置远程 URL 显示、起始日志、启动 30 秒轮询定时器
- `handle_actions`：Poll Now / Checkout (HTTP) 按钮
- `handle_timer`：30 秒间隔自动 poll
- `handle_http_response` / `handle_http_request_error` / `handle_http_progress`：HTTP 响应处理

## 关键 API

- `makepad_git::build_ls_refs_head_request()` — 构建 Git 协议 v2 ls-refs 请求
- `makepad_git::build_info_refs_request()` — 构建 info/refs 请求
- `makepad_git::build_upload_pack_request()` — 构建 upload-pack 请求
- `makepad_git::parse_ls_refs_head_response()` / `parse_info_refs_response()` — 响应解析
- `makepad_git::extract_pack_from_response()` — 从响应中提取 pack 数据
- `makepad_git::apply_pack_and_checkout()` — 应用 pack 数据并检出文件
- `cx.http_request(request_id, req)` — Makepad 异步 HTTP 请求
