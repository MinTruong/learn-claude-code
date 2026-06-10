# s01: Agent Loop — Một loop là đủ

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

`s01` → [s02](../s02_tool_use/) → s03 → s04 → ... → s20

> *"Một loop & Bash là tất cả cần để bắt đầu"* — một tool + một loop = một agent
>
> **Harness layer**: loop — kết nối đầu tiên giữa model và thế giới thực.

---

## Vấn đề

Bạn hỏi LLM: "Liệt kê các file trong thư mục của tôi và chạy XXX.py đi".

Model xuất ra được một lệnh bash, nhưng xong lệnh là dừng. Nó không tự chạy cho bạn được, không xem kết quả rồi tiếp tục suy luận cũng không.

Bạn có thể tự chạy lệnh đó, paste output lại vào cho nó xem, rồi chờ lệnh tiếp theo. Làm đi làm lại như vậy nhiều lần.

Mỗi lần quay lại đó bạn đang là một lớp điều phối (middle layer). Và quá trình đó hoàn toàn có thể tự động hóa.

---

## Giải pháp

![Agent Loop](images/agent-loop.en.svg)

Một vòng lặp `while True`: model gọi tool thì tiếp tục, không gọi thì dừng. Toàn bộ chỉ có hai tín hiệu:

| Tín hiệu | Ý nghĩa | Hành động |
|------|------|---------|
| `stop_reason == "tool_use"` | Model giơ tay: "Tôi cần dùng tool" | Thực thi → nhét kết quả vào → tiếp tục |
| `stop_reason != "tool_use"` | Model nói: "Xong rồi" | Thoát loop |

---

## Nguyên lý hoạt động

Dịch quá trình trên thành code:

**Bước 1**: Đặt câu hỏi của user làm message đầu tiên.

```python
messages = [{"role": "user", "content": query}]
```

**Bước 2**: Gửi messages và tool definitions cho LLM.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

**Bước 3**: Nối câu trả lời của model, kiểm tra xem có gọi tool không. Không → kết thúc.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

**Bước 4**: Thực thi tool mà model yêu cầu, thu thập kết quả.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

**Bước 5**: Nối tool result làm message mới, quay lại bước 2.

```python
messages.append({"role": "user", "content": results})
```

Lắp ghép thành hàm hoàn chỉnh:

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

Chưa tới 30 dòng, đây là lõi harness agent nhỏ nhất có thể chạy. Nó không phải intelligence, mà là khung vận hành tối thiểu để model có thể hành động liên tục. Model chịu trách nhiệm quyết định (có gọi tool không, gọi tool nào), harness chịu trách nhiệm thực thi (gọi rồi thì chạy, kết quả nhét lại). 18 chương sau đều chồng mechanism lên loop này, còn loop thì không đổi.

---

## Thử nghiệm

> **Lưu ý demo giảng dạy**: Code sẽ thực thi shell command mà model sinh ra. Nên chạy trong thư mục test tạm thời để tránh ảnh hưởng file dự án. s03 sẽ nói về permission system thực sự.

**Chuẩn bị** (chạy lần đầu):

```sh
pip install -r requirements.txt
cp .env.example .env
# Sửa .env, điền ANTHROPIC_API_KEY và MODEL_ID
```

**Chạy**:

```sh
python s01_agent_loop/code.py
```

Thử các prompt:

1. `Tạo file hello.py in ra "Hello, World!"`
2. `Liệt kê tất cả file Python trong thư mục này`
3. `Hiện tại đang ở git branch nào?`

Quan sát trọng tâm: Model khi nào gọi tool (loop tiếp tục), khi nào không gọi (loop kết thúc)?

---

## Tiếp theo

Hiện tại model chỉ có bash là tool duy nhất. Đọc file phải `cat`, viết file phải `echo ... >`, tìm file phải `find`. Vừa xấu vễ dễ sai.

s02 Tool Use → Cấp cho nó 5 tool thực sự, rồi xem chuyện gì xảy ra. Model có gọi nhiều tool cùng lúc không? Nhiều tool chạy đồng thời có đè lên nhau không?

<details>
<summary>Đi sâu vào CC source code</summary>

> Nội dung dựa trên kiểm tra CC source code `src/query.ts` (1729 dòng). Khác biệt cốt lõi chỉ có hai: CC không dùng `stop_reason` field mà kiểm tra xem content có tool_use block không (vì stop_reason trong streaming response không reliable); CC có nhiều exit path và recovery strategy hơn cho production protection.

**30 dòng `while True` của bản giảng dạy chính là cốt lõi 1729 dòng của CC.** Mỗi mục dưới đây đều là protection mechanism chồng lên cốt lõi đó.

<details>
<summary>1. Khác biệt cấu trúc loop</summary>

Bản giảng dạy kiểm tra `response.stop_reason`. CC không dùng nó làm căn cứ duy nhất để tiếp tục loop -- trong streaming response, `stop_reason` có thể chưa update nhưng content đã có `tool_use` block. CC dùng flag `needsFollowUp`: khi nhận streaming message (`query.ts:830-834`), chỉ cần phát hiện `tool_use` block thì set là `true`; `QueryEngine.ts` sẽ bắt real `stop_reason` từ `message_delta` cho logic khác, nhưng query loop tự nó dựa vào `needsFollowUp` để quyết định có tiếp tục không.

```typescript
// query.ts:554-558
// stop_reason === 'tool_use' là không reliable.
// Set trong khi streaming mỗi khi tool_use block đến.
let needsFollowUp = false
```

</details>

<details>
<summary>2. State object 10 fields (bản giảng dạy chỉ dùng messages)</summary>

| # | Field | Mục đích | Chapter tương ứng |
|---|------|------|---------|
| 1 | `messages` | Message array của iteration hiện tại | s01 |
| 2 | `toolUseContext` | Tool, signal, permission context | s02 |
| 3 | `autoCompactTracking` | Tracking compaction state | s08 |
| 4 | `maxOutputTokensRecoveryCount` | Số lần thử token recovery (max 3) | s11 |
| 5 | `hasAttemptedReactiveCompact` | Round này đã thử reactive compact chưa | s08 |
| 6 | `maxOutputTokensOverride` | Override nâng cấp 8K→64K | s11 |
| 7 | `pendingToolUseSummary` | Tool use summary do Haiku nền tạo | s08 |
| 8 | `stopHookActive` | Stop hook có tạo blocking error không | s04 |
| 9 | `turnCount` | Đếm số turn (maxTurns check) | s01 |
| 10 | `transition` | Lý do tiếp tục lần trước | s11 |

> Lưu ý: `taskBudgetRemaining` (`query.ts:291`) là biến local của loop, không nằm trên State. Source code comment ghi rõ "Loop-local (not on State)".

</details>

<details>
<summary>3. Nhiều exit và continue paths</summary>

Bản giảng dạy chỉ có 1 exit path (model không gọi tool là kết thúc). Production version có nhiều exit và continue paths, bao phủ blocking limit, prompt too long, model error, abort, hook stop, max turns, token budget continuation, reactive compact retry và các scenario khác. Mỗi scenario có recovery hoặc exit strategy tương ứng.

</details>

<details>
<summary>4. Streaming tool execution và QueryEngine</summary>

`StreamingToolExecutor` của CC (`query.ts:561`) cho phép tool bắt đầu thực thi song song trong khi model vẫn đang generate output (quyết định theo việc tool có concurrency-safe hay không để chạy song song hoặc exclusive). `QueryEngine.ts` bổ sung các protection như vượt quá giới hạn phí, structured output validation fail, v.v. Bản giảng dạy không implement những điều này -- mục tiêu là concept clear, không phải performance tối đa.

</details>

**Một câu**: Cốt lõi 1729 dòng query.ts chính là 30 dòng `while True`. Tất cả complex fields và exit paths đều là protection mechanisms. Hiểu core loop trước, mọi thứ sau đó tự nhiên mở ra.

</details>

<!-- translation-sync: zh@v1, en@v0, ja@v0, vi@v1 -->
