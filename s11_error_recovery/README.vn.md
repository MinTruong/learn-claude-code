# s11: Khôi phục lỗi — Lỗi không phải là kết thúc, mà là điểm bắt đầu để thử lại

[Tiếng Việt](README.vn.md) · [中文](README.md) · [English](README.en.md) · [日本語](README.ja.md)

s01 → ... → s09 → s10 → `s11` → [s12](../s12_task_system/) → s13 → ... → s20
> *"Lỗi không phải là điểm cuối, mà là điểm bắt đầu để thử lại"* — nâng cấp token, nén ngữ cảnh, chuyển đổi mô hình.
>
> **Tầng Harness**: Khả năng phục hồi — phân loại và khôi phục khi vòng lặp chính gặp lỗi.

---

## Vấn đề

Agent chạy chạy thì báo lỗi:

```
Error: 529 overloaded
```

Agent sập. Nó không thử lại, không đổi mô hình, không giảm ngữ cảnh — sập thẳng.

Trong môi trường sản xuất, lỗi API là chuyện thường xảy ra. Ba kiểu lỗi phổ biến nhất: **đầu ra bị cắt cụt** (mô hình nói dở dang khi hết token), **ngữ cảnh vượt quá giới hạn** (vẫn quá dài sau khi nén), **lỗi tạm thời** (429 rate limit / 529 quá tải). Một Agent không xử lý lỗi giống như chiếc xe chết máy ngay khi chạm phải.

---

## Giải pháp

![Tổng quan Khôi phục Lỗi](images/error-recovery-overview.svg)

s10 giữ nguyên vòng lặp, ghép prompt. Thay đổi duy nhất: gọi LLM được bọc trong try/except, tuỳ loại lỗi đi theo đường khôi phục khác nhau. Sau khi khôi phục thì `continue` quay lại đầu vòng lặp để gọi lại LLM.

Ba kiểu khôi phục phổ biến nhất (phiên bản dạy chỉ xử lý 429/529; hệ thống thực còn xử lý lỗi kết nối, timeout, xác thực nhà cung cấp đám mây... CC thực có 13+ mã lý do, xem thêm trong Deep dive):

| Kiểu | Kích hoạt | Hành động khôi phục |
|------|------|---------|
| Đầu ra bị cắt cụt | `max_tokens` | Nâng cấp 8K→64K / prompt tiếp tục |
| Ngữ cảnh vượt quá giới hạn | `prompt_too_long` | reactive compact → thử lại |
| Lỗi tạm thời | 429 / 529 | Exponential backoff + jitter, liên tục 529 có thể chuyển sang mô hình dự phòng |

---

## Cách hoạt động

### Đường 1: Đầu ra bị cắt cụt

Mô hình nói dở dang, `max_tokens` hết. Mặc định 8000 token không đủ để nó xuất câu trả lời đầy đủ.

Lần đầu xảy ra, ngay lập tức nâng `max_tokens` từ 8K lên 64K (gấp 8 lần không gian), thử lại cùng request — lúc này không thêm đầu ra bị cắt cụt vào messages, giữ nguyên request gốc. Nếu 64K vẫn không đủ, mới lưu đầu ra bị cắt cụt và thêm prompt tiếp tục để mô hình nói tiếp, tối đa 3 lần:

```python
if response.stop_reason == "max_tokens":
    # First escalation: don't append truncated output, retry same request
    if not state.has_escalated:
        max_tokens = ESCALATED_MAX_TOKENS
        state.has_escalated = True
        continue  # messages unchanged, same request with more tokens
    # 64K still truncated: save output + continuation prompt
    messages.append({"role": "assistant", "content": response.content})
    if state.recovery_count < MAX_RECOVERY_RETRIES:
        messages.append({"role": "user", "content":
            "Output token limit hit. Resume directly — "
            "no apology, no recap. Pick up mid-thought."})
        state.recovery_count += 1
        continue
    return  # still truncated after 3 continuations
# Normal: append after max_tokens check
messages.append({"role": "assistant", "content": response.content})
```

Chỉ có một cơ hội nâng cấp, tiếp tục tối đa 3 lần. Vượt quá thì thoát — tiếp tục cũng không có kết quả thực chất.

### Đường 2: Ngữ cảnh vượt quá giới hạn

LLM nói "ngữ cảnh của bạn quá dài" (`prompt_too_long`). Bốn lớp nén của s08 chạy hết rồi, vẫn quá.

Kích hoạt reactive compact — mạnh mẽ hơn auto compact. Phiên bản dạy chỉ giữ lại 5 tin nhắn cuối để mô phỏng hiệu quả nén; bản thực sẽ gọi LLM sinh tóm tắt compact rồi thử lại. Nén xong thử lại. Nhưng nếu đã nén một lần rồi vẫn quá giới hạn, chỉ có thể thoát — nén nữa cũng không nhỏ hơn:

```python
except PromptTooLongError:
    if not state.has_attempted_reactive_compact:
        messages[:] = reactive_compact(messages)
        state.has_attempted_reactive_compact = True
        continue
    return  # 压缩过了还是超限，只能退出
```

### Đường 3: Lỗi tạm thời

Mạng chập chờn, 429 rate limit, 529 quá tải — đây không phải bug, mà là chuyện thường trong hệ thống phân tán.

429 và 529 cùng đi theo exponential backoff + jitter: lần đầu chờ 0.5 giây, lần hai 1 giây, lần ba 2 giây, tối đa 10 lần. Thêm random jitter để request đồng thời không retry cùng lúc. Liên tiếp 3 lần 529 quá tải → chuyển sang mô hình dự phòng (nếu có cài biến môi trường `FALLBACK_MODEL_ID`):

```python
def retry_delay(attempt, retry_after=None):
    if retry_after:
        return retry_after
    base = min(500 * (2 ** attempt), 32000) / 1000
    return base + random.uniform(0, base * 0.25)

def with_retry(fn, state, max_retries=10):
    for attempt in range(max_retries):
        try:
            return fn()
        except (RateLimitError, OverloadedError):
            delay = retry_delay(attempt)
            time.sleep(delay)
            if is_overloaded:
                state.consecutive_529 += 1
                if state.consecutive_529 >= 3 and FALLBACK_MODEL:
                    state.current_model = FALLBACK_MODEL
    raise MaxRetriesExceeded()
```

Công thức backoff: `min(500 × 2^attempt, 32000) + random(0~25%)`. Nếu server trả về header `Retry-After`, ưu tiên dùng giá trị đó.

### Tổng hợp lại

```python
def agent_loop(messages, context):
    system = get_system_prompt(context)
    state = RecoveryState()
    max_tokens = 8000

    while True:
        try:
            response = with_retry(
                lambda: client.messages.create(
                    model=state.current_model, system=system,
                    messages=messages, tools=TOOLS,
                    max_tokens=max_tokens),
                state)
        except Exception as e:
            if is_prompt_too_long_error(e):
                if not state.has_attempted_reactive_compact:
                    messages[:] = reactive_compact(messages)
                    state.has_attempted_reactive_compact = True
                    continue
                return
            log_error(e)
            return

        # max_tokens check BEFORE appending to messages
        if response.stop_reason == "max_tokens":
            if not state.has_escalated:
                max_tokens = 64000
                state.has_escalated = True
                continue  # retry same request, messages unchanged
            # save truncated output + continuation prompt
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": CONTINUATION_PROMPT})
            continue
        # Normal completion
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return
        # ... tool execution ...
```

Lớp ngoài try/except bắt lỗi API (prompt_too_long v.v.), `with_retry` xử lý lỗi tạm thời (429/529), kiểm tra `stop_reason` xử lý cắt cụt. Ba cơ chế khôi phục, mỗi cái quản một kiểu lỗi.

---

## Thay đổi so với s10

| Thành phần | Trước (s10) | Sau (s11) |
|------|-----------|-----------|
| Xử lý lỗi | Không (sập ngay) | Ba kiểu khôi phục + exponential backoff |
| Hằng số mới | — | ESCALATED_MAX_TOKENS=64000, MAX_RETRIES=10, BASE_DELAY_MS=500, FALLBACK_MODEL |
| Hàm mới | — | with_retry, retry_delay, reactive_compact, is_prompt_too_long_error, RecoveryState |
| Công cụ | bash, read_file, write_file (3) | bash, read_file, write_file (3) — không đổi |
| Vòng lặp | Gọi LLM trực tiếp | Bọc try/except + continue thử lại |

---

## Thử ngay

```sh
cd learn-claude-code
python s11_error_recovery/code.py
```

Thử các prompt này:

1. Để Agent sinh một đoạn code dài, quan sát xem khi bị cắt cụt có tự động tiếp tục không (xem log `[max_tokens] escalating`)
2. Liên tục đọc nhiều file để làm lớn ngữ cảnh, quan sát reactive compact
3. Nếu gặp 429/529, quan sát log output của exponential backoff

---

## Tiếp theo

Agent giờ đã tự khôi phục khi gặp lỗi. Nhưng nhiệm vụ nó xử lý vẫn là "một lần" — bạn giao nhiệm vụ, nó làm xong, kết thúc.

Có thể để Agent quản lý **danh sách nhiệm vụ** — có quan hệ phụ thuộc, lưu trữ trên đĩa, khôi phục được qua các phiên không? Danh sách TODO không phải là hệ thống nhiệm vụ.

s12 Hệ thống Nhiệm vụ → Nhiệm vụ là đồ thị có phụ thuộc, có trạng thái, lưu trữ được. Đây là nền tảng cho cộng tác đa Agent.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Phân tích dựa trên mã nguồn CC `query.ts` (1729 dòng), `services/api/withRetry.ts` (822 dòng), `query/tokenBudget.ts` (93 dòng), `utils/tokenBudget.ts` (73 dòng).

### Một. Hơn chục reason/transition (không chỉ 3)

Phiên bản dạy nói 3 kiểu khôi phục phổ biến nhất. CC thực có hơn chục reason/transition, mỗi lần gọi LLM xong đều đánh giá:

| reason/transition | Phiên bản dạy tương ứng | Hành vi CC |
|---|---|---|
| `completed` | Hoàn thành bình thường | Trả kết quả |
| `next_turn` | Gọi công cụ bình thường | Tiếp tục vòng thực thi công cụ |
| `max_output_tokens_escalate` | Đường 1 | Nâng cấp 8K→64K |
| `max_output_tokens_recovery` | Đường 1 tiếp tục | Prompt tiếp tục (tối đa 3 lần) |
| `reactive_compact_retry` | Đường 2 | reactive compact → thử lại |
| `prompt_too_long` | Đường 2 | Tương tự |
| `collapse_drain_retry` | Không mở rộng | context collapse gửi tạm trước |
| `model_error` | Không mở rộng | Thử lại |
| `image_error` | Không mở rộng | `ImageSizeError` / `ImageResizeError` xử lý riêng |
| `aborted_streaming` | Không mở rộng | Khôi phục stream bị hủy |
| `aborted_tools` | Không mở rộng | Công cụ bị hủy |
| `stop_hook_blocking` | Không mở rộng | Chèn blocking error → mô hình tự sửa |
| `stop_hook_prevented` | Không mở rộng | hooks ngăn cản |
| `hook_stopped` | Không mở rộng | Hook dừng thực thi |
| `token_budget_continuation` | Không mở rộng | Tiếp tục khi token < 90% |
| `blocking_limit` | Không mở rộng | Giới hạn blocking |
| `max_turns` | Không mở rộng | Đạt số vòng tối đa |

Phiên bản dạy chỉ mở rộng 5 kiểu đầu (phổ biến nhất), các kiểu khác có logic xử lý riêng.

### Hai. Công thức exponential backoff chính xác

Delay backoff của CC (`withRetry.ts:530-548`):

```
delay = min(500 × 2^(attempt-1), 32000) + random(0~25%)
```

| Lần thử | Delay cơ bản | + Jitter |
|------|---------|--------|
| 1 | 500ms | 0-125ms |
| 2 | 1000ms | 0-250ms |
| 4 | 4000ms | 0-1000ms |
| 7+ | 32000ms (giới hạn trên) | 0-8000ms |

Nếu server trả về header `Retry-After`, ưu tiên dùng giá trị đó.

### Ba. CONTINUATION prompt gốc

Prompt tiếp tục của CC (`query.ts:1225-1227`):

```
Output token limit hit. Resume directly — no apology, no recap of what
you were doing. Pick up mid-thought if that is where the cut happened.
Break remaining work into smaller pieces.
```

Prompt gợi ý token budget (`tokenBudget.ts:72`):

```
Stopped at {pct}% of token target. Keep working — do not summarize.
```

### Bốn. Xử lý lỗi stream

Trong đường stream của CC, lỗi có thể khôi phục (413, max_tokens, media error) **bị giữ lại không hiển thị** trong streaming (`query.ts:788-822`) — SDK consumer không thấy, chỉ có logic khôi phục thấy. Sau khi streaming kết thúc mới đánh giá có cần khôi phục không.

### Năm. 529 → Chuyển Fallback Model

Sau 3 lần liên tiếp lỗi 529 quá tải (`MAX_529_RETRIES = 3`), CC tự động chuyển sang fallback model (ví dụ Opus → Sonnet). Khi chuyển xóa tất cả pending messages và kết quả công cụ, hiển thị cho người dùng "Switched to {model} due to high demand".

### Sáu. Phát hiện Diminishing Returns

"Tiếp tục" của token budget không vô hạn. Khi tiếp tục 3 lần liên tiếp và token tăng thêm < 500, hệ thống đánh giá "tiếp tục cũng không có kết quả thực chất", dừng continuation (`tokenBudget.ts:60-62`).

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
