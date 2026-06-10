# s17: Tác Nhân Tự Chủ — Bảng công việc của riêng mình, tự ghi danh

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s15 → s16 → `s17` → [s18](../s18_worktree_isolation/) → s19 → s20

> *"Bảng công việc của riêng mình, tự ghi danh"* — Khi rảnh rỡi thì kiểm tra, có việc thì làm.
>
> **Lớp Harness**: Tự chủ — Các đồng nghiệp tự tổ chức, không phụ thuộc vào sự phân công của Lead.

---

## Vấn đề

Ở s16, các đồng nghiệp có thể giao tiếp, có thể bắt tay tắt máy. Nhưng mỗi đồng nghiệp đều chờ Lead phân công nhiệm vụ — nếu bảng nhiệm vụ có 10 nhiệm vụ chưa được ghi danh, Lead phải ghi danh 10 lần. Điều này không thể mở rộng. Các đồng nghiệp nên tự xem bảng nhiệm vụ, phát hiện nhiệm vụ không có ai làm thì tự ghi danh, sau khi hoàn thành thì tìm nhiệm vụ tiếp theo.

---

## Giải pháp

![Tổng quan Tác Nhân Tự Chủ](images/autonomous-agents-overview.svg)

Sử dụng lại MessageBus và công cụ giao thức từ S16. Chương này thêm mới: **idle_poll**（kiểm tra mỗi 5 giây khi rảnh）、**scan_unclaimed_tasks**（quét bảng để tìm nhiệm vụ có thể ghi danh）、**tự động ghi danh**（tìm được nhiệm vụ thì ghi danh, không phải Lead lo).

Vòng đời của đồng nghiệp từ hai giai đoạn thành ba giai đoạn:

| Giai đoạn | Hành động | Điều kiện thoát |
|------|------|---------|
| WORK | inbox → LLM → vòng lặp công cụ | `stop_reason != tool_use` |
| IDLE | mỗi 5s kiểm tra inbox + bảng nhiệm vụ | 60s quá thời |
| SHUTDOWN | gửi tóm tắt, thoát | — |

---

## Nguyên tắc hoạt động

### idle_poll: kiểm tra khi rảnh

Khi đồng nghiệp hoàn thành nhiệm vụ hiện tại, họ không thoát mà vào giai đoạn IDLE — kiểm tra mỗi 5 giây có công việc mới nào không:

```python
IDLE_POLL_INTERVAL = 5   # seconds
IDLE_TIMEOUT = 60         # seconds

def idle_poll(agent_name, messages, name, role) -> str:
    """Return 'work', 'shutdown', or 'timeout'."""
    for _ in range(IDLE_TIMEOUT // IDLE_POLL_INTERVAL):
        time.sleep(IDLE_POLL_INTERVAL)

        # ① Kiểm tra hộp thư (ưu tiên)
        inbox = BUS.read_inbox(agent_name)
        if inbox:
            # shutdown_request xử lý ngay
            for msg in inbox:
                if msg.get("type") == "shutdown_request":
                    # ... trả lời shutdown_response
                    return "shutdown"
            # Tin nhắn bình thường được thêm vào bối cảnh, quay lại WORK
            messages.append(...)
            return "work"

        # ② Quét bảng nhiệm vụ
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            task = unclaimed[0]
            result = claim_task(task["id"], agent_name)
            if "Claimed" in result:
                messages.append(...)
                return "work"
    return "timeout"
```

Hộp thư được ưu tiên (có thể chứa shutdown_request và các tin nhắn giao thức khác), bảng nhiệm vụ được ưu tiên tiếp theo. Giai đoạn IDLE nhận được shutdown_request sẽ trả lời ngay và thoát, không đợi vòng lặp WORK tiếp theo.

### scan_unclaimed_tasks: quét bảng nhiệm vụ

Tìm kiếm các nhiệm vụ có trạng thái pending, không có chủ sở hữu, tất cả các phụ thuộc đã hoàn thành (`can_start`):

```python
def scan_unclaimed_tasks() -> list[dict]:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and can_start(task["id"])):
            unclaimed.append(task)
    return unclaimed
```

Ba điều kiện: phải là pending, không có chủ sở hữu, tất cả các phụ thuộc blockedBy đã hoàn thành. `can_start` kiểm tra trạng thái của các nhiệm vụ phụ thuộc — có phụ thuộc không có nghĩa là không thể làm, chỉ khi bị các nhiệm vụ chưa hoàn thành chặn thì không thể làm. Phiên bản giáo dục sắp xếp theo tên tệp và lấy cái đầu tiên; CC sử dụng khóa tệp để ngăn chặn nhiều đồng nghiệp ghi danh cùng một nhiệm vụ.

### claim_task: kiểm tra chủ sở hữu

Khi tự động ghi danh, hãy kiểm tra kết quả ghi danh, không coi thất bại là thành công:

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    if task.owner:
        return f"Task {task_id} already owned by {task.owner}"
    if not can_start(task_id):
        return f"Blocked by: {deps}"
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
    return f"Claimed {task.id} ({task.subject})"
```

Phiên bản giáo dục không có khóa tệp, ghi danh đồng thời có thể gây tranh chấp. Nhưng ít nhất kiểm tra `task.owner` tránh được vấn đề "ghi lên sau" rõ ràng nhất. CC sử dụng `proper-lockfile` để bảo vệ tệp nhiệm vụ, `claimTask` hoàn thành đọc-sửa-ghi trong khóa tệp (`utils/tasks.ts:541-612`).

### Vòng đời của đồng nghiệp: WORK → IDLE → SHUTDOWN

Ở s16, đồng nghiệp hoàn thành nhiệm vụ thì thoát. s17 thêm giai đoạn IDLE, đồng nghiệp lặp lại WORK → IDLE trong vòng lặp bên ngoài:

```python
# Vòng lặp ngoài: vòng lặp WORK → IDLE
while True:
    # Giai đoạn WORK: vòng lặp bên trong (tối đa 10 lần gọi LLM)
    for _ in range(10):
        # Kiểm tra inbox, xử lý tin nhắn giao thức, gọi LLM, thực thi công cụ
        ...
        if response.stop_reason != "tool_use":
            break  # Giai đoạn WORK kết thúc

    # Giai đoạn IDLE
    idle_result = idle_poll(name, messages, name, role)
    if idle_result == "shutdown":
        break
    if idle_result == "timeout":
        break  # 60s quá thời → SHUTDOWN

# SHUTDOWN: gửi tóm tắt cho Lead
BUS.send(name, "lead", summary, "result")
```

Các thiết kế chính:
- **Vòng lặp ngoài while True**: WORK và IDLE xen kẽ nhau, cho đến khi quá thời hoặc nhận được yêu cầu tắt máy
- **Vòng lặp bên trong for 10**: giai đoạn WORK tối đa 10 lần gọi LLM (ngăn chặn vòng lặp vô hạn)
- **IDLE quá thời 60 giây**: 12 lần kiểm tra × 5 giây = 60 giây. Sau quá thời thì gửi tóm tắt và thoát
- **shutdown_request có thể phản hồi ở cả hai giai đoạn**: giai đoạn WORK thông qua `handle_inbox_message` phân phối; giai đoạn IDLE `idle_poll` kiểm tra trực tiếp và trả lời

### Tái nhập lại bản sắc

Sau autoCompact (s08), danh sách messages của đồng nghiệp có thể được nén thành một tóm tắt. Mỗi khi vào giai đoạn WORK mới, hãy kiểm tra:

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}. "
                   f"Continue your work.</identity>"})
```

Nếu tin nhắn quá ngắn có nghĩa là đã xảy ra nén, lúc này tái nhập lại thông tin bản sắc. Trong CC thực tế, context compaction sẽ giữ lại system prompt, phiên bản giáo dục đơn giản hóa cần xử lý thủ công.

### consume_lead_inbox: hộp thư Lead thống nhất

Công cụ `check_inbox` và cuối vòng lặp chính đều gọi cùng một hàm `consume_lead_inbox()`: trước hết định tuyến các phản hồi giao thức để cập nhật trạng thái, sau đó thêm tất cả tin nhắn vào lịch sử cuộc trò chuyện của Lead. Tóm tắt/kết quả được gửi từ đồng nghiệp không chỉ được in ra ở bảng điều khiển, mà LLM của Lead có thể thấy và điều phối bước tiếp theo.

### Chạy cùng nhau

```
1. Lead: "Xây dựng backend — quá nhiều nhiệm vụ, hãy để đồng nghiệp tự ghi danh"
2. Lead → create_task("Tạo schema cơ sở dữ liệu")
3. Lead → create_task("Viết đường dẫn API")
4. Lead → create_task("Viết bài kiểm tra đơn vị")
5. Lead → spawn_teammate("alice", "backend", "Bạn là nhà phát triển backend")
6. Lead → spawn_teammate("bob", "backend", "Bạn là nhà phát triển backend")

7. Thread alice khởi động → WORK: không có inbox ban đầu → không có gì → IDLE
8. Thread bob khởi động → WORK: không có inbox ban đầu → không có gì → IDLE

9. alice IDLE lần quét thứ 1 → scan_unclaimed → phát hiện "Tạo schema cơ sở dữ liệu"
10. alice → claim_task → "Tạo schema cơ sở dữ liệu" → quay lại WORK
11. bob IDLE lần quét thứ 1 → scan_unclaimed → phát hiện "Viết đường dẫn API"
12. bob → claim_task → "Viết đường dẫn API" → quay lại WORK

13. alice WORK: write_file("schema.sql", ...) → complete_task → WORK kết thúc
14. alice IDLE → scan → "Viết bài kiểm tra đơn vị" → ghi danh → WORK
15. alice WORK: write_file("test_api.py", ...) → complete_task → WORK kết thúc
16. alice IDLE → 60s không có nhiệm vụ mới → SHUTDOWN

17. bob quá trình tương tự → hoàn thành → SHUTDOWN
18. Lead consume_lead_inbox → xem tóm tắt của alice và bob
```

Hai đồng nghiệp song song ghi danh, song song làm việc. Lead chỉ cần tạo nhiệm vụ và khởi động đồng nghiệp, không cần phân công thủ công.

---

## Thay đổi so với s16

| Thành phần | Trước đó (s16) | Sau đó (s17) |
|------|-----------|-----------|
| Phân công nhiệm vụ | Lead ghi danh thủ công | Đồng nghiệp tự động ghi danh (kiểm tra can_start phụ thuộc) |
| Trạng thái đồng nghiệp | WORK hoặc thoát | WORK → IDLE (kiểm tra 60s) → SHUTDOWN |
| claim_task | không kiểm tra chủ sở hữu | từ chối nhiệm vụ đã có chủ sở hữu |
| Tắt máy giai đoạn IDLE | không xử lý shutdown_request | phân phối shutdown trực tiếp và thoát |
| Lead inbox | chỉ in, không vào bối cảnh | consume_lead_inbox thêm thống nhất vào lịch sử |
| Hàm mới | — | idle_poll, scan_unclaimed_tasks, consume_lead_inbox |
| Duy trì bản sắc | chỉ system prompt | tự động tái nhập lại sau nén |
| Công cụ Lead | 14 (s16) | 14 (không thay đổi) |
| Công cụ đồng nghiệp | 5 | 8 (+ list_tasks, claim_task, complete_task) |
| Điều kiện thoát đồng nghiệp | Hoàn thành nhiệm vụ thì thoát | 60s không có nhiệm vụ mới thì thoát |

---

## Thử một lần

```sh
cd learn-claude-code
python s17_autonomous_agents/code.py
```

Thử prompt này:

`Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim and work.`

Các điểm quan trọng để quan sát: Đồng nghiệp có tự động ghi danh nhiệm vụ chưa phân công không? Nhiệm vụ có phụ thuộc blockedBy có được ghi danh chính xác sau khi phần phụ thuộc hoàn thành không? Sau khi rảnh quá thời có tự động tắt máy không? Giai đoạn IDLE nhận được shutdown_request có phản hồi ngay không? Trạng thái nhiệm vụ trong thư mục `.tasks/` thay đổi như thế nào?

---

## Tiếp theo

Đồng nghiệp tự tổ chức rồi. Nhưng Alice và Bob đều làm việc trong cùng một thư mục — Alice sửa `config.py`, Bob cũng sửa `config.py`, họ viết đè lên nhau.

s18 Worktree Isolation → mỗi nhiệm vụ có thư mục làm việc riêng, không can thiệp lẫn nhau.

<details>
<summary>Sâu vào mã nguồn CC</summary>

> Lưu ý giáo dục: cơ chế idle_poll + auto-claim của chương này là thiết kế giáo dục, sử dụng hàm kiểm tra thống nhất để trình diễn "tìm công việc khi rảnh". Việc thực hiện thực tế của CC là sự kết hợp của nhiều cơ chế, nhưng mục tiêu là giống nhau — giảm bớt gánh nặng phân công thủ công của Lead.

### Một, cơ chế rảnh rỡi của CC: kết hợp các đường dẫn, không phải kiểm tra duy nhất

Phiên bản giáo dục sử dụng một `idle_poll()` thống nhất để xử lý kiểm tra hộp thư rảnh rỡi và ghi danh nhiệm vụ. Việc thực hiện thực tế của CC là sự kết hợp của bốn cơ chế:

**idle_notification**: Sau khi đồng nghiệp hoàn thành một vòng làm việc, `sendIdleNotification()`（`inProcessRunner.ts:569-589`）gửi thông báo rảnh rỡi cho Lead. Lead biết rằng đồng nghiệp có sẵn, có thể phân công nhiệm vụ mới hoặc yêu cầu tắt máy.

**mailbox polling**: `waitForNextPromptOrShutdown()`（`inProcessRunner.ts:689-868`）là một **vòng lặp kiểm tra 500ms**, liên tục kiểm tra ba nguồn: pending user messages, tin nhắn tệp mailbox, danh sách nhiệm vụ. shutdown_request được xử lý ưu tiên（`inProcessRunner.ts:768-804`），sẽ không bị các tin nhắn thông thường đói kém.

**task watcher**: `useTaskListWatcher`（`hooks/useTaskListWatcher.ts:34-189`）sử dụng `fs.watch()` để giám sát thay đổi thư mục `.claude/tasks/`, debounce 1 giây, khi có nhiệm vụ mới hoặc phụ thuộc mở khóa thì kích hoạt kiểm tra. Phán xét phụ thuộc（`L197-207`）là "blockedBy không có nhiệm vụ chưa hoàn thành", không phải "blockedBy trống".

**chủ động claim**: bên trong vòng lặp kiểm tra cũng sẽ gọi `tryClaimNextTask()`（`inProcessRunner.ts:853-860`）— chủ động lấy nhiệm vụ từ danh sách nhiệm vụ trong thời gian chờ. Vì vậy, "đồng nghiệp không chủ động kiểm tra nhiệm vụ" là không chính xác, CC có cả thông báo bị động và ghi danh chủ động.

### Hai, ghi danh nhiệm vụ: khóa tệp + hoạt động nguyên tử

`claimTask()`（`utils/tasks.ts:541-612`）sử dụng khóa tệp nhiệm vụ của `proper-lockfile`, hoàn thành đọc-kiểm tra-sửa-ghi trong khóa. Các mục kiểm tra: chủ sở hữu đã tồn tại（`L575-576`），đã hoàn thành（`L580-581`），blockedBy có nhiệm vụ chưa hoàn thành（`L585-594`）. `claimTaskWithBusyCheck()`（`utils/tasks.ts:614-692`）sử dụng khóa cấp danh sách nhiệm vụ, làm cho busy check và ghi danh trở thành hoạt động nguyên tử, tránh TOCTOU.

`findAvailableTask()`（`inProcessRunner.ts:595-604`）phán xét phụ thuộc cũng là "tất cả blockedBy đã hoàn thành", được triển khai bằng `task.blockedBy.every(id => !unresolvedTaskIds.has(id))`. `tryClaimNextTask()`（`inProcessRunner.ts:624-657`）cập nhật trạng thái thành `in_progress` sau khi ghi danh, cho phép giao diện người dùng phản ánh thay đổi ngay lập tức.

### Ba, so sánh phiên bản giáo dục vs CC

| Chiều | Phiên bản giáo dục (s17) | CC |
|------|-------------|-----|
| Cơ chế rảnh | idle_poll kiểm tra thống nhất (5s) | idle_notification + kiểm tra mailbox 500ms + task watcher |
| Khám phá nhiệm vụ | scan_unclaimed_tasks (kiểm tra) | useTaskListWatcher (giám sát tệp) + tryClaimNextTask (kiểm tra chủ động) |
| Phán xét phụ thuộc | can_start (tất cả blockedBy đã hoàn thành) | findAvailableTask (ngữ nghĩa giống nhau) |
| An toàn đồng thời | kiểm tra chủ sở hữu (không có khóa tệp) | khóa nhiệm vụ proper-lockfile + khóa danh sách nhiệm vụ |
| Xử lý shutdown | phân phối trực tiếp IDLE, xử lý WORK qua handle_inbox_message | xử lý ưu tiên shutdown_request trong vòng lặp kiểm tra 500ms |
| Quá thời thoát | 60s không có nhiệm vụ mới | không quá thời cố định, Lead tắt máy thủ công |
| Duy trì bản sắc | phát hiện độ dài messages | context compaction giữ lại system prompt |
| Xử lý thất bại claim | kiểm tra giá trị trả về, thất bại không thêm | khóa tệp đảm bảo tính nguyên tử |

Phiên bản giáo dục `idle_poll()` kết hợp bốn cơ chế của CC thành một hàm kiểm tra — đơn giản hóa hợp lý, vì ngữ nghĩa cốt lõi（rảnh rỡi tìm công việc, phụ thuộc mở khóa có thể ghi danh, shutdown ưu tiên）là nhất quán.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
