# s19: Công Cụ MCP — Công Cụ Bên Ngoài, Giao Thức Chuẩn

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s17 → s18 → `s19` → [s20](../s20_comprehensive/)

> *"Công cụ bên ngoài, giao thức chuẩn"* — Khám phá, lắp ráp, gọi, Agent không cần biết công cụ được viết bởi ai.
>
> **Lớp Harness**: Plugin — Khả năng bên ngoài tiếp cận thông qua giao thức chuẩn.

---

## Vấn đề

Từ s01 đến s18, tất cả công cụ của Agent đều được viết thủ công — bash, read, write, task, worktree. Xác thực đầu vào, logic thực thi, xử lý lỗi của mỗi công cụ, đều là bạn viết từng dòng.

Bây giờ bạn có 3 dịch vụ bên ngoài muốn tiếp cận: Jira API của công ty (tra cứu issue, tạo ticket), hệ thống triển khai tự xây dựng (kích hoạt deploy, xem nhật ký), kho kiến thức Notion của nhóm (tìm kiếm tài liệu, tạo trang). Bạn không muốn viết lại một bộ công cụ cho mỗi dịch vụ.

Bạn cần một giao thức chuẩn — dịch vụ bên ngoài chỉ cần triển khai nó, Agent sẽ có thể gọi trực tiếp, bất kể dịch vụ được viết bằng ngôn ngữ nào.

---

## Giải pháp

![Kiến trúc MCP](images/mcp-architecture.svg)

MCP（Model Context Protocol）định nghĩa cách Agent khám phá và gọi công cụ bên ngoài. Các khái niệm cốt lõi:

| Khái niệm | Tác dụng |
|------|------|
| MCPClient | Máy khách ở phía Agent, kết nối server, khám phá công cụ, gọi công cụ |
| MCP Server | Dịch vụ bên ngoài, triển khai `tools/list` + `tools/call` |
| assemble_tool_pool | Lắp ráp công cụ nội tạo và công cụ MCP thành một bể công cụ |
| mcp\_\_server\_\_tool naming | Tránh xung đột tên công cụ giữa các server khác nhau |

Sử dụng lại phiên bản giáo dục cách ly worktree từ s18, tự động ghi danh, kiểm tra rảnh rỗi, hệ thống giao thức. Chương này thêm mới: công cụ `connect_mcp` — kết nối dịch vụ bên ngoài, khám phá công cụ, thêm vào bể công cụ.

Phiên bản giáo dục sử dụng mock handler để mô phỏng server bên ngoài. Phiên bản thực sẽ khởi động quy trình con, gửi yêu cầu JSON-RPC thông qua stdin/stdout. Lợi ích của mock là không phụ thuộc vào dịch vụ bên ngoài có thể chạy toàn bộ luồng; chi phí là bạn không thể thấy giao tiếp mạng và quản lý quy trình thực sự.

---

## Nguyên tắc hoạt động

### MCPClient: Khám phá + Gọi

```python
class MCPClient:
    def __init__(self, name: str):
        self.name = name
        self.tools: list[dict] = []
        self._handlers: dict[str, callable] = {}

    def register(self, tool_defs, handlers):
        """Mô phỏng khám phá tools/list."""
        self.tools = tool_defs
        self._handlers = handlers

    def call_tool(self, tool_name: str, args: dict) -> str:
        """Mô phỏng tools/call."""
        handler = self._handlers.get(tool_name)
        if not handler:
            return f"MCP error: unknown tool '{tool_name}'"
        return handler(**args)
```

Phiên bản giáo dục sử dụng hàm Python để mô phỏng triển khai công cụ của server. Phiên bản thực giao tiếp với quy trình con thông qua JSON-RPC stdio.

### connect_mcp: Kết nối + Khám phá

```python
def connect_mcp(name: str) -> str:
    if name in mcp_clients:
        return f"MCP server '{name}' already connected"
    factory = MOCK_SERVERS.get(name)
    if not factory:
        return f"Unknown server '{name}'. Available: ..."
    mcp_client = factory()
    mcp_clients[name] = mcp_client
    return f"Connected to '{name}'. Discovered: ..."
```

Sau khi kết nối, công cụ do server cung cấp có sẵn ngay.

### normalize_mcp_name: Chuẩn hóa tên

```python
_DISALLOWED_CHARS = re.compile(r'[^a-zA-Z0-9_-]')

def normalize_mcp_name(name: str) -> str:
    return _DISALLOWED_CHARS.sub('_', name)
```

Thay thế tất cả các ký tự không phải `[a-zA-Z0-9_-]` bằng `_`. Ngăn chặn tên server hoặc tên công cụ chứa ký tự đặc biệt dẫn đến xung đột đặt tên hoặc vấn đề tiêm.

### assemble_tool_pool: Lắp ráp bể công cụ

```python
def assemble_tool_pool() -> tuple[list[dict], dict]:
    tools = list(BUILTIN_TOOLS)
    handlers = dict(BUILTIN_HANDLERS)
    for server_name, mcp_client in mcp_clients.items():
        safe_server = normalize_mcp_name(server_name)
        for tool_def in mcp_client.tools:
            safe_tool = normalize_mcp_name(tool_def["name"])
            prefixed = f"mcp__{safe_server}__{safe_tool}"
            tools.append(...)
            handlers[prefixed] = (
                lambda *, c=mcp_client, t=tool_def["name"], **kw:
                    c.call_tool(t, kw))
    return tools, handlers
```

Tiền tố `mcp__{server}__{tool}` tránh xung đột tên công cụ giữa các server khác nhau. Tên được chuẩn hóa thông qua `normalize_mcp_name`.

Mô tả công cụ MCP có ghi chú `(readOnly)` hoặc `(destructive)` — phiên bản giáo dục sử dụng ghi chú văn bản, CC thực sẽ sử dụng cấu trúc tool annotations để hệ thống quyền hạn đánh giá.

### Không bộ nhớ cache: Bể công cụ thay đổi, prompt cũng thay đổi

Agent loop từ s10-s18 sử dụng prompt cache để tránh tuần tự hóa lại. s19 bỏ đi bộ nhớ cache:

```python
def agent_loop(messages, context):
    tools, handlers = assemble_tool_pool()     # Xây dựng lại mỗi lần
    system = assemble_system_prompt(context)    # Tạo lại mỗi lần
    ...
    if any(b.name == "connect_mcp" ...):
        tools, handlers = assemble_tool_pool()  # Xây dựng lại sau kết nối
        system = assemble_system_prompt(context)
```

Lý do: sau `connect_mcp`, bể công cụ đã thay đổi — thêm các công cụ như `mcp__docs__search`. Danh sách công cụ trong bộ nhớ cache là cũ, tiếp tục sử dụng sẽ khiến model không thể gọi được công cụ mới. Phiên bản giáo dục bỏ đi bộ nhớ cache trực tiếp, chi phí là thêm một chút thời gian tuần tự hóa.

### Công cụ MCP chỉ Lead có thể sử dụng

Trong phiên bản giáo dục, `connect_mcp` là công cụ Lead, `assemble_tool_pool` cũng chỉ phục vụ agent_loop của Lead. Teammate vẫn sử dụng 8 công cụ con cố định (bash, read_file, write_file, send_message, submit_plan, list_tasks, claim_task, complete_task).

Đây là đơn giản hóa giáo dục. Trong CC thực, công cụ MCP có sẵn cho cả agent chính và sub agent — sub agent kế thừa cấu hình MCP của cấp cha.

---

## Thay đổi so với s18

| Thành phần | Trước đó (s18) | Sau đó (s19) |
|------|-----------|-----------|
| Nguồn công cụ | Toàn bộ nội tạo thủ công | Nội tạo thủ công + công cụ bên ngoài MCP khám phá động |
| Bể công cụ | BUILTIN_TOOLS cố định | assemble_tool_pool lắp ráp động công cụ với tiền tố mcp\_\_ |
| An toàn tên | Không có | normalize_mcp_name chuẩn hóa |
| Loại mới | — | Lớp MCPClient (mô phỏng tools/list + tools/call) |
| Không gian tên | — | mcp\_\_server\_\_tool tránh xung đột |
| Mô tả công cụ | Không có ghi chú | (readOnly)/(destructive) ghi chú |
| prompt cache | Có (từ s10) | Bỏ đi — bể công cụ thay đổi động khiến bộ nhớ cache vô hiệu |
| Công cụ Lead | 17 (s18) | 18 (+connect_mcp) |
| Công cụ Teammate | 8 (s18) | 8 (không thay đổi, công cụ MCP chỉ Lead có) |
| Phương thức mở rộng | Viết code thêm công cụ | Giao thức chuẩn, bất kỳ ngôn ngữ nào triển khai server |

---

## Thử một lần

```sh
cd learn-claude-code
python s19_mcp_plugin/code.py
```

Thử những prompt này:

1. `Connect to the docs MCP server and search for something`
2. `Connect to the deploy server and trigger a deployment`
3. `Connect both servers — what tools are now available?`

Các điểm quan trọng để quan sát: Sau khi kết nối MCP server, tên công cụ có mang tiền tố `mcp__docs__` hoặc `mcp__deploy__` không? Công cụ của hai server có cùng có sẵn không? Mô tả công cụ MCP có mang ghi chú (readOnly)/(destructive) không?

---

## Tiếp theo

Bây giờ Agent có thể tiếp cận công cụ bên ngoài thông qua giao thức chuẩn rồi. Nhưng trước 19 chương, mỗi chương chỉ thêm một cơ chế, agent thực không chạy riêng lẻ như vậy.

Công cụ, quyền hạn, hooks, todo, biểu đồ nhiệm vụ, ký ức, nén, nền, cron, nhóm, worktree, MCP — những cơ chế này nên gắn vào cùng một vòng lặp, chứ không phải rải rác trong 19 demo.

s20 Comprehensive Agent → kết hợp các cơ chế từ 19 chương trước vào một harness hoàn chỉnh. Cơ chế rất nhiều, vòng lặp chỉ có một.

<details>
<summary>Sâu vào mã nguồn CC</summary>

> Phân tích sau đây dựa trên mã nguồn CC `services/mcp/client.ts`, `auth.ts`, `config.ts`, `channelNotification.ts`.

### Một, 6 loại Transport

Phiên bản giáo dục chỉ trình diễn mock stdio. CC hỗ trợ 6 loại truyền tải (`types.ts:23-25`):

| Transport | Phương thức giao tiếp |
|-----------|---------|
| `stdio` | stdin/stdout quy trình con (mặc định đa nền tảng) |
| `sse` | HTTP Server-Sent Events |
| `http` | Streamable HTTP (POST/SSE hai chiều) |
| `ws` | WebSocket |
| `sse-ide` | Truyền tải SSE nhúng IDE |
| `sdk` | Truyền tải SDK trong quy trình |

Khi kết nối, máy chủ cục bộ (stdio) và từ xa (http/sse/ws) xử lý song song theo batch: 3 cục bộ, 20 từ xa.

### Hai, Thuật toán lắp ráp bể công cụ

`assembleToolPool()`（`tools.ts:345-364`）:

```typescript
// Loại bỏ trùng lặp ưu tiên giữ công cụ nội tạo (công cụ nội tạo ở trước nếu tên giống)
return uniqBy(
  [...builtInTools.sort(byName), ...filteredMcpTools.sort(byName)],
  'name',
)
```

Công cụ nội tạo và công cụ MCP được sắp xếp riêng, không phải hợp nhất sắp xếp. Lý do là `claude_code_system_cache_policy` của CC đặt điểm ngắt bộ nhớ cache toàn cục ở một vị trí nào đó sau công cụ nội tạo cuối cùng — sắp xếp hỗn hợp sẽ phá vỡ thiết kế này.

### Ba, Quy tắc đặt tên: `mcp__server__tool`

`buildMcpToolName()`（`mcpStringUtils.ts:50-52`）:

```
mcp__<normalizedServerName>__<normalizedToolName>
```

Thay thế tất cả các ký tự không phải `[a-zA-Z0-9_-]` bằng `_`（`normalization.ts:17-23`）. Phiên bản giáo dục `normalize_mcp_name` sử dụng quy tắc giống nhau.

### Bốn, Kiểm tra quyền hạn

CC có hệ thống quyền hạn độc lập cho công cụ MCP. Logic kiểm tra `checkPermissions()` của công cụ MCP khác với công cụ nội tạo — công cụ MCP có thể khai báo yêu cầu quyền hạn của riêng nó (readOnly, destructive, v.v.), CC quyết định dựa trên khai báo có cần xác nhận người dùng không. Phiên bản giáo dục chỉ dùng ghi chú văn bản `(readOnly)` / `(destructive)` trong mô tả, không chặn quyền hạn.

### Năm, Nguồn cấu hình và mức độ ưu tiên

Cấu hình máy chủ MCP đến từ nhiều nguồn. Mức độ ưu tiên cấu hình của CC từ thấp đến cao:

```
Trình kết nối claude.ai < plugin < user settings.json < approved project .mcp.json < local settings.local.json
```

Trình kết nối `claude.ai` được kéo riêng, loại bỏ trùng lặp theo chữ ký nội dung, hợp nhất với mức độ ưu tiên thấp nhất（`config.ts:1267-1289`）. Khi `managed-mcp.json` của doanh nghiệp tồn tại, loại trừ hoàn toàn các cấu hình khác.

Phiên bản giáo dục truyền trực tiếp tên server cho từ điển `MOCK_SERVERS`, không hợp nhất cấu hình.

### Sáu, Channel Notification: Server gửi tin nhắn ngược

Phiên bản giáo dục chỉ nói về gọi một chiều Agent → MCP Server. CC còn hỗ trợ thông báo ngược（`channelNotification.ts`）:

1. Server khai báo `capabilities.experimental['claude/channel']`
2. Server gửi thông báo `notifications/claude/channel` cho Agent thông qua MCP
3. Tin nhắn được bao bọc trong thẻ XML `<channel source="serverName">...</channel>`
4. Agent được SleepTool đánh thức (trong 1 giây)

Server còn có thể yêu cầu quyền hạn: `notifications/claude/channel/permission_request` → Agent trả lời `notifications/claude/channel/permission`. Người dùng xác nhận/từ chối qua ID ngắn 5 chữ cái.

### Bảy, Luồng xác thực OAuth

Xác thực MCP của CC（`auth.ts`）hỗ trợ luồng OAuth 2.0 + PKCE hoàn chỉnh:
- Khám phá siêu dữ liệu OAuth thông qua máy khách công khai + PKCE（RFC 8414 / RFC 9728）
- Máy chủ gọi lại cục bộ nhận mã ủy quyền
- Token được lưu trữ bền vững thông qua `getSecureStorage()`（macOS Keychain / tệp mã hóa Linux / Trình quản lý thông tin đăng nhập Windows）
- Tự động làm mới 5 phút trước hết hạn
- Hỗ trợ truy cập ứng dụng chéo（XAA）: trình duyệt lấy id_token → RFC 8693 + RFC 7523 trao đổi → không cần bật lại trình duyệt

### Tám, Xử lý lỗi vòng đời kết nối

CC có phân loại lỗi tốt và thử lại cho kết nối MCP（`client.ts:1266-1402`）:
- Lỗi cuối cùng（ECONNRESET, ETIMEDOUT, EPIPE, v.v.）: liên tiếp 3 lần → đóng + kết nối lại
- Gọi công cụ 401: Token hết hạn → ném `McpAuthError` → kích hoạt xác thực lại
- Gọi công cụ hết thời: `Promise.race` hết thời（có thể cấu hình, mặc định khoảng 28 giờ）
- Stdio ngắt kết nối: giết quy trình theo thứ tự SIGINT → SIGTERM → SIGKILL

### Đơn giản hóa phiên bản giáo dục

- 6 loại transport → 1 loại（mock stdio）: khối lượng khái niệm có thể kiểm soát
- Channel thông báo ngược → bỏ qua: phiên bản giáo dục Agent là bên chủ động
- Luồng OAuth → bỏ qua: phiên bản giáo dục giả định server không cần xác thực
- Mức độ ưu tiên cấu hình nhiều lớp → bỏ qua: phiên bản giáo dục truyền trực tiếp tên server
- Phân loại lỗi phức tạp → bỏ qua: phiên bản giáo dục dùng try/except bao lại
- Công cụ MCP chỉ cho Lead → bỏ qua kế thừa sub agent: đơn giản hóa cấu trúc code

</details>

<!-- translation-sync: zh@v2, en@v0, ja@v0, vi@v1 -->
