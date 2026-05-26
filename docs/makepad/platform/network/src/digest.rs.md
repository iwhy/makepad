# digest.rs — 无依赖哈希与编码实现

**File path**: `platform/network/src/digest.rs` (597 行)
**Core purpose**: 纯 Rust 实现的 SHA1（流式 API）、MD5、SHA256 哈希和 Base64 编码，无外部依赖。

## 结构体

### Sha1
- `state: [u32; 5]`, `block: [u8; 64]`, `total: usize`, `in_block: usize`
- 流式 API: `new()` → `update(&[u8])` → `finalise() -> [u8; 20]`
- 实现了 `Default`

## 公有函数

### md5_hash(input: &[u8]) -> [u8; 16]
- 完整 MD5 哈希计算（PKCS#7 填充，64 轮压缩函数）
- 使用标准 MD5 常量（`0x67452301` 等）

### sha256_hash(input: &[u8]) -> [u8; 32]
- 完整 SHA-256 实现
- 64 轮压缩，使用 SHA-256 原始常量

### base64_encode(input: &[u8]) -> String
- 标准 Base64 编码（`+` 和 `/`，`=` 填充）

## 内部细节

### SHA1 核心函数
- `sha1_digest_bytes(state, block)` — 处理 64 字节块
- `sha1_digest_block_u32(state, block_u32)` — 80 轮压缩
- 4 轮不同的布尔函数: `sha1rnds4c` (Choose), `sha1rnds4p` (Parity), `sha1rnds4m` (Majority)
- 消息调度宏 `schedule!` 和 `schedule_rounds4!`
- 使用 SIMD 友好风格（4 路并行处理）

### 常量
- `SHA256_K` — 64 个 SHA-256 轮常量
- `BASE64_TABLE` — Base64 字符表
- `SHA1_INIT_STATE`, `K0`-`K3`
