# scratchpad/src/main.rs

演示 Makepad 脚本层的完整 HTTP 请求、JSON 解析和 UI 动态更新的能力 — 实现了一个图片搜索引擎 UI。

## 整体结构

- **第8行**：`use mod.net` — 导入网络模块
- **第11行**：`let results = []` — 脚本层数组，存储搜索结果
- **第13-19行**：`ImageCard` 模板定义 — 一个 RoundedView 包含缩略图、标题和来源标签
- **第21-30行**：`fn fetch(url, extra_headers)` — 脚本层面的 HTTP GET 请求函数，返回 Promise 支持异步
- **第32-53行**：`fn do_search(query)` — 搜索函数：
  1. URL 编码查询词
  2. 请求 DuckDuckGo 图片搜索 HTML 页面
  3. 从 HTML 中提取 `vqd` 参数（DuckDuckGo 的 API 令牌）
  4. 请求 JSON 数据 API
  5. 解析 JSON 并将结果填入 `results` 数组
  6. 调用 `ui.results_view.render()` 触发 UI 更新
- **第55-106行**：UI 定义 — 搜索输入框、搜索按钮、结果列表（`ScrollYView`），结果列表通过 `on_render` 回调动态渲染 `ImageCard`
- **第108-128行**：Rust 侧代码 — App 结构体、MatchEvent（无操作）、AppMain 标准实现

## 关键特性

- 脚本层 `promise()` + `.await()`：非阻塞 HTTP 请求
- `net.http_request()`：脚本网络 API
- `parse_json()`：脚本层 JSON 解析
- `http_resource(result.thumbnail)`：从 URL 加载图片资源
- `on_render` 回调实现虚拟列表效果
- `on_return` 和 `on_click` 回调实现输入框回车和按钮点击的事件绑定
