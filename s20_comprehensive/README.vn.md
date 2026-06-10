# s20: Tác Nhân Toàn Diện — Tất cả cơ chế, quy về một vòng lặp

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s18 → s19 → `s20`

> *"Cơ chế rất nhiều, vòng lặp chỉ có một"* — Công cụ, quyền hạn, ký ức, nhiệm vụ, nhóm, plugin đều gắn vào cùng một while True.
>
> **Lớp Harness**: Toàn diện — đặt các cơ chế từ 19 chương trước vào cùng một hệ thống có thể chạy.

---

## Vấn đề

19 chương trước, mỗi chương chỉ thêm một cơ chế. Như vậy thích hợp để học, nhưng tác nhân thực sẽ không chỉ mang một cơ chế để chạy.

Một coding agent có thể làm việc lâu dài cần đồng thời có:

- Phân phối công cụ và ranh giới quyền hạn
- Điểm mở rộng hooks
- Kế hoạch todo và biểu đồ nhiệm vụ
- Kỹ năng, ký ức, lắp ráp system prompt
- Nén và khôi phục lỗi
- Nhiệm vụ nền và lập lịch cron
- Nhóm, giao thức, tự động ghi danh
- Cách ly worktree
- Tiếp cận công cụ bên ngoài MCP

Khó khăn không phải là chất đống chức năng, mà là xem rõ chúng đều gắn vào vị trí nào của vòng lặp. S20 chính là chương kết thúc: đưa tất cả thành phần về đúng vị trí.

---

## Giải pháp

![Kiến trúc hệ thống](images/system-architecture.svg)

S20 không phải phát minh lại một cơ chế mới, mà là tổng hợp các thành phần giáo dục trước thành một harness hoàn chỉnh:

```text
Đầu vào người dùng
  → UserPromptSubmit hooks
  → cron/background thông báo thêm vào
  → nén ngữ cảnh
  → ký ức + kỹ năng + trạng thái MCP lắp ráp system prompt
  → LLM
  → có tool_use block không?
       không → Stop hooks → trả về
       có → PreToolUse hooks + quyền hạn
          → TOOL_HANDLERS / MCP handlers / background dispatch
          → PostToolUse hooks
          → tool_result / task_notification quay lại messages
          → vòng tiếp theo
```

Vòng lặp bản thân vẫn là cùng một cấu trúc: gọi model, kiểm tra xem có xuất hiện `tool_use` block trong phản hồi không, thực thi công cụ, thêm kết quả vào `messages`. Trong mã nguồn CC cũng không tin tưởng trực tiếp `stop_reason == "tool_use"`, mà lấy tool_use block thực sự xuất hiện làm tín hiệu có tiếp tục vòng công cụ không. Sự thay đổi là harness xung quanh vòng lặp trở nên hoàn chỉnh.

---

## Vị trí của thành phần trong vòng lặp

| Vị trí | Thành phần | Tác dụng |
|------|------|------|
| Trước/sau đầu vào người dùng | `UserPromptSubmit` hooks | Ghi lại, thêm vào, kiểm toán đầu vào người dùng |
| Trước LLM | cron queue | Thêm prompt được kích hoạt định kỳ vào `messages` |
| Trước LLM | background notifications | Sau khi nhiệm vụ nền hoàn thành, thêm vào dưới dạng `<task_notification>` |
| Trước LLM | compaction pipeline | Trước nén đầu ra lớn, sau cắt lịch sử, sau nén tool_result cũ, tóm tắt khi cần thiết |
| Trước LLM | memory / skills / MCP state | Lắp ráp system prompt, để model thấy khả năng hiện tại và ngữ cảnh dài hạn |
| Gọi LLM | error recovery | 429/529 thử lại, nâng cấp `max_tokens`, prompt quá dài kích hoạt reactive compact |
| Trước thực thi công cụ | `PreToolUse` hooks + permission | Chặn lệnh nguy hiểm, viết vượt giới hạn, công cụ MCP phá hoại |
| Phân phối công cụ | `assemble_tool_pool` | Lắp ráp công cụ nội tạo và công cụ MCP động |
| Thực thi công cụ | background dispatch | Thao tác bash chậm đưa vào daemon thread, vòng lặp chính trả về kết quả placeholder trước |
| Sau thực thi công cụ | `PostToolUse` hooks | Cảnh báo đầu ra lớn, nhật ký và xử lý sau |
| Quay lại vòng lặp | tool_result | Mỗi `tool_use` tương ứng một `tool_result`, sau đó quay lại vòng tiếp theo |
| Vòng này không có tool_use / dừng lại | `Stop` hooks | Thống kê, dọn dẹp, kiểm toán |

---

## code.py bao gồm những gì

### Công cụ và phân phối

Bể công cụ nội tạo bao gồm 27 công cụ:

```text
bash, read_file, write_file, edit_file, glob
todo_write, task, load_skill, compact
create_task, list_tasks, get_task, claim_task, complete_task
schedule_cron, list_crons, cancel_cron
spawn_teammate, send_message, check_inbox
request_shutdown, request_plan, review_plan
create_worktree, remove_worktree, keep_worktree
connect_mcp
```

`assemble_tool_pool()` lắp ráp mỗi vòng:

```text
BUILTIN_TOOLS + connected MCP tools
BUILTIN_HANDLERS + mcp__server__tool handlers
```

Vì vậy sau `connect_mcp("docs")`, vòng tiếp theo trong bể công cụ sẽ xuất hiện `mcp__docs__search`.

### Quyền hạn và hooks

Quyền hạn không viết cứng vào dòng thực thi công cụ, mà như `PreToolUse` hook:

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append(tool_result(block.id, blocked))
    continue
```

Như vậy permission, log, kiểm toán đều có thể gắn vào cùng một điểm hook. Sau khi thực thi kích hoạt `PostToolUse`.

### Kế hoạch và nhiệm vụ

S20 đồng thời giữ lại hai lớp kế hoạch:

- `todo_write`: Kế hoạch nhẹ trong phiên hiện tại, lưu trong bộ nhớ
- task graph: Tệp nhiệm vụ vượt phiên, có thể phụ thuộc, có thể ghi danh, viết vào `.tasks/task_*.json`

Loại trước giúp Agent đơn lẻ không trôi dạt; loại sau hỗ trợ hợp tác nhóm.

### Sub agent và nhóm

S20 có hai loại delegation:

- `task`: subagent một lần. `messages[]` độc lập, quá trình trung gian bỏ đi, chỉ trả về tóm tắt cuối cùng.
- `spawn_teammate`: luồng đồng nghiệp lâu dài. Thu phát tin nhắn thông qua MessageBus, có thể kiểm tra rảnh rỗi bảng nhiệm vụ và tự động ghi danh.

Subagent một lần giải quyết "cách ly ngữ cảnh"; đồng nghiệp lâu dài giải quyết "hợp tác song song lâu dài".

### Ký ức, kỹ năng và prompt

`assemble_system_prompt(context)` lắp ráp mỗi vòng:

- Nhận dạng và giải thích công cụ
- workspace
- skills catalog
- `.memory/MEMORY.md`
- Đã kết nối MCP server

Kỹ năng chỉ đặt danh mục trong system prompt. Nội dung đầy đủ được tải theo nhu cầu thông qua `load_skill(name)`.

### Nén và khôi phục

Trước LLM chạy pipeline nén trước:

```text
tool_result_budget → snip_compact → micro_compact → compact_history
```

Khi gọi model lại bao thêm một lớp khôi phục:

- 429: Thử lại với backoff theo cấp số nhân
- 529: Thử lại với backoff theo cấp số nhân, thất bại liên tiếp có thể chuyển sang fallback model
- `max_tokens`: Trước nâng max_tokens, sau yêu cầu continuation
- prompt quá dài: reactive compact sau đó thử lại

### Nền và cron

Thao tác bash chậm sẽ không chặn vòng lặp chính:

```text
should_run_background → start_background_task → placeholder tool_result
Nền hoàn thành → task_notification → vòng tiếp theo thêm vào messages
```

Bộ lập lịch cron daemon thread độc lập kiểm tra mỗi giây. CLI sẽ giám sát `cron_queue`, khi trúng chủ động thêm `[Scheduled] ...` và chạy một vòng Agent.

### Worktree và MCP

worktree chịu trách nhiệm cách ly thư mục:

- `create_worktree(name, task_id)` tạo nhánh và thư mục độc lập
- Trường `worktree` của task liên kết thư mục
- Đồng nghiệp ghi danh task có worktree, bash/read/write tự động thực thi trong thư mục tương ứng

MCP chịu trách nhiệm khả năng bên ngoài:

- `connect_mcp(name)` kết nối mock server
- `assemble_tool_pool()` lắp ráp công cụ MCP vào bể công cụ
- Tên công cụ thống nhất là `mcp__server__tool`

---

## Thay đổi so với s19

| Thành phần | s19 | s20 |
|------|-----|-----|
| Bể công cụ | Nội tạo + MCP | Nội tạo + MCP, bổ sung công cụ từ s01-s18 |
| Quyền hạn | Chủ thể giáo dục bỏ qua | Thực thi trong `PreToolUse` hook |
| hooks | Bỏ qua | UserPromptSubmit / PreToolUse / PostToolUse / Stop |
| todo | Bỏ qua | `todo_write` + reminder |
| skill | Bỏ qua | catalog in system prompt + `load_skill` |
| compact | Bỏ qua | Nén trước LLM + công cụ `compact` + reactive compact |
| error recovery | Đơn giản hóa try/except | thử lại / max_tokens / prompt quá dài |
| background | Bỏ qua | Thread nền thao tác chậm + task notification |
| cron | Bỏ qua | daemon scheduler + durable jobs |
| multi-agent | Giữ lại | Giữ lại; đồng nghiệp sử dụng công cụ cơ bản trong thư mục cách ly |
| worktree | Giữ lại | Giữ lại |
| MCP | Thêm mới | Giữ lại, như một phần của bể công cụ cuối cùng |

---

## Thử một lần

```sh
cd learn-claude-code
python s20_comprehensive/code.py
```

Có thể thử:

1. `Create a todo list for inspecting this repo, then list Python files`
2. `Connect to the docs MCP server and search for agent loop`
3. `Create two tasks, create worktrees for them, then spawn alice and bob. Ask them to submit plans before claiming tasks.`
4. `remind me of the meeting in 3 minutes.`
5. `Run npm install in the background and continue reading README.md`

Các điểm quan trọng để quan sát:

- Trước khi gọi công cụ có qua hooks/permission không
- Sau `connect_mcp` vòng tiếp theo có xuất hiện công cụ MCP không
- Thao tác chậm có trả về background placeholder không
- Đến giờ có tự động nhắc họp không
- Đồng nghiệp có nộp plan không, và có tạm dừng trước khi được approval không
- Sau khi plan được chấp thuận, đồng nghiệp có thể ghi danh nhiệm vụ không
- Sau khi liên kết worktree, đồng nghiệp có chuyển sang thư mục tương ứng không

---

## Kết thúc cũng là bắt đầu

Từ s01 đến s20, bề mặt code ngày càng phức tạp, nhưng cốt lõi luôn không thay đổi:

```python
while True:
    response = LLM(messages, tools)
    if not has_tool_use(response.content):
        return
    results = execute_tools(response.content)
    messages.append(tool_results)
```

Độ phức tạp của Claude Code không phải là "một bộ não agent khác", mà là độ phức tạp của một harness trưởng thành. Model chịu trách nhiệm đánh giá và lựa chọn hành động; harness chịu trách nhiệm tổ chức môi trường, công cụ, quyền hạn, ký ức, nhóm và khả năng bên ngoài.

Đây là điểm kết thúc của toàn bộ sách: cơ chế rất nhiều, vòng lặp chỉ có một.
