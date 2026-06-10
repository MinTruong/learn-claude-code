# S01 Agent Loop — Ví dụ chi tiết

## Prompt: "Tạo file hello.py in Hello World"

---

## Call 1: User gửi prompt

**Request gửi đi:**
```json
{
  "model": "claude-sonnet-4-6",
  "system": "You are a coding agent at d:\\MinhTh_code\\learn-claude-code. Use bash to solve tasks. Act, don't explain.",
  "messages": [
    {
      "role": "user",
      "content": "Tạo file hello.py in Hello World"
    }
  ],
  "tools": [
    {
      "name": "bash",
      "description": "Run a shell command.",
      "input_schema": {
        "type": "object",
        "properties": {
          "command": { "type": "string" }
        },
        "required": ["command"]
      }
    }
  ],
  "max_tokens": 8000
}
```

**Response trả về:**
```json
{
  "id": "msg_01ABC...",
  "model": "claude-sonnet-4-6",
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01xyz789",
      "name": "bash",
      "input": {
        "command": "echo 'print(\"Hello, World!\")' > hello.py"
      }
    }
  ],
  "usage": {
    "input_tokens": 185,
    "output_tokens": 42
  }
}
```

**Phân tích:**
- `stop_reason: "tool_use"` — model không trả text, mà yêu cầu chạy lệnh
- `content` chỉ có **1 block** duy nhất: `tool_use`
- Model chọn lệnh `echo 'print("Hello, World!")' > hello.py` — redirect output vào file

**Harness thực thi:**
```
$ echo 'print("Hello, World!")' > hello.py
(no output)
```

**Sau Call 1, messages trông thế này:**
```python
messages = [
    {"role": "user", "content": "Tạo file hello.py in Hello World"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_01xyz789", "name": "bash",
         "input": {"command": "echo 'print(\"Hello, World!\")' > hello.py"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01xyz789",
         "content": "(no output)"}
    ]}
]
```

---

## Call 2: Harness gửi lại với tool result

**Request gửi đi:**
```json
{
  "model": "claude-sonnet-4-6",
  "system": "You are a coding agent at d:\\MinhTh_code\\learn-claude-code. Use bash to solve tasks. Act, don't explain.",
  "messages": [
    {
      "role": "user",
      "content": "Tạo file hello.py in Hello World"
    },
    {
      "role": "assistant",
      "content": [
        {
          "type": "tool_use",
          "id": "toolu_01xyz789",
          "name": "bash",
          "input": { "command": "echo 'print(\"Hello, World!\")' > hello.py" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01xyz789",
          "content": "(no output)"
        }
      ]
    }
  ],
  "tools": [ "...same as above..." ],
  "max_tokens": 8000
}
```

**Response trả về:**
```json
{
  "id": "msg_01DEF...",
  "model": "claude-sonnet-4-6",
  "role": "assistant",
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "Đã tạo file hello.py với nội dung in ra \"Hello, World!\"."
    }
  ],
  "usage": {
    "input_tokens": 310,
    "output_tokens": 28
  }
}
```

**Phân tích:**
- `stop_reason: "end_turn"` — model đã xong, không cần gọi tool nữa
- `content` chỉ có **1 block** duy nhất: `text`
- Model thấy `"(no output)"` (lệnh echo thành công, không có error) — kết luận hoàn thành

**Loop kết thúc.**

---

## Tổng kết 2 API calls

| | Call 1 | Call 2 |
|---|---|---|
| **Input tokens** | 185 | 310 |
| **Output tokens** | 42 | 28 |
| **stop_reason** | `tool_use` | `end_turn` |
| **Content** | 1× tool_use block | 1× text block |
| **Hành động** | `echo ... > hello.py` | Hoàn thành |

**Tổng: ~565 tokens** cho một task đơn giản.

---

## So sánh: Nếu không dùng loop

**1 API call duy nhất, không có tools:**
```json
{
  "messages": [
    { "role": "user", "content": "Tạo file hello.py in Hello World" }
  ]
}
```

**Response:**
```json
{
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "Bạn có thể dùng lệnh sau:\n\necho 'print(\"Hello, World!\")' > hello.py\n\nHoặc tạo file thủ công..."
    }
  ],
  "usage": {
    "input_tokens": 25,
    "output_tokens": 55
  }
}
```

**Tổng: ~80 tokens** — nhưng bạn phải tự chạy lệnh, tự kiểm tra, tự xử lý lỗi.

---

## Trường hợp: Model gọi nhiều tool cùng lúc

Với prompt phức tạp hơn, response có thể chứa **nhiều tool_use blocks cùng lúc**:

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01abc",
      "name": "bash",
      "input": { "command": "mkdir -p src" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01def",
      "name": "bash",
      "input": { "command": "echo 'print(\"Hello\")' > src/hello.py" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01ghi",
      "name": "bash",
      "input": { "command": "python src/hello.py" }
    }
  ]
}
```

Claude Sonnet 4.6 có thể sinh **3-5 tool calls song song** trong 1 response. Harness sẽ chạy tất cả, gom kết quả, rồi gửi lại Call tiếp theo.
