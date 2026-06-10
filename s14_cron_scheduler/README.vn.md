# s14: Cron Scheduler — Lịch trình định时任务

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s13 → `s14` → [s15](../s15_agent_teams/) → s16 → ... → s20
> *"Kickstart định时任务, không cần can thiệp thủ công"* — Agent có thể tự động kích hoạt công việc theo lịch.

---

## Vấn đề

Agent thường phải chờ lệnh chậm (như `npm install`, `pip install`) hoặc làm việc rườm qua thời gian. Nếu muốn **tự động** thực hiện các thao tác định kỳ (ví dụ: kiểm tra trạng thái server mỗi 5 phút, chạy backup vào 2h sáng), việc phải ngồi đợi hoặc tự nhắc nhở là không hiệu quả.

---

## Giải pháp

![Cron Overview](images/cron-overview.svg)

**Cron** là hệ thống lịch trình phổ biến trong Unix/Linux, cho phép định nghĩa các sự kiện sẽ xảy ra tại thời điểm cụ thể. Trong Agent, chúng ta sử dụng cron để:

- Tự động khởi chạy các công cụ chậm
- Kiểm tra trạng thái hệ thống định kỳ
- Gửi thông báo định kỳ
- Thực hiện các công việc bảo trì

Công cụ cron trong Agent hoạt động như sau:

1. **Định nghĩa lịch trình**: Sử dụng cú pháp cron (`minute hour day month weekday`)
2. **Gán công việc**: Mỗi lịch trình tương ứng với một công cụ (tool) và các tham số
3. **Kiểm tra định kỳ**: Cron chạy nền, khi khớp thời điểm thì khởi chạy công cụ
4. **Thông báo kết quả**: Kết quả được chèn vào cuộc hội thoại như một `<task_notification>`

---

## Nguyên lý hoạt động

### 1. Cú pháp cron

Cron sử dụng 5-6 trường để định nghĩa lịch:

```
* * * * * command
- minute (0-59)
- hour (0-23)
- day of month (1-31)
- month (1-12)
- day of week (0-6, 0 = Sunday)
- year (optional)

* → mọi giá trị
*/5 → mỗi 5 phút
0 9 * * 1-5 → 9h sáng thứ 2-6 (thứ Hai đến thứ Sáu)
```

**Ví dụ**:
- `0 0 * * * echo "Backup completed"` → Chạy lúc 0:00 mỗi ngày
- `30 14 * * 1-5 echo "Check server"` → 14:30 thứ 2-6

---

## Cấu trúc file cron

Trong Agent, cron jobs được định nghĩa trong file `.claude/cron.json`:

```json
{
  "cron_jobs": [
    {
      "schedule": "0 0 * * *",
      "tool": "bash",
      "input": {
        "command": "echo 'Daily backup completed'"
      },
      "description": "Daily backup at midnight"
    },
    {
      "schedule": "30 14 * * 1-5",
      "tool": "bash",
      "input": {
        "command": "echo 'Check server health at 2:30PM on weekdays'"
      },
      "description": "Weekday health check at 2:30PM"
    }
  ]
}
```

---

## Cách sử dụng

### 1. Định nghĩa lịch trình

Bạn có thể định nghĩa cron job qua:

- **CLI**: `cron add "0 0 * * * echo 'Daily backup'"`
- **File**: Chỉnh sửa `.claude/cron.json` trực tiếp
- **Skill**: Dùng skill `cron` để tạo lịch trình qua giao diện

### 2. Chạy nền

Cron chạy **ngoài vòng lặp chính** của Agent. Nó không chặn việc xử lý các lệnh khác, chỉ khi khớp thời điểm thì mới thực thi công cụ.

### 3. Kết quả và thông báo

Khi cron kích hoạt công cụ:

1. Agent nhận được thông báo từ `collect_background_results()`
2. Thông báo được định dạng thành `<task_notification>`
3. Thông báo được chèn vào cùng một message với kết quả công cụ

**Ví dụ**:
```
<task_notification>
  <task_id>bg_0005</task_id>
  <status>completed</status>
  <command>npm install</command>
  <summary>npm install completed successfully (took 45s)</summary>
</task_notification>
```

---

## Thay đổi so với s13

| Thành phần | Trước (s13) | Sau (s14) |
|----------|------------|----------|
| Chạy công cụ | Chỉ khi LLM yêu cầu hoặc khi `should_run_background` trả về true | Cron tự động kiểm tra định kỳ, không cần LLM can thiệp |
| Kiểm soát | LLM quyết định khi nào chạy | Cron tự động quyết định, LLM chỉ cần định nghĩa lịch trình |
| Kiểu công cụ | Chạy khi có lệnh `run_in_background` | Cron có thể khởi chạy bất kỳ công cụ nào (cũng như s13) |
| Tính năng mới | — | **Lịch trình tự động**, **độ tinh tế thời gian**, **quản lý lịch sử** |

---

## Thử nghiệm

```sh
cd learn-claude-code
python s14_cron_scheduler/code.py
```

Thử các prompt:

1. `Lên lịch: chạy "echo 'Hello'" lúc 9h sáng mỗi ngày`
2. `Lên lịch: chạy "echo 'Check health'" lúc 14:30 thứ 2, 3, 3, 4, 5`
3. `Liệt kê tất cả lịch trình đã được định nghĩa`

Quan sát trọng tâm: Cron job có chạy khi LLM không yêu cầu, và kết quả được thông báo qua `<task_notification>` không?

---

## Tiếp theo

s15 Agent Teams → nhiều agent cùng làm việc, cần có cơ chế trao đổi và phân công.

---

## Tài liệu tham khảo

- [Cron Tutorial](https://en.wikipedia.org/wiki/Cron)
- [Cron.Guru](https://crontab.guru/) - Công cụ trực quan hóa cron expression
- [Anthropic Cron Guide](https://docs.anthropic.com/claude/code/scheduling)