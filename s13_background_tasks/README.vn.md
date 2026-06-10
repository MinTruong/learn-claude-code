# s13: Background Tasks — Đưa các thao tác chậm ra nền

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s11 → s12 → `s13` → [s14](../s14_cron_scheduler/) → s15 → ... → s20

> *"Vứt thao tác chậm ra nền, agent tiếp tục xử lý"* — luồng nền chạy lệnh, xong thì chèn thông báo.
>
> **Tầng Harness**: nền — thực thi bất đồng bộ, không chặn vòng lặp chính.

---

## Vấn đề

Bạn từng dùng máy giặt chưa? Ném quần áo vào, bấm nút chạy, rồi đi làm việc khác — nấu ăn, trả lời tin nhắn, đọc paper. 30 phút sau máy giặt "bíp bíp" nhắc bạn: xong rồi. Bạn sẽ không đứng chờ trước máy giặt suốt 30 phút.

Bash tool của Agent cũng vậy. `pip install torch` mất 10 phút, `npm run build` mất 3 phút. Khi các lệnh đó chạy, Agent chỉ biết đứng chờ bash trả về, không thể tận dụng khoảng thời gian đó để xử lý việc khác.

Đọc file là mức mili giây, không cần chờ. `git status` trả về trong chưa đầy một giây, cũng không cần chờ. Nhưng `npm install` thì ở mức phút. Agent chờ 10 phút không làm gì cả, trong khi LLM lại tính phí theo token, chạy rỗng là lãng phí.

---

## Giải pháp

![Background Tasks Overview](images/background-tasks-overview.svg)

Code giảng dạy tiếp tục dùng hệ thống tác vụ giản lược của s12 và cơ chế lắp prompt; để tập trung vào background tasks, nó lược bỏ phần phục hồi lỗi đầy đủ, memory và skill system. Thay đổi duy nhất: các thao tác chậm được ném vào luồng nền, Agent tiếp tục chạy loop, và khi nền chạy xong thì chèn thông báo vào cuộc hội thoại.

Đồng bộ vs nền:

| | Đồng bộ (s12) | Nền (s13) |
|---|---|---|
| Thao tác chậm | Agent đứng chờ | Luồng nền thực thi |
| Agent rảnh | Có | Không, nó tiếp tục xử lý |
| Kết quả | Trả về ngay | Chèn thông báo ở vòng sau |
| Tiêu chí quyết định | — | Tham số `run_in_background` (model yêu cầu rõ ràng), heuristic làm phương án dự phòng |

---

## Nguyên lý hoạt động

### should_run_background: yêu cầu tường minh ưu tiên trước, heuristic chỉ là dự phòng

Model có thể dùng tham số `run_in_background` của bash tool để yêu cầu chạy nền một cách rõ ràng. Nếu model không chỉ định, bản giảng dạy dùng heuristic từ khóa để dự phòng:

```python
def is_slow_operation(tool_name: str, tool_input: dict) -> bool:
    """Fallback heuristic: commands likely to take > 30s."""
    if tool_name != "bash":
        return False
    cmd = tool_input.get("command", "").lower()
    slow_keywords = ["install", "build", "test", "deploy", "compile",
                     "docker build", "pip install", "npm install",
                     "cargo build", "pytest", "make"]
    return any(kw in cmd for kw in slow_keywords)

def should_run_background(tool_name: str, tool_input: dict) -> bool:
    """Model explicit request takes priority; fallback to heuristic."""
    if tool_input.get("run_in_background"):
        return True
    return is_slow_operation(tool_name, tool_input)
```

Trong CC, schema của bash tool có tham số `run_in_background: boolean` (`BashTool.tsx:241`). Model tự quyết định lệnh nào nên đẩy ra nền, không dựa vào việc đoán qua từ khóa. Bản giảng dạy vẫn giữ heuristic làm đường lui, nhưng đường chính là model yêu cầu tường minh.

### start_background_task: thực thi nền và vòng đời

Ta bọc lời gọi tool vào một hàm worker, rồi ném nó vào luồng daemon để chạy. Mỗi background task có một ID duy nhất, và trạng thái được lưu trong từ điển `background_tasks`:

```python
_bg_counter = 0
background_tasks: dict[str, dict] = {}   # bg_id → {tool_use_id, command, status}
background_results: dict[str, str] = {}   # bg_id → output
background_lock = threading.Lock()

def start_background_task(block) -> str:
    """Run tool in a daemon thread. Returns background task ID."""
    global _bg_counter
    _bg_counter += 1
    bg_id = f"bg_{_bg_counter:04d}"

    def worker():
        result = execute_tool(block)
        with background_lock:
            background_tasks[bg_id]["status"] = "completed"
            background_results[bg_id] = result

    with background_lock:
        background_tasks[bg_id] = {
            "tool_use_id": block.id,
            "command": block.input.get("command", ""),
            "status": "running",
        }
    thread = threading.Thread(target=worker, daemon=True)
    thread.start()
    return bg_id
```

Hàm này trả về `bg_id`, chứ không chỉ trả về kiểu `[Running in background...]`. `daemon=True` đảm bảo khi tiến trình Agent thoát thì luồng cũng thoát theo. Bản giảng dạy dùng từ điển trong bộ nhớ để theo dõi trạng thái; CC thật có `LocalShellTaskState`, redirect output ra file, hỗ trợ stop task, đọc output tiếp theo và quản lý vòng đời đầy đủ hơn.

### collect_background_results: thu thập thông báo

Sau khi background task hoàn thành, ta thu kết quả và định dạng nó thành thông báo `<task_notification>`:

```python
def collect_background_results() -> list[str]:
    """Collect completed results as task_notification messages."""
    with background_lock:
        ready_ids = [bid for bid, task in background_tasks.items()
                     if task["status"] == "completed"]
    notifications = []
    for bg_id in ready_ids:
        with background_lock:
            task = background_tasks.pop(bg_id)
            output = background_results.pop(bg_id, "")
        notifications.append(
            f"<task_notification>\n"
            f"  <task_id>{bg_id}</task_id>\n"
            f"  <status>completed</status>\n"
            f"  <command>{task['command']}</command>\n"
            f"  <summary>{output[:200]}</summary>\n"
            f"</task_notification>")
    return notifications
```

Thông báo này không tái sử dụng `tool_use_id` gốc. Tool call ban đầu đã được phản hồi bằng một `tool_result` placeholder, còn việc hoàn tất ở nền là một sự kiện độc lập, nên được chèn dưới dạng `task_notification`. Điều này phù hợp với ngữ nghĩa ghép cặp tool của Messages API: một `tool_use` chỉ tương ứng với một `tool_result`.

### Tích hợp vào vòng lặp

Trong `agent_loop`, thực thi công cụ đi theo hai nhánh, còn thông báo và kết quả được hợp nhất thành một message của user:

```python
results = []
for block in response.content:
    if block.type != "tool_use":
        continue
    if should_run_background(block.name, block.input):
        bg_id = start_background_task(block)
        results.append({"type": "tool_result",
            "tool_use_id": block.id,
            "content": f"[Background task {bg_id} started] "
                       f"Result will be available when complete."})
    else:
        output = execute_tool(block)
        results.append({"type": "tool_result",
            "tool_use_id": block.id, "content": output})

# Hợp thông báo và kết quả công cụ vào cùng một message user
user_content = []
bg_notifications = collect_background_results()
if bg_notifications:
    for notif in bg_notifications:
        user_content.append({"type": "text", "text": notif})
user_content.extend(results)
messages.append({"role": "user", "content": user_content})
```

Thao tác chậm trước tiên trả về một `tool_result` placeholder có `bg_id`, để LLM biết lệnh đó vẫn đang chạy và có thể đi làm việc khác trước. Khi nền hoàn tất, thông báo sẽ được chèn như một text block độc lập, rồi cùng với `tool_result` của lượt hiện tại tạo thành message user.

Bản giảng dạy sẽ poll kết quả nền khi agent loop tiếp tục chạy. CC thật dùng notification queue (`messageQueueManager.ts`) để đẩy sự kiện hoàn thành nền vào các turn kế tiếp, không cần đợi vòng lặp công cụ.

### Chạy toàn bộ cùng nhau

```
Turn 1:
  LLM → bash "npm install" (run_in_background=true)
  → start_background_task → bg_0001
  → tool_result: "[Background task bg_0001 started]..."
  → LLM: "OK, I'll check later. Let me also read the config."

Turn 2:
  LLM → read_file "package.json" (nhanh, sync)
  → tool_result: file content
  → collect: bg_0001 done! inject <task_notification>
  → LLM sees: config file + install notification in one message
```

Agent không đứng chờ vô ích; trong lúc `npm install` chạy nền, nó đã đi đọc file cấu hình.

---

## Thay đổi so với s12

| Thành phần | Trước đó (s12) | Sau đó (s13) |
|------|-----------|-----------|
| Mô hình thực thi | Tất cả đồng bộ | Thao tác chậm chạy luồng nền + chèn thông báo |
| Bash schema | `command` | `command` + `run_in_background` |
| Hàm mới | — | `should_run_background`, `is_slow_operation`, `start_background_task`, `collect_background_results` |
| Kiểu mới | — | `background_tasks: dict`, `background_results: dict`, `background_lock: Lock` |
| Định dạng thông báo | — | `<task_notification>` (không tái sử dụng `tool_use_id`) |
| Hành vi vòng lặp | Công cụ chạy tuần tự | Chậm thì async, nhanh thì sync, thông báo được thu mỗi vòng |
| Công cụ | 8 (s12) | 8 (không đổi, chỉ đổi chiến lược thực thi) |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s13_background_tasks/code.py
```

Thử các prompt sau:

1. `Chạy pip list ở chế độ nền và tìm tất cả file Python trong thư mục này`
2. `Chạy npm install (dùng run_in_background) và trong lúc chờ thì đọc package.json`
3. `Tạo một task để setup project, sau đó chạy pip list ở chế độ nền`

Quan sát trọng tâm: Thao tác chậm có được đẩy ra nền không? Có trả về `bg_id` không? Thông báo nền có được chèn dưới dạng `<task_notification>` không?

---

## Tiếp theo

Background tasks giải quyết được chuyện "thao tác chậm không làm block". Nhưng nếu muốn làm việc gì đó theo lịch thì sao? Ví dụ: "mỗi sáng 9 giờ chạy test", hay "cứ 5 phút kiểm tra trạng thái server một lần".

s14 Cron Scheduler → gắn cho Agent một chiếc đồng hồ báo thức.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Nội dung dưới đây dựa trên phân tích đầy đủ mã nguồn CC `query.ts` (211, 1054-1060, 1411-1482), `services/toolUseSummary/toolUseSummaryGenerator.ts` (L15 prompt text), `LocalShellTask.tsx` (L24-25 hằng số, L59-98 logic watchdog), `messageQueueManager.ts` (notification queue), `utils/task/framework.ts` (L267 `enqueueTaskNotification`).

### 1. pendingToolUseSummary: Haiku sinh nền

Sau mỗi batch tool execution, CC khởi động một side-query bằng Haiku để tạo tóm tắt việc dùng công cụ. Mã khởi chạy nằm ở `query.ts:1411-1482`, còn prompt nằm ở `services/toolUseSummary/toolUseSummaryGenerator.ts:15` (biến `TOOL_USE_SUMMARY_SYSTEM_PROMPT`). Prompt yêu cầu kiểu: "Write a short summary label... think git-commit-subject, not sentence", dùng thì quá khứ, khoảng 30 ký tự.

Tóm tắt do Haiku sinh (~1s) sẽ hoàn tất trong lúc model chính vẫn đang stream output (5-30s). Trước khi vòng tiếp theo bắt đầu, phần tóm tắt này sẽ được yield ra. SDK dùng chúng để hiển thị tiến độ trên thiết bị di động.

### 2. Mô hình thread: thực ra không có thread thật

CC chạy trên Node.js/Bun event loop đơn luồng. Cái gọi là "background" thực chất là "không await". `ShellCommand.background(taskId)` redirect stdout/stderr ra file để tiến trình có thể chạy độc lập.

### 3. Bảy loại background task

CC định nghĩa 7 loại background task (`Task.ts:7-13`): `local_bash`, `local_agent`, `remote_agent`, `in_process_teammate`, `local_workflow`, `monitor_mcp`, `dream`. Mỗi loại có cơ chế đăng ký, vòng đời và thông báo riêng.

### 4. Chèn thông báo: hàng đợi lệnh

Sau khi background task xong, nó được đưa vào hàng đợi lệnh chung qua `enqueueTaskNotification` (`utils/task/framework.ts:267`) hoặc `enqueuePendingNotification` (`messageQueueManager.ts`). Định dạng thông báo là XML có cấu trúc:

```xml
<task_notification>
  <status>completed</status>
  <summary>Background command "npm test" completed (exit code 0)</summary>
</task_notification>
```

Mức ưu tiên được chia thành `next` > `later` (`messageQueueManager.ts`). Background tasks mặc định là `later` (không chặn user input). Điểm tiêu thụ nằm ở `query.ts:1566-1593`.

### 5. Watchdog chống treo

Background bash task có watchdog (`LocalShellTask.tsx` dòng hằng số L24-25, logic L59-98), định kỳ kiểm tra output có bị đứng không. Nếu 45 giây không tăng thêm output, nó sẽ kiểm tra xem có prompt tương tác kiểu `(y/n)` hay không, để tránh nền bị kẹt ở hộp thoại tương tác không ai trả lời.

### 6. Giới hạn đồng thời

Lời gọi tool foreground: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (mặc định 10 tool an toàn song song). Background bash task: không có hard limit, vì chúng là các tiến trình con độc lập.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
