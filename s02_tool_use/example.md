# Ví dụ API Response S02 — Dispatch Map & Tool Execution

## Ví dụ 1: Song song — Nhiều tool độc lập trong 1 response

**Prompt:** *"Đọc file README.md và tìm tất cả file Python trong thư mục này"*

### Call 1: User gửi prompt

**Request gửi đi:**
```json
{
  "model": "claude-sonnet-4-6",
  "system": "You are a coding agent at d:\\MinhTh_code\\learn-claude-code. Use tools to solve tasks. Act, don't explain.",
  "messages": [
    {
      "role": "user",
      "content": "Đọc file README.md và tìm tất cả file Python trong thư mục này"
    }
  ],
  "tools": [
    {"name": "bash", ...},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "glob", "description": "Find files matching a glob pattern.",
     "input_schema": {"type": "object", "properties": {"pattern": {"type": "string"}}, "required": ["pattern"]}}
  ],
  "max_tokens": 8000
}
```

**Response trả về:**
```json
{
  "id": "msg_01ABC...",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01read",
      "name": "read_file",
      "input": {"path": "README.md"}
    },
    {
      "type": "tool_use",
      "id": "toolu_01glob",
      "name": "glob",
      "input": {"pattern": "**/*.py"}
    }
  ],
  "usage": {"input_tokens": 250, "output_tokens": 68}
}
```

> **Model gọi 2 tool cùng lúc** — `read_file` và `glob` độc lập, không cần đợi nhau.

**Harness thực thi (qua dispatch map):**
```python
handler = TOOL_HANDLERS["read_file"]   # → run_read
output = run_read(**{"path": "README.md"})

handler = TOOL_HANDLERS["glob"]        # → run_glob
output = run_glob(**{"pattern": "**/*.py"})
```

**Messages sau Call 1:**
```python
messages = [
    {"role": "user", "content": "Đọc file README.md và tìm tất cả file Python..."},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_01read", "name": "read_file",
         "input": {"path": "README.md"}},
        {"type": "tool_use", "id": "toolu_01glob", "name": "glob",
         "input": {"pattern": "**/*.py"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01read",
         "content": "# learn-claude-code\n\n..."},
        {"type": "tool_result", "tool_use_id": "toolu_01glob",
         "content": "s01_agent_loop/code.py\ns02_tool_use/code.py\ntests/..."}
    ]}
]
```

### Call 2: Model tổng hợp

**Response:**
```json
{
  "id": "msg_01DEF...",
  "stop_reason": "end_turn",
  "content": [{
    "type": "text",
    "text": "Dự án này là repo học tập về harness engineering...\n\nCác file Python:\n- s01_agent_loop/code.py\n- s02_tool_use/code.py\n..."
  }],
  "usage": {"input_tokens": 1200, "output_tokens": 85}
}
```

### Dòng chảy
```
read_file ──┐
            ├──> gom kết quả → LLM → end_turn
glob ───────┘
```

---

## Ví dụ 2: Tuần tự — Tool trước cần kết quả để gọi tool sau

**Prompt:** *"Đọc file code.py, tìm hàm run_bash, sửa timeout từ 120s thành 60s"*

### Call 1: Đọc file trước

**Response:**
```json
{
  "stop_reason": "tool_use",
  "content": [{
    "type": "tool_use",
    "id": "toolu_01read",
    "name": "read_file",
    "input": {"path": "s02_tool_use/code.py"}
  }]
}
```

**Kết quả:** Model đọc được nội dung file, thấy `timeout=120` trong `run_bash`.

### Call 2: Đã biết nội dung → gọi edit_file

```json
{
  "stop_reason": "tool_use",
  "content": [{
    "type": "tool_use",
    "id": "toolu_01edit",
    "name": "edit_file",
    "input": {
      "path": "s02_tool_use/code.py",
      "old_text": "timeout=120",
      "new_text": "timeout=60"
    }
  }]
}
```

**Kết quả:** `"Edited s02_tool_use/code.py"`

### Call 3: Xác nhận → kết thúc

```json
{
  "stop_reason": "end_turn",
  "content": [{
    "type": "text",
    "text": "Đã sửa timeout trong run_bash từ 120s xuống 60s."
  }]
}
```

### Dòng chảy
```
read_file → LLM → edit_file → LLM → end_turn
    ↑                        ↑
cần kết quả đọc        cần kết quả sửa
để biết sửa gì         để xác nhận
```

---

## Ví dụ 3: Meta — Model viết file dựa vào thông tin đã đọc

**Prompt:** *"Dựa vào 2 ví dụ trên viết cho tôi file example.md"*

### Call 1: Đọc 2 example.md để hiểu cấu trúc

Model cần biết 2 file cũ viết gì, format ra sao → gọi `read_file` cho cả 2.

**Response:**
```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01ex1",
      "name": "read_file",
      "input": {"path": "s01_agent_loop/example.md"}
    },
    {
      "type": "tool_use",
      "id": "toolu_01ex2",
      "name": "read_file",
      "input": {"path": "s02_tool_use/example.md"}
    }
  ]
}
```

**Kết quả song song:**
```
toolu_01ex1 → "# S01 Agent Loop — Ví dụ chi tiết\n\n..."
toolu_01ex2 → "# Ví dụ API Response S02 — Dispatch Map...\n\n..."
```

### Call 2: Đã biết cấu trúc → ghi file mới

Model đã đọc cả 2 example, biết format, biết nội dung cần viết → gọi `write_file`.

**Response:**
```json
{
  "stop_reason": "tool_use",
  "content": [{
    "type": "tool_use",
    "id": "toolu_01write",
    "name": "write_file",
    "input": {
      "path": "s02_tool_use/example_combined.md",
      "content": "# Ví dụ kết hợp S01 & S02\n\n..."
    }
  }]
}
```

### Call 3: Xác nhận hoàn thành

```json
{
  "stop_reason": "end_turn",
  "content": [{
    "type": "text",
    "text": "Đã tạo file example_combined.md dựa trên cấu trúc từ 2 file example trước đó."
  }]
}
```

### Dòng chảy
```
Call 1 (song song):  read_file(s01/example.md)
                      read_file(s02/example.md)     → LLM hiểu cấu trúc
Call 2 (tuần tự):                                    → write_file(file mới) → LLM xác nhận
Call 3:
```

---

## So sánh 3 dạng

| | Song song (VD1) | Tuần tự (VD2) | Hỗn hợp (VD3) |
|---|---|---|---|
| **Mô tả** | 2 tool độc lập trong 1 turn | 1 tool/turn, kết quả quyết định tool sau | Đọc song song → ghi tuần tự |
| **Số tool calls** | 2 | 2 | 3 (2 song song + 1 tuần tự) |
| **Số API call** | 1 | 2 | 2 |
| **Thứ tự quan trọng?** | Không | Có — phải đọc trước mới sửa được | Có — phải đọc trước mới viết được |
| **Tổng token (ước)** | ~1,600 | ~1,800 | ~2,000 |
