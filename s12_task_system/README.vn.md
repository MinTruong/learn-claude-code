# s12: Task System — Mục tiêu quá lớn, hãy tách thành các tác vụ nhỏ

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s10 → s11 → `s12` → [s13](../s13_background_tasks/) → s14 → ... → s20

> *"Mục tiêu lớn tách thành tác vụ nhỏ, sắp xếp thứ tự, lưu bền vững"* — đồ thị tác vụ được lưu bằng file, là nền tảng cho cộng tác đa agent.
>
> **Tầng Harness**: tác vụ — mục tiêu được lưu bền vững, tiến độ có thể phục hồi.

---

## Vấn đề

Agent nhận một dự án: dựng database, viết API, thêm test. Nó dùng TodoWrite của s05 để liệt kê một danh sách, rồi bắt đầu viết API, viết nửa chừng mới phát hiện chưa có bảng database nên quay lại bổ sung; đến lúc thêm test thì lại phát hiện chữ ký API đã đổi...

Xây nhà không thể dựng mái trước rồi mới đổ móng. Giữa các tác vụ có quan hệ trước sau. Quan hệ phụ thuộc giữa các tác vụ nên tạo thành một đồ thị có hướng không chu trình (DAG); bản giảng dạy chỉ minh họa kiểm tra `blockedBy`, chưa triển khai phát hiện vòng.

TodoWrite ở s05 là danh sách thực thi cho **nhiệm vụ hiện tại**, được lưu trong bộ nhớ phiên. Ở đây điều cần là **hệ thống tác vụ**: mỗi tác vụ là một file JSON, giữa các tác vụ có phụ thuộc `blockedBy`, được lưu bền vững trên đĩa xuyên phiên.

---

## Giải pháp

![Task System Overview](images/task-system-overview.svg)

Code giảng dạy giữ lại agent loop cơ bản, và để tập trung vào hệ thống tác vụ thì lược bỏ phần phục hồi lỗi đầy đủ của s11 (RecoveryState, backoff, escalation, reactive compact, fallback model). Phần mới thêm vào gồm 5 tool tác vụ + thư mục `.tasks/` để lưu bền vững + kiểm tra phụ thuộc `blockedBy`. Hệ thống tác vụ và phục hồi lỗi là hai tầng độc lập: trong mã nguồn CC, `utils/tasks.ts` chỉ xử lý CRUD, còn `query.ts` với `with_retry`/RecoveryState lo phục hồi lỗi, không ghép cứng với nhau.

TodoWrite vs Task System:

| | TodoWrite (s05) | Task System (s12) |
|---|---|---|
| Vai trò | Danh sách thực thi của nhiệm vụ hiện tại | Hệ thống tác vụ có thể phục hồi |
| Lưu trữ | Trong tiến trình / trạng thái phiên | `.tasks/{id}.json` |
| Phụ thuộc | Không có | Đồ thị phụ thuộc `blockedBy` / `blocks` |
| Vòng đời | Phiên hiện tại / nhiệm vụ hiện tại | Giữ lại xuyên phiên |
| Phân công | Không phụ trách claim tác vụ | `owner` / claim |
| Trạng thái | pending / in_progress / completed | pending / in_progress / completed |
| Độ hạt | Các bước riêng của Agent | Tác vụ có thể được claim, theo dõi và mở khóa |

---

## Nguyên lý hoạt động

![Task DAG](images/task-dag.svg)

### Task: cấu trúc dữ liệu

Mỗi tác vụ là một file JSON, nằm trong thư mục `.tasks/`:

```python
@dataclass
class Task:
    id: str
    subject: str
    description: str
    status: str          # pending | in_progress | completed
    owner: str | None    # Tên Agent (trong bối cảnh đa Agent)
    blockedBy: list[str] # Danh sách ID tác vụ phụ thuộc
```

ID được tạo bằng `timestamp + random hex`, đơn giản nhưng đủ dùng. CC dùng ID tuần tự + file highwatermark để ngăn tái sử dụng ID, đó là thiết kế chặt chẽ hơn.

### create_task: tạo tác vụ

```python
def create_task(subject: str, description: str = "",
                blockedBy: list[str] | None = None) -> Task:
    task = Task(
        id=f"task_{int(time.time())}_{random_hex(4)}",
        subject=subject, description=description,
        status="pending", owner=None,
        blockedBy=blockedBy or [],
    )
    save_task(task)
    return task
```

Khi tạo sẽ tự động `save_task` vào `.tasks/{id}.json`. `blockedBy` dùng để khai báo phụ thuộc, ví dụ `blockedBy` của "viết API" có thể là `["task_schema"]`.

### can_start: kiểm tra phụ thuộc

Một tác vụ chỉ có thể bắt đầu khi **toàn bộ** `blockedBy` của nó đều đã ở trạng thái `completed`:

```python
def can_start(task_id: str) -> bool:
    task = load_task(task_id)
    for dep_id in task.blockedBy:
        if not _task_path(dep_id).exists():
            return False  # thiếu dependency = vẫn bị chặn
        dep = load_task(dep_id)
        if dep.status != "completed":
            return False
    return True
```

`can_start` là bước kiểm tra trước của `claim_task`: nếu trong `blockedBy` có bất kỳ tác vụ nào chưa `completed`, thì không thể claim. Nếu dependency không tồn tại cũng xem là blocked, để tránh crash khi tham chiếu sai ID.

### claim_task: nhận tác vụ

Khi Agent bắt đầu làm một tác vụ, nó gọi `claim_task`: đặt `owner`, đổi trạng thái từ `pending` → `in_progress`. Trường `owner` ghi lại ai đang làm tác vụ này, giúp tránh bị claim trùng trong bối cảnh đa Agent:

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    if task.status != "pending":
        return f"Task {task_id} is {task.status}, cannot claim"
    if not can_start(task_id):
        deps = [d for d in task.blockedBy
                if load_task(d).status != "completed"]
        return f"Blocked by: {deps}"
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
    return f"Claimed {task_id} ({task.subject})"
```

Nếu tác vụ đã bị người khác claim (`status != "pending"`), hoặc dependency chưa hoàn tất (`can_start` trả về False), thì sẽ từ chối claim.

### complete_task: hoàn thành và mở khóa

Khi tác vụ làm xong, đặt thành `completed`. Đồng thời quét qua các tác vụ khác để tìm **các tác vụ hạ nguồn vừa được mở khóa**:

```python
def complete_task(task_id: str) -> str:
    task = load_task(task_id)
    task.status = "completed"
    save_task(task)
    # tìm các tác vụ hạ nguồn vừa được mở khóa
    unblocked = [t.subject for t in list_tasks()
                 if t.status == "pending" and t.blockedBy
                 and can_start(t.id)]
    msg = f"Completed {task_id} ({task.subject})"
    if unblocked:
        msg += f"\nUnblocked: {', '.join(unblocked)}"
    return msg
```

Sau khi hoàn thành "schema", `can_start` của "endpoints" và "docs" sẽ trả về True, và chúng có thể bắt đầu.

### get_task: xem đầy đủ chi tiết

`list_tasks` chỉ hiển thị tóm tắt một dòng. `get_task` trả về toàn bộ JSON của tác vụ, bao gồm description và chi tiết dependency. Khi khôi phục xuyên phiên, Agent cần đọc mô tả đầy đủ để tiếp tục công việc:

```python
def get_task(task_id: str) -> str:
    task = load_task(task_id)
    return json.dumps(asdict(task), indent=2)
```

### Máy trạng thái: hai hành động, ba trạng thái

```
pending ──claim──→ in_progress ──complete──→ completed
```

Ở đây `claim` / `complete` là hành động, còn `pending` / `in_progress` / `completed` là trạng thái:

- **claim_task**: `pending` → `in_progress`. Đặt owner, bắt đầu làm việc.
- **complete_task**: `in_progress` → `completed`. Đánh dấu tác vụ hoàn thành và mở khóa hạ nguồn.

CC không có đường `in_progress → pending` theo kiểu release trực tiếp. Nếu teammate bị terminate hoặc shutdown, CC sẽ unassign tác vụ chưa hoàn thành của nó (xóa owner) và reset trạng thái về `pending`, để agent khác có thể claim lại. Bản giảng dạy lược bỏ đường phục hồi này.

### Chạy toàn bộ cùng nhau

```python
# Tạo các tác vụ có phụ thuộc
schema = create_task("setup database schema")
endpoints = create_task("create API endpoints", blockedBy=[schema.id])
tests = create_task("write tests", blockedBy=[endpoints.id])
docs = create_task("write docs", blockedBy=[schema.id])

# Agent claim tác vụ đầu tiên có thể làm
claim_task(schema.id)       # ✓ Claimed (không có phụ thuộc)
complete_task(schema.id)    # ✓ Completed → mở khóa endpoints, docs

claim_task(endpoints.id)    # ✓ Claimed (schema đã xong)
complete_task(endpoints.id) # ✓ Completed → mở khóa tests

claim_task(docs.id)         # ✓ Claimed (schema đã xong)
complete_task(docs.id)      # ✓ Completed

claim_task(tests.id)        # ✓ Claimed (endpoints đã xong)
complete_task(tests.id)     # ✓ Completed
```

Mỗi `create_task` sẽ ghi ra một file JSON, mỗi `claim_task` / `complete_task` sẽ cập nhật file. Khi qua phiên khác, thư mục `.tasks/` vẫn còn, Agent chỉ cần đọc file là có thể khôi phục tiến độ.

---

## Thay đổi so với s11

| Thành phần | Trước đó (s11) | Sau đó (s12) |
|------|-----------|-----------|
| Quản lý tác vụ | Không có | Task dataclass + 5 tool |
| Kiểu mới | — | Task (id, subject, description, status, owner, blockedBy) |
| Lưu trữ | Không có persistence | `.tasks/{id}.json` xuyên phiên |
| Phụ thuộc | Không có | Đồ thị `blockedBy` + kiểm tra `can_start` |
| Công cụ | bash, read_file, write_file (3) | + create_task, list_tasks, get_task, claim_task, complete_task (8) |
| Vòng đời | — | pending → in_progress → completed (không có đường release quay lại) |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s12_task_system/code.py
```

Thử các prompt sau:

1. `Tạo các tác vụ: setup database schema, create API endpoints (phụ thuộc vào schema), write tests (phụ thuộc vào endpoints), write docs (phụ thuộc vào schema)`
2. `Liệt kê tất cả tác vụ và trạng thái của chúng`
3. `Claim tác vụ đầu tiên không bị chặn và hoàn thành nó`
4. `Liệt kê lại các tác vụ — tác vụ nào bây giờ đã được mở khóa?`

Quan sát trọng tâm: Có file JSON nào được tạo trong thư mục `.tasks/` không? Sau khi hoàn thành một tác vụ, các tác vụ bị chặn có được mở khóa không?

---

## Tiếp theo

Đồ thị tác vụ đã có. Nhưng có những tác vụ chạy rất lâu — ví dụ full test, deploy lên server. Agent gọi LLM là tính phí theo lượng dùng, nên không thể ngồi chờ một thao tác chậm.

s13 Background Tasks → đưa thao tác chậm ra chạy nền. Agent tiếp tục xử lý việc khác, khi nền chạy xong thì thông báo lại.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Nội dung dưới đây dựa trên phân tích mã nguồn CC `utils/tasks.ts` (862 dòng), `tools/TaskCreateTool/TaskCreateTool.ts` (138 dòng), `tools/TaskUpdateTool/TaskUpdateTool.ts` (406 dòng), `tools/TaskGetTool/TaskGetTool.ts` (128 dòng), `tools/TaskListTool/TaskListTool.ts` (116 dòng), `hooks/useTaskListWatcher.ts` (221 dòng).

### 1. Các trường đầy đủ của TaskRecord

Bản giảng dạy chỉ nói về id, subject, status, owner, blockedBy. Thực tế CC có 9 trường (`utils/tasks.ts:76-89`):

| Trường | Kiểu | Mục đích |
|------|------|------|
| `id` | string | ID số nguyên tăng dần |
| `subject` | string | Tiêu đề ngắn |
| `description` | string | Mô tả tự do |
| `activeForm` | string? | Dạng đang làm, hiển thị ở spinner khi in_progress |
| `owner` | string? | ID agent được gán |
| `status` | pending/in_progress/completed | Vòng đời |
| `blocks` | string[] | ID các tác vụ bị tác vụ này chặn (hạ nguồn) |
| `blockedBy` | string[] | ID các tác vụ chặn tác vụ này (thượng nguồn) |
| `metadata` | Record? | Cặp key/value mở rộng tùy ý |

Vị trí lưu trữ: `~/.claude/tasks/{taskListId}/{id}.json`. Mỗi tác vụ một file.

### 2. Không phải bản nâng cấp của TodoWrite, mà là hai hệ thống độc lập

Trong CC, Task System và TodoWrite **cùng tồn tại**, được chuyển bằng `isTodoV2Enabled()` (`utils/tasks.ts:133`) — phiên tương tác mặc định bật Task (V2), còn non-interactive/SDK mặc định dùng TodoWrite. Biến môi trường `CLAUDE_CODE_ENABLE_TASKS` có thể ép bật Task. Task có những thứ TodoWrite không có: khóa file chống cạnh tranh, thực thi phụ thuộc bắt buộc, ownership, fs.watch dạng phản ứng, lifecycle hooks.

### 3. Cơ chế khóa khi claim song song

`claimTask()` (`utils/tasks.ts:541-612`) dùng khóa kép để tránh race condition:

**Khóa file tác vụ**: dùng `proper-lockfile` khóa `{taskId}.json` (retry tối đa 30 lần, exponential backoff 5-100ms). Bên trong lock:
1. Đọc lại tác vụ (tránh TOCTOU)
2. Kiểm tra đã bị người khác claim → `already_claimed`
3. Kiểm tra đã hoàn thành → `already_resolved`
4. Kiểm tra thượng nguồn chưa xong → `blocked`
5. Đặt owner

**Khóa cấp danh sách** (khi kiểm tra agent đang bận): file `.lock`, quét nguyên tử tất cả tác vụ và kiểm tra agent đó đã có open task khác chưa.

Lưu ý: bản giảng dạy gộp claim và bắt đầu làm thành một bước (claim = set owner + in_progress); còn CC thật thì `claimTask` chủ yếu giải quyết tranh chấp owner, chỉ set owner chứ không đổi status, việc cập nhật trạng thái do `TaskUpdate` xử lý.

### 4. Highwatermark ngăn tái sử dụng ID

File `.highwatermark` ghi lại ID tác vụ lớn nhất từng được cấp. Dù tác vụ bị xóa, ID vẫn không bị tái sử dụng.

### 5. Bốn Task tool

Hệ thống tác vụ của CC có bốn tool (không phải một Task tool chung như bản giảng dạy): `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList`. Tất cả đều đặt `isConcurrencySafe: true` và `shouldDefer: true` (schema của tool không nằm trong prompt ban đầu, chỉ hiển thị sau khi ToolSearch tìm thấy).

Việc bản giảng dạy cho `create_task(blockedBy=...)` khai báo phụ thuộc ngay lúc tạo là một cách đơn giản hóa hợp lý. Trong CC thật, `TaskCreate` chỉ nhận subject/description/activeForm/metadata, còn quan hệ phụ thuộc được quản lý qua `TaskUpdate` với `addBlocks/addBlockedBy`.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
