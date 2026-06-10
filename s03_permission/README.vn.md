# s03: Permission — Kiểm tra quyền trước khi thực thi

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → `s03` → [s04](../s04_hooks/) → s05 → ... → s20

> *"Tool phải qua cửa permission rồi mới được chạy"* — permission pipeline quyết định thao tác nào cần approval.
>
> **Harness layer**: permission — thêm một cánh cửa trước khi tool được thực thi.

---

## Vấn đề

Agent ở s02 có 5 tools. File tools được `safe_path` bảo vệ, nhưng bash thì không bị giới hạn. Chỉ cần bảo nó "dọn project đi" là có thể sinh ra `rm -rf /`.

An toàn không thể dựa vào việc “tin model”. Nó phải đến từ code -- tức là phải kiểm tra trước khi tool thật sự chạy.

---

## Giải pháp

![Permission Overview](images/permission-overview.svg)

Loop của s02 được giữ nguyên hoàn toàn. Thay đổi duy nhất là chèn `check_permission()` vào trước khi tool chạy. Mỗi tool call phải đi qua ba cánh cửa theo đúng thứ tự: **từ chối cứng trước**, **hỏi mềm sau**, nếu không trúng gì thì mới **cho chạy**.

Ba cánh cửa tương ứng với ba kiểu quyết định:

| Cánh cửa | Vai trò | Nếu trúng |
|------|------|--------|
| 1. Deny list | Những thao tác cấm tuyệt đối (`rm -rf /`, `sudo`) | Từ chối ngay, không chạy |
| 2. Rule match | Những thao tác phụ thuộc ngữ cảnh (ghi ra ngoài workspace, `rm` file) | Chuyển sang cánh cửa 3 |
| 3. User approval | Sau khi rule trúng, dừng lại để hỏi user | User quyết định allow hay deny |

Nếu không trúng cánh cửa nào → chạy luôn. Phần lớn thao tác thông thường đi theo luồng này.

---

## Nguyên lý hoạt động

![Permission Pipeline](images/permission-pipeline.svg)

### Cửa 1: deny list

Một danh sách chặn cứng. Kiểm tra trước, trúng là chặn ngay. (Lưu ý: đây chỉ là minh họa giảng dạy; string matching đơn giản không phải cơ chế bảo mật đáng tin cậy, vì command variant và shell expansion có thể đi vòng qua nó. Cách làm thật của CC ở appendix.)

```python
DENY_LIST = [
    "rm -rf /", "sudo", "shutdown", "reboot",
    "mkfs", "dd if=", "> /dev/sda",
]

def check_deny_list(command: str) -> str | None:
    for pattern in DENY_LIST:
        if pattern in command:
            return f"Blocked: '{pattern}' is on the deny list"
    return None
```

### Cửa 2: rule matching

Dùng để mô tả **khi nào cần hỏi user**. Mỗi rule chỉ rõ tool nào áp dụng và điều kiện kiểm tra là gì.

```python
PERMISSION_RULES = [
    {
        "tools": ["write_file", "edit_file"],
        "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
        "message": "Writing outside workspace",
    },
    {
        "tools": ["bash"],
        "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777"]),
        "message": "Potentially destructive command",
    },
]

def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]
    return None
```

### Cửa 3: hỏi user

Khi rule trúng, dừng lại để user quyết định.

```python
def ask_user(tool_name: str, args: dict, reason: str) -> str:
    print(f"\n⚠  {reason}")
    print(f"   Tool: {tool_name}({args})")
    choice = input("   Allow? [y/N] ").strip().lower()
    return "allow" if choice in ("y", "yes") else "deny"
```

### Ghép ba cánh cửa lại với nhau

Ba bước này được xâu lại thành `check_permission()` và chèn vào ngay trước khi tool chạy:

```python
def check_permission(block) -> bool:
    # Cửa 1: deny cứng
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(f"\n⛔ {reason}")
            return False

    # Cửa 2 + 3: rule match → user approval
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False

    return True

# Trong agent_loop — loop của s02 chỉ thêm đúng một dòng:
for block in response.content:
    if block.type == "tool_use":
        if not check_permission(block):           # ← mới thêm
            results.append({... "content": "Permission denied."})
            continue
        output = TOOL_HANDLERS[block.name](**block.input)  # phần cũ của s02
        results.append(...)
```

---

## Thay đổi so với s02

| Thành phần | Trước (s02) | Sau (s03) |
|------|-----------|-----------|
| Mô hình an toàn | Không có, tin model | Permission pipeline 3 cánh cửa |
| Hàm mới | — | `check_deny_list`, `check_rules`, `ask_user`, `check_permission` |
| Loop | Mọi tool chạy trực tiếp | Chèn `check_permission()` trước khi thực thi |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s03_permission/code.py
```

Thử các prompt sau:

1. `Tạo file test.txt trong thư mục hiện tại` (nên chạy thẳng)
2. `Xóa toàn bộ file tạm trong /tmp` (bash + rm sẽ đụng cửa 2)
3. `Trong thư mục hiện tại có những file nào?` (chỉ đọc, đi qua toàn bộ)
4. `Thử ghi một file vào /etc/something` (ghi ngoài workspace, đụng cửa 2)

Điểm cần quan sát: thao tác nào đi qua trực tiếp? Thao tác nào cần bạn xác nhận? Thao tác nào bị chặn ngay lập tức?

---

## Tiếp theo

Permission check đã có. Nhưng nếu mỗi lần đều hard-code `check_permission()` ngay trong loop thì rất nhanh sẽ rối. Ví dụ bạn muốn log trước/sau mỗi tool call thì sao? Muốn sau một số thao tác tự động `git commit` thì sao? Những logic mở rộng như vậy mà nhét hết vào loop thì loop sẽ phình lên rất nhanh.

s04 Hooks → cho loop một cơ chế hook. Logic mở rộng được treo ở hook, còn loop vẫn giữ sạch.

<details>
<summary>Đi sâu vào CC source code</summary>

> Nội dung dựa trên kiểm tra CC source code `types/permissions.ts`, `utils/permissions/permissions.ts`, `toolExecution.ts`, `utils/permissions/yoloClassifier.ts`, `tools/AgentTool/forkSubagent.ts`.

### 1. PermissionResult: không phải 3 loại, mà là 4

Ba cánh cửa của bản giảng dạy (deny → ask → allow) không trùng hoàn toàn với CC. `PermissionResult` của CC có 4 behavior (`types/permissions.ts:241-266`):

| behavior | Ý nghĩa | Tương ứng trong bản giảng dạy |
|----------|------|-----------|
| `allow` | Cho phép trực tiếp | Cửa 3 đi qua |
| `deny` | Từ chối trực tiếp | Cửa 1 trúng |
| `ask` | Bật hộp thoại hỏi user | Cửa 2 trúng |
| `passthrough` | Tool không tự quyết, giao cho pipeline chung | Bản giảng dạy không có |

### 2. Pipeline validation trong production

Tool call trong CC không chỉ đi qua ba cánh cửa, mà đi qua nhiều phase rải trong `checkPermissionsAndCallTool()` (`toolExecution.ts:599-1745`), hooks, `hasPermissionsToUseToolInner()` (`utils/permissions/permissions.ts:1158-1310`) và classifier logic:

1. **Zod schema validation** (`toolExecution.ts:614-680`) — kiểm tra kiểu của argument
2. **`validateInput()`** (`toolExecution.ts:682-733`) — kiểm tra semantics ở cấp tool
3. **`backfillObservableInput()`** (`toolExecution.ts:784`) — bổ sung các field legacy
4. **PreToolUse hooks** (`toolExecution.ts:800-862`) — hook có thể trả về allow/deny/ask
5. **`resolveHookPermissionDecision()`** (`toolExecution.ts:921-931`) — phối hợp giữa hook và permission pipeline
6. **`hasPermissionsToUseToolInner()`** (`permissions.ts:1158-1310`) — kiểm tra đa tầng:
   - Tool bị deny toàn cục → `deny`
   - Tool bị đánh dấu ask → `ask`
   - `tool.checkPermissions()` tự kiểm tra riêng
   - Tool tự trả về deny → `deny`
   - `requiresUserInteraction()` → `ask`
   - Content-based ask rule → `ask` (không bypass được)
   - Safety violation → `ask` (không bypass được)
   - `bypassPermissions` mode → `allow`
   - Tool được allow rule toàn cục → `allow`
   - `passthrough` → chuyển thành `ask`

### 3. Deny list thật không chỉ là một file

CC không có một deny list duy nhất. Permission rule đến từ 8 nguồn (`types/permissions.ts:54-62`):

| Nguồn | Vị trí cấu hình |
|------|---------|
| `userSettings` | `~/.claude/settings.json` |
| `projectSettings` | `.claude/settings.json` |
| `localSettings` | `settings.local.json` |
| `flagSettings` | Feature flags |
| `policySettings` | Chính sách doanh nghiệp |
| `cliArg` | `--allowedTools` / `--deniedTools` |
| `command` | Inline command |
| `session` | Ủy quyền tạm trong session |

Mỗi rule có dạng: `{ toolName: "Bash", ruleBehavior: "deny", ruleContent: "npm publish:*" }`. Nhiều nguồn được merge lại, nguồn ưu tiên cao override nguồn thấp (từ thấp đến cao: user < project < local < flag < policy, cộng thêm cliArg, command, session).

### 4. `isDestructive()` là gì

Trong CC, `isDestructive` (`Tool.ts:405-406`) **chỉ dùng cho UI** — hiển thị nhãn `[destructive]` trong danh sách tool. Nó không tham gia vào quyết định permission. Mặc định mọi tool đều trả về `false`. Chỉ ExitWorktree (khi remove) và MCP tool (dựa vào `annotations.destructiveHint`) mới override nó.

### 5. YoloClassifier (auto-approval)

Ở chế độ auto, CC không hỏi user mỗi lần. `classifyYoloAction` (`utils/permissions/yoloClassifier.ts:1012`) gửi tool call + conversation context cho một classifier LLM để quyết định có an toàn không. Trước tiên nó thử mô phỏng trong `acceptEdits` mode (`permissions.ts:620-656`, nếu acceptEdits cho phép thì approve luôn), sau đó tra whitelist tool an toàn (`permissions.ts:658-686`), cuối cùng mới gọi classifier. Nếu classifier từ chối quá nhiều lần liên tiếp thì fallback về manual approval.

### 6. Permission bubbling

Subagent được fork ra qua AgentTool sẽ có `permissionMode = 'bubble'` (`forkSubagent.ts:50`). Nghĩa là permission dialog **nổi lên terminal của parent agent**, thay vì bị deny âm thầm trong subagent. Bash classifier vẫn tiếp tục chạy trong background để có thể auto-approve nếu đủ điều kiện.

### Vì sao bản giảng dạy cố ý đơn giản hóa

- Multi-stage pipeline → 3 cánh cửa: giảm mạnh ngưỡng tiếp cận
- 8 nguồn rule → 1 DENY_LIST local: giữ lượng khái niệm vừa đủ
- `isDestructive` → bỏ qua (bản giảng dạy không có UI layer, mà CC cũng không dùng nó để quyết định permission)
- YoloClassifier → bỏ qua (vì cần thêm LLM call và telemetry)
- Permission bubbling → bỏ qua (đến s15 mới bắt đầu có multi-agent)

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
