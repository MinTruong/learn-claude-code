# s15: Agent Teams — Một người không xong, hãy lập nhóm

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s13 → s14 → `s15` → [s16](../s16_team_protocols/) → s17 → s18 → s19 → s20
> *"Một người không xong, hãy lập nhóm"* — hộp thư file + luồng đồng đội.
>
> **Tầng Harness**: đội — cộng tác đa Agent, message bus.

---

## Vấn đề

"Refactor toàn bộ backend" liên quan đến module authentication, tầng database, API routes, test. Khi Agent đang sửa API routes, chi tiết module authentication đã không còn trong ngữ cảnh nữa. Cửa sổ ngữ cảnh chỉ có bấy nhiêu, sự tập trung của một Agent không thể bao phủ tất cả các module.

Sub Agent ở s06 là công nhân tạm, gọi đến làm một việc rồi đi. Nhưng có những nhiệm vụ cần đồng đội có thể giao tiếp và hợp tác được.

---

## Giải pháp

![Agent Teams Overview](images/agent-teams-overview.svg)

Code giảng dạy tiếp tục dùng khả năng của s14 (lắp prompt, hệ thống tác vụ, thực thi nền, cron scheduler). Để tập trung vào cơ chế đội, nó lược bỏ phần phục hồi lỗi đầy đủ, memory và skill system. Thêm mới ba thứ: **MessageBus** (hộp thư file), **spawn_teammate_thread** (khởi tạo luồng đồng đội), **inbox injection** (Lead nhận tin nhắn từ đồng đội và chèn vào history).

Sub Agent vs đồng đội:

| | s06 Sub Agent | s15 Đồng đội |
|---|---|---|
| Vòng đời | Một lần, dùng xong hủy | Đa vòng (bản giảng dạy giới hạn 10 vòng, CC thật dùng idle loop) |
| Giao tiếp | Chỉ trả lại kết luận | Hộp thư bất đồng bộ, giao tiếp bất kỳ lúc nào |
| Ngữ cảnh | Cách ly hoàn toàn | Chia sẻ thông tin qua tin nhắn |
| Số lượng | Một Agent chính + thỉnh thoảng Sub Agent | Một Lead + nhiều đồng đội |

---

## Nguyên lý hoạt động

![Team Topology](images/team-topology.svg)

### MessageBus: hộp thư file

Mỗi Agent (bao gồm Lead và đồng đội) có một hộp thư `.jsonl`. Gửi tin nhắn = append một dòng JSON vào file của người kia. Đọc tin nhắn = đọc file + xóa (consume):

```python
class MessageBus:
    def send(self, from_agent: str, to_agent: str,
             content: str, msg_type: str = "message"):
        msg = {"from": from_agent, "to": to_agent,
               "content": content, "type": msg_type,
               "ts": time.time()}
        inbox = MAILBOX_DIR / f"{to_agent}.jsonl"
        with open(inbox, "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, agent: str) -> list[dict]:
        inbox = MAILBOX_DIR / f"{agent}.jsonl"
        if not inbox.exists():
            return []
        msgs = [json.loads(line) for line in inbox.read_text().splitlines()]
        inbox.unlink()  # Consume: đọc xong xóa
        return msgs
```

Tại sao dùng file thay vì hàng đợi bộ nhớ? Bản giảng dạy chọn file vì trực quan, có thể quan sát xuyên luồng. CC thật cũng dùng hộp thư file (`~/.claude/teams/{team}/inboxes/`), nhưng thêm `proper-lockfile` để chống xung đột ghi đồng thời. `read_inbox` của bản giảng dạy có race condition giữa read + unlink, nhiều luồng đọc cùng lúc có thể bỏ lỡ tin nhắn, nhưng đối với bối cảnh giảng dạy thì có thể chấp nhận được.

### spawn_teammate_thread: khởi tạo đồng đội

Lead gọi tool `spawn_teammate` để khởi tạo một đồng đội. Đồng đội chạy trong luồng daemon của riêng mình, có system prompt riêng, messages riêng, bộ công cụ giản lược riêng:

```python
def spawn_teammate_thread(name: str, role: str, prompt: str) -> str:
    system = f"You are '{name}', a {role}. Use tools to complete tasks."

    def run():
        messages = [{"role": "user", "content": prompt}]
        sub_tools = [bash, read_file, write_file, send_message]
        for _ in range(10):           # Tối đa 10 vòng
            inbox = BUS.read_inbox(name)
            if inbox:
                messages.append({"role": "user",
                    "content": f"<inbox>{json.dumps(inbox)}</inbox>"})
            response = client.messages.create(
                model=MODEL, system=system, messages=messages[-20:],
                tools=sub_tools, max_tokens=8000)
            # ... thực thi công cụ, xử lý kết quả
        # Xong thì gửi summary cho Lead
        BUS.send(name, "lead", summary, "result")

    threading.Thread(target=run, daemon=True).start()
```

Thiết kế quan trọng:
- **Đồng đội có bộ công cụ giản lược**: bash, read, write, send_message. Bản giảng dạy lược bỏ task và cron, tập trung vào cơ chế giao tiếp. CC thật đồng đội cũng có TaskCreate, TaskUpdate và các tool khác, hệ thống tác vụ là chia sẻ giữa các thành viên
- **Bản giảng dạy giới hạn 10 vòng**: để tránh đồng đội lặp vô hạn. CC thật dùng idle loop: chạy xong một vòng thì gửi `idle_notification`, đợi tin nhắn trong inbox, nhận được thì tiếp tục, cho đến khi nhận được `shutdown_request` mới thoát
- **Tự động báo cáo khi hoàn thành**: `BUS.send(name, "lead", summary)` gửi kết quả cuối cùng vào inbox của Lead

### Inbox injection của Lead

Lead kiểm tra inbox sau mỗi vòng của vòng lặp chính. Tin nhắn từ đồng đội được chèn vào history, để LLM nhìn thấy và phản ứng:

```python
# Sau khi vòng lặp chính kết thúc
inbox = BUS.read_inbox("lead")
if inbox:
    inbox_text = "\n".join(
        f"From {m['from']}: {m['content'][:200]}" for m in inbox)
    history.append({"role": "user",
                    "content": f"[Inbox]\n{inbox_text}"})
```

Bản giảng dạy chèn ngoài vòng lặp nhập liệu người dùng. CC chi tiết hơn, `useInboxPoller` của Lead kiểm tra mỗi 1 giây, có tin nhắn thì submit thành turn mới, không cần đợi người dùng nhập.

### Permission bubbling

Bản giảng dạy lược bỏ permission bubbling. Quy trình thật của CC (`permissionSync.ts`, `useSwarmPermissionPoller.ts`):

1. Đồng đội gặp thao tác cần phê duyệt → gửi `permission_request` vào inbox của Lead
2. `useInboxPoller` của Lead phát hiện request → route vào hàng đợi phê duyệt
3. Sau khi user phê duyệt → Lead gửi `permission_response` về cho đồng đội
4. `useSwarmPermissionPoller` của đồng đội (poll mỗi 500ms) nhận được reply → tiếp tục hoặc từ chối

### Chạy toàn bộ cùng nhau

```
1. Lead: "Dựng backend: một người không xong, lập nhóm đi"
2. Lead → spawn_teammate("alice", "backend dev", "Tạo database schema")
3. Lead → spawn_teammate("bob", "frontend dev", "Viết API client")
4. Alice thread khởi động → LLM gọi riêng → bash "python manage.py migrate"
5. Bob thread khởi động → LLM gọi riêng → write_file("client.ts", ...)
6. Alice xong → BUS.send("alice", "lead", "Schema done: users, orders tables")
7. Bob xong → BUS.send("bob", "lead", "Client written with types")
8. Vòng tiếp theo của Lead → inbox chèn vào history → LLM nhìn thấy kết quả của alice và bob
```

Hai đồng đội làm việc song song.

---

## Thay đổi so với s14

| Thành phần | Trước đó (s14) | Sau đó (s15) |
|------|-----------|-----------|
| Số lượng Agent | 1 | 1 Lead + N luồng đồng đội |
| Giao tiếp | Không có | MessageBus + .mailboxes/*.jsonl |
| Class mới | — | MessageBus, active_teammates dict |
| Hàm mới | — | spawn_teammate_thread, run_send_message, run_check_inbox |
| Tool của Lead | 11 (s14) | + spawn_teammate, send_message, check_inbox (14) |
| Tool của đồng đội | — | bash, read_file, write_file, send_message (4) |
| Quyền hạn | Quyết định cục bộ | Bản giảng dạy lược bỏ (CC thật có cơ chế bubbling) |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s15_agent_teams/code.py
```

Thử các prompt sau:

1. `Spawn alice as a backend developer. Ask her to create a file called schema.sql with a users table.`
2. `Check your inbox for alice's result.`
3. `Spawn bob as a tester. Ask him to check if schema.sql exists and list its contents.`

Quan sát trọng tâm: Lead khởi tạo đồng đội như thế nào? Các file JSONL trong thư mục `.mailboxes/` trông như thế nào? Sau khi đồng đội hoàn thành, inbox có được chèn vào history của Lead không?

---

## Tiếp theo

Đồng đội có thể làm việc và giao tiếp được. Nhưng nếu Lead muốn tắt Alice, kill thread trực tiếp sẽ để lại file đang viết dở. Cần một giao thức tắt máy lịch sự: Lead gửi shutdown_request, đồng đội dọn dẹp rồi mới thoát.

s16 Team Protocols → giao thức handshake tắt máy và quy ước tin nhắn.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Nội dung dưới đây dựa trên phân tích đầy đủ mã nguồn CC `spawnMultiAgent.ts`, `useInboxPoller.ts` (969 dòng), `useSwarmPermissionPoller.ts` (330 dòng), `teammateMailbox.ts`, `teamHelpers.ts`.

### 1. Không có message bus trung tâm, mà là hệ thống file

Bản giảng dạy dùng class `MessageBus` để gửi/nhận tin nhắn. Cách tiếp cận của CC trực tiếp hơn, mỗi Agent trực tiếp ghi vào file inbox của Agent khác.

Đường dẫn inbox: `~/.claude/teams/{teamName}/inboxes/{agentName}.json`

Khi ghi dùng `proper-lockfile` để khóa file đảm bảo an toàn ghi đồng thời (tối đa retry 10 lần). Mỗi file là một mảng JSON, khi append tin nhắn mới thì đọc→thêm→ghi lại.

### 2. 15 loại tin nhắn

CC có 15 loại tin nhắn có cấu trúc cho giao tiếp đội (`teammateMailbox.ts`):

| Loại | Hướng | Mục đích |
|------|------|------|
| `plain text` | Hai chiều | Giao tiếp bình thường giữa đồng đội |
| `idle_notification` | Đồng đội→Lead | Đồng đội hoàn thành một vòng làm việc, vào trạng thái idle |
| `permission_request` | Đồng đội→Lead | Đồng đội cần phê duyệt thao tác |
| `permission_response` | Lead→Đồng đội | Kết quả phê duyệt từ Lead |
| `plan_approval_request` | Đồng đội→Lead | Đồng đội gửi kế hoạch đợi phê duyệt |
| `plan_approval_response` | Lead→Đồng đội | Lead phê duyệt kế hoạch |
| `shutdown_request` | Lead→Đồng đội | Yêu cầu tắt máy lịch sự |
| `shutdown_approved` | Đồng đội→Lead | Xác nhận tắt máy |
| `shutdown_rejected` | Đồng đội→Lead | Từ chối tắt máy (kèm lý do) |
| `task_assignment` | Lead→Đồng đội | Giao việc |
| `team_permission_update` | Lead→Đồng đội | Broadcast thay đổi quyền hạn |
| `mode_set_request` | Lead→Đồng đội | Sửa mode quyền hạn của đồng đội |
| `sandbox_permission_*` | Hai chiều | Yêu cầu/phản hồi quyền mạng |
| `teammate_terminated` | Hệ thống | Thông báo đồng đội bị gỡ bỏ |

Tin nhắn text được bọc trong thẻ XML `<teammate-message>` để đưa cho model.

### 3. Permission bubbling: poll hai chiều

Bản giảng dạy lược bỏ permission bubbling. Quy trình thực tế của CC (`permissionSync.ts`):

1. **Đồng đội** gặp thao tác cần phê duyệt → gửi `permission_request` vào inbox của Lead
2. **Lead** của `useInboxPoller` (poll mỗi 1 giây) phát hiện request → route vào `ToolUseConfirmQueue`
3. UI của Lead hiển thị hộp thoại phê duyệt, kèm tên và màu của đồng đội
4. Sau khi user phê duyệt → Lead gửi `permission_response` vào inbox của đồng đội
5. **Đồng đội** của `useSwarmPermissionPoller` (poll mỗi 500ms) nhận được reply → tiếp tục hoặc từ chối thực thi

### 4. Vòng đời đồng đội

Đồng đội của CC được tạo bởi `spawnTeammate()` (`spawnMultiAgent.ts`):

1. **Spawn**: Tạo cửa sổ tmux (hoặc trong tiến trình), gán màu, ghi vào team config
2. **Work**: `useInboxPoller` mỗi 1 giây kiểm tra inbox → có tin nhắn thì submit thành turn mới
3. **Idle**: Stop hook kích hoạt → gửi `idle_notification` cho Lead
4. **Shutdown**: Lead gửi `shutdown_request` → đồng đội reply `shutdown_approved` → Lead dọn dẹp

### 5. Team Config

Team registry nằm ở `~/.claude/teams/{teamName}/config.json` (`teamHelpers.ts`):

```json
{
  "name": "my-team",
  "leadAgentId": "lead@my-team",
  "members": [{
    "agentId": "researcher@my-team",
    "name": "researcher",
    "agentType": "general-purpose",
    "color": "blue",
    "isActive": true
  }]
}
```

Đồng đội không thể lồng nhau (`AgentTool.tsx:273` rõ ràng cấm "teammates spawning other teammates").

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->