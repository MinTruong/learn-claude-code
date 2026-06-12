# Ví dụ API Response S02 — Tool Dispatch Map & Tool Execution

Mỗi ví dụ dưới đây sẽ trace từng bước từ **model trả response** → **code nào chạy** → **kết quả trả về**.

---

## Ví dụ 1: Song song — 2 tool độc lập trong 1 response

**Prompt:** *"Đọc file README.md và tìm tất cả file Python trong thư mục này"*

---

### Bước 1: Model trả response — 2 tool_use blocks

```python
# Code — agent_loop() dòng 150-170
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )                                          # dòng 152-155
```

**Request gửi lên API:**
```json
{
  "model": "claude-sonnet-4-6",
  "system": "You are a coding agent at d:\\MinhTh_code\\learn-claude-code. Use tools to solve tasks. Act, don't explain.",
  "messages": [
    {"role": "user", "content": "Đọc file README.md và tìm tất cả file Python trong thư mục này"}
  ],
  "tools": [
    {"name": "bash", "description": "Run a shell command.", "input_schema": {...}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to a file.", ...},
    {"name": "edit_file", "description": "Replace exact text in a file once.", ...},
    {"name": "glob", "description": "Find files matching a glob pattern.",
     "input_schema": {"type": "object", "properties": {"pattern": {"type": "string"}}, "required": ["pattern"]}}
  ],
  "max_tokens": 8000
}
```

**Response trả về:**
```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_01read", "name": "read_file",
#    "input": {"path": "README.md"}},
#   {"type": "tool_use", "id": "toolu_01glob", "name": "glob",
#    "input": {"pattern": "**/*.py"}},
# ]
# response.stop_reason = "tool_use"
```

---

### Bước 2: Vòng lặp kiểm tra stop_reason — dòng 156-159

```python
    messages.append({"role": "assistant", "content": response.content})  # dòng 156

    if response.stop_reason != "tool_use":    # "tool_use" != "tool_use" → False → KHÔNG return
        return
```

---

### Bước 3: Dispatch map — TOOL_HANDLERS[block.name]() — dòng 161-170

```python
    results = []
    for block in response.content:
        if block.type == "tool_use":
            print(f"\033[33m> {block.name}\033[0m")
            #   lần 1: in "> read_file"
            #   lần 2: in "> glob"

            # ── ĐÂY LÀ ĐIỂM KHÁC BIỆT CỦA S02 ──────────────
            #   s01: output = run_bash(block.input["command"])          — hardcode
            #   s02: handler = TOOL_HANDLERS.get(block.name)           — dispatch map
            #        output = handler(**block.input) if handler else...
            # ─────────────────────────────────────────────────

            handler = TOOL_HANDLERS.get(block.name)     # dòng 165

            # Đây là TOOL_HANDLERS — dòng 138-141:
            # TOOL_HANDLERS = {
            #     "bash": run_bash, "read_file": run_read, "write_file": run_write,
            #     "edit_file": run_edit, "glob": run_glob,
            # }

            # Lần 1: block.name = "read_file"
            #   → TOOL_HANDLERS.get("read_file") → run_read          ← dispatch!
            # Lần 2: block.name = "glob"
            #   → TOOL_HANDLERS.get("glob") → run_glob              ← dispatch!
```

**Tool thực thi qua dispatch — run_read() — dòng 73-80:**
```python
def run_read(path: str, limit: int | None = None) -> str:
    try:
        lines = safe_path(path).read_text().splitlines()
        #   safe_path("README.md") → (WORKDIR / "README.md").resolve()
        #     → Path("d:\\MinhTh_code\\learn-claude-code\\README.md")
        #   is_relative_to(WORKDIR)? → True → OK
        #   read_text() → đọc toàn bộ nội dung
        #   .splitlines() → ["# learn-claude-code", "", "## Introduction", ...]
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        return "\n".join(lines)   # trả về nội dung file
    except Exception as e:
        return f"Error: {e}"
```

**Tool thực thi — run_glob() — dòng 105-114:**
```python
def run_glob(pattern: str) -> str:
    import glob as g
    try:
        results = []
        for match in g.glob(pattern, root_dir=WORKDIR):
            #   g.glob("**/*.py", root_dir="d:\\MinhTh_code\\learn-claude-code")
            #   → ["s01_agent_loop/code.py", "s02_tool_use/code.py", "tests/...", ...]
            if (WORKDIR / match).resolve().is_relative_to(WORKDIR):
                results.append(match)
        return "\n".join(results) if results else "(no matches)"
    except Exception as e:
        return f"Error: {e}"
```

**Gom kết quả — dòng 166-168:**
```python
            output = handler(**block.input) if handler else f"Unknown: {block.name}"
            # Lần 1: run_read(**{"path": "README.md"}) → nội dung README
            # Lần 2: run_glob(**{"pattern": "**/*.py"}) → danh sách file .py

            print(str(output)[:200])               # in 200 ký tự đầu mỗi kết quả
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": output,                 # nội dung thật
            })

    messages.append({"role": "user", "content": results})  # dòng 170
```

---

### Bước 4: Trạng thái messages sau Call 1

```python
messages = [
    {"role": "user", "content": "Đọc file README.md và tìm tất cả file Python..."},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_01read", "name": "read_file",
         "input": {"path": "README.md"}},
        {"type": "tool_use", "id": "toolu_01glob", "name": "glob",
         "input": {"pattern": "**/*.py"}},
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01read",
         "content": "# learn-claude-code\n\n## Introduction\n..."},
        {"type": "tool_result", "tool_use_id": "toolu_01glob",
         "content": "s01_agent_loop/code.py\ns02_tool_use/code.py\ntests/test_agents_smoke.py\n..."},
    ]}
]
```

---

### Bước 5: Call 2 — model kết thúc

```python
# while True — lần lặp thứ 2
response = client.messages.create(...)   # dòng 152
#   stop_reason = "end_turn"
#   content = [{"type": "text", "text": "Dự án này là...\n\nCác file Python:\n- s01_agent_loop/code.py\n..."}]

messages.append({"role": "assistant", "content": response.content})   # dòng 156

if response.stop_reason != "tool_use":   # "end_turn" != "tool_use" → True
    return                               # ← THOÁT
```

---

### Sơ đồ code chạy

```
agent_loop()
  │
  ├── VÒNG LẶP #1
  │   ├── response.content = [
  │   │     tool_use(read_file, path="README.md"),
  │   │     tool_use(glob, pattern="**/*.py")
  │   │   ]
  │   │   stop_reason = "tool_use"
  │   │
  │   ├── for block in response.content:
  │   │   │
  │   │   ├── Block 1: name = "read_file"
  │   │   │    ├── handler = TOOL_HANDLERS.get("read_file")    → dòng 165
  │   │   │    │              → run_read
  │   │   │    ├── output = run_read(**{"path": "README.md"})  → dòng 166
  │   │   │    │    ├── safe_path("README.md")                 → dòng 66-70
  │   │   │    │    ├── read_text()                            → dòng 75
  │   │   │    │    └── return "# learn-claude-code\n..."
  │   │   │    └── results.append(tool_use_id="toolu_01read", content="...")
  │   │   │
  │   │   └── Block 2: name = "glob"
  │   │        ├── handler = TOOL_HANDLERS.get("glob")          → dòng 165
  │   │        │              → run_glob
  │   │        ├── handler = run_glob(**{"pattern": "**/*.py"}) → dòng 166
  │   │        │    ├── glob.glob("**/*.py", root_dir=WORKDIR)  → dòng 109
  │   │        │    └── return "s01_agent_loop/code.py\n..."
  │   │        └── results.append(tool_use_id="toolu_01glob", content="...")
  │   │
  │   └── messages.append(user, results)                       → dòng 170
  │
  └── VÒNG LẶP #2
      ├── response.stop_reason = "end_turn"
      └── return → THOÁT
```

---

## Ví dụ 2: Tuần tự — Kết quả tool trước quyết định tool sau

**Prompt:** *"Đọc code.py, tìm hàm run_bash, sửa timeout từ 120 thành 60"*

Đây là task **2 bước bắt buộc**: model không thể sửa nếu chưa đọc nội dung.

---

### Call 1: Model đọc file trước

```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_read", "name": "read_file",
#    "input": {"path": "s02_tool_use/code.py"}}
# ]
# stop_reason = "tool_use"
```

**Kiểm tra stop_reason — dòng 158-159:**
```python
if response.stop_reason != "tool_use":    # "tool_use" != "tool_use" → False → ở lại loop
```

**Dispatch + thực thi — dòng 165-166:**
```python
handler = TOOL_HANDLERS.get("read_file")           # → run_read
output = run_read(**{"path": "s02_tool_use/code.py"})
#   safe_path("s02_tool_use/code.py") → Path(".../s02_tool_use/code.py")
#   read_text().splitlines() → list các dòng trong code.py
#   → trả về nội dung file (chứa "timeout=120" ở dòng 53)
```

**Gom kết quả:**
```python
results.append({
    "type": "tool_result",
    "tool_use_id": "toolu_read",
    "content": "# code.py\n\n...\ntimeout=120\n...",
})
messages.append({"role": "user", "content": results})   # dòng 170
# while True → lần lặp thứ 2
```

---

### Call 2: Model đã biết nội dung → gọi edit_file

```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_edit", "name": "edit_file",
#    "input": {"path": "s02_tool_use/code.py", "old_text": "timeout=120", "new_text": "timeout=60"}}
# ]
# stop_reason = "tool_use"
```

**Dispatch — dòng 165:**
```python
handler = TOOL_HANDLERS.get("edit_file")           # → run_edit
```

**run_edit() — dòng 93-102:**
```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        file_path = safe_path(path)
        #   safe_path("s02_tool_use/code.py") → OK (trong WORKDIR)

        text = file_path.read_text()
        #   đọc nội dung hiện tại

        if old_text not in text:                    # "timeout=120" in text? → True
            return f"Error: text not found in {path}"

        file_path.write_text(text.replace(old_text, new_text, 1))
        #   text.replace("timeout=120", "timeout=60", 1)
        #   → thay thế "timeout=120" đầu tiên bằng "timeout=60"
        #   → ghi đè file

        return f"Edited {path}"                     # → "Edited s02_tool_use/code.py"

    except Exception as e:
        return f"Error: {e}"
```

**Kết quả:** `"Edited s02_tool_use/code.py"`

---

### Call 3: Model xác nhận → end_turn

```python
# response.content = [
#   {"type": "text", "text": "Đã sửa timeout trong run_bash từ 120s xuống 60s."}
# ]
# stop_reason = "end_turn"

if response.stop_reason != "tool_use":    # "end_turn" != "tool_use" → True
    return                                 # ← THOÁT
```

---

### Tại sao không thể gộp làm 1?

Nếu model gộp `read_file` và `edit_file` trong 1 response:
```python
# Response lý thuyết (nếu model cố gắng):
content = [
    {"type": "tool_use", "name": "read_file", "input": {"path": "code.py"}},
    {"type": "tool_use", "name": "edit_file", "input": {...}},   # ❓ sửa gì?
]
```

- `edit_file` cần `old_text` chính xác (ví dụ `timeout=120`)
- Model **chưa đọc file** → không biết nội dung hiện tại → không thể điền `old_text`
- **Bắt buộc 2 API calls**: đọc → biết nội dung → sửa

---

### Sơ đồ code chạy

```
agent_loop()
  │
  ├── CALL 1
  │   ├── response = tool_use(read_file, path="s02_tool_use/code.py")
  │   ├── handler = TOOL_HANDLERS["read_file"] → run_read
  │   ├── output = run_read(path="s02_tool_use/code.py")
  │   │    └── return nội dung (có "timeout=120")
  │   └── messages += tool_result
  │
  ├── CALL 2
  │   ├── response = tool_use(edit_file, path="...", old_text="timeout=120", new_text="timeout=60")
  │   │                                   └── model biết được từ Call 1
  │   ├── handler = TOOL_HANDLERS["edit_file"] → run_edit
  │   ├── output = run_edit(path="...", old_text="timeout=120", new_text="timeout=60")
  │   │    ├── safe_path → OK
  │   │    ├── read_text() → tìm "timeout=120"
  │   │    ├── replace("timeout=120", "timeout=60")
  │   │    ├── write_text() → ghi đè
  │   │    └── return "Edited s02_tool_use/code.py"
  │   └── messages += tool_result
  │
  └── CALL 3
      ├── response = text → end_turn
      └── return
```

---

## Ví dụ 3: Hỗn hợp — Đọc song song → Ghi tuần tự

**Prompt:** *"Dựa vào file cũ ở s02_tool_use/example.md, viết một file example_summary.md tóm tắt các khái niệm chính. Đọc thêm README.md để biết context."*

---

### Call 1: Model đọc 2 file song song

```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_rd1", "name": "read_file",
#    "input": {"path": "s02_tool_use/example.md"}},
#   {"type": "tool_use", "id": "toolu_rd2", "name": "read_file",
#    "input": {"path": "README.md"}},
# ]
# stop_reason = "tool_use"
```

**Vòng lặp — dòng 162-168:**

Cả 2 block đều là `read_file`:

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)      # → run_read (cả 2 lần)
        output = handler(**block.input)
        # Lần 1: run_read(**{"path": "s02_tool_use/example.md"})
        #   → nội dung example.md cũ
        # Lần 2: run_read(**{"path": "README.md"})
        #   → nội dung README.md
        results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})

messages.append({"role": "user", "content": results})   # gom 2 tool_result → gửi lại model
```

**Trạng thái messages sau Call 1:**
```python
messages = [
    {"role": "user", "content": "Dựa vào file cũ... viết file example_summary.md..."},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_rd1", "name": "read_file",
         "input": {"path": "s02_tool_use/example.md"}},
        {"type": "tool_use", "id": "toolu_rd2", "name": "read_file",
         "input": {"path": "README.md"}},
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_rd1",
         "content": "# Ví dụ API Response S02 — Dispatch Map...\n\n..."},
        {"type": "tool_result", "tool_use_id": "toolu_rd2",
         "content": "# learn-claude-code\n\n## Introduction\n..."},
    ]}
]
```

---

### Call 2: Model đã biết nội dung → ghi file

```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_wr", "name": "write_file",
#    "input": {"path": "s02_tool_use/example_summary.md", "content": "# Tóm tắt S02\n\n...\n- Dispatch map: ...\n- safe_path: ...\n"}}
# ]
# stop_reason = "tool_use"
```

**Dispatch — dòng 165:**
```python
handler = TOOL_HANDLERS.get("write_file")           # → run_write
```

**run_write() — dòng 83-90:**
```python
def run_write(path: str, content: str) -> str:
    try:
        file_path = safe_path(path)
        #   safe_path("s02_tool_use/example_summary.md")
        #   → (WORKDIR / "s02_tool_use/example_summary.md").resolve()
        #   → Path("d:\\MinhTh_code\\learn-claude-code\\s02_tool_use\\example_summary.md")
        #   is_relative_to(WORKDIR)? → True → OK

        file_path.parent.mkdir(parents=True, exist_ok=True)
        #   tạo s02_tool_use/ nếu chưa tồn tại

        file_path.write_text(content)
        #   ghi nội dung summary vào file

        return f"Wrote {len(content)} bytes to {path}"
        #   → "Wrote 456 bytes to s02_tool_use/example_summary.md"

    except Exception as e:
        return f"Error: {e}"
```

---

### Call 3: Model xác nhận → end_turn

```python
# response.content = [
#   {"type": "text", "text": "Đã tạo file example_summary.md dựa trên nội dung example.md cũ và README.md."}
# ]
# stop_reason = "end_turn"

if response.stop_reason != "tool_use":    # "end_turn" != "tool_use" → True
    return                                 # ← THOÁT
```

---

### Sơ đồ code chạy — hỗn hợp song song + tuần tự

```
agent_loop()
  │
  ├── CALL 1 — SONG SONG
  │   ├── response = [
  │   │     tool_use(read_file, path="s02_tool_use/example.md"),
  │   │     tool_use(read_file, path="README.md")
  │   │   ]
  │   │
  │   ├── for block in response.content:                    ← duyệt 2 block
  │   │   ├── handler = TOOL_HANDLERS["read_file"] → run_read
  │   │   │   └── run_read(path="s02_tool_use/example.md")
  │   │   │        → "# Ví dụ API Response S02..."
  │   │   │
  │   │   └── handler = TOOL_HANDLERS["read_file"] → run_read
  │   │       └── run_read(path="README.md")
  │   │            → "# learn-claude-code\n..."
  │   │
  │   └── messages += 2 tool_results
  │
  ├── CALL 2 — TUẦN TỰ (cần kết quả từ Call 1)
  │   ├── response = tool_use(write_file, path="...", content="...")
  │   │                        └── model đã đọc cả 2 file → biết viết gì
  │   ├── handler = TOOL_HANDLERS["write_file"] → run_write
  │   ├── run_write(path="s02_tool_use/example_summary.md", content="...")
  │   │    ├── safe_path → OK
  │   │    ├── mkdir(parents=True) → tạo thư mục
  │   │    ├── write_text(content) → ghi file
  │   │    └── return "Wrote 456 bytes to s02_tool_use/example_summary.md"
  │   └── messages += tool_result
  │
  └── CALL 3
      ├── response = text → end_turn
      └── return
```

---

## So sánh 3 ví dụ — Code path

| Yếu tố | VD1: Song song | VD2: Tuần tự | VD3: Hỗn hợp |
|---|---|---|---|
| **API calls** | 2 | 3 | 3 |
| **Tool calls** | 2 (cùng lúc) | 2 (nối tiếp) | 3 (2 song song + 1 tuần tự) |
| **Dispatch map** | `read_file` → `run_read`; `glob` → `run_glob` | `read_file` → `run_read`; `edit_file` → `run_edit` | `read_file` × 2 → `run_read`; `write_file` → `run_write` |
| **safe_path()** | `run_read` gọi safe_path; `run_glob` tự check | Cả 2 đều gọi safe_path | Cả 3 đều gọi safe_path |
| **Tính song song** | Có — 2 tool không phụ thuộc | Không — phải đọc trước mới sửa được | Cả hai — đọc song song, ghi tuần tự |
| **dòng code dispatch** | `TOOL_HANDLERS.get(block.name)` — dòng 165 | Giống | Giống |
