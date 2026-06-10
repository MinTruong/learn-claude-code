# s08: Context Compact — Ngữ cảnh luôn đầy, phải có cách tạo chỗ trống

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → s03 → s04 → s05 → s06 → s07 → `s08` → [s09](../s09_memory/) → s10 → ... → s20
> *"Ngữ cảnh luôn đầy, phải có cách tạo chỗ trống"* — Bốn tầng compress, cái rẻ chạy trước cái đắt chạy sau.
>
> **Tầng Harness**: Compress — Bộ nhớ sạch, phiên làm việc vô hạn.

---

## Vấn đề

Agent chạy chạy mà dừng lại.

Tay có bash, có read, có write, khả năng đủ rồi. Nhưng nó đọc một file 1000 dòng（~4000 token），rồi đọc 30 file, chạy 20 lệnh. Mỗi lệnh output, mỗi file content, tất cả xếp chồng trong danh sách `messages`. 

Cửa sổ ngữ cảnh có giới hạn. Đầy rồi, API từ chối thẳng: `prompt_too_long`.

Không compress, Agent hoàn toàn không thể làm việc trong dự án lớn.

---

## Giải pháp

![Compact Overview](images/compact-overview.svg)

Giữ nguyên cấu trúc hook từ s07, tải kỹ năng, sub Agent vv, bỏ qua một số chi tiết công cụ để tập trung vào compress. Thay đổi cốt lõi: trước mỗi lệnh gọi LLM chèn ba tầng tiền xử lý（0 API），nếu token vẫn vượt ngưỡng kích hoạt tóm tắt LLM（1 API），khi API báo lỗi thì compress khẩn cấp.

Thiết kế cốt lõi: cái rẻ chạy trước, cái đắt chạy sau.

---

## Nguyên lý hoạt động

![Bốn tầng compress pipeline](images/compaction-layers.svg)

### L1: snip_compact — Cắt bỏ hội thoại cũ không liên quan

Agent chạy 80 vòng hội thoại, `messages` tích lũy 160 dòng. Ở phía trước "giúp tôi tạo hello.py" và công việc hiện tại gần như không liên quan, nhưng toàn chiếm chỗ.

Số tin nhắn vượt 50 → giữ lại đầu 3 dòng（ngữ cảnh ban đầu）và cuối 47 dòng（công việc hiện tại），cắt bỏ ở giữa; điều kiện biên duy nhất thêm là, không thể cắt rời `assistant(tool_use)` và `user(tool_result)` phía sau:

```python
def snip_compact(messages, max_messages=50):
    if len(messages) <= max_messages:
        return messages
    head_end, tail_start = 3, len(messages) - (max_messages - 3)
    if _message_has_tool_use(messages[head_end - 1]):
        while head_end < len(messages) and _is_tool_result_message(messages[head_end]):
            head_end += 1
    if _is_tool_result_message(messages[tail_start]) and _message_has_tool_use(messages[tail_start - 1]):
        tail_start -= 1
    snipped = tail_start - head_end
    placeholder = {"role": "user", "content": f"[snipped {snipped} messages from conversation middle]"}
    return messages[:head_end] + [placeholder] + messages[tail_start:]
```

Cắt bỏ là chính nó tin nhắn, chỉ là thêm một bước bảo vệ ở điểm cắt; những tin nhắn còn lại trong `tool_result` content vẫn tích lũy——tin nhắn thứ 34 có thể nằm 30KB nội dung file cũ. → L2.

### L2: micro_compact — Kết quả công cụ cũ chiếm chỗ

![Kết quả cũ chiếm chỗ](images/micro-compact.svg)

Agent liên tiếp đọc 10 file. Nội dung đầy đủ lần 1-7 vẫn nằm trong ngữ cảnh, từ lâu không cần nữa, nhưng chiếm nhiều chỗ.

Chỉ giữ nội dung đầy đủ 3 `tool_result` gần nhất, cái cũ hơn thay bằng một dòng chiếm chỗ:

```python
KEEP_RECENT_TOOL_RESULTS = 3

def micro_compact(messages):
    tool_results = collect_tool_result_blocks(messages)
    if len(tool_results) <= KEEP_RECENT_TOOL_RESULTS:
        return messages
    for _, _, block in tool_results[:-KEEP_RECENT_TOOL_RESULTS]:
        if len(block.get("content", "")) > 120:
            block["content"] = "[Earlier tool result compacted. Re-run if needed.]"
    return messages
```

Kết quả cũ xóa sạch, nhưng kết quả mới đơn lẻ có thể 500KB——một lần `cat` file lớn output đủ lấp đầy ngữ cảnh. → L3.

### L3: tool_result_budget — Kết quả lớn lưu đĩa

![Kết quả lớn lưu đĩa](images/layer1-budget.svg)

Mô hình một lần đọc 5 file lớn, tất cả `tool_result` trong một tin nhắn user 500KB.

Thống kê tổng kích thước tất cả `tool_result` trong tin nhắn user cuối cùng. Vượt 200KB → sắp xếp theo kích thước, từ cái lớn nhất bắt đầu lưu đĩa vào `.task_outputs/tool-results/`, ngữ cảnh chỉ giữ `<persisted-output>` tag + preview 2000 ký tự đầu. Mô hình thấy tag sau biết nội dung đầy đủ trên đĩa, khi cần có thể đọc lại.

```python
def tool_result_budget(messages, max_bytes=200_000):
    last = messages[-1]
    blocks = [(i, b) for i, b in enumerate(last["content"])
              if b.get("type") == "tool_result"]
    total = sum(len(str(b.get("content", ""))) for _, b in blocks)
    if total <= max_bytes:
        return messages
    ranked = sorted(blocks, key=lambda p: len(str(p[1].get("content", ""))), reverse=True)
    for idx, block in ranked:
        if total <= max_bytes:
            break
        block["content"] = persist_large_output(block["tool_use_id"], str(block["content"]))
        total = recalculate_total(blocks)
    return messages
```

Ba tầng trước đều là thao tác text/cấu trúc, 0 lệnh gọi API, nhưng cũng không thể "hiểu" nội dung hội thoại. Ngữ cảnh có thể vẫn quá lớn. → L4.

### L4: compact_history — Tóm tắt LLM đầy đủ

![Tóm tắt LLM đầy đủ](images/auto-compact.svg)

Ba tầng trước chạy xong, nhưng trong dự án cực lớn sau 30 phút làm việc liên tục, token vẫn vượt ngưỡng.

Ba bước quy trình:

1. **Lưu transcript**: Hội thoại đầy đủ ghi vào `.transcripts/`, định dạng JSONL. Transcript giữ lại bản ghi có thể phục hồi, nhưng ngữ cảnh sắc hoạt động của mô hình chỉ còn tóm tắt. Với suy luận hiện tại của mô hình, chi tiết đã không còn trong ngữ cảnh. Mã giảng dạy không cung cấp công cụ truy xuất transcript.
2. **LLM tạo tóm tắt**: Gửi lịch sử hội thoại cho LLM, yêu cầu giữ lại mục tiêu hiện tại, phát hiện quan trọng, file đã sửa, công việc còn lại, ràng buộc người dùng v.v. thông tin chính.
3. **Thay thế danh sách tin nhắn**: Tất cả tin nhắn cũ được thay thế bằng một tóm tắt. Phiên bản giảng dạy chỉ giữ tóm tắt; Claude Code thực tế sau compact sẽ tái gắn một số file gần đây, kế hoạch, agent/skill/tool v.v. ngữ cảnh.

```python
def compact_history(messages):
    transcript_path = write_transcript(messages)  # trước tiên lưu hội thoại đầy đủ
    summary = summarize_history(messages)          # LLM tạo tóm tắt
    return [{"role": "user",
             "content": f"[Compacted]\n\n{summary}"}]
```

**Cầu chì**: Sau 3 lần thất bại liên tiếp dừng thử lại, ngăn chặn vòng lặp chết lãng phí lệnh gọi API.

### Khẩn cấp: reactive_compact

Đôi khi API vẫn trả về `prompt_too_long`（413），khi tốc độ tăng ngữ cảnh nhanh hơn kích hoạt compress.

Lúc đó kích hoạt **reactive_compact**: thậm chí còn cảnh báo hơn compact_history, quay lui từ cuối, nhưng vẫn tránh để lại `tool_result` cô độc.

```python
def reactive_compact(messages):
    transcript = write_transcript(messages)
    summary = summarize_history(messages)
    tail_start = max(0, len(messages) - 5)
    if _is_tool_result_message(messages[tail_start]) and _message_has_tool_use(messages[tail_start - 1]):
        tail_start -= 1
    return [{"role": "user",
             "content": f"[Reactive compact]\n\n{summary}"}, *messages[tail_start:]]
```

reactive compact có giới hạn thử lại（mặc định 1 lần）. Thất bại nữa thì ném ngoại lệ, không lặp vô hạn. Logic phục hồi lỗi đầy đủ để lại cho s11.

### Chạy cùng nhau

```python
def agent_loop(messages):
    reactive_retries = 0
    while True:
        # Ba tiền xử lý（0 lệnh gọi API）
        # Thứ tự: budget chạy trước, đảm bảo nội dung lớn lưu đĩa trước rồi mới chiếm chỗ và cắt
        messages[:] = tool_result_budget(messages)    # L3: kết quả lớn lưu đĩa
        messages[:] = snip_compact(messages)          # L1: cắt giữa
        messages[:] = micro_compact(messages)         # L2: kết quả cũ chiếm chỗ

        # Vẫn không đủ? Tóm tắt LLM（1 lệnh gọi API）
        if estimate_token_count(messages) > THRESHOLD:
            messages[:] = compact_history(messages)

        try:
            response = client.messages.create(...)
        except PromptTooLongError:
            if reactive_retries < MAX_REACTIVE_RETRIES:
                messages[:] = reactive_compact(messages)  # khẩn cấp
                reactive_retries += 1
                continue
            raise  # vượt giới hạn thử lại, ném ngoại lệ
        # ... thực thi công cụ ...

        # công cụ compact: khi mô hình gọi chủ động kích hoạt compact_history
        if block.name == "compact":
            messages[:] = compact_history(messages)
            results.append({..., "content": "[Compacted. History summarized.]"})
            messages.append({"role": "user", "content": results})
            break  # kết thúc turn hiện tại, bắt đầu vòng mới với ngữ cảnh nén
```

**Không được đổi thứ tự.** L3（budget）trước L2（micro），vì micro sẽ thay thế `tool_result` cũ lớn bằng một dòng chiếm chỗ, budget phải làm trước để lưu nội dung đầy đủ lên đĩa. Đây là lý do tại sao CC source đặt `applyToolResultBudget` ở đầu tiên.

---

## Thay đổi so với s07

| Thành phần | Trước đó (s07) | Sau đó (s08) |
|-----------|--------------|------------|
| Quản lý ngữ cảnh | Không（ngữ cảnh膨胀vô hạn） | Bốn tầng compress pipeline + khẩn cấp |
| Hàm mới | — | snip_compact, micro_compact, tool_result_budget, compact_history, reactive_compact |
| Công cụ | bash, read, write, edit, glob, todo_write, task, load_skill (8) | 8 + compact (9) |
| Vòng lặp | Lệnh gọi LLM → thực thi công cụ | Ba tầng tiền xử lý trước mỗi vòng + kích hoạt ngưỡng compact_history |
| Nguyên tắc thiết kế | — | Cái rẻ chạy trước, cái đắt chạy sau |

---

## Hãy thử

```sh
cd learn-claude-code
python s08_context_compact/code.py
```

Thử những prompt này:

1. `Read the file README.md, then read code.py, then read s01_agent_loop/README.md`（đọc liên tiếp nhiều file, quan sát compress L2 kết quả cũ）
2. `Read every file in s08_context_compact/`（đọc một lần lượng nội dung lớn, quan sát L3 lưu đĩa）
3. Nói chuyện liên tiếp 20+ vòng, quan sát có xuất hiện `[auto compact]` hay `[reactive compact]` không

Điểm quan sát: sau mỗi lần thực thi công cụ, tool_result cũ có bị compress không? Sau hội thoại liên tiếp nếu token vượt ngưỡng, tóm tắt có được kích hoạt tự động không?

---

## Tiếp theo

Compress ngữ cảnh cho phép Agent chạy lâu không bị sập. Nhưng mỗi lần compress, những điều người dùng nói cho nó về sở thích, ràng buộc cũng mất theo. Có thể để Agent có chọn lọc nhớ lại những điều quan trọng không?

s09 Memory → Ba hệ thống con: chọn nhớ cái gì, trích xuất thông tin chính, tổng chỉnh lập lại. Xuyên suốt compress, xuyên suốt phiên.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Dưới đây dựa trên phân tích `compact.ts`, `autoCompact.ts`, `microCompact.ts`, `query.ts` của CC source.

### So sánh thứ tự thực thi

Phiên bản giảng dạy vì tiện giảng dạy theo L1/L2/L3/L4 đánh số, nhưng thứ tự thực thi thực tế và đánh số không hoàn toàn tương ứng:

| Chiều | Phiên giảng dạy | Claude Code |
|------|--------|-------------|
| Thứ tự thực thi | budget → snip → micro → auto | budget → snip → micro → collapse → auto（`query.ts:379-468`） |
| snip_compact | Giữ lại đầu 3 + cuối 47 | CC chỉ bật trên main thread; triển khai không trong repo mở（`HISTORY_SNIP` feature gate），nhưng interface nhìn thấy: `snipCompactIfNeeded(messages)` → `{ messages, tokensFreed, boundaryMessage? }`，còn lộ `SnipTool` công cụ cho mô hình gọi chủ động. Tham số 3/47 của phiên giảng dạy là đơn giản hóa |
| micro_compact | Thay thế text chiếm chỗ | Hai đường dẫn: time-based xóa nội dung trực tiếp, cached gọi API `cache_edits`（legacy path đã xóa） |
| micro_compact whitelist | Theo vị trí（3 dòng gần nhất） | time-based theo ngưỡng thời gian; cached theo lượng（`microCompact.ts`） |
| tool_result_budget | 200KB ký tự | 200,000 ký tự（`toolLimits.ts:49`） |
| compact_history ngưỡng | Ước tính ký tự | Token chính xác: `contextWindow - maxOutputTokens - 13_000` |
| Yêu cầu tóm tắt | 5 loại thông tin | 9 phần + `<analysis>`/`<summary>` tag kép |
| Prompt compress | Prompt đơn giản | Bảo vệ hai đầu cấm gọi công cụ |
| Thử lại PTL | Có（đơn giản） | `truncateHeadForPTLRetry()` quay lui theo nhóm tin nhắn（`compact.ts:243-290`） |
| Phục hồi sau compress | Không（phiên giảng dạy chỉ giữ tóm tắt） | Tự động đọc lại file gần đây, kế hoạch, agent/skill/tool v.v. |
| Cầu chì | 3 lần | 3 lần（`autoCompact.ts:70`） |
| Thử lại reactive | 1 lần | CC có phân loại thử lại tinh vi hơn |

### Chi tiết thứ tự thực thi

Thứ tự thực tế trong `query.ts` của CC source:

1. `applyToolResultBudget`（L379）: trước tiên xử lý kết quả lớn, đảm bảo nội dung đầy đủ lưu đĩa
2. `snipCompact`（L403）: cắt tin nhắn giữa
3. `microcompact`（L414）: kết quả cũ chiếm chỗ
4. `contextCollapse`（L441）: hệ thống quản lý ngữ cảnh độc lập（phiên giảng dạy không có）
5. `autoCompact`（L454）: tóm tắt LLM đầy đủ

Thứ tự budget → snip → micro của phiên giảng dạy phù hợp với đây. Phiên giảng dạy không có cơ chế contextCollapse.

### Cân nhắc read_file

`micro_compact` của phiên giảng dạy sẽ thay thế tất cả `tool_result` cũ bằng chiếm chỗ, bao gồm `read_file`. Điều này thường không ảnh hưởng tính đúng đắn chức năng: nếu sau đó vẫn cần nội dung file, mô hình có thể đọc lại. Chi phí là có thể thêm lần gọi công cụ, cũng có thể hạ tỷ lệ trúng prompt cache.

Claude Code không giải quyết vấn đề này bằng cách đơn giản như phiên giảng dạy. Nó đặt `Read` cũng vào tập công cụ có thể microcompact, nhưng đồng thời duy trì `readFileState`: khi đọc lại file không thay đổi trả về `FILE_UNCHANGED_STUB`, sau compact theo ngân sách phục hồi nội dung file đọc gần đây（ví dụ tối đa 5 file, mỗi 5K token, tổng ngân sách 50K token）. Đây là cơ chế cache và phục hồi trong triển khai mức sản xuất, phiên giảng dạy không mở rộng, giữ lại trade-off đơn giản "compress kết quả cũ, khi cần đọc lại".

### Tham chiếu hằng số đầy đủ

| Hằng số | Giá trị | File nguồn |
|--------|-------|-----------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | `autoCompact.ts:62` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | `autoCompact.ts:70` |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | `autoCompact.ts:30` |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | `compact.ts:123` |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | `compact.ts:122` |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | `compact.ts:124` |
| Khoảng micro_compact theo thời gian | 60 phút | `timeBasedMCConfig.ts` |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | `compact.ts:131` |

### contextCollapse và sessionMemoryCompact

Còn hai cơ chế trong CC source mà phiên giảng dạy này không mở rộng:

- **contextCollapse**: hệ thống quản lý ngữ cảnh độc lập, khi bật sẽ kìm hãm proactive autocompact（`autoCompact.ts:215-222`），do collapse's commit/blocking flow tiếp quản quản lý ngữ cảnh. Nhưng manual `/compact` và reactive fallback vẫn là đường dẫn độc lập, không chịu ảnh hưởng contextCollapse.
- **sessionMemoryCompact**: trước compact_history, CC sẽ thử dùng session memory có sẵn（s09 sẽ nói）làm tóm tắt nhẹ, không gọi LLM. Cơ chế này chờ học xong s09 sau quay lại xem sẽ rõ hơn.

### Prompt compress trông như thế nào?

Prompt compress của CC có hai yêu cầu cứng:

1. **Tuyệt đối cấm gọi công cụ**: bắt đầu là `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.`，cuối còn REMINDER lại một lần
2. **Trước phân tích rồi tóm tắt**: mô hình cần trước tiên suy luận trong tag `<analysis>`, rồi sau đó trong tag `<summary>` xuất ra tóm tắt chính thức. analysis bị tách rời khi định dạng

### Đơn giản hóa phiên giảng dạy là cố ý

- micro_compact dùng text chiếm chỗ → chúng ta không có quyền API `cache_edits`
- read_file không xử lý đặc biệt → phiên giảng dạy chấp nhận đọc lại khi cần, tránh đưa vào readFileState và cơ chế phục hồi sau compress
- token ước tính ký tự → tokenizer chính xác không trong phạm vi giảng dạy
- phục hồi sau compress bỏ qua → phiên giảng dạy chỉ giữ tóm tắt, không tự động tái gắn file
- hai cơ chế phụ không mở rộng → thuộc 10% chi tiết

Thiết kế cơ bản, cái rẻ chạy trước cái đắt chạy sau, hoàn toàn giữ lại.

</details>

<!-- translation-sync: zh@v2, en@v2, ja@v2, vi@v1 -->
