# s02: Tool Use — Thêm một tool chỉ cần thêm một handler

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → `s02` → [s03](../s03_permission/) → s04 → ... → s20
> *"Thêm một tool, chỉ thêm một handler"* — loop không cần động; tool mới chỉ cần register vào dispatch map.
>
> **Harness layer**: tool dispatch — mở rộng ranh giới mà model có thể chạm tới.

---

## Chỉ có bash thì quá chật

Agent ở s01 chỉ có một tool: bash. Đọc file phải `cat`, viết file phải `echo "..." > file.py`, sửa file phải `sed`.

Model muốn làm rất đơn giản: "đọc file này". Nhưng để làm được, nó phải biến ý định đó thành `cat path/to/file`. Thêm một lớp dịch tay, vừa tốn token, vừa dễ sai.

---

## Góc nhìn toàn cục: tool dispatch

![Tool Dispatch](images/tool-dispatch.svg)

Loop của s01 được giữ nguyên: gọi LLM, kiểm tra `stop_reason`, append message. Thay đổi duy nhất nằm ở đúng một dòng thực thi tool: thay `run_bash()` bằng lookup qua `TOOL_HANDLERS[block.name]()`.

Thêm một tool cho Agent chỉ cần hai việc:

1. **Định nghĩa tool**: thêm một mô tả vào mảng `TOOLS`
2. **Register handler**: thêm một mapping vào dict `TOOL_HANDLERS`

---

## Từ 1 tool lên 5 tools

s01 chỉ có bash:

```python
TOOLS = [{"name": "bash", ...}]

def run_bash(command): ...
```

s02 tăng lên 5 tools, mỗi tool được định nghĩa độc lập:

```python
TOOLS = [
    {"name": "bash",       "description": "Run a shell command.", ...},
    {"name": "read_file",  "description": "Read file contents.",  ...},
    {"name": "write_file", "description": "Write content to file.", ...},
    {"name": "edit_file",  "description": "Replace text in file once.", ...},
    {"name": "glob",       "description": "Find files by pattern.", ...},
]
```

Mỗi tool có hàm implementation riêng:

```python
def run_read(path, limit=None):
    lines = safe_path(path).read_text().splitlines()
    if limit:
        lines = lines[:limit]
    return "\n".join(lines)

def run_write(path, content):
    safe_path(path).write_text(content)
    return f"Wrote {len(content)} bytes to {path}"

def run_edit(path, old_text, new_text):
    text = safe_path(path).read_text()
    if old_text not in text:
        return "Error: text not found"
    safe_path(path).write_text(text.replace(old_text, new_text, 1))
    return f"Edited {path}"

def run_glob(pattern):
    import glob as g
    return "\n".join(g.glob(pattern, root_dir=WORKDIR))
```

---

## Tool dispatch

```python
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
    "edit_file":  run_edit,
    "glob":       run_glob,
}

# Trong loop chỉ đổi một dòng: từ hard-code run_bash sang lookup:
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS[block.name]    # lookup
        output = handler(**block.input)         # call
        results.append(...)
```

Thêm một tool = thêm một entry trong `TOOLS` + thêm một dòng trong `TOOL_HANDLERS`. Loop không đổi.

---

## Nhiều tool calls

Model thường trả về nhiều `tool_use` trong một lần: "đọc a.py và b.py, rồi liệt kê tất cả file .py".

Bản giảng dạy chạy tuần tự theo thứ tự gốc của `response.content`. CC phức tạp hơn: cắt theo continuous batches theo thứ tự gốc, trong batch tool nào concurrency-safe thì chạy song song, giữa các batch vẫn giữ thứ tự nghiêm ngặt (xem appendix).

---

## Quick notes

| Khái niệm | Một câu |
|------|--------|
| TOOL_HANDLERS | Dict tool name → handler. Thêm tool = thêm một mapping |
| Tool definition | JSON schema nói cho model biết "tôi làm được gì" |
| Nhiều tool calls | Model có thể trả về nhiều `tool_use` cùng lúc; bản giảng dạy chạy tuần tự theo thứ tự |
| Loop không đổi | `while True` của s01 không sửa một dòng nào |

---

## Thay đổi so với s01

| Thành phần | Trước (s01) | Sau (s02) |
|------|-----------|-----------|
| Số tool | 1 (bash) | 5 (+read, write, edit, glob) |
| Tool execution | Hard-code `run_bash()` | Lookup qua `TOOL_HANDLERS` |
| Path safety | Không có | `safe_path` check (chỉ cho file tools) |
| Loop | `while True` + `stop_reason` | Giống hệt s01 |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s02_tool_use/code.py
```

Thử các prompt:

1. `Đọc file README.md và nói cho tôi dự án này nói về gì`
2. `Tạo file test.py in ra "hello", rồi đọc lại`
3. `Tìm tất cả file Python trong thư mục này`
4. `Đọc cả README.md và requirements.txt, rồi tạo một file tóm tắt`

Quan sát trọng tâm: Khi nào model chỉ gọi một tool, khi nào gọi nhiều tool cùng lúc? Thứ tự và kết quả của nhiều tool calls có đúng không?

---

## Tiếp theo

Giờ Agent đã có 5 tools chuyên dụng. File tools được `safe_path` bảo vệ, nhưng bash vẫn không bị giới hạn, `rm -rf /` vẫn có thể chạy.

s03 Permission → thêm một cánh cửa trước khi tool chạy: thao tác này an toàn không? Có cần user approve không?

<details>
<summary>Đi sâu vào CC source code</summary>

> Nội dung dựa trên kiểm tra CC source code `Tool.ts`, `tools.ts`, `toolOrchestration.ts`, `toolExecution.ts`, `StreamingToolExecutor.ts`.

### 1. Cách định nghĩa tool

**Bản giảng dạy**: mảng `TOOLS` + dict `TOOL_HANDLERS`. Định nghĩa và implementation tách riêng.
**CC**: mỗi tool là object riêng do `buildTool()` tạo, bao gồm schema, validation, permission, execution. `getAllBaseTools()` gom tất cả tools.

Cách tách của bản giảng dạy rõ hơn cho việc học -- người đọc thấy ngay "thêm tool = thêm hai chỗ định nghĩa".

### 2. Kiểm tra concurrency safety: `isConcurrencySafe()`

![Tool Concurrency](images/concurrency-comparison.svg)

Bản giảng dạy chạy tuần tự theo thứ tự gốc, không concurrency. CC dùng `isConcurrencySafe(input)` để quyết định có chạy song song được không -- lưu ý đây không đơn giản là "read vs write", mà xét theo input cụ thể:

| | isReadOnly | isConcurrencySafe |
|---|---|---|
| FileRead | true | true |
| Glob | true | true |
| Bash `ls` | true | **true** ← khác biệt chính |
| Bash `rm` | false | false |
| TaskCreate | false | **true** ← đổi state nhưng vẫn chạy song song được (TaskCreate xuất hiện ở s12) |

`isConcurrencySafe` của Bash tool trong CC bằng với `isReadOnly` -- lệnh chỉ đọc có thể song song, lệnh write thì không. TaskCreate tuy sửa file task, nhưng mỗi lần ghi vào file khác nhau nên vẫn có thể song song.

### 3. Partition algorithm

`partitionToolCalls()` của CC (`toolOrchestration.ts:91-115`) không chia thành 2 nhóm đơn giản, mà chia tool calls theo **continuous blocks**:

```
[read A, read B, glob *.py, bash "rm x", read C]
  → batch1(concurrent): [read A, read B, glob *.py]
  → batch2(serial): [bash "rm x"]
  → batch3(concurrent): [read C]
```

Các tool call liên tiếp mà concurrency-safe sẽ vào cùng một batch, bên trong batch chạy song song thật (`toolOrchestration.ts:152-176`, có concurrency limit). Gặp tool không concurrency-safe thì mở batch mới và chạy serial. Giữa các batch luôn giữ thứ tự nghiêm ngặt.

### 4. Validation pipeline

Mỗi tool call trong CC đi qua pipeline 5 bước nghiêm ngặt (`toolExecution.ts`):

1. **Zod schema validation** (`614-680`, bản giảng dạy dùng JSON Schema thay thế): kiểm tra kiểu/cấu trúc argument
2. **Tool-level `validateInput()`** (`682-733`): kiểm tra giá trị argument (ví dụ path có nằm trong workspace không)
3. **PreToolUse hooks** (`800-862`, s04 giải thích chi tiết): hook có thể trả về message, sửa input, chặn execution
4. **Permission check** (`921-931`, nội dung cốt lõi của s03): `canUseTool` + `checkPermissions` → allow/deny/ask
5. **Execute `tool.call()`** (`1207-1222`)

Bản giảng dạy bỏ qua Zod (dùng JSON Schema), bỏ qua `validateInput` (dùng safe function), nhưng giữ khái niệm permission check và hook.

### 5. Streaming tool execution

`StreamingToolExecutor` của CC (`StreamingToolExecutor.ts`) cho phép tool bắt đầu chạy khi model vẫn đang generate output -- không cần đợi model nói xong. `read_file` có thể chạy xong ngay khi model còn đang in "để tôi phân tích". Bản giảng dạy không implement phần này, mục tiêu giống s01 -- concept clear, không chase performance tối đa.

### 6. Tool result persistence

Mỗi tool có field `maxResultSizeChars`. Nếu result vượt ngưỡng này thì ghi ra disk, model chỉ nhìn thấy preview + file path. FileRead là ngoại lệ -- set thành `Infinity`, để tránh output của đọc file lại bị coi là file và ghi xuống disk. Cụ thể, nếu FileRead result vượt threshold và bị ghi file, model lần sau đọc file đó lại vượt threshold → ghi tiếp → loop vô hạn (đọc file → ghi disk → đọc lại → ghi lại → ...).

</details>

<!-- translation-sync: zh@v1, en@v0, ja@v0, vi@v1 -->
