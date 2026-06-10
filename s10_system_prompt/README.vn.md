# s10: System Prompt — Lắp ráp lúc chạy, không mã hóa cứng

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s08 → s09 → `s10` → [s11](../s11_error_recovery/) → s12 → ... → s20
> *"Prompt được lắp ráp, không được viết chết"* — Chia đoạn + ghép nối theo nhu cầu + cache.
>
> **Tầng Harness**: Prompt — Lắp ráp lúc chạy, không mã hóa cứng.

---

## Vấn đề

Từ s01 đến s09, system prompt đều là một dòng mã hóa cứng:

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use tools to solve tasks."
```

s01 đủ dùng, chỉ có bash, read, write ba công cụ. Nhưng đến s09, Agent đã có bộ nhớ, có compress, có tải kỹ năng. Prompt nên nhắc đến khả năng ngày càng nhiều:

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Use tools to solve tasks. Act, don't explain. "
    "Before starting any multi-step task, use todo_write. "
    "Skills are available via list_skills and load_skill. "
    "Relevant memories are injected below when available. "
    # ... thêm một khả năng là thêm một đoạn
)
```

Ba vấn đề:

1. **Đổi dự án phải viết lại toàn bộ prompt**, không biết cái nào nên đổi, cái nào nên giữ
2. **Sửa một chỗ có thể ảnh hưởng toàn bộ**, thêm một đoạn mô tả công cụ có thể xung đột với chỉ thị trước đó
3. **Mỗi lần request đều mang toàn bộ nội dung**, dù hội thoại hiện tại không dùng một số đoạn cũng lãng phí token

System prompt nên là cấu hình được lắp ráp lúc chạy dựa theo trạng thái hiện tại: công cụ nào bật, ngữ cảnh nào nhìn thấy, bộ nhớ nào liên quan, nội dung nào phải giữ ổn định để trúng prompt cache.

---

## Giải pháp

![System Prompt Overview](images/system-prompt-overview.svg)

s10 tập trung vào cơ chế lắp ráp prompt. Lấy khả năng s08-s09 làm nền, nhưng không triển khai lại hệ thống compress và bộ nhớ. Thay đổi cốt lõi: tách `SYSTEM` mã hóa cứng thành các đoạn độc lập（section），lúc chạy ghép nối theo nhu cầu dựa theo trạng thái thực, cache kết quả tránh lắp ráp lặp lại.

Bốn section, hai chiến lược tải:

| Section | Chiến lược tải | Nội dung | Căn cứ quyết định |
|---------|-------------|---------|----------------|
| identity | Luôn luôn | Bạn là ai, cách làm việc | Luôn tồn tại |
| tools | Luôn luôn | Danh sách công cụ khả dụng | `enabled_tools` |
| workspace | Luôn luôn | Thư mục làm việc | Luôn tồn tại |
| memory | Theo nhu cầu | Nội dung bộ nhớ liên quan | `.memory/MEMORY.md` có tồn tại không |

Thiết kế chính: có tải section hay không phụ thuộc vào trạng thái thực（công cụ có tồn tại không, file có tồn tại không），không phải từ khóa trong tin nhắn.

---

## Cách hoạt động

### PROMPT_SECTIONS: Định nghĩa chia đoạn

Tách một đoạn chuỗi lớn thành từ điển, mỗi key là một chủ đề:

```python
PROMPT_SECTIONS = {
    "identity": "You are a coding agent. Act, don't explain.",
    "tools": "Available tools: bash, read_file, write_file.",
    "workspace": f"Working directory: {WORKDIR}",
    "memory": "Relevant memories are injected below when available.",
}
```

Mỗi section duy trì độc lập. Sửa `tools` không ảnh hưởng `identity`, thêm `memory` không động `workspace`.

### assemble_system_prompt: Ghép nối theo nhu cầu

Không phải tất cả section mỗi lần đều cần. Hiện tại không có file bộ nhớ, tải section memory chỉ lãng phí token. Dựa theo trạng thái thực của context quyết định tải cái nào:

```python
def assemble_system_prompt(context: dict) -> str:
    sections = []

    # Luôn tải
    sections.append(PROMPT_SECTIONS["identity"])
    sections.append(PROMPT_SECTIONS["tools"])
    sections.append(PROMPT_SECTIONS["workspace"])

    # Tải theo nhu cầu — dựa trên trạng thái thực, không phải từ khóa
    memories = context.get("memories", "")
    if memories:
        sections.append(f"Relevant memories:\n{memories}")

    return "\n\n".join(sections)
```

"Luôn tải" là mỗi vòng đều cần: danh tính, công cụ, thư mục làm việc. "Tải theo nhu cầu" chỉ trong điều kiện cụ thể mới hữu ích.

Tại sao không tải hết? Token có chi phí（system prompt mỗi vòng tính tiền），thông tin càng ít LLM càng tập trung（chỉ thị không liên quan là nhiễu）.

### get_system_prompt: Cache tránh ghép nối lặp lại

Khi ngữ cảnh không thay đổi（nhiều lần gọi LLM cùng vòng hội thoại, context giống nhau），ghép nối lại là lãng phí. Dùng serialization chắc chắn phát hiện thay đổi, trúng cache trả về trực tiếp:

```python
def get_system_prompt(context: dict) -> str:
    global _last_context_key, _last_prompt
    key = json.dumps(context, sort_keys=True, ensure_ascii=False, default=str)
    if key == _last_context_key and _last_prompt:
        return _last_prompt
    _last_context_key = key
    _last_prompt = assemble_system_prompt(context)
    return _last_prompt
```

Dùng `json.dumps` không phải `hash()`: `hash()` nội bộ Python có random hóa tiến trình, không phù hợp làm cache key ổn định, và khi gặp list/dict sẽ báo `unhashable type`.

Chú ý: cache ở đây chỉ "tránh ghép nối chuỗi lặp lại", và prompt cache của API CC không phải một chuyện. Prompt cache của CC qua `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` phân tách phần tĩnh và động, phần tĩnh trúng global cache, không vô hiệu hóa vì nội dung động thay đổi.

### context: Trạng thái thực, không phải đoán từ khóa

context phản ánh trạng thái thực lúc chạy hiện tại:

```python
def update_context(context: dict, messages: list) -> dict:
    memories = ""
    if MEMORY_INDEX.exists():
        content = MEMORY_INDEX.read_text().strip()
        if content:
            memories = content
    return {
        "enabled_tools": list(TOOL_HANDLERS.keys()),
        "workspace": str(WORKDIR),
        "memories": memories,
    }
```

`enabled_tools` liệt kê công cụ thực tế đăng ký. `memories` kiểm tra `.memory/MEMORY.md` có tồn tại không. Tải section dựa trên trạng thái thực này, không tìm từ khóa trong tin nhắn.

### Chạy cùng nhau

```python
def agent_loop(messages: list, context: dict):
    system = get_system_prompt(context)
    while True:
        response = client.messages.create(
            model=MODEL, system=system, messages=messages,
            tools=TOOLS, max_tokens=8000)
        # ... thực thi công cụ ...
        context = update_context(context, messages)
        system = get_system_prompt(context)
```

Mỗi vòng lặp đầu lấy một lần system prompt. Context thay đổi thì lắp ráp lại, không đổi thì trả về cache.

---

## Thay đổi so với s09

| Thành phần | Trước đó (s09) | Sau đó (s10) |
|-----------|--------------|------------|
| prompt | Chuỗi SYSTEM mã hóa cứng | PROMPT_SECTIONS + assemble_system_prompt |
| Cache | Không | get_system_prompt（phát hiện json.dumps + cache） |
| Hàm mới | — | assemble_system_prompt, get_system_prompt, update_context |
| Công cụ | bash, read_file, write_file (3) | bash, read_file, write_file (3) — không thay đổi |
| Vòng lặp | Dùng SYSTEM cố định | Dùng get_system_prompt(context) |

---

## Hãy thử

```sh
cd learn-claude-code
python s10_system_prompt/code.py
```

Điểm quan sát:

1. Output thấy section nào được tải（tag `[assembled] sections: ...`）
2. Khi hội thoại liên tiếp, trúng cache hiển thị `[cache hit]`
3. Sau tạo file `.memory/MEMORY.md`, vòng sau section memory tự động tải

Thử những prompt này:

1. `Read the file README.md`（quan sát ba section luôn tải）
2. `Create a file called .memory/MEMORY.md with content "- [test](test.md) — test memory"`（ghi chỉ mục bộ nhớ）
3. `Read the file code.py`（quan sát section memory có xuất hiện không）

---

## Tiếp theo

System prompt có thể lắp ráp lúc chạy rồi, nhưng Agent gặp lỗi vẫn sẽ sập. Rung mạng, giới hạn API, output bị cắt, ngữ cảnh vượt hạn, những cái này không phải bug, là trạng thái thường.

s11 Error Recovery → Bốn đường dẫn phục hồi. Nâng cấp token, compress ngữ cảnh, lùi theo số mũ, chuyển mô hình.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Dưới đây dựa trên phân tích CC source `constants/prompts.ts`（914 dòng）, `constants/systemPromptSections.ts`（68 dòng）, `context.ts`（189 dòng）, `utils/api.ts`（718 dòng）, `utils/systemPrompt.ts`（123 dòng）, `bootstrap/state.ts`.

### System prompt của CC có bao nhiêu section?

Số lượng không cố định, chịu ảnh hưởng feature flag, output style, chế độ KAIROS/Proactive, loại người dùng, ngân sách token v.v. Đại khái phân hai loại:

**Section tĩnh**（luôn tải）: identity, system, doing_tasks, actions, using_tools, tone_style, output_efficiency v.v.

**Section động**（tải theo trạng thái）: session_guidance, memory, ant_model_override, env_info_simple, language, output_style, mcp_instructions, scratchpad, frc, summarize_tool_results, numeric_length_anchors, token_budget, brief v.v.

`mcp_instructions` là section dễ bay hơi duy nhất（qua `DANGEROUS_uncachedSystemPromptSection()` tạo），vì MCP server có thể kết nối và ngắt kết nối giữa các vòng.

### Hàm lắp ráp

```typescript
getSystemPrompt(tools, model, additionalWorkingDirs?, mcpClients?): Promise<string[]>
```

Trả về `string[]`（mỗi phần tử là một section），bởi `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` phân tách phần tĩnh và động.

### cache scope

Khi bật global cache boundary, section tĩnh hợp nhất thành một global cache block, section động không dùng global cache（`cacheScope: null`）. Không có boundary hoặc bỏ qua global cache đường dẫn mới đi org scope.

Cache phiên giảng dạy chỉ tránh ghép nối chuỗi lặp lại. Ba tầng cache của CC:

1. **lodash memoize**: `getSystemContext` và `getUserContext` cache trong phiên（`context.ts`）
2. **cache đăng ký section**: `STATE.systemPromptSectionCache` cache kết quả section động, xóa khi `/clear` hoặc `/compact`
3. **Cache cấp API**: `splitSysPromptPrefix()`（`api.ts`）chia prompt theo boundary thành block cache scope khác nhau

### getUserContext vs getSystemContext

| | getSystemContext | getUserContext |
|---|---|---|
| Nội dung | gitStatus, cacheBreaker | Nội dung CLAUDE.md, currentDate |
| Cách nhồi | Thêm vào mảng system prompt | Đặt trước là tin nhắn người dùng `<system-reminder>` |
| Khi nào bỏ qua | Khi system prompt tùy chỉnh | Luôn chạy |

### Chế độ thay đổi prompt như thế nào

- **CLAUDE_CODE_SIMPLE**: Toàn bộ prompt chỉ 2 dòng
- **Proactive/KAIROS**: Dùng phiên bản compact thay thế tất cả section chuẩn
- **Coordinator**: Dùng prompt chuyên cho coordinator thay thế hoàn toàn
- **Chế độ Agent**: Prompt định nghĩa Agent thay thế hoặc thêm vào prompt mặc định

### Tổng kích thước

Dưới chế độ tương tác chuẩn system prompt cốt lõi khoảng 20-30KB text. CLAUDE_CODE_SIMPLE khoảng 150 ký tự. User context（CLAUDE.md）và system context（git status）tích lũy trên cơ sở này.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
