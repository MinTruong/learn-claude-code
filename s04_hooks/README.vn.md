# s04: Hooks — Gắn quanh loop, đừng nhét vào trong loop

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → s03 → `s04` → [s05](../s05_todo_write/) → s06 → ... → s20

> *"Hook bọc quanh loop, chứ không chui vào loop"* — hook dùng để inject logic mở rộng trước và sau khi tool chạy.
>
> **Harness layer**: hook — extension point không xâm nhập trực tiếp vào loop.

---

## Vấn đề

Agent ở s03 đã có permission check. Nhưng mỗi lần muốn thêm một kiểm tra mới, ví dụ như “log mọi bash call”, hay “sau thao tác thì tự động git add”, bạn lại phải sửa thẳng vào hàm `agent_loop`.

Rất nhanh, loop sẽ biến thành thế này:

```python
def agent_loop(messages):
    while True:
        # ... LLM call ...
        for block in response.content:
            if block.type != "tool_use":
                continue
            log_to_file(block)          # thêm một dòng
            check_permission(block)     # thêm một dòng
            notify_slack(block)         # lại thêm một dòng
            output = execute(block)
            auto_git_add(block)         # thêm tiếp một dòng
            # ... chẳng mấy chốc không còn nhận ra loop nữa
```

Bạn muốn mở rộng hành vi của Agent, nhưng thứ bị sửa lại chính là loop. Trong khi loop nên là phần lõi ổn định; extension thì nên treo ở bên ngoài.

---

## Giải pháp

![Hooks Overview](images/hooks-overview.svg)

Loop và logic permission của s03 được giữ nguyên gần như hoàn toàn. Khác biệt duy nhất là `check_permission()` được chuyển từ trong thân loop ra thành hook. Loop không còn gọi trực tiếp các hàm kiểm tra nữa, mà thay bằng `trigger_hooks("PreToolUse", block)`. Việc chạy cái gì được quyết định bởi hook registry.

Bốn event là đủ để bao trùm một agent cycle hoàn chỉnh:

| Event | Kích hoạt khi nào | Dùng để làm gì |
|------|---------|---------|
| UserPromptSubmit | Sau khi user submit input, trước khi vào LLM | Validate input, inject context |
| PreToolUse | Trước khi tool chạy | Permission check, logging |
| PostToolUse | Sau khi tool chạy | Side-effect (như auto git add), kiểm tra output |
| Stop | Ngay trước khi loop thoát | Dọn dẹp, kết thúc (CC còn có thể ép chạy tiếp) |

Extension được thêm bằng `register_hook()`, còn loop chỉ cần gọi `trigger_hooks()`.

---

## Nguyên lý hoạt động

### Hook registry

Một dict ánh xạ event name → danh sách callback:

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # return ≠ None → hook nói "dừng"
            return result
    return None
```

Trong bản giảng dạy, nếu PreToolUse trả về giá trị khác `None` thì tool call đó sẽ bị chặn; còn nếu Stop trả về khác `None` thì loop sẽ bị buộc tiếp tục. Return value của UserPromptSubmit và PostToolUse thì không dùng.

### UserPromptSubmit

Event này chạy sau khi user nhập xong và trước khi prompt đi vào LLM. Trong CC, hook có thể chặn hoặc sửa input. Ở bản giảng dạy, nó chỉ dùng để demo logging:

```python
def context_inject_hook(query: str) -> str | None:
    """Inject current working directory info into every prompt."""
    print(f"\033[90m[HOOK] UserPromptSubmit: working in {WORKDIR}\033[0m")
    return None   # return None = không sửa gì, để prompt đi tiếp

register_hook("UserPromptSubmit", context_inject_hook)
```

Trong main loop, nó được gọi ngay sau khi user nhập:

```python
query = input("s04 >> ")
trigger_hooks("UserPromptSubmit", query)   # ← trước khi vào LLM
history.append({"role": "user", "content": query})
agent_loop(history)
```

### PreToolUse / PostToolUse

Đây là hai hook trước và sau khi tool chạy. Logic permission của s03 giờ được bọc thành PreToolUse hook; ngoài ra thêm một hook để log và một hook để cảnh báo khi output quá lớn:

```python
# PreToolUse: permission check (logic từ s03, chuyển từ loop ra hook)
def permission_hook(block):
    if block.name == "bash":
        for pattern in DENY_LIST:
            if pattern in block.input.get("command", ""):
                return "Permission denied by deny list"
    if block.name in ("write_file", "edit_file"):
        path = block.input.get("path", "")
        if not (WORKDIR / path).resolve().is_relative_to(WORKDIR):
            choice = input("   Allow? [y/N] ").strip().lower()
            if choice not in ("y", "yes"):
                return "Permission denied by user"
    return None

# PreToolUse: logging
def log_hook(block):
    print(f"[HOOK] {block.name}(...)")

# PostToolUse: nhắc khi output lớn
def large_output_hook(block, output):
    if len(str(output)) > 100000:
        print(f"[HOOK] ⚠ Large output from {block.name}")

register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
register_hook("PostToolUse", large_output_hook)
```

### Stop

Event này chạy khi loop chuẩn bị thoát (`stop_reason != "tool_use"`). Trong bản giảng dạy, nó dùng để in thống kê cuối session:

```python
def summary_hook(messages: list) -> str | None:
    """Print a summary when the loop is about to stop."""
    tool_count = sum(1 for m in messages
                     for b in (m.get("content") if isinstance(m.get("content"), list) else [])
                     if isinstance(b, dict) and b.get("type") == "tool_result")
    print(f"\033[90m[HOOK] Stop: session used {tool_count} tool calls\033[0m")
    return None   # return None = cho phép dừng, return string = ép tiếp tục

register_hook("Stop", summary_hook)
```

Trong `agent_loop`, nó được gọi ngay trước khi thoát:

```python
if response.stop_reason != "tool_use":
    force = trigger_hooks("Stop", messages)   # ← trước khi thoát
    if force:
        # hook trả về một message → inject vào và tiếp tục
        messages.append({"role": "user", "content": force})
        continue
    return
```

### Loop chỉ thay đổi một chỗ

Ở s03, loop gọi trực tiếp `check_permission(block)`. Sang s04, nó đổi sang `trigger_hooks("PreToolUse", block)`:

```python
for block in response.content:
    if block.type != "tool_use":
        continue

    # s03: if not check_permission(block): ...
    # s04: dùng hook thay cho hard-code
    blocked = trigger_hooks("PreToolUse", block)
    if blocked:
        results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": str(blocked)})
        continue

    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown: {block.name}"

    trigger_hooks("PostToolUse", block, output)

    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": output})
```

Bốn hook này vừa đủ bao phủ các mốc quan trọng trong agent cycle: input → trước thực thi → sau thực thi → thoát. Loop chỉ còn nhiệm vụ gọi `trigger_hooks()`, còn logic cụ thể được dời ra callback.

---

## Thay đổi so với s03

| Thành phần | Trước (s03) | Sau (s04) |
|------|-----------|-----------|
| Cách mở rộng | `check_permission()` hard-code trong loop | Hook registry `HOOKS` + `trigger_hooks()` |
| Hàm mới | — | `register_hook`, `trigger_hooks` |
| Hook callback | — | `context_inject_hook`, `permission_hook`, `log_hook`, `large_output_hook`, `summary_hook` |
| Loop | Gọi trực tiếp `check_permission()` | Gọi `trigger_hooks("PreToolUse", ...)` |
| Điều khiển thoát | Không có | `trigger_hooks("Stop", ...)` có thể ngăn thoát |
| Input interception | Không có | `trigger_hooks("UserPromptSubmit", ...)` có thể inject context |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s04_hooks/code.py
```

Thử các prompt sau:

1. `Read the file README.md` (nên đi qua ngay, quan sát log hook)
2. `Create a file called test.txt` (sau khi chạy, xem PostToolUse có được kích hoạt không)
3. `Delete all temporary files in /tmp` (bash + rm sẽ đụng permission hook)

Điểm cần quan sát: trước mỗi tool execution có xuất hiện log `[HOOK]` không? Khi permission bị deny, đó là do hook chặn hay loop hard-code?

---

## Tiếp theo

Giờ Agent đã có thể thực thi thao tác một cách an toàn hơn. Nhưng nó có bao giờ dừng lại để nghĩ “mình nên làm bước nào trước, bước nào sau” không? Nếu giao cho nó một task phức tạp, nó sẽ lao vào làm ngay hay sẽ plan trước?

s05 TodoWrite → thêm cho Agent một planning tool. Viết checklist trước, rồi mới làm.

<details>
<summary>Đi sâu vào CC source code</summary>

> Nội dung dựa trên phân tích đầy đủ CC source code `toolHooks.ts` (650 dòng), `hooks.ts`, `stopHooks.ts`, `coreTypes.ts`.

### 1. Hook event: không chỉ 4 mà là 27

Bản giảng dạy chỉ nói về PreToolUse và PostToolUse. Thực tế CC có 27 hook event (`coreTypes.ts:25-53`):

| Nhóm | Event |
|------|------|
| Liên quan tool | `PreToolUse`, `PostToolUse`, `PostToolUseFailure` |
| Liên quan session | `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `Setup` |
| Tương tác user | `UserPromptSubmit`, `Notification`, `PermissionRequest`, `PermissionDenied` |
| Subagent | `SubagentStart`, `SubagentStop` |
| Liên quan compaction | `PreCompact`, `PostCompact` |
| Liên quan team | `TeammateIdle`, `TaskCreated`, `TaskCompleted` |
| Khác | `Elicitation`, `ElicitationResult`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, `InstructionsLoaded`, `CwdChanged`, `FileChanged` |

Bản giảng dạy chỉ giữ lại 4 event cốt lõi (UserPromptSubmit, PreToolUse, PostToolUse, Stop), vì chúng đã đủ để bao trùm các điểm mấu chốt của một agent cycle hoàn chỉnh. 23 event còn lại vẫn đi theo cùng một pattern.

### 2. Một số field hay dùng của HookResult

`HookResult` của CC (`types/hooks.ts:260-275`) có 14 field. Những field hay dùng nhất gồm:

| Field | Kiểu | Mục đích |
|------|------|------|
| `message` | Message | UI message tùy chọn |
| `blockingError` | HookBlockingError | Blocking error → inject vào hội thoại để model tự sửa |
| `outcome` | success/blocking/non_blocking_error/cancelled | Kết quả thực thi |
| `preventContinuation` | boolean | Ngăn bước tiếp theo |
| `stopReason` | string | Mô tả lý do dừng |
| `permissionBehavior` | allow/deny/ask/passthrough | Quyết định permission do hook trả về |
| `updatedInput` | Record | Sửa tool input |
| `additionalContext` | string | Context bổ sung |
| `updatedMCPToolOutput` | unknown | Chỉnh sửa output của MCP tool |

### 3. Bất biến quan trọng: hook `allow` không được vượt deny/ask rule

Đây là thiết kế bảo mật quan trọng nhất của permission system trong CC (`toolHooks.ts:325-331`): **dù hook trả về `allow`, hệ thống vẫn phải tiếp tục kiểm tra deny/ask rule trong settings.json**. Tức là ngay cả khi hook script của user nói “cho phép”, nếu tool đó bị cấm trong settings.json thì thao tác vẫn bị chặn.

Bản giảng dạy không có tầng này, nên chỉ diễn giải đơn giản: PreToolUse trả về khác `None` thì chặn tool. Với teaching purpose như vậy là đủ, nhưng đem nguyên sang production thì sẽ thành lỗ hổng bảo mật.

### 4. Cơ chế `stopHookActive`

Stop hook trong CC có cơ chế chống infinite loop (`query.ts:212,1300`): field trạng thái `stopHookActive`. Khi stop hook tạo ra `blockingError`, loop sẽ re-enter lượt tiếp theo với `stopHookActive: true`. Ở các lượt sau, nếu stop hook thấy flag này thì nó sẽ không tự kích hoạt lại nữa. Cơ chế này ngăn một bug kiểu: model tự sửa → stop hook lại báo lỗi → model lại tự sửa → stop hook lại báo lỗi...

### 5. `hook_stopped_continuation`

Nếu PostToolUse hook trả về `preventContinuation: true`, CC sẽ tạo một attachment tên `hook_stopped_continuation` (`toolHooks.ts:117-130`). `query.ts` (L1388-1393) bắt được cái này rồi set `shouldPreventContinuation = true`, khiến loop thoát. Đây là cách để hook “cho Agent dừng một cách lịch sự”, chứ không phải crash.

### Vì sao bản giảng dạy cố ý đơn giản hóa

- 27 event → 4 event (UserPromptSubmit / PreToolUse / PostToolUse / Stop): đủ bao trùm agent cycle
- 14 field → return value đơn giản (`None = tiếp tục`, khác `None = chặn hoặc ép chạy tiếp`): giảm gánh nặng tư duy tối đa
- Bất biến hook-allow vs deny/ask → bỏ qua: bản giảng dạy không có tầng settings.json
- `stopHookActive` → bỏ qua: Stop hook trong bản giảng dạy chỉ dùng cho continue đơn giản, không xử lý loop protection

</details>

<!-- translation-sync: zh@v1, en@v0, ja@v0, vi@v1 -->
