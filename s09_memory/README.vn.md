# s09: Memory — Compress sẽ mất chi tiết, cần một tầng không mất

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → ... → s07 → s08 → `s09` → [s10](../s10_system_prompt/) → s11 → ... → s20
> *"Compress sẽ mất chi tiết, cần một tầng không mất"* — Kho file + chỉ mục + tải theo nhu cầu, xuyên suốt compress, xuyên suốt phiên.
>
> **Tầng Harness**: Bộ nhớ — Tích lũy kiến thức xuyên suốt compress, xuyên suốt phiên.

---

## Vấn đề

autoCompact của s08 sẽ viết mục tiêu hiện tại, công việc còn lại, ràng buộc người dùng vào tóm tắt, nhưng chi tiết sẽ mất: "dùng tab để thụt lề không dùng khoảng trắng" có thể được đơn giản hóa thành "người dùng có sở thích kiểu code". Và khi mở phiên mới, thậm chí tóm tắt cũng không còn nữa.

LLM không có trạng thái lâu dài, tất cả thông tin đều trong cửa sổ ngữ cảnh. Ngữ cảnh đầy phải compress, compress thì mất mát. Cần một tầng không tham gia compress, giữ lại xuyên suốt phiên.

---

## Giải pháp

![Memory Overview](images/memory-overview.svg)

Pipeline compress của s08 giữ nguyên, tập trung vào bộ nhớ. Lưu trữ chọn file system: thư mục `.memory/` dưới, mỗi bộ nhớ một file `.md`, có YAML frontmatter（`name` / `description` / `type`）. File nhiều cần chỉ mục: `MEMORY.md` một dòng một liên kết, nhồi vào SYSTEM.

Thiết kế chính: chỉ mục luôn ở SYSTEM prompt（có thể được prompt cache lưu trữ），nội dung file theo nhu cầu nhồi vào user turn hiện tại（khớp theo filename/description với hội thoại hiện tại, không làm hỏng cache）. Ghi vào hoàn thành bởi bộ trích xuất sau mỗi vòng: khi người dùng nói rõ "nhớ lại" hoặc thể hiện sở thích bền vững, bộ trích xuất sẽ lưu như bộ nhớ. File tích lũy nhiều, định kỳ tổng chỉnh loại bỏ trùng.

Bốn loại bộ nhớ, mỗi loại có mục đích:

| Loại | Trả lời cái gì | Ví dụ |
|------|---------|------|
| user | Bạn là ai | "dùng tab không dùng khoảng trắng" |
| feedback | Cách làm việc | "đừng mock database" |
| project | Đang xảy ra cái gì | "rewrite auth là vì quy định" |
| reference | Đồ vật ở đâu | "bug pipeline ở Linear INGEST" |

---

## Cách hoạt động

![Memory Subsystems](images/memory-subsystems.svg)

### Lưu trữ: File Markdown + Chỉ mục

Mỗi bộ nhớ là một file `.md`, frontmatter YAML ghi metadata:

```markdown
---
name: user-preference-tabs
description: User prefers tabs for indentation
type: user
---

User prefers using tabs, not spaces, for indentation.
**Why:** Consistency with existing codebase conventions.
**How to apply:** Always use tabs when writing or editing files.
```

`MEMORY.md` là chỉ mục, một dòng một liên kết:

```markdown
- [user-preference-tabs](user-preference-tabs.md) — User prefers tabs for indentation
```

Khi ghi bộ nhớ mới tự động xây dựng lại chỉ mục:

```python
def write_memory_file(name, mem_type, description, body):
    slug = name.lower().replace(" ", "-")
    filepath = MEMORY_DIR / f"{slug}.md"
    filepath.write_text(
        f"---\nname: {name}\ndescription: {description}\ntype: {mem_type}\n---\n\n{body}\n"
    )
    _rebuild_index()
```

### Tải: Hai đường dẫn

**Đường dẫn một: Chỉ mục luôn ở SYSTEM.** `build_system()` đầu mỗi yêu cầu người dùng đọc `MEMORY.md`, nhồi danh sách bộ nhớ vào. Trích xuất bộ nhớ và tổng chỉnh chỉ kích hoạt cuối vòng hiện tại, do đó trong cùng một vòng yêu cầu người dùng không cần xây dựng lại SYSTEM lặp lại.

**Đường dẫn hai: Bộ nhớ liên quan theo nhu cầu nhồi vào.** Đầu mỗi yêu cầu người dùng, `load_memories()` gửi hội thoại gần đây và thư mục bộ nhớ（name + description）cùng với nhau cho LLM làm side-query nhẹ, chọn ra tên file liên quan, rồi đọc nội dung file tạm nhồi vào user turn hiện tại. Tối đa 5 cái, kiểm soát chi phí.

```python
def select_relevant_memories(messages, max_items=5):
    files = list_memory_files()
    if not files:
        return []

    # Build catalog: "0: user-preference-tabs — User prefers tabs..."
    catalog = "\n".join(f"{i}: {f['name']} — {f['description']}" for i, f in enumerate(files))

    response = client.messages.create(model=MODEL, messages=[{"role": "user",
        "content": f"Select relevant memory indices. Return JSON array.\n\n"
                   f"Recent conversation:\n{recent}\n\nMemory catalog:\n{catalog}"}],
        max_tokens=200)
    text = extract_text(response.content).strip()
    indices = json.loads(re.search(r'\[.*?\]', text).group())
    return [files[i]["filename"] for i in indices if 0 <= i < len(files)]
```

Nếu side-query thất bại（lỗi API, thất bại phân tích JSON），hạ cấp xuống khớp từ khóa name + description.

### Ghi: Trích xuất sau mỗi vòng

Người dùng sẽ không mỗi lần đều nói "nhớ lại". Sở thích thường phân tán trong hội thoại bình thường: "dùng tab tốt hơn khoảng trắng", "sau này dùng dấu ngoặc đơn hết".

`extract_memories()` chạy cuối mỗi vòng, điều kiện là mô hình dừng lại và không có tool_use（chỉ ra hội thoại kết thúc một giai đoạn）:

```python
# In agent_loop:
if response.stop_reason != "tool_use":
    extract_memories(pre_compress)   # từ snapshot trước compress trích xuất bộ nhớ mới
    consolidate_memories()       # kiểm tra có cần tổng chỉnh không
    return
```

Trích xuất trước kiểm tra bộ nhớ có sẵn, tránh trùng. Prompt trích xuất yêu cầu LLM trả về JSON array `{name, type, description, body}`, chỉ ghi file khi thực sự có thông tin mới.

```python
def extract_memories(messages):
    dialogue = format_recent_messages(messages[-10:])
    existing = "\n".join(f"- {m['name']}: {m['description']}" for m in list_memory_files())

    prompt = (
        "Extract user preferences, constraints, or project facts.\n"
        "Return JSON array: [{name, type, description, body}].\n"
        "If nothing new or already covered, return [].\n\n"
        f"Existing memories:\n{existing}\n\nDialogue:\n{dialogue[:4000]}"
    )
    # ... parse response, write files ...
```

### Tổng chỉnh: Hợp nhất loại bỏ trùng tần suất thấp

File bộ nhớ sẽ tích lũy. `consolidate_memories()` khi số file đạt ngưỡng（mặc định 10）kích hoạt, để LLM loại bỏ trùng, hợp nhất mâu thuẫn, loại bỏ bộ nhớ cũ:

```python
CONSOLIDATE_THRESHOLD = 10

def consolidate_memories():
    files = list_memory_files()
    if len(files) < CONSOLIDATE_THRESHOLD:
        return  # quá ít, không đáng tổng chỉnh
    # Send all memories to LLM, get back deduplicated list
    # Replace all files with consolidated results
```

CC gọi quá trình này Dream, thực tế có bốn tầng cổng: khoảng thời gian, throttle quét, số phiên, khóa file. Phiên giảng dạy đơn giản hóa thành ngưỡng số file.

### Bộ nhớ thích lưu cái gì

Bộ nhớ lưu thông tin xuyên suốt phiên vẫn hữu ích: sở thích người dùng, phản hồi lặp lại, nền tảng dự án, lối vào thường dùng và dòng theo dõi sự cố. Nó tập trung "sau này sẽ dùng cái gì", và thông qua chỉ mục + tải theo nhu cầu mang những thông tin này trở lại hội thoại hiện tại.

session memory tập trung tính liên tục trong cùng phiên: sau compact, phiên hiện tại vẫn cần giữ lại ngữ cảnh nào. Hai cái phối hợp dùng: Memory quản lý kiến thức dài hạn, session memory quản lý compress tiếp nối phiên hiện tại.

---

## Thay đổi so với s08

| Thành phần | Trước đó (s08) | Sau đó (s09) |
|-----------|--------------|------------|
| Khả năng nhớ | Không（sở thích thoái hóa theo tóm tắt sau compress） | Lưu + tải + trích xuất + tổng chỉnh |
| Hàm mới | — | write_memory_file, select_relevant_memories, load_memories, extract_memories, consolidate_memories |
| Lưu trữ | — | .memory/MEMORY.md chỉ mục + .memory/*.md file |
| Công cụ | bash, read, write, edit, glob, todo_write, task, load_skill, compact (9) | bash, read_file, write_file, edit_file, glob, task (6) |
| Vòng lặp | Mỗi vòng chỉ compress | Mỗi vòng nhồi bộ nhớ + compress + cuối vòng trích xuất + định kỳ tổng chỉnh |

---

## Hãy thử

```sh
cd learn-claude-code
python s09_memory/code.py
```

Thử những prompt này（ghi từng phần, quan sát tích lũy và tải bộ nhớ）:

1. `I prefer using tabs for indentation, not spaces. Remember that.`
2. `Create a Python file called test.py`（quan sát Agent có dùng tab không）
3. `What did I tell you about my preferences?`（quan sát Agent có nhớ không）
4. `I also prefer single quotes over double quotes for strings.`

Điểm quan sát: cuối mỗi vòng có xuất hiện `[Memory: extracted N new memories]` không? Thư mục `.memory/` dưới có tạo file `.md` không? Chỉ mục `MEMORY.md` có cập nhật không? Vòng hội thoại mới Agent có tự động tải bộ nhớ trước đây không?

---

## Tiếp theo

Bộ nhớ, compress, công cụ đã sẵn. Nhưng system prompt vẫn là một đoạn chuỗi lớn được mã hóa cứng. Thêm công cụ mới phải thêm mô tả tay, đổi dự án phải viết lại toàn bộ prompt. Prompt nên được lắp ráp lúc chạy.

s10 System Prompt → Chia thành đoạn + lắp ráp lúc chạy. Dự án khác, công cụ khác, lắp ra prompt khác.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Dưới đây dựa trên phân tích CC source `src/` dưới `memdir/`, `services/`, `utils/`, `query/`, số dòng đã đối chiếu chính xác.

### Đường dẫn source

| File | Dòng | Trách nhiệm |
|------|------|-----------|
| `memdir/memdir.ts` | 507 | Lõi: định nghĩa MEMORY.md（`34-38`）, chỉ thị hành vi bộ nhớ phân biệt memory/plan/tasks（`199-266`）, `loadMemoryPrompt()` ba đường dẫn（`419-490`） |
| `memdir/findRelevantMemories.ts` | 141 | Side-query Sonnet chọn bộ nhớ（`18-24` system prompt, `97-122` logic gọi） |
| `memdir/memoryTypes.ts` | 271 | Định nghĩa loại, trường frontmatter |
| `memdir/memoryScan.ts` | — | Quét file .md, loại trừ MEMORY.md, đọc frontmatter, tối đa 200 cái, theo mtime giảm dần（`35-94`） |
| `services/extractMemories/extractMemories.ts` | 615 | Forked agent trích xuất bộ nhớ, quyền hạn chế, `skipTranscript: true`, `maxTurns: 5`（`371-427`） |
| `services/autoDream/autoDream.ts` | 324 | Tổng chỉnh Dream, bốn tầng cổng（`63-66` giá trị mặc định, `130-190` cổng, `224-233` forked agent） |
| `services/SessionMemory/sessionMemory.ts` | 495 | Quản lý bộ nhớ phiên |
| `services/compact/sessionMemoryCompact.ts` | — | Tóm tắt nhẹ bộ nhớ phiên, ngưỡng 10K/5/40K（`56-61`） |
| `utils/attachments.ts` | — | Ngân sách nhồi: 200 dòng / 4096 byte mỗi file, 60KB mỗi session（`269-288`）; tìm bộ nhớ liên quan theo query（`2196-2241`） |
| `query.ts` | — | Prefetch bộ nhớ mỗi vòng khởi động（`301-304`）, thu thập phi chặn（`1592-1614`） |
| `query/stopHooks.ts` | — | Stop hook kích hoạt fire-and-forget trích xuất và Dream（`141-155`） |

### Chọn bộ nhớ: LLM chọn, không phải embedding

CC dùng **chính Sonnet để chọn**（`findRelevantMemories.ts`），không phải vector similarity embedding:

1. `memoryScan.ts` quét tất cả file `.md` dưới `.memory/`（loại trừ MEMORY.md），tối đa 200 cái, theo mtime giảm dần
2. Liệt kê `name` + `description` thành danh mục
3. Gửi cho Sonnet side-query: "theo tên và mô tả chọn bộ nhớ thực sự hữu ích（tối đa 5 cái）. Không chắc chắn thì không chọn."
4. Sonnet trả về `{ selected_memories: ["file1.md", ...] }`
5. File chọn đọc nội dung đầy đủ（mỗi file ≤ 200 dòng / 4096 byte），nhồi vào ngữ cảnh. Ngân sách tổng mỗi session 60KB

Mỗi vòng user turn khởi động, `query.ts:301-304` khởi động memory prefetch（bất đồng bộ）; sau thực thi công cụ `1592-1614` thu thập phi chặn, không cản main flow.

### Thời điểm trích xuất: stop hook, không phải sau autoCompact

Vị trí kích hoạt（`stopHooks.ts:141-155`）: trong `handleStopHooks()`, kích hoạt fire-and-forget trích xuất và Dream. Phiên giảng dạy đặt trích xuất trong nhánh `stop_reason != "tool_use"`, hướng tương tự.

Trích xuất của CC qua forked agent thực thi（`extractMemories.ts:371-427`）: quyền hạn chế, `skipTranscript: true`, `maxTurns: 5`. Còn bảo vệ chồng lặp: nếu Agent chính đã ghi file bộ nhớ, bỏ qua trích xuất.

### Định dạng file bộ nhớ

CC dùng Markdown + YAML frontmatter, giống phiên giảng dạy. Bốn loại: `user`, `feedback`, `project`, `reference`.

`memdir.ts:34-38` định nghĩa ràng buộc chỉ mục: `MEMORY.md` tối đa 200 dòng / 25KB. `memdir.ts:199-266` xây dựng hướng dẫn hành vi bộ nhớ, phân biệt rõ memory, plan, tasks. Vị trí lưu: `~/.claude/projects/<sanitized-git-root>/memory/`.

### Dream: Bốn tầng cổng

Không phải "khi rảnh kích hoạt" hay "đủ số lượng thì hợp nhất", mà là bốn tầng cổng（`autoDream.ts`，giá trị mặc định `63-66`，logic cổng `130-190`）:

1. **Cổng thời gian**: từ lần hợp nhất trước ≥ 24 giờ
2. **Throttle quét**: tránh quét file system thường xuyên
3. **Cổng phiên**: từ lần hợp nhất trước sửa đổi ≥ 5 phiên transcript
4. **Cổng khóa**: không có tiến trình khác đang hợp nhất（file `.consolidate-lock`）

Hợp nhất thực tế qua forked agent thực thi（`224-233`）: định vị → thu thập tín hiệu gần đây → ghi file hợp nhất → cắt cúc cập nhật chỉ mục. Mtime file khóa là lastConsolidatedAt. Phục hồi sự cố: khóa tự động hết hạn sau 1 giờ.

### User Memory vs Session Memory

| | User Memory | Session Memory |
|---|---|---|
| Tính lâu dài | Xuyên suốt phiên | Một phiên |
| Lưu trữ | Nhiều file .md dưới `memory/` | `session-memory/<id>/memory.md` |
| Tải vào | system prompt | tóm tắt compact |
| Mục đích | Tích lũy kiến thức xuyên suốt phiên | Tính liên tục ngữ cảnh xuyên suốt compact |

sessionMemoryCompact（cơ chế được nêu trong s08）đúng là sử dụng Session Memory: trước autoCompact đọc file session memory, nếu nội dung đủ（≥ 10K token, ≥ 5 dòng text, ≤ 40K token，`sessionMemoryCompact.ts:56-61`），dùng nó làm tóm tắt, không gọi LLM.

### Thực tế phức tạp hơn phiên giảng dạy chỗ nào

- **Feature flags**: chức năng bộ nhớ liên quan có nhiều tầng feature gate kiểm soát
- **Team memory**: bộ nhớ chia sẻ đội, `loadMemoryPrompt()` có đường dẫn đặc biệt（phiên giảng dạy không liên quan）
- **KAIROS**: chiến lược trích xuất bộ nhớ nhận biết thời gian, trong `loadMemoryPrompt()` chế độ daily-log
- **Prompt cache**: nhồi bộ nhớ cần xem xét TTL của prompt cache, tránh mỗi lần rewrite phần lớn system prompt
- **Khóa file**: cơ chế khóa khi nhiều tiến trình đồng thời
- **Memory prefetch**: tải trước bất đồng bộ, không cản main flow

### Đơn giản hóa phiên giảng dạy là cố ý

- LLM side-query → LLM side-query + hạ cấp từ khóa: phiên giảng dạy giữ chọn LLM, thêm đường dẫn hạ cấp
- Bộ nhớ JSON → Markdown + frontmatter: phiên giảng dạy giống CC
- Stop hook kích hoạt → nhánh `stop_reason != "tool_use"`: hướng tương tự
- Bốn tầng cổng → ngưỡng số file: phiên giảng dạy không có hệ thống transcript và khái niệm nhiều phiên
- Forked agent + quyền hạn chế → gọi trực tiếp: phiên giảng dạy không có cách ly tiến trình con

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1, vi@v1 -->
