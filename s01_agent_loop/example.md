# Ví dụ API Response S01 — Agent Loop (vòng lặp tool)

Mỗi ví dụ dưới đây sẽ trace từng bước từ **model trả response** → **code nào chạy** → **kết quả trả về**.

---

## Ví dụ 1: Single tool — Tạo file hello.py

**Prompt:** *"Tạo file hello.py in Hello World"*

---

### Bước 1: Model trả response — stop_reason = "tool_use"

```python
# Code — agent_loop() dòng 85-113
def agent_loop(messages: list):
    while True:                                    # dòng 86
        response = client.messages.create(          # dòng 87-90
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
            max_tokens=8000,
        )
```

**Request gửi lên API:**
```json
{
  "model": "claude-sonnet-4-6",
  "system": "You are a coding agent at d:\\MinhTh_code\\learn-claude-code. Use bash to solve tasks. Act, don't explain.",
  "messages": [
    {"role": "user", "content": "Tạo file hello.py in Hello World"}
  ],
  "tools": [{"name": "bash", "description": "Run a shell command.", "input_schema": {
    "type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]
  }}],
  "max_tokens": 8000
}
```

**Response trả về:**
```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_01xyz", "name": "bash",
#    "input": {"command": "echo 'print(\"Hello, World!\")' > hello.py"}}
# ]
# response.stop_reason = "tool_use"
```

**Vòng lặp kiểm tra stop_reason — dòng 93-97:**
```python
    # Append assistant turn
    messages.append({"role": "assistant", "content": response.content})  # dòng 93

    # Nếu model không gọi tool → kết thúc
    if response.stop_reason != "tool_use":    # "tool_use" != "tool_use" → False
        return                                 # → KHÔNG return, tiếp tục loop
```

---

### Bước 2: Vòng lặp xử lý block — gọi run_bash — dòng 100-113

```python
    # Thực thi từng tool call, gom kết quả
    results = []
    for block in response.content:               # block = tool_use của bash
        if block.type == "tool_use":             # "tool_use" == "tool_use" → True
            print(f"\033[33m$ {block.input['command']}\033[0m")
            #   in ra: $ echo 'print("Hello, World!")' > hello.py

            output = run_bash(block.input["command"])
            #   block.input["command"] = "echo 'print(\"Hello, World!\")' > hello.py"
```

**Bên trong run_bash() — dòng 69-81:**
```python
def run_bash(command: str) -> str:
    # command = "echo 'print(\"Hello, World!\")' > hello.py"

    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):    # không chứa lệnh nguy hiểm → False
        return "Error: Dangerous command blocked"

    try:
        r = subprocess.run(
            command,                             # "echo 'print(...)' > hello.py"
            shell=True, cwd=os.getcwd(),
            capture_output=True, text=True, timeout=120,
        )                                        # dòng 74-75
        #   subprocess chạy lệnh echo → tạo file hello.py
        #   stdout = "" (echo redirect vào file, không in ra terminal)
        #   stderr = ""
        out = (r.stdout + r.stderr).strip()      # out = ""
        return out[:50000] if out else "(no output)"   # → "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
    except (FileNotFoundError, OSError) as e:
        return f"Error: {e}"
```

**Quay lại vòng lặp — gom kết quả:**
```python
            print(output[:200])                   # in "(no output)"
            results.append({                      # dòng 106-110
                "type": "tool_result",
                "tool_use_id": block.id,          # "toolu_01xyz"
                "content": output,                # "(no output)"
            })

    # Feed tool results back, loop continues
    messages.append({"role": "user", "content": results})  # dòng 113
```

---

### Bước 3: Trạng thái messages sau Call 1

```python
messages = [
    {"role": "user", "content": "Tạo file hello.py in Hello World"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_01xyz", "name": "bash",
         "input": {"command": "echo 'print(\"Hello, World!\")' > hello.py"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01xyz",
         "content": "(no output)"}
    ]}
]
```

---

### Bước 4: Vòng lặp chạy tiếp — Call 2, model kết thúc

```python
# while True (dòng 86) — lần lặp thứ 2
response = client.messages.create(         # dòng 87-90
    model=MODEL, system=SYSTEM,
    messages=messages,                     # messages đã có 3 phần
    tools=TOOLS, max_tokens=8000,
)
# response:
#   stop_reason = "end_turn"
#   content = [{"type": "text", "text": "Đã tạo file hello.py với nội dung in ra \"Hello, World!\"."}]

messages.append({"role": "assistant", "content": response.content})  # dòng 93

if response.stop_reason != "tool_use":    # "end_turn" != "tool_use" → True
    return                                 # ← THOÁT vòng lặp while True
```

---

### Sơ đồ toàn bộ code chạy

```
agent_loop(messages)
  │
  ├── VÒNG LẶP #1
  │   ├── client.messages.create(...)               → dòng 87
  │   │    ├── gửi: [user prompt]
  │   │    ├── stop_reason = "tool_use"
  │   │    └── content = [tool_use(bash)]
  │   │
  │   ├── messages.append(assistant, content)       → dòng 93
  │   ├── "tool_use" == "tool_use"? → ở lại loop    → dòng 96
  │   │
  │   ├── for block in content:
  │   │    └── run_bash("echo... > hello.py")       → dòng 104
  │   │         ├── subprocess.run(...)              → dòng 74
  │   │         └── return "(no output)"
  │   │
  │   └── messages.append(user, tool_result)        → dòng 113
  │
  ├── VÒNG LẶP #2
  │   ├── client.messages.create(...)               → dòng 87
  │   │    ├── gửi: [user, assistant, tool_result]
  │   │    ├── stop_reason = "end_turn"
  │   │    └── content = [text]
  │   │
  │   ├── "end_turn" != "tool_use"? → True
  │   └── return                                     → THOÁT
  │
  └── entry point in ra text cuối cùng
```

---

## Ví dụ 2: Multiple tools — 3 lệnh bash trong 1 response

**Prompt:** *"Tạo thư mục src, tạo file test.py trong đó, chạy file đó"*

Model thấy 3 việc độc lập → gom làm 3 tool_use blocks trong 1 response.

---

### Bước 1: Model trả 3 tool_use blocks

```python
# response.content = [
#   {"type": "tool_use", "id": "toolu_a", "name": "bash",
#    "input": {"command": "mkdir -p src"}},
#   {"type": "tool_use", "id": "toolu_b", "name": "bash",
#    "input": {"command": "cat > src/test.py << 'EOF'\nprint('Hello from test')\nEOF"}},
#   {"type": "tool_use", "id": "toolu_c", "name": "bash",
#    "input": {"command": "python src/test.py"}},
# ]
# stop_reason = "tool_use"
```

---

### Bước 2: Vòng lặp duyệt 3 block — dòng 101-113

```python
results = []
for block in response.content:   # duyệt lần lượt 3 blocks
    if block.type == "tool_use":

        # ── Block 1: mkdir ──
        print("$ mkdir -p src")
        output = run_bash("mkdir -p src")
        #   → subprocess.run("mkdir -p src") → src/ được tạo
        #   → stdout="" → stderr="" → return "(no output)"
        print("(no output)")
        results.append({"type": "tool_result", "tool_use_id": "toolu_a", "content": "(no output)"})

        # ── Block 2: cat file ──
        print("$ cat > src/test.py ...")
        output = run_bash("cat > src/test.py << 'EOF'\nprint('Hello from test')\nEOF")
        #   → tạo file src/test.py với nội dung print(...)
        #   → stdout="" → return "(no output)"
        print("(no output)")
        results.append({"type": "tool_result", "tool_use_id": "toolu_b", "content": "(no output)"})

        # ── Block 3: chạy Python ──
        print("$ python src/test.py")
        output = run_bash("python src/test.py")
        #   → python thực thi file → stdout = "Hello from test\n" → .strip() → "Hello from test"
        #   → out có nội dung → return out[:50000]
        print("Hello from test")
        results.append({"type": "tool_result", "tool_use_id": "toolu_c", "content": "Hello from test"})

# Gom 3 kết quả — dòng 113
messages.append({"role": "user", "content": results})
```

---

### Bước 3: Trạng thái messages

```python
messages = [
    {"role": "user", "content": "Tạo thư mục src, tạo file test.py trong đó, chạy file đó"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_a", "name": "bash",
         "input": {"command": "mkdir -p src"}},
        {"type": "tool_use", "id": "toolu_b", "name": "bash",
         "input": {"command": "cat > src/test.py << 'EOF'\nprint('Hello from test')\nEOF"}},
        {"type": "tool_use", "id": "toolu_c", "name": "bash",
         "input": {"command": "python src/test.py"}},
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_a", "content": "(no output)"},
        {"type": "tool_result", "tool_use_id": "toolu_b", "content": "(no output)"},
        {"type": "tool_result", "tool_use_id": "toolu_c", "content": "Hello from test"},
    ]}
]
```

---

### Sơ đồ: 3 tool trong 1 vòng lặp

```
agent_loop()
  │
  ├── response.content = [bash#1, bash#2, bash#3]    (1 API call, 3 blocks)
  │   stop_reason = "tool_use"
  │
  ├── for block in response.content:
  │    ├── Block 1: run_bash("mkdir -p src")
  │    │              → "(no output)"
  │    ├── Block 2: run_bash("cat > src/test.py ...")
  │    │              → "(no output)"
  │    └── Block 3: run_bash("python src/test.py")
  │                   → "Hello from test"
  │
  ├── 3 tool_result → messages.append(user, results)   → dòng 113
  │
  ├── Call 2: model thấy kết quả của cả 3 lệnh
  │    └── stop_reason = "end_turn" → return
```

---

## Ví dụ 3: Model không cần tool — 1 lần gọi API duy nhất

**Prompt:** *"Viết cho tôi một đoạn Python in Hello World"*

---

### Bước 1: Model trả text thuần — stop_reason = "end_turn"

```python
# response.content = [
#   {"type": "text", "text": "Bạn có thể dùng lệnh:\n\necho 'print(\"Hello, World!\")' > hello.py\n\nHoặc đơn giản hơn:\n```python\nprint(\"Hello, World!\")\n```"}
# ]
# response.stop_reason = "end_turn"
```

---

### Bước 2: Vòng lặp phát hiện kết thúc — dòng 93-97

```python
messages.append({"role": "assistant", "content": response.content})   # dòng 93

if response.stop_reason != "tool_use":    # "end_turn" != "tool_use" → True
    return                                 # ← THOÁT NGAY, for loop không chạy
```

**Kết quả:** Chỉ 1 API call. Không tool nào được gọi. Loop chạy 1 lần rồi thoát.

---

### So sánh 3 ví dụ

| Yếu tố | VD1: 1 tool | VD2: 3 tool (1 response) | VD3: Không tool |
|---|---|---|---|
| **Vòng lặp** | 2 lần | 2 lần | 1 lần (thoát ngay) |
| **API calls** | 2 | 2 | 1 |
| **Tool calls** | 1 | 3 | 0 |
| **stop_reason đầu** | `tool_use` | `tool_use` | `end_turn` |
| **Lần 1** | tool_use → chạy bash | 3× tool_use → chạy 3 bash | end_turn → return |
| **Lần 2** | end_turn → return | end_turn → return | — |
| **run_bash() có chạy?** | ✅ 1 lần | ✅ 3 lần | ❌ không |
