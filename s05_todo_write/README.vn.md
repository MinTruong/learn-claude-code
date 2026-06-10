# s05: TodoWrite — Agent không có plan thì rất dễ lệch hướng

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → s03 → s04 → `s05` → [s06](../s06_subagent/) → s07 → ... → s20

> *"Agent không có plan thì sẽ đi đâu hay đó"* — liệt kê các bước trước khi bắt tay vào làm, task dài sẽ ít bị sót hơn.
>
> **Harness layer**: planning — giúp Agent nghĩ rõ trước khi hành động.

---

## Vấn đề

Giao cho Agent một task khá phức tạp: "Đổi toàn bộ file Python sang cách đặt tên snake_case, sau đó chạy test và sửa những chỗ bị fail."

Agent bắt đầu làm, sửa 3 file, chạy test, thấy 2 chỗ fail, rồi lao vào sửa tiếp. Sửa một lúc, nó quên mất mục tiêu ban đầu là "đổi sang snake_case". Toàn bộ attention bị hút sang chuyện sửa test.

Càng nói chuyện lâu thì chuyện này càng tệ: tool result liên tục đổ vào context, ảnh hưởng của system prompt bị loãng dần. Một refactor 10 bước, làm xong bước 1-3 là Agent bắt đầu improvise, vì bước 4-10 đã bị đẩy khỏi vùng chú ý từ lâu.

---

## Giải pháp

![Todo Overview](images/todo-overview.svg)

Giữ nguyên hook structure tối thiểu từ chương trước, nhưng thêm một tool mới là `todo_write` cùng một cơ chế reminder. `todo_write` không làm việc thực tế nào cả: không đọc file, không chạy command. Việc duy nhất của nó là buộc Agent phải làm rõ mình định làm gì trước khi bắt đầu.

Dispatch mechanism không đổi, tool mới vẫn đi qua `TOOL_HANDLERS[block.name]`. Để minh họa reminder, loop được gắn thêm một counter: nếu 3 rounds liên tiếp không gọi `todo_write` thì hệ thống tự inject một lời nhắc.

---

## Cách hoạt động

### `todo_write` tool

Tool này nhận vào một danh sách task có state, lưu trong process memory, đồng thời hiển thị progress ở terminal:

```python
CURRENT_TODOS: list[dict] = []

def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    CURRENT_TODOS = todos

    lines = ["\n## Current Tasks"]
    for t in CURRENT_TODOS:
        icon = {"pending": " ", "in_progress": "▸", "completed": "✓"}[t["status"]]
        lines.append(f"  [{icon}] {t['content']}")
    print("\n".join(lines))
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

Tool definition được thêm chung với 5 tool còn lại trong dispatch map:

```python
TOOLS = [
    {"name": "bash",       ...},
    {"name": "read_file",  ...},
    {"name": "write_file", ...},
    {"name": "edit_file",  ...},
    {"name": "glob",       ...},
    # s05: thêm mới
    {"name": "todo_write", "description": "Create and manage a task list ...",
     "input_schema": {
         "type": "object",
         "properties": {
             "todos": {
                 "type": "array",
                 "items": {
                     "type": "object",
                     "properties": {
                         "content": {"type": "string"},
                         "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]},
                     },
                 },
             },
         },
     },
    },
]

TOOL_HANDLERS["todo_write"] = run_todo_write
```

### Nag reminder

Khi model liên tiếp 3 rounds không gọi `todo_write`, hệ thống sẽ tự inject một reminder. Đây là cơ chế chỉ có trong bản giảng dạy; CC source code không có rule cố định kiểu “3 rounds”.

```python
if rounds_since_todo >= 3 and messages:
    messages.append({
        "role": "user",
        "content": "<reminder>Update your todos.</reminder>",
    })
    rounds_since_todo = 0
```

Flow điển hình là:

1. Agent gọi `todo_write` để liệt kê tất cả steps (ban đầu đều là `pending`)
2. Bắt đầu một step → đổi thành `in_progress`
3. Làm xong → đổi thành `completed`
4. Chuyển sang step `pending` tiếp theo
5. Tiếp tục cho đến khi hoàn tất

Nếu 3 rounds liên tiếp không cập nhật TODO, loop sẽ append reminder trước lần LLM call tiếp theo.

**Điểm quan trọng**: `todo_write` không tăng **execution ability** nào cho Agent. Nó chỉ tăng **planning ability**.

---

## Thay đổi so với s04

| Thành phần | Trước (s04) | Sau (s05) |
|-----------|-------------|-----------|
| Số tool | 5 (bash, read, write, edit, glob) | 6 (+todo_write) |
| Planning ability | Không có | TODO list có state + nag reminder |
| SYSTEM prompt | Generic | Thêm hướng dẫn "plan trước khi execute" |
| Loop | Không đổi | Dispatch không đổi, thêm `rounds_since_todo` counter và reminder injection |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s05_todo_write/code.py
```

Thử các prompt sau:

1. `Refactor s05_todo_write/example/hello.py: add type hints, docstrings, and a main guard` (hãy xem Agent có liệt kê 3 bước trước khi làm không)
2. `Create a Python package under s05_todo_write/example/demo_pkg with __init__.py, utils.py, and tests/test_utils.py`
3. `Review Python files under s05_todo_write/example and fix any style issues`

Điểm cần quan sát: lần tool call đầu tiên có phải là `todo_write` không? Danh sách TODO có bao nhiêu bước? Trong lúc execute, status có chuyển từ `pending` → `in_progress` → `completed` không?

---

## Tiếp theo

Giờ Agent đã biết lên plan. Nhưng nếu task quá lớn, ví dụ như "refactor toàn bộ authentication module", chỉ TODO list thôi là chưa đủ. Một task như vậy thực chất là tập hợp của rất nhiều subtask; nếu giữ hết trong một hội thoại, context sẽ nhanh chóng bị tràn và lẫn lộn.

s06 Subagent → tách big task thành subtasks, mỗi subtask giao cho một Agent độc lập. Mỗi subagent có clean context riêng, không pollute lẫn nhau.

<details>
<summary>Đi sâu vào CC source code</summary>

CC có hai task system tồn tại song song (`tasks.ts:133-139`):

- **TodoWrite (V1)**: một list tool đơn giản, dữ liệu giữ trong AppState memory (`TodoWriteTool.ts:65-103`). Bản giảng dạy cũng giữ trong process memory, exit là mất.
- **Task System (V2 = s12)**: file persistence, dependency graph, concurrency lock, ownership.

Việc chuyển đổi do `isTodoV2Enabled()` điều khiển. Logic hiện tại trong source code: V2 mặc định bật trong interactive session, còn non-interactive session (SDK) mặc định dùng V1; set biến môi trường `CLAUDE_CODE_ENABLE_TASKS` có thể force bật V2. Lưu ý comment trong source code "Force-enable tasks in non-interactive mode" mô tả mục đích của env var path, chứ không phải semantics của nhánh mặc định, nên khi đọc phải tách hai thứ đó ra.

Bản giảng dạy bỏ qua field `activeForm` có trong source code thật (`utils/todo/types.ts:8-15`). CC dùng field này để UI hiển thị spinner kiểu “đang làm gì”, còn bản giảng dạy chỉ có terminal output nên không cần.

Nag reminder của bản giảng dạy (3 rounds chưa update thì inject nhắc) là một teaching mechanism. CC source code không có logic “3 rounds” cố định; gần nhất là `TodoWriteTool.ts:72-107`, khi có 3 hoặc nhiều todos đã hoàn thành nhưng không có verification item thì tool sẽ append thêm một verification nudge.

Task System so với TodoWrite có những bước tiến chính:
- File persistence (dưới Claude config directory `tasks/{taskListId}/{taskId}.json`) thay vì in-memory list
- `blockedBy` dependency graph thay vì flat list
- `proper-lockfile` để concurrency-safe thay vì không khóa
- Bốn tool độc lập (Create / Get / Update / List) thay vì một tool duy nhất
- TaskCreated / TaskCompleted hooks (`TaskCreateTool.ts:80-129`, `TaskUpdateTool.ts:231-260`) để tích hợp với external system

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
