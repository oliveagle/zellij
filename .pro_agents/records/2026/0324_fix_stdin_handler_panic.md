# 修复 stdin_handler panic: assertion failed: mid <= self.len()

> **记录时间**: 2026-03-24 11:30
> **参与 Agent**: hardclaw-Debugger

---

## 问题描述

Zellij 在运行时出现 panic：
```
thread 'stdin_handler' panicked at /rust/lib/rustlib/src/rust/library/core/src/slice/mod.rs:3672:9:
assertion failed: mid <= self.len()
```

### 日志位置
- `/tmp/zellij-1000/zellij-log/zellij.log`

### 错误时间
- 2026-03-24 11:23:38.575
- 2026-03-24 11:23:41.961

---

## 根本原因

问题出在 `zellij-utils/src/vendored/termwiz/input.rs` 的 `decode_one_char` 函数：

```rust
fn decode_one_char(bytes: &[u8]) -> Option<(char, usize)> {
    let bytes = &bytes[..bytes.len().min(4)];  // 裁剪最多 4 字节
    match std::str::from_utf8(bytes) {
        Ok(s) => { ... }
        Err(err) => {
            // 问题：err.valid_up_to() 是相对于原始输入的索引
            // 当原始输入 > 4 字节时，valid_up_to() 可能 > 4
            // 但 bytes 已经被裁剪为最多 4 字节
            let (valid, _after_valid) = bytes.split_at(err.valid_up_to());
            // 如果 valid_up_to() > bytes.len()，触发断言失败
        }
    }
}
```

### 触发条件

当输入的 UTF-8 字节序列超过 4 字节且包含无效编码时：
1. `bytes` 被裁剪为最多 4 字节
2. `from_utf8(bytes)` 返回 `Err(err)`
3. `err.valid_up_to()` 返回相对于**原始输入**的有效字节数（可能 > 4）
4. `bytes.split_at(err.valid_up_to())` 触发断言失败

---

## 修复方案

在 `split_at` 之前，将 `valid_up_to()` 的值钳制到 `bytes.len()`：

```rust
Err(err) => {
    // Clamp valid_up_to() to the length of the sliced bytes
    // to avoid panic in split_at when valid_up_to() > bytes.len()
    let valid_len = err.valid_up_to().min(bytes.len());
    let (valid, _after_valid) = bytes.split_at(valid_len);
    if !valid.is_empty() {
        let s = unsafe { std::str::from_utf8_unchecked(valid) };
        let (c, len) = Self::first_char_and_len(s);
        Some((c, len))
    } else {
        None
    }
}
```

### 修复文件
- `zellij-utils/src/vendored/termwiz/input.rs:1431`

### 修改内容
```diff
- let (valid, _after_valid) = bytes.split_at(err.valid_up_to());
+ let valid_len = err.valid_up_to().min(bytes.len());
+ let (valid, _after_valid) = bytes.split_at(valid_len);
```

---

## 验证

```bash
cargo check --package zellij-utils
```

结果：编译成功，仅存 4 个无关警告。

---

## 相关文件

- `zellij-utils/src/vendored/termwiz/input.rs` - 修复位置
- `zellij-client/src/stdin_handler.rs` - stdin 处理线程
- `zellij-client/src/keyboard_parser.rs` - Kitty 键盘解析器

---

*记录者：hardclaw-Debugger*
