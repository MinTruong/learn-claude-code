# Ví dụ API Response S03 — Permission Pipeline chi tiết (code → code)

Mỗi ví dụ dưới đây sẽ trace từng bước từ **model trả response** → **code nào chạy** → **kết quả trả về**.

## Ví dụ 1: Auto-allow — Tool an toàn, không cần hỏi

**Prompt:** *"Đọc file README.md"*

---

### Bước 1: Model trả response

```python
# Code chạy ở agent_loop() — dòng 222-228
response = client.messages.create(...)

# response.content trông như sau:
# [{"type": "tool_use", "id": "toolu_01", "name": "read_file",
#   "input": {"path": "README.md"}}]

messages.append({"role": "assistant", "content": response.content})

# response.stop_reason = "tool_use" → không return, tiếp tục loop
```

---

### Bước 2: Vòng lặp xử lý block

```python
# Code — agent_loop() dòng 232-250
results = []
for block in response.content:       # block = tool_use của read_file
    if block.type != "tool_use":
        continue

    print(f"> {block.name}")         # in > read_file

    # ── GỌI check_permission(block) ──────────────────────
    if not check_permission(block):  # (1) chạy vào đây
        results.append({...})
        continue

    handler = TOOL_HANDLERS.get(block.name)     # (2) nếu True
    output = handler(**block.input)              # (3) run_read(**{"path": "README.md"})
```

---

### Bước 3: check_permission() — dòng 203-214

```python
def check_permission(block) -> bool:
    # block.name = "read_file", block.input = {"path": "README.md"}

    # ── Gate 1: block.name == "bash"? ──
    if block.name == "bash":                   # "read_file" != "bash" → False → BỎ QUA
        reason = check_deny_list(block.input.get("command", ""))
        ...

    # ── Gate 2: check_rules() ──
    reason = check_rules(block.name, block.input)   # gọi check_rules("read_file", {"path": "README.md"})
```

---

### Bước 4: check_rules() — dòng 178-182

```python
def check_rules(tool_name: str, args: dict) -> str | None:
    # tool_name = "read_file", args = {"path": "README.md"}

    for rule in PERMISSION_RULES:
        # PERMISSION_RULES = [
        #     {"tools": ["write_file", "edit_file"], ...},     # Rule 1
        #     {"tools": ["bash"], ...},                         # Rule 2
        # ]

        # Rule 1: tool_name = "read_file" ∈ ["write_file", "edit_file"]? → KHÔNG → skip
        # Rule 2: tool_name = "read_file" ∈ ["bash"]? → KHÔNG → skip

        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]             # không chạy đến đây

    return None   # ← trả về None
```

---

### Bước 5: Quay lại check_permission()

```python
    reason = None    # không có rule nào match

    # ── Gate 3: reason is truthy? ──
    if reason:                                # reason = None → False → BỎ QUA
        decision = ask_user(block.name, block.input, reason)
        ...

    return True       # ← trả về True → tool được phép chạy
```

---

### Bước 6: Tool thực thi

```python
# Quay lại agent_loop() — dòng 247-250
handler = TOOL_HANDLERS.get(block.name)      # handler = TOOL_HANDLERS["read_file"] = run_read
output = handler(**block.input)               # run_read(path="README.md")

# bên trong run_read() — dòng 77-84
def run_read(path: str, limit: int | None = None) -> str:
    try:
        lines = safe_path(path).read_text().splitlines()
        #   safe_path("README.md") → path = WORKDIR / "README.md" → nằm trong WORKDIR → OK
        #   read_text() → đọc nội dung file
        ...
        return "\n".join(lines)
    except Exception as e:
        return f"Error: {e}"
```

---

### Bước 7: Model nhận kết quả

```python
# agent_loop() — dòng 249-250
print(str(output)[:200])                     # in 200 ký tự đầu
results.append({                              # gói vào tool_result
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": output,                         # nội dung README.md
})

# Sau loop, results được append vào messages (dòng 252)
messages.append({"role": "user", "content": results})
```

---

### Sơ đồ code chạy

```
agent_loop()
  ├── client.messages.create(...)           → response.content = [tool_use]
  ├── messages.append(assistant, response)
  │
  ├── for block in response.content:        → block.name = "read_file"
  │     │
  │     ├── check_permission(block)         → dòng 203
  │     │    ├── Gate 1: block.name == "bash"?          → False → skip
  │     │    ├── check_rules("read_file", args)         → dòng 178
  │     │    │    ├── Rule 1: "read_file" ∈ [... ]?     → False → skip
  │     │    │    └── Rule 2: "read_file" ∈ ["bash"]?   → False → skip
  │     │    ├── reason = None → không gọi ask_user
  │     │    └── return True
  │     │
  │     ├── handler = TOOL_HANDLERS["read_file"]        = run_read
  │     ├── output = run_read(path="README.md")         → dòng 77
  │     │    ├── safe_path("README.md")                 → OK
  │     │    ├── read_text()
  │     │    └── return nội dung file
  │     └── results.append(tool_result)
  │
  └── messages.append(user, results)        → loop tiếp hoặc end_turn
```

---

## Ví dụ 2: Hard Deny — Lệnh nguy hiểm bị chặn cứng

**Prompt:** *"Xóa toàn bộ file trong thư mục này"*

---

### Bước 1: Model trả response

```python
# response.content = [{"type": "tool_use", "id": "toolu_02", "name": "bash",
#                       "input": {"command": "rm -rf *"}}]
```

---

### Bước 2: Vòng lặp → check_permission()

```python
block.name = "bash"
block.input = {"command": "rm -rf *"}

if not check_permission(block):   # ← đi vào đây
```

---

### Bước 3: check_permission() — dòng 203-214

```python
def check_permission(block) -> bool:
    # ── Gate 1: block.name == "bash"? ────────── YES
    if block.name == "bash":
        # gọi check_deny_list("rm -rf *")          ← dòng 205
        reason = check_deny_list(block.input.get("command", ""))
```

---

### Bước 4: check_deny_list() — dòng 156-160

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]

def check_deny_list(command: str) -> str | None:
    # command = "rm -rf *"

    for pattern in DENY_LIST:
        # pattern = "rm -rf /"
        # "rm -rf /" in "rm -rf *" ?
        #   → "rm -rf /" là chuỗi con của "rm -rf *"?
        #   → "rm -rf *".find("rm -rf /") → tìm thấy vì "rm -rf *" bắt đầu bằng "rm -rf "
        #   → "rm -rf /".startswith("rm -rf ") và "*" != "/"
        #   → phân tích: "rm -rf /" gồm 7 ký tự, "rm -rf *" gồm 7 ký tự
        #   → "rm -rf " (6 ký tự) + "/" (1 ký tự) = 7 ký tự
        #   → "rm -rf " (6 ký tự) + "*" (1 ký tự) = 7 ký tự
        #   → 6 ký tự đầu giống nhau "rm -rf ", ký tự thứ 7: "/" vs "*"
        #   → KHÔNG giống nhau! Kiểm tra lại...
        #   → Python: "rm -rf /" in "rm -rf *" ?
        #   → "rm -rf /" là một chuỗi, cần tìm chính xác "rm -rf /" trong "rm -rf *"
        #   → "rm -rf *" có chứa "rm -rf " nhưng không có "/" sau nó
        #   → KẾT QUẢ: False!
        #
        # pattern = "sudo"
        # "sudo" in "rm -rf *"? → False
        # pattern = "shutdown" → False
        # ... không pattern nào match

        if pattern in command:
            return f"Blocked: '{pattern}' is on the deny list"

    return None   # ← trả về None
```

> ⚠️ **Quan trọng:** `"rm -rf /"` KHÔNG phải là substring của `"rm -rf *"`!  
> `"rm -rf /"` có 7 ký tự, `"rm -rf *"` có 7 ký tự, ký tự cuối khác nhau (`/` vs `*`).  
> **Pattern này không bắt được lệnh `rm -rf *`!**

```python
    # Vì reason = None (không pattern nào match Gate 1)...

    # ── Gate 2: check_rules() ──
    reason = check_rules(block.name, block.input)
    # gọi check_rules("bash", {"command": "rm -rf *"})
```

---

### Bước 5: check_rules() — dòng 178-182

```python
def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        # Rule 1: tool = ["write_file", "edit_file"]
        #   "bash" ∈ ["write_file", "edit_file"]? → Không → skip

        # Rule 2: tool = ["bash"]
        #   "bash" ∈ ["bash"]? → YES
        #   check(args) = lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"])
        #     → args.get("command") = "rm -rf *"
        #     → "rm " in "rm -rf *" ? → YES! (3 ký tự đầu là "rm ")
        #     → any() → True
        #   → rule["check"](args) → True

        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]      # ← trả về "Potentially destructive command"

    return None
```

**📌 Gate 1 không bắt được nhưng Gate 2 bắt được!** Đây là lý do cần nhiều gate.

---

### Bước 6: Quay lại check_permission()

```python
    reason = "Potentially destructive command"

    # ── Gate 3: reason is truthy? ──
    if reason:                                    # "Potentially destructive command" → truthy → YES
        decision = ask_user(block.name, block.input, reason)
        # gọi ask_user("bash", {"command": "rm -rf *"}, "Potentially destructive command")
```

---

### Bước 7: ask_user() — dòng 190-194

```python
def ask_user(tool_name: str, args: dict, reason: str) -> str:
    # in ra màn hình:
    # ⚠  Potentially destructive command
    #    Tool: bash({'command': 'rm -rf *'})
    #    Allow? [y/N]
    print(f"\n\033[33m⚠  {reason}\033[0m")
    print(f"   Tool: {tool_name}({args})")
    choice = input("   Allow? [y/N] ").strip().lower()

    # User gõ Enter (hoặc "n") → choice = "n"
    # "n" in ("y", "yes")? → False
    return "deny"           # ← trả về "deny"
```

---

### Bước 8: Quay lại check_permission() — kết thúc

```python
    if reason:
        decision = ask_user(...)    # decision = "deny"
        if decision == "deny":      # True → YES
            return False            # ← trả về False → CHẶN tool
```

---

### Bước 9: Loop nhận False

```python
# Quay lại agent_loop() — dòng 242-245
if not check_permission(block):      # check_permission() → False
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,     # "toolu_02"
        "content": "Permission denied.",
    })
    continue     # ← bỏ qua handler(...), không gọi run_bash
                 #   run_bash KHÔNG BAO GIỜ được chạy
```

---

### Sơ đồ code chạy

```
agent_loop()
  ├── response.content = [{"name": "bash", "command": "rm -rf *"}]
  │
  ├── check_permission(block)                      → dòng 203
  │    ├── Gate 1: block.name == "bash"? → YES
  │    │    └── check_deny_list("rm -rf *")        → dòng 156
  │    │         └── "rm -rf /" in "rm -rf *"? → FALSE
  │    │         └── return None                    → không block
  │    │
  │    ├── Gate 2: check_rules("bash", args)       → dòng 178
  │    │    ├── Rule 1: skip
  │    │    ├── Rule 2: "rm " in "rm -rf *"? → YES
  │    │    └── return "Potentially destructive command"
  │    │
  │    ├── Gate 3: reason = "Potentially destructive command"
  │    │    └── ask_user("bash", args, reason)     → dòng 190
  │    │         ├── in "⚠ Potentially destructive command"
  │    │         ├── in "Tool: bash({'command': 'rm -rf *'})"
  │    │         ├── chờ input → user Enter → choice = ""
  │    │         └── return "deny"
  │    └── return False
  │
  ├── results.append({"content": "Permission denied."})
  ├── continue     → KHÔNG gọi run_bash
  │
  └── messages.append(user, results)
      └── Call 2: model thấy "Permission denied."
           └── Response: "Không thể xóa vì bị từ chối."
```

---

## Ví dụ 3: User Approval — Ghi file ra ngoài workspace

**Prompt:** *"Ghi file log ra C:\temp\output.txt"*

---

### Bước 1: Model trả response

```python
# response.content = [{"name": "write_file", "input": {"path": "C:\\temp\\output.txt", "content": "Log data"}}]
```

---

### Bước 2: check_permission() — Gate 1 skip, Gate 2 match

```python
def check_permission(block) -> bool:
    # ── Gate 1: block.name == "bash"? ──
    if block.name == "bash":         # "write_file" != "bash" → False → skip
        ...

    # ── Gate 2: check_rules() ──
    reason = check_rules("write_file", {"path": "C:\\temp\\output.txt", "content": "Log data"})
```

---

### Bước 3: check_rules() — dòng 178-182

```python
def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        # Rule 1: tools = ["write_file", "edit_file"]
        #   "write_file" ∈ ["write_file", "edit_file"]? → YES
        #   check(args) = lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR)
        #     → args.get("path") = "C:\\temp\\output.txt"
        #     → WORKDIR / "C:\\temp\\output.txt" → Path("d:\\MinhTh_code\\learn-claude-code\\C:\\temp\\output.txt")
        #     → .resolve() → "C:\\temp\\output.txt" (vì là absolute path, resolve chuẩn hóa)
        #     → .is_relative_to(WORKDIR) → False (C:\temp không phải con của D:\...)
        #     → not False → True
        #   → rule["check"](args) → True

        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]     # ← "Writing outside workspace"

    return None
```

---

### Bước 4: Gate 3 — ask_user()

```python
    reason = "Writing outside workspace"

    if reason:    # YES
        decision = ask_user("write_file", {"path": "C:\\temp\\output.txt", "content": "Log data"}, reason)
```

#### Nhánh 1: User gõ `y` (allow)

```python
def ask_user(tool_name, args, reason):
    print("⚠ Writing outside workspace")
    print("   Tool: write_file({'path': 'C:\\temp\\output.txt', 'content': 'Log data'})")
    choice = input("   Allow? [y/N] ")    # user gõ "y"
    return "allow" if choice in ("y", "yes") else "deny"    # → "allow"
```

```python
    if reason:
        decision = ask_user(...)    # "allow"
        if decision == "deny":      # False → skip
            return False
    return True   # ← tool được phép chạy
```

Sau đó trong loop:

```python
handler = TOOL_HANDLERS["write_file"]    # = run_write
output = run_write(**{"path": "C:\\temp\\output.txt", "content": "Log data"})

# bên trong run_write() — dòng 87-94
def run_write(path: str, content: str) -> str:
    try:
        file_path = safe_path(path)      # ← RAISE ERROR!
        #   safe_path("C:\\temp\\output.txt")
        #   → (WORKDIR / p).resolve() = "C:\\temp\\output.txt"
        #   → is_relative_to(WORKDIR) = False
        #   → raise ValueError("Path escapes workspace: C:\\temp\\output.txt")
```

**Kết quả:** Permission pipeline cho phép, nhưng `safe_path` vẫn chặn ở layer tool.

#### Nhánh 2: User gõ Enter (deny)

```python
def ask_user(tool_name, args, reason):
    print("⚠ Writing outside workspace")
    print("   Tool: write_file({'path': 'C:\\temp\\output.txt', ...})")
    choice = input("   Allow? [y/N] ")    # user Enter → choice = ""
    return "allow" if choice in ("y", "yes") else "deny"    # → "deny"
```

```python
    if reason:
        decision = ask_user(...)    # "deny"
        if decision == "deny":      # True
            return False            # ← chặn tool

# Loop nhận False
results.append({
    "content": "Permission denied.",
    "tool_use_id": "toolu_01write",
})
continue    # KHÔNG chạy run_write
```

---

### Sơ đồ code chạy — nhánh Deny

```
agent_loop()
  ├── check_permission(block)                      → dòng 203
  │    ├── Gate 1: "write_file" != "bash" → skip
  │    │
  │    ├── Gate 2: check_rules("write_file", args)  → dòng 178
  │    │    ├── Rule 1: "write_file" ∈ ["write_file", "edit_file"] → YES
  │    │    │    └── check: not native_path.is_relative_to(WORKDIR)?
  │    │    │         → "C:\\temp\\output.txt" ngoài WORKDIR → True
  │    │    └── return "Writing outside workspace"
  │    │
  │    ├── Gate 3: ask_user(...)                   → dòng 190
  │    │    ├── print cảnh báo + chi tiết
  │    │    ├── chờ input → "y" → return "allow"
  │    │    │    → check_permission return True
  │    │    │    → handler = run_write
  │    │    │    → run_write(path="C:\\temp\\output.txt")
  │    │    │         → safe_path() → ValueError
  │    │    │         → return "Error: Path escapes workspace..."
  │    │    │    → model thấy error, biết path bị chặn
  │    │    │
  │    │    └── chờ input → Enter → return "deny"
  │    │         → check_permission return False
  │    │         → results.append("Permission denied.")
  │    │         → continue → KHÔNG chạy run_write
  │    │
  │    └── model thấy "Permission denied." → kết thúc
```

---

## So sánh 3 ví dụ — Code path

| Bước | Auto-allow (VD1) | Hard Deny (VD2) | User Ask (VD3) |
|---|---|---|---|
| **check_permission** | dòng 203 | dòng 203 | dòng 203 |
| **Gate 1** | `block.name == "bash"?` → False → skip | `check_deny_list("rm -rf *")` → None | `block.name == "bash"?` → False → skip |
| **Gate 2** | `check_rules("read_file", ...)` → None | `check_rules("bash", ...)` → `"Potentially destructive command"` | `check_rules("write_file", ...)` → `"Writing outside workspace"` |
| **Gate 3** | reason = None → skip | `ask_user("bash", ...)` → user deny | `ask_user("write_file", ...)` → user quyết định |
| **Kết quả** | `return True` | `return False` | `True` hoặc `False` |
| **Tool chạy?** | ✅ `run_read()` dòng 77 | ❌ `continue` dòng 245 — không chạy | ✅ `run_write()` dòng 87 / ❌ `continue` |
| **Lớp bảo vệ thứ 2** | — | — | `safe_path()` dòng 60 vẫn chặn nếu allow |
