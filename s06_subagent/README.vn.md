# s06: Subagent — Tách việc lớn thành việc nhỏ, mỗi phần có context riêng

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → s03 → s04 → s05 → `s06` → [s07](../s07_skill_loading/) → s08 → ... → s20

> *"Task lớn thì chia nhỏ, mỗi task con có clean context riêng"* — Subagent dùng `messages[]` độc lập, không làm bẩn cuộc hội thoại chính.
>
> **Harness layer**: subagent — cách ly context để attention không bị trôi.

---

## Vấn đề

Agent đang sửa một bug. Nó đã đọc 30 file để lần theo call chain, rồi trao đổi qua lại hơn 60 lượt. Lúc này `messages` đã phình lên hơn 100 entries, mà phần lớn trong số đó chỉ là dấu vết trung gian của việc “đi lần call chain”, chứ không còn liên quan trực tiếp đến mục tiêu cuối cùng là sửa bug.

Những thông tin trung gian đó chiếm chỗ trong context, khiến Agent ngày càng “đãng trí”: nó không còn giữ được trọng tâm ban đầu nữa.

Nếu nhìn theo cách con người làm việc thì chuyện này rất tự nhiên: khi debug, bạn thường mở một terminal khác để truy vết. Truy xong thì đóng terminal đó lại, ghi kết luận vào ghi chú, rồi quay về terminal chính để tiếp tục xử lý bug. Agent cũng cần đúng khả năng đó: mở một tiến trình phụ độc lập, cho nó một danh sách message riêng, để nó chỉ tập trung vào đúng một việc.

---

## Giải pháp

![Subagent Overview](images/subagent-overview.svg)

Giữ lại hook structure tối thiểu và `todo_write` tool từ chương trước, rồi thêm một tool mới: `task`.

Khi Agent gọi `task`, hệ thống sẽ spawn một subagent với `messages[]` hoàn toàn mới. Subagent chạy loop của riêng nó, làm xong thì chỉ trả lại **một đoạn kết luận cuối cùng** cho parent agent. Toàn bộ context trung gian của subagent sẽ bị bỏ đi, nhưng các side effect trên filesystem — như tạo file, sửa file, chạy lệnh — vẫn được giữ lại trong working directory.

Subagent cũng bị giới hạn toolset: nó có bash/read/write/edit/glob, nhưng **không có** `task`, nên không thể tiếp tục spawn thêm subagent khác theo kiểu đệ quy. Và dù context đã được tách riêng, mọi tool call của subagent vẫn phải đi qua permission hook như bình thường.

---

## Nguyên lý hoạt động

### `spawn_subagent()`

Hàm này tạo một `messages[]` hoàn toàn mới cho subagent, chạy loop riêng, và cuối cùng chỉ trả về kết luận:

```python
def spawn_subagent(description: str) -> str:
    # Toolset của subagent: chỉ có tool cơ bản, không có task (chặn đệ quy)
    sub_tools = [
        {"name": "bash", ...}, {"name": "read_file", ...},
        {"name": "write_file", ...}, {"name": "edit_file", ...},
        {"name": "glob", ...},
    ]
    messages = [{"role": "user", "content": description}]  # messages[] hoàn toàn mới

    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=sub_tools, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                blocked = trigger_hooks("PreToolUse", block)
                if blocked:
                    results.append({... "content": str(blocked)})
                    continue
                handler = SUB_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown"
                trigger_hooks("PostToolUse", block, output)
                results.append({... "content": output})
        messages.append({"role": "user", "content": results})

    # Chỉ trả về kết luận cuối cùng, bỏ toàn bộ quá trình trung gian
    return extract_text(messages[-1]["content"])
```

Parent agent gọi nó giống hệt như gọi các tool khác:

```python
TOOLS = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},
    {"name": "glob", ...},
    {"name": "todo_write", ...},
    # s06: thêm task tool
    {"name": "task",
     "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
     "input_schema": {"type": "object", "properties": {"description": {"type": "string"}}, "required": ["description"]}},
]

TOOL_HANDLERS["task"] = spawn_subagent
```

### Ba quyết định thiết kế quan trọng

| Quyết định | Chọn cách nào | Vì sao |
|------|------|------|
| Cách ly context | Tạo `messages[]` mới hoàn toàn | Để quá trình trung gian của subagent không làm bẩn context của parent |
| Chỉ trả về kết luận | `extract_text(last_message)` | Parent không cần cả transcript, chỉ cần kết quả cuối |
| Cấm đệ quy | Subagent không có `task` tool | Tránh việc subagent lại tiếp tục spawn subagent khác |
| Không bỏ qua bảo mật | Tool call của subagent vẫn qua PreToolUse hook | Context isolation không có nghĩa là permission isolation |

Dispatch mechanism vẫn y nguyên: `task` cũng chỉ là một tool khác trong `TOOL_HANDLERS`. Subagent có `SUB_SYSTEM` prompt riêng, trong đó ghi rõ: **hãy tự xử lý task này, đừng tiếp tục ủy quyền xuống agent khác**.

---

## Thay đổi so với s05

| Thành phần | Trước (s05) | Sau (s06) |
|------|-----------|-----------|
| Số tool | 6 (bash, read, write, edit, glob, todo_write) | 7 (+task) |
| Hàm mới | — | `spawn_subagent` (messages[] độc lập + giới hạn an toàn 30 vòng) |
| Cách ly context | Mọi thứ nằm trong hội thoại chính | Subagent dùng `messages[]` mới hoàn toàn |
| Loop | Không đổi | Dispatch không đổi, subagent có loop riêng với `SUB_SYSTEM` và hook protection |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s06_subagent/code.py
```

Thử các prompt sau:

1. `Use a subtask to find what testing framework this project uses`  
   (subagent sẽ đi đọc file, parent chỉ nhận kết luận)
2. `Delegate: read all .py files in agents/ and summarize what each one does`
3. `Use a task to create s06_subagent/example/string_tools.py with a slugify(text: str) function, then verify it from the parent agent`

Điểm cần quan sát:
- Có xuất hiện `[Subagent spawned]` / `[Subagent done]` không?
- Tool call của subagent có được in dưới dạng `[sub] ...` không?
- Parent agent có tiếp tục làm việc chỉ dựa trên phần summary mà subagent trả về không?

---

## Tiếp theo

Giờ Agent đã biết cách chia task ra. Nhưng mỗi task lại cần một loại kiến thức khác nhau: sửa frontend component thì cần React convention, viết SQL thì cần schema database. Nếu nhồi hết đống knowledge đó vào system prompt thì context sẽ nổ rất nhanh.

s07 Skill Loading → chỉ inject skill khi cần, thay vì nhồi tài liệu vào system prompt ngay từ đầu. Lúc cần mới load, tự nhiên như đọc file.

<details>
<summary>Đi sâu vào CC source code</summary>

> Nội dung dưới đây dựa trên phân tích CC source code `AgentTool.tsx`, `runAgent.ts`, `forkSubagent.ts`, `forkedAgent.ts`.

### 1. Không chỉ có một kiểu subagent, mà có ba kiểu

Bản giảng dạy chỉ nói về mô hình “fresh `messages[]`”. Nhưng CC thực tế có ba chế độ thực thi:

| Chế độ | Kích hoạt khi nào | Context |
|------|---------|--------|
| **Normal Subagent** | Có chỉ định `subagent_type` | `messages[]` mới hoàn toàn, chỉ có prompt |
| **Fork Subagent** | Không chỉ định `subagent_type`, và fork gate mở | Dùng `buildForkedMessages()` để tạo prefix thân thiện với prompt cache |
| **General-Purpose** | Không chỉ định `subagent_type`, nhưng fork gate đóng | Giống Normal |

### 2. Fork mode tồn tại để chia sẻ Prompt Cache

Đây là điểm mà bản giảng dạy bỏ qua. Ở fork mode (`forkSubagent.ts:60-71`), hệ thống **không** tạo context hoàn toàn mới, mà dùng `buildForkedMessages()` (`forkSubagent.ts:107-168`) để tạo một message prefix có thể hit prompt cache.

Mục tiêu không phải là isolation, mà là tối ưu API cost: để parent và child agent có cùng system prompt, tool list và message prefix, giúp phía Anthropic API không phải tính lại từ đầu.

Năm yếu tố quan trọng để cache hit (`forkedAgent.ts:57-68`) là: system prompt, tools, model, message prefix, và thinking config — tất cả phải giống nhau ở mức byte.

### 3. Context isolation thật sự không tuyệt đối

`createSubagentContext()` (`forkedAgent.ts:345-462`) tạo `ToolUseContext` cho subagent như sau:

| Trường | Hành vi |
|------|------|
| `abortController` | Tạo child controller mới, abort từ cha truyền xuống con |
| `setAppState` | Mặc định là no-op; nhưng sync agent có thể share qua `shareSetAppState` (`runAgent.ts:697-714`) |
| `readFileState` | **Clone từ parent** để tránh đọc lại cùng một file |
| `queryTracking` | chainId mới, `depth = parentDepth + 1` |

Tức là subagent không hoàn toàn “cô lập tuyệt đối”: read-file state vẫn có thể được chia sẻ. Mức độ tách biệt của UI và notification còn phụ thuộc vào execution path (sync / async / fork / teammate khác nhau).

### 4. Recursive fork protection trong CC tinh vi hơn

Bản giảng dạy đơn giản hóa thành “subagent không có `task` tool”. CC làm chi tiết hơn:

- `isInForkChild()` (`forkSubagent.ts:78-89`) kiểm tra trong conversation history có `FORK_BOILERPLATE_TAG` không; nếu có thì từ chối tiếp
- `constants/tools.ts:36-46` mặc định cấm `Agent` tool trong phần lớn agent
- `forkSubagent.ts:73-89` có protection riêng cho fork child
- `agentToolUtils.ts:100-110` lại có ngoại lệ đặc biệt cho teammate

Nói cách khác: đây không chỉ là chuyện “không cho spawn subagent mới”, mà là cả một policy phân tầng.

### 5. Permission bubbling

Fork agent dùng `permissionMode: 'bubble'` (`forkSubagent.ts:67`), nghĩa là mọi permission prompt của subagent sẽ **nổi lên terminal của parent agent** để user duyệt tại đó.

### 6. Async và sync là hai đường khác nhau

Bản giảng dạy chỉ mô tả subagent chạy đồng bộ: parent chờ child làm xong. CC còn có async path (`AgentTool.tsx:686-764`): khi `run_in_background: true`, subagent được khởi chạy bất đồng bộ, trả về ngay `{ status: 'async_launched' }` cho parent. Đến khi child làm xong thì kết quả được báo lại qua cơ chế notification.

Ngoài `run_in_background`, trên thực tế còn có thêm các nhánh như auto-background, assistant force async, coordinator/proactive, v.v.

### Vì sao bản giảng dạy cố ý đơn giản hóa

- 3 chế độ → 1 chế độ (`fresh messages[]`): để người học nắm được khái niệm cốt lõi trước
- Prompt cache sharing → bỏ qua: vì đây là optimization ở tầng API
- Recursive fork protection → đơn giản hóa thành “subagent không có task tool”
- Async path → để dành cho s13: s06 chỉ cần hiểu mô hình sync trước

</details>

<!-- translation-sync: zh@v1, en@v0, ja@v0, vi@v1 -->
