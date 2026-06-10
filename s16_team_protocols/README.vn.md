# s16: Team Protocols — Đồng đội cần có quy ước

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s14 → s15 → `s16` → [s17](../s17_autonomous_agents/) → s18 → s19 → s20
> *"Đồng đội cần có quy ước"* — request-response pattern thúc đẩy thương lượng.
>
> **Tầng Harness**: giao thức — handshake có cấu trúc giữa các Agent.

---

## Vấn đề

Đồng đội ở s15 đã có thể làm việc, nhưng việc điều phối còn lỏng lẻo: Lead gửi tin nhắn, đồng đội trả lời, không có giao thức có cấu trúc. Hai tình huống bộc lộ vấn đề:

**Tắt máy**: Lead muốn tắt Alice. Kill thread trực tiếp, file Alice đang viết dở sẽ nằm trên đĩa. Cần handshake: Lead gửi request, Alice xác nhận dọn dẹp rồi mới tắt máy.

**Phê duyệt kế hoạch**: Bob muốn refactor module authentication, đây là thao tác rủi ro cao. Nên để Lead xem kế hoạch của Bob trước, phê duyệt rồi mới được làm.

Hai tình huống này có cấu trúc hoàn toàn giống nhau: một bên gửi request, bên kia trả lời, request và reply liên kết qua cùng một ID. Có state machine để theo dõi: pending → approved / rejected.

---

## Giải pháp

![Team Protocols Overview](images/team-protocols-overview.svg)

Code giảng dạy tiếp nối khả năng Agent từ các chương trước, thêm giao thức có cấu trúc dựa trên nền tảng giao tiếp đội của s15. Để tập trung vào cơ chế giao thức, nó lược bỏ phần phục hồi lỗi đầy đủ, memory và skill system. Thêm mới ba thứ: **ProtocolState** (theo dõi trạng thái request), **dispatch_message** (route theo loại tin nhắn đến handler), **match_response** (liên kết reply với request qua request_id, có kiểm tra loại).

Hai giao thức, một cơ chế:

| Giao thức | Hướng | Mục đích |
|------|------|------|
| shutdown_request / response | Lead → đồng đội | Handshake tắt máy lịch sự |
| plan_approval_request / response | Đồng đội → Lead | Ví dụ giao thức phê duyệt kế hoạch |

> Bản giảng dạy minh họa luồng tin nhắn request-response của plan approval, chưa triển khai gating thực thi (chặn bash/write_file khi chưa approved). Đồng đội thật của CC có cơ chế permission gating.

---

## Nguyên lý hoạt động

### ProtocolState: trạng thái request

Mỗi request protocol tạo một bản ghi trạng thái, ghi lại ai gửi, gửi cho ai, trạng thái hiện tại, nội dung đi kèm:

```python
@dataclass
class ProtocolState:
    request_id: str      # ID duy nhất, ví dụ "req_004281"
    type: str            # "shutdown" | "plan_approval"
    sender: str          # Bên gửi
    target: str          # Bên nhận
    status: str          # pending | approved | rejected
    payload: str         # Text kế hoạch hoặc lý do tắt máy
    created_at: float    # Timestamp

pending_requests: dict[str, ProtocolState] = {}
```

Gửi request thì tạo bản ghi, nhận reply thì tìm bản ghi tương ứng qua `request_id`, cập nhật trạng thái.

### Bốn bước luồng protocol

Lấy ví dụ tắt máy, luồng đầy đủ:

```
① Lead gửi request
   req_id = new_request_id()           # "req_004281"
   pending_requests[req_id] = ProtocolState(type="shutdown", status="pending", ...)
   BUS.send("lead", "alice", "shutdown_request", metadata={"request_id": req_id})

② Đồng đội nhận → dispatch
   inbox = BUS.read_inbox("alice")
   msg_type = msg["type"]              # "shutdown_request"
   → route đến handle_shutdown_request()

③ Đồng đội reply
   BUS.send("alice", "lead", "shutdown_response",
            metadata={"request_id": req_id, "approve": True})

④ Lead nhận response → match
   match_response("shutdown_response", req_id, approve=True)
   pending_requests[req_id].status = "approved"
```

`request_id` là khóa liên kết xuyên suốt cả chain, request mang nó đi, reply mang nó về.

> Bản giảng dạy dùng `shutdown_response` thống nhất (trường approve phân biệt đồng ý/từ chối). Mã nguồn thật tách thành hai loại tin nhắn độc lập là `shutdown_approved` và `shutdown_rejected` (`teammateMailbox.ts:720-763`).

### dispatch_message: route theo loại

Inbox của đồng đội không chỉ nhận tin nhắn thường mà còn nhận tin nhắn protocol. `handle_inbox_message` phân phối theo loại tin nhắn:

```python
def handle_inbox_message(name, msg, messages):
    msg_type = msg.get("type", "message")
    req_id = msg.get("metadata", {}).get("request_id", "")

    if msg_type == "shutdown_request":
        BUS.send(name, "lead", "Shutting down.", "shutdown_response",
                 {"request_id": req_id, "approve": True})
        return True   # Dừng vòng lặp

    if msg_type == "plan_approval_response":
        approve = msg["metadata"].get("approve", False)
        messages.append({"role": "user",
            "content": "[Plan approved]" if approve else "[Plan rejected]"})
    return False       # Tiếp tục vòng lặp
```

Thêm loại protocol mới chỉ cần thêm nhánh `if` mới.

### match_response: kiểm tra loại

`match_response` không chỉ tìm trạng thái theo `request_id`, mà còn kiểm tra loại response có khớp với loại request không:

```python
def match_response(response_type, request_id, approve):
    state = pending_requests.get(request_id)
    if not state:
        return
    if state.type == "shutdown" and response_type != "shutdown_response":
        return  # type mismatch, skip
    if state.type == "plan_approval" and response_type != "plan_approval_response":
        return
    if state.status != "pending":
        return  # already resolved, skip duplicate
    state.status = "approved" if approve else "rejected"
```

Một shutdown_response không thể vô tình approve một plan_approval request.

### Tiêu thụ inbox thống nhất: consume_lead_inbox

Cả tool `check_inbox` và cuối vòng lặp chính đều gọi cùng một hàm `consume_lead_inbox()`, route protocol message trước rồi mới trả lại phần còn lại, tránh tin nhắn bị đọc mà trạng thái protocol không cập nhật:

```python
def consume_lead_inbox(route_protocol=True) -> list[dict]:
    msgs = BUS.read_inbox("lead")
    if route_protocol:
        for msg in msgs:
            meta = msg.get("metadata", {})
            req_id = meta.get("request_id", "")
            msg_type = msg.get("type", "")
            if req_id and msg_type.endswith("_response"):
                match_response(msg_type, req_id, meta.get("approve", False))
    return msgs
```

Cuối vòng lặp chính còn chèn tin nhắn inbox vào `history`, để LLM nhìn thấy và phản ứng.

### Vòng lặp idle của đồng đội: đợi thay vì thoát

Đồng đội ở s15 chạy 10 vòng rồi thoát. Đồng đội ở s16 sau khi LLM trả về non-tool_use thì vào trạng thái idle đợi: poll inbox, nhận được shutdown_request thì reply rồi thoát, nhận được tin nhắn mới thì tiếp tục làm việc.

```
LLM trả về non-tool_use
  → idle: mỗi giây poll inbox
  → nhận được shutdown_request → reply shutdown_response → thoát
  → nhận được tin nhắn mới → chèn vào messages → tiếp tục LLM turn
```

Bản giảng dạy lược bỏ thông báo idle_notification gửi cho Lead. CC thật khi idle sẽ gửi `idle_notification`, Lead nhận được biết đồng đội rảnh, có thể giao việc mới.

### Chạy toàn bộ cùng nhau

```
1. Lead: "Cho Alice tạo một file rồi tắt máy"
2. Lead → spawn_teammate("alice", "backend", "Tạo config.py")
3. Alice thread khởi động → write_file("config.py", "...") → xong → idle
4. Lead → request_shutdown("alice")
   → BUS.send("shutdown_request", {request_id: "req_000142"})
5. Alice idle poll nhận được → handle_shutdown_request
   → BUS.send("shutdown_response", {request_id: "req_000142", approve: True})
6. Lead consume_lead_inbox → match_response("req_000142", approve=True)
   → pending_requests["req_000142"].status = "approved"
   → Tin nhắn inbox chèn vào history, LLM nhận được kết quả tắt máy
```

Tắt máy handshake hoàn chỉnh: request → confirm → tắt máy. Mỗi bước có `request_id` để trace.

---

## Thay đổi so với s15

| Thành phần | Trước đó (s15) | Sau đó (s16) |
|------|-----------|-----------|
| Cách điều phối | Tin nhắn text lỏng lẻo | Giao thức request-response có cấu trúc |
| Theo dõi request | Không có | ProtocolState + dict pending_requests |
| Route tin nhắn | Xử lý tất cả như text | dispatch_message route theo loại |
| Tắt máy | Thoát tự nhiên hoặc kill thread | Cơ chế handshake với request_id |
| Phê duyệt kế hoạch | Không có | Luồng tin nhắn ví dụ (chưa triển khai gating thực thi) |
| Loại tin nhắn mới | message, result | + shutdown_request/response, plan_approval_request/response |
| Vòng đời đồng đội | Tối đa 10 vòng | idle loop (đợi tin nhắn inbox) |
| Inbox của Lead | check_inbox và vòng lặp chính đọc riêng | consume_lead_inbox thống nhất |
| Tool của Lead | 14 (s15) | 14 (thêm request_shutdown, request_plan, review_plan vào bộ tool cốt lõi) |
| Tool của đồng đội | 4 (s15) | + submit_plan (5) |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s16_team_protocols/code.py
```

Thử các prompt sau:

1. `Spawn alice as a backend dev. Ask her to create a file. Then request her shutdown.`
2. `Spawn bob with a refactoring task. Have him submit a plan first. Then review and approve it.`

Quan sát trọng tâm: Handshake tắt máy có hoàn chỉnh không (request → confirm → tắt máy)? Trạng thái của `pending_requests` có chuyển đổi đúng không? `request_id` có giữ nhất quán giữa request và response không? Đồng đội idle sau đó có nhận được shutdown_request không?

---

## Tiếp theo

Trong s15-s16, Lead phải gán việc cho từng đồng đội. "Alice làm cái này, Bob làm cái kia". Trên bảng tác vụ có 10 tác chưa ai nhận, Lead phải assign thủ công.

Có thể để đồng đội tự nhìn bảng, tự nhận việc không? Lead chỉ cần tạo tác vụ, đồng đội tự tìm, tự nhận, tự làm.

s17 Autonomous Agents → đồng đội tự tổ chức, không cần Lead gán.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

Triển khai team protocol của CC (`teammateMailbox.ts`, 1184 dòng) và bản giảng dạy nhất quán về cấu trúc cốt lõi: request_id + pattern request-response với approve/reject. Khác biệt nằm ở:

**Giao thức tắt máy**: shutdown của CC là giao tiếp ba chiều (`teammateMailbox.ts:720-763`, `SendMessageTool.ts:268-430`). Lead gửi `shutdown_request`, đồng đội reply `shutdown_approved` (hoặc `shutdown_rejected` kèm lý do), hệ thống gửi thông báo `teammate_terminated` cho tất cả bên liên quan. Sau khi xác nhận tắt máy, hệ thống tự động dọn dẹp pane (tmux/iTerm2), unassign tác vụ, xóa thành viên khỏi team config (`useInboxPoller.ts:677-800`). Bản giảng dạy dùng `shutdown_response` thống nhất, mã nguồn thật tách approved/rejected thành hai loại tin nhắn độc lập.

**Phê duyệt kế hoạch**: Trong mã nguồn thật, plan approval request được tạo bởi `ExitPlanModeV2Tool.ts:263-312` khi đồng đội ở chế độ plan-mode-required thoát khỏi plan mode. `useInboxPoller.ts:599-661` hiện tự động ghi lại approval và đưa request cho Lead làm ngữ cảnh (regular message). `SendMessageTool.ts:434-518` vẫn giữ khả năng reply approve/reject tường minh, khi phê duyệt có thể đồng thời đặt `permissionMode` (ví dụ "approved but run in plan mode"), phản hồi có thể chứa `feedback` string để đồng đội sửa rồi submit lại. Không phải luồng "Lead dùng tool review_plan thủ công" đơn giản.

**Định dạng tin nhắn**: Tin nhắn protocol của CC là JSON có cấu trúc (có Zod schema validation), bản giảng dạy dùng type + metadata dictionary đơn giản. Tên trường cũng không đồng nhất: permission dùng `request_id` (`teammateMailbox.ts:453-462`), shutdown và plan approval dùng `requestId` (`teammateMailbox.ts:684-763`).

**Gating thực thi**: Đồng đội của CC có permission gating đầy đủ. Thao tác rủi ro cao chưa được phê duyệt sẽ bị chặn, không phải tùy chọn. Bản giảng dạy chỉ minh họa luồng tin nhắn, chưa triển khai chặn thực thi.

**Tính phổ quát**: Một FSM của bản giảng dạy (pending → approved | rejected) tương ứng với hai giao thức, cách đơn giản hóa này hoàn toàn đúng. Tất cả tin nhắn protocol của CC dùng chung cơ chế liên kết request id.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->