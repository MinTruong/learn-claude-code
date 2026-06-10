# s18: Cách Ly Worktree — Mỗi người làm của mình, không can thiệp

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s16 → s17 → `s18` → [s19](../s19_mcp_plugin/) → s20

> *"Mỗi người một thư mục, không can thiệp"* — Nhiệm vụ quản lý mục tiêu, worktree quản lý thư mục, liên kết theo ID.
>
> **Lớp Harness**: Cách ly — Cách ly thư mục cho việc thực thi song song.

---

## Vấn đề

Ở s17, Alice và Bob đều làm việc trong cùng một thư mục. Nhiệm vụ của Alice là "tái cấu trúc mô-đun xác thực", nhiệm vụ của Bob là "tái cấu trúc trang đăng nhập UI".

Alice `write_file("config.py", ...)`. Bob cũng `write_file("config.py", ...)`. Hai người sửa cùng một tệp, viết đè lên nhau. Và không thể hoàn tác sạch sẽ — không biết thay đổi nào là của ai.

s15-s17 đã giải quyết "ai làm gì" (hệ thống nhiệm vụ) và "giao tiếp thế nào" (bus tin nhắn), nhưng chưa giải quyết "làm ở đâu".

---

## Giải pháp

![Tổng quan Worktree](images/worktree-overview.svg)

Git worktree cho phép bạn tạo nhiều thư mục làm việc độc lập trong cùng một repository, mỗi thư mục có nhánh riêng. Alice làm việc ở `.worktrees/auth-refactor/`, Bob làm việc ở `.worktrees/ui-login/` — không can thiệp lẫn nhau.

Sử dụng lại MessageBus, giao thức và cơ chế tự động ghi danh từ S17. Chương này thêm mới:

| Khả năng | Tác dụng |
|------|------|
| create_worktree | Tạo thư mục độc lập + nhánh độc lập cho nhiệm vụ |
| bind_task_to_worktree | Liên kết nhiệm vụ và thư mục làm việc (không thay đổi trạng thái) |
| remove_worktree / keep_worktree | Dọn dẹp hoặc giữ lại sau khi hoàn thành |
| validate_worktree_name | Từ chối path traversal và ký tự không hợp lệ |

---

## Nguyên tắc hoạt động

### Tạo: Liên kết Nhiệm vụ-Worktree

```python
def create_worktree(name: str, task_id: str = "") -> str:
    validate_worktree_name(name)       # Chỉ cho phép [A-Za-z0-9._-]{1,64}
    path = WORKTREES_DIR / name
    ok, result = run_git(["worktree", "add", str(path), "-b", f"wt/{name}", "HEAD"])
    if not ok:
        return f"Git error: {result}"
    if task_id:
        bind_task_to_worktree(task_id, name)
    log_event("create", name, task_id)
    return f"Worktree '{name}' created at {path}"

def bind_task_to_worktree(task_id: str, worktree_name: str):
    task = load_task(task_id)
    task.worktree = worktree_name       # Chỉ ghi trường worktree
    save_task(task)                     # Trạng thái giữ pending, đợi đồng nghiệp ghi danh
```

Quy tắc liên kết: một nhiệm vụ liên kết với một worktree. Liên kết không thay đổi trạng thái nhiệm vụ — nhiệm vụ vẫn là `pending`, chỉ khi đồng nghiệp tự động ghi danh thì mới chuyển sang `in_progress`. Như vậy Lead có thể tạo trước nhiệm vụ và worktree, đồng nghiệp rảnh rỗi thì tự nhiên ghi danh nhiệm vụ có worktree.

### Chuyển đổi cwd của công cụ đồng nghiệp

Phiên bản giáo dục duy trì một từ điển `wt_ctx` cho mỗi đồng nghiệp, ghi lại đường dẫn worktree hiện tại. Khi đồng nghiệp ghi danh nhiệm vụ có worktree, `wt_ctx` tự động được đặt là đường dẫn worktree; `bash`, `read_file`, `write_file` của đồng nghiệp thực thi trong thư mục worktree:

```python
# Bên trong luồng đồng nghiệp
wt_ctx = {"path": None}

def _run_claim_task(task_id):
    result = claim_task(task_id, owner=name)
    if "Claimed" in result:
        task = load_task(task_id)
        if task.worktree:
            wt_ctx["path"] = str(WORKTREES_DIR / task.worktree)
    return result

def _run_bash(command):
    return run_bash(command, cwd=wt_ctx["path"])  # Thực thi trong worktree
```

Đây là đơn giản hóa giáo dục. EnterWorktree của CC thực sự sử dụng `process.chdir()` để chuyển toàn bộ thư mục quy trình, cách ly AgentTool sử dụng `cwdOverride` để bao bọc việc thực thi sub agent.

### Kết thúc: Keep hay Remove

Sau khi hoàn thành nhiệm vụ, có hai lựa chọn:

```python
def remove_worktree(name: str, discard_changes: bool = False) -> str:
    # Kiểm tra an toàn: mặc định từ chối khi có thay đổi
    if not discard_changes:
        files, commits = _count_worktree_changes(path)
        if files > 0 or commits > 0:
            return "Có thay đổi chưa commit, sử dụng discard_changes=true để xóa cưỡng bức, hoặc keep_worktree để giữ lại"
    ok, _ = run_git(["worktree", "remove", str(path), "--force"])
    if not ok:
        return "Xóa thất bại"
    run_git(["branch", "-D", f"wt/{name}"])
    log_event("remove", name)

def keep_worktree(name: str) -> str:
    log_event("keep", name)
    return f"Worktree '{name}' kept for review (branch: wt/{name})"
```

Keep = giữ lại nhánh, đợi review thủ công rồi merge vào nhánh chính. Remove = mặc định từ chối khi có thay đổi, cần `discard_changes=true` để xác nhận. Không tự động complete task — hoàn thành nhiệm vụ được kích hoạt rõ ràng bởi `complete_task` của đồng nghiệp.

### Luồng sự kiện: Có thể kiểm tra

Mỗi thao tác vòng đời ghi vào nhật ký, tiện lợi cho việc điều tra:

```python
def log_event(event_type: str, worktree_name: str, task_id: str = ""):
    event = {"type": event_type, "worktree": worktree_name,
             "task_id": task_id, "ts": time.time()}
    # append to .worktrees/events.jsonl
```

Các loại sự kiện: `create` (tạo), `remove` (xóa), `keep` (giữ lại). Phiên bản giáo dục chỉ ghi lại sự kiện để điều tra thủ công; khôi phục hoàn chỉnh còn cần index hoặc quét `git worktree list`.

### run_git: Trả về thành công/thất bại

```python
def run_git(args: list[str]) -> tuple[bool, str]:
    r = subprocess.run(["git"] + args, cwd=WORKDIR, ...)
    return r.returncode == 0, output
```

`create_worktree` và `remove_worktree` chỉ ghi nhật ký sự kiện sau khi lệnh git thành công, đảm bảo nhật ký phản ánh trạng thái thực tế.

---

## Thay đổi so với s17

| Thành phần | Trước đó (s17) | Sau đó (s18) |
|------|-----------|-----------|
| Thư mục làm việc | Tất cả Agent chia sẻ WORKDIR | Mỗi nhiệm vụ có thể liên kết git worktree độc lập |
| Dữ liệu Task | id/subject/status/owner/blockedBy | + trường worktree |
| Công cụ đồng nghiệp cwd | Luôn luôn WORKDIR | Tự động chuyển đổi khi ghi danh nhiệm vụ có worktree |
| Hàm mới | — | create_worktree, bind_task_to_worktree, remove_worktree, keep_worktree, validate_worktree_name |
| An toàn worktree | Không có | Xác thực tên + từ chối xóa khi có thay đổi |
| Nhật ký sự kiện | Không có | events.jsonl kiểm tra vòng đời |
| Công cụ Lead | 14 (s17) | + create_worktree, remove_worktree, keep_worktree (17) |
| Công cụ đồng nghiệp | 8 (s17) | 8 (bash/read/write thực thi trong worktree cwd) |

---

## Thử một lần

```sh
cd learn-claude-code
python s18_worktree_isolation/code.py
```

Thử prompt này:

`Create two tasks, then create worktrees for each (bind with task_id). Spawn alice and bob. Watch them auto-claim and work in isolated directories.`

Các điểm quan trọng để quan sát: Đầu ra `git status` của hai worktree có hiển thị các nhánh khác nhau không? Sau khi đồng nghiệp ghi danh nhiệm vụ có worktree, lệnh bash có thực thi trong thư mục worktree không? `remove_worktree` có từ chối worktree có thay đổi không? Trạng thái nhiệm vụ trong `.tasks/` sau khi liên kết có vẫn là `pending` không?

---

## Tiếp theo

Nhóm agent có thể tự tổ chức trong không gian làm việc cách ly rồi. Nhưng khả năng của Agent bị giới hạn bởi các công cụ chúng ta viết cho nó — bash, read, write, task...

Nếu người dùng đã có công cụ riêng thì sao? Ví dụ như Jira API nội bộ công ty, một hệ thống triển khai tự xây dựng?

s19 MCP Plugin → Trang bị hệ thống plugin cho Agent. Công cụ bên ngoài tiếp cận thông qua giao thức chuẩn, Agent không cần biết ai viết chúng.

<details>
<summary>Sâu vào mã nguồn CC</summary>

Hệ thống worktree của CC có hai đường dẫn: **EnterWorktree** (vào phiên hiện tại) và **AgentTool isolation** (cách ly sub agent).

### EnterWorktree: Chuyển đổi phiên hiện tại

`EnterWorktreeTool.ts:92-97` tạo worktree sau đó ngay lập tức `process.chdir(worktreePath)`, `setCwd()`, `setOriginalCwd()`, `saveWorktreeState()`. Thư mục làm việc của phiên hiện tại chuyển trực tiếp sang worktree — không phải nhắc nhở prompt, mà là thay đổi thư mục cấp quy trình.

`ExitWorktreeTool.ts:261-320` cả keep và remove đều sẽ `restoreSessionToOriginalCwd()` khôi phục thư mục gốc. Khi Remove, kiểm tra thay đổi chưa commit (`ExitWorktreeTool.ts:190-220`), không có `discard_changes: true` thì từ chối xóa.

### AgentTool isolation: Cách ly sub agent

`AgentTool.tsx:590-641` khi `isolation: "worktree"` gọi `createAgentWorktree()` tạo worktree, sử dụng `cwdOverridePath` để bao bọc việc thực thi sub agent. Tất cả các hoạt động của sub agent tự động diễn ra trong thư mục worktree. `AgentTool/prompt.ts:272` nói với model: đây là worktree tạm thời, không có thay đổi thì tự động dọn dẹp, có thay đổi thì trả về đường dẫn và nhánh.

`worktree.ts:902-951` của `createAgentWorktree()` không sửa đổi session cwd toàn cục, chỉ dành cho sub agent. `worktree.ts:961-1020` của `removeAgentWorktree()` xóa từ main repo root.

### Xác thực tên

`worktree.ts:76-84` xác thực slug: từ chối `.`/`..`, cho phép `[a-zA-Z0-9._-]`. `worktree.ts:48` định nghĩa `VALID_WORKTREE_SLUG_SEGMENT`. Phiên bản giáo dục `validate_worktree_name` sử dụng quy tắc giống nhau.

### Đặt tên đường dẫn và nhánh

Đường dẫn thực là `.claude/worktrees/`, tên nhánh `worktree-{slug}` (`worktree.ts:204-227`, dấu gạch chéo thay bằng `+`). Phiên bản giáo dục sử dụng `.worktrees/` và `wt/{name}` để đơn giản hóa.

Khi tạo sử dụng `git worktree add -B` (`worktree.ts:326-328`), ưu tiên dựa trên `origin/<defaultBranch>` chứ không phải HEAD hiện tại.

### Quản lý trạng thái

CC không có liên kết task-worktree. Trạng thái Worktree được quản lý thông qua `PersistedWorktreeSession` (`worktree.ts:756-768`), các trường bao gồm `originalCwd`, `worktreePath`, `worktreeName`, `worktreeBranch`, `originalBranch`, `originalHeadCommit`, `sessionId`, v.v. — không có taskId. `saveWorktreeState()` (`sessionStorage.ts:2883-2920`) ghi vào session transcript với `type: 'worktree-state'`.

Phiên bản giáo dục sử dụng trường `worktree` của task để liên kết, đây là đơn giản hóa giáo dục. CC coi worktree và task là hai hệ thống độc lập, liên kết thông qua Agent hiểu bối cảnh.

</details>

<!-- translation-sync: zh@v1, en@v0, ja@v0, vi@v1 -->
