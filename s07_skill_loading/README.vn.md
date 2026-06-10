# s07: Skill Loading — Khi nào cần dùng mới tải

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [Tiếng Việt](README.vn.md)

s01 → s02 → s03 → s04 → s05 → s06 → `s07` → [s08](../s08_context_compact/) → s09 → ... → s20
> *"Khi cần dùng mới tải, đừng nhồi tất cả vào prompt"* — Qua tool_result injection, không nhồi vào system prompt.
>
> **Tầng Harness**: Kiến thức — Tải theo nhu cầu, không lấp ngữ cảnh.

---

## Vấn đề

Dự án của bạn có một bộ quy chuẩn React component, một bản hướng dẫn kiểu SQL, một tài liệu thiết kế API. Bạn muốn Agent tự động tuân thủ những quy chuẩn này. Ý tưởng trực tiếp nhất, nhồi tất cả vào system prompt:

```python
SYSTEM = (
    f"You are a coding agent. "
    + open("docs/react-style.md").read()       # 2000 dòng
    + open("docs/sql-style.md").read()         # 1500 dòng
    + open("docs/api-design.md").read()        # 3000 dòng
)
```

6500 dòng system prompt. Agent mỗi lần gọi LLM đều mang theo những tài liệu này — dù đang sửa màu CSS hay chỉnh truy vấn SQL. 99% nội dung không liên quan đến nhiệm vụ hiện tại, lãng phí token.

---

## Giải pháp

![Skill Overview](images/skill-overview.svg)

Giữ nguyên cấu trúc hook tối thiểu từ chương trước, `todo_write` và sub Agent, chương này tập trung vào công cụ `load_skill` mới được thêm. Khởi động thêm thư mục kỹ năng vào SYSTEM prompt, lúc chạy đăng ký thêm một công cụ tải nội dung đầy đủ, chỉ tốn token khi cần.

Hai tầng thiết kế:

| Tầng | Vị trí | Thời điểm | Chi phí |
|-----|--------|---------|---------|
| 1. Thư mục | system prompt | Khởi động nhồi（harness quét skills/） | ~100 tokens/skill，mỗi lần đều mang |
| 2. Nội dung | tool_result | Khi Agent gọi load_skill; SKILL.md có thể hướng dẫn gọi read_file/bash tiếp theo, để truy cập thêm tài nguyên theo nhu cầu | ~2000 tokens/skill，theo nhu cầu |

Cơ chế dispatch không thay đổi, load_skill qua `TOOL_HANDLERS[block.name]` phân phối.

---

## Cách hoạt động

**Thư mục skills/**, mỗi kỹ năng một thư mục con, chứa file `SKILL.md`:

```
skills/
  agent-builder/SKILL.md
  code-review/SKILL.md
  mcp-builder/SKILL.md
  pdf/SKILL.md
```

**Tầng một: Nhồi thư mục khi khởi động**: harness khởi động gọi `_scan_skills()` quét thư mục skills/, phân tích frontmatter YAML của mỗi SKILL.md（`name`、`description`），lưu vào từ điển `SKILL_REGISTRY`. `list_skills()` từ bảng đăng ký tạo thư mục, nhồi vào SYSTEM prompt. Agent mỗi lần đều thấy "tôi có những kỹ năng nào", không tốn thêm lệnh gọi API:

```python
SKILL_REGISTRY: dict[str, dict] = {}

def _scan_skills():
    if not SKILLS_DIR.exists():
        return
    for d in sorted(SKILLS_DIR.iterdir()):
        if not d.is_dir():
            continue
        manifest = d / "SKILL.md"
        if manifest.exists():
            raw = manifest.read_text()
            meta, body = _parse_frontmatter(raw)
            name = meta.get("name", d.name)
            desc = meta.get("description", raw.split("\n")[0].lstrip("#").strip())
            SKILL_REGISTRY[name] = {"name": name, "description": desc, "content": raw}

_scan_skills()  # runs once at startup

def list_skills() -> str:
    return "\n".join(f"- **{s['name']}**: {s['description']}" for s in SKILL_REGISTRY.values())

def build_system() -> str:
    catalog = list_skills()
    return (
        f"You are a coding agent at {WORKDIR}. "
        f"Skills available:\n{catalog}\n"
        "Use load_skill to get full details when needed."
    )

SYSTEM = build_system()
```

**Tầng hai: load_skill**: Agent quyết định "tôi cần hướng dẫn kiểu SQL", gọi `load_skill("sql-style")`. Tìm qua bảng đăng ký, không đi qua đường dẫn file, không có rủi ro path traversal. Nội dung SKILL.md qua `tool_result` nhồi vào, và có thể qua công cụ file và bash hiện có tiếp cộttruy cập `references/`、`scripts/` hoặc `assets/` được tham chiếu.

```python
def load_skill(name: str) -> str:
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

Khác biệt chính: nội dung kỹ năng không phải một phần của system prompt, nó như một kết quả công cụ vào messages hiện tại. Gọi tiếp theo sẽ mang theo lịch sử, cho đến khi ngữ cảnh compress, cắt ngắn hoặc kết thúc phiên. Điều này tự nhiên kết nối với s08 compact: tải theo nhu cầu giải quyết "không nên mang trước thì không mang", compact giải quyết "cái nào nên vứt là vứt thế nào".

---

## Thay đổi so với s06

| Thành phần | Trước đó (s06) | Sau đó (s07) |
|-----------|--------------|------------|
| Số lượng công cụ | 7 (bash, read, write, edit, glob, todo_write, task) | 8 (+load_skill) |
| Tải kiến thức | Không | Hai tầng: nhồi thư mục khi khởi động vào SYSTEM + load_skill lúc chạy; SKILL.md có thể hướng dẫn truy cập tài nguyên tiếp theo |
| SYSTEM prompt | Chuỗi tĩnh | Quét skills/ khởi động nhồi thư mục |
| Bảng đăng ký kỹ năng | Không | SKILL_REGISTRY（nhồi lúc khởi động, chặn path traversal） |
| Vòng lặp | Không thay đổi | Không thay đổi（công cụ skill phân phối tự động） |

---

## Hãy thử

```sh
cd learn-claude-code
python s07_skill_loading/code.py
```

Thử những prompt này:

1. `What skills are available?`
2. `Load the code-review skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`

Điểm quan sát: Agent có trực tiếp biết từ thư mục trong SYSTEM có những kỹ năng nào không? Khi cần quy chuẩn đầy đủ có xuất hiện `[HOOK] load_skill` không? Sau khi tải, trả lời có sử dụng hướng dẫn từ skill tương ứng không?

---

## Tiếp theo

Tải theo nhu cầu giải quyết "không nên mang thì không mang". Nhưng vấn đề khác đến: Agent làm việc liên tục 30 phút, danh sách messages lấp đầy quá trình trung gian. tool_result cũ, nội dung file lỗi thời, chiếm ngữ cảnh nhưng không tạo giá trị.

s08 Context Compact → Bốn tầng compress. Cái rẻ chạy trước, cái đắt chạy sau.

<details>
<summary>Đi sâu vào mã nguồn CC</summary>

> Dưới đây là phân tích dựa trên mã nguồn CC `loadSkillsDir.ts`, `SkillTool.ts`, `bundledSkills.ts`, `commands.ts`.

### Một. Nguồn kỹ năng: không phải chỉ một thư mục skills/

Phiên bản giảng dạy giả sử tất cả kỹ năng trong thư mục `skills/`. CC thực tế tải từ nhiều nguồn, phân bố trong nhiều file: `loadSkillsDir.ts` tải từ user/project/`--add-dir` thư mục và legacy commands（`.claude/commands/`）; `bundledSkills.ts` tải kỹ năng nội bộ; `SkillTool.ts` xử lý kỹ năng remote MCP; `commands.ts` tập hợp lệnh. Loại bao gồm managed/policy skills, user skills（`~/.claude/skills/`）, project skills（`.claude/skills/`）, `--add-dir` skills, legacy commands, dynamic skills, conditional skills（có frontmatter `paths`, kích hoạt theo đường dẫn file）, bundled skills, plugin skills, MCP skills.

### Hai. Các trường Frontmatter SKILL.md phổ biến

Frontmatter YAML của SKILL.md của CC được `parseSkillFrontmatterFields()` phân tích（`loadSkillsDir.ts`），các trường phổ biến bao gồm:

| Trường | Mục đích |
|-------|---------|
| `name` / `description` | Hiển thị tên và mô tả |
| `when_to_use` | Hướng dẫn mô hình khi nào gọi |
| `allowed-tools` | Danh sách công cụ tự động cho phép của kỹ năng |
| `context` | `inline`（mặc định）hoặc `fork`（chạy như sub Agent） |
| `model` | Ghi đè mô hình（haiku/sonnet/opus/inherit） |
| `hooks` | Cấu hình hook cấp kỹ năng |
| `paths` | Mô hình glob kích hoạt điều kiện |
| `user-invocable` | Người dùng có thể gọi qua `/name` |

Danh sách trường đầy đủ thay đổi theo phiên bản, trên chỉ liệt kê các trường cốt lõi mà phiên bản giảng dạy liên quan.

### Ba. Triển khai chính xác tải hai tầng

1. **Catalog（khởi động）**: `getSkillDirCommands()` quét thư mục → đăng ký như `Command` object, chỉ chứa metadata. `getSkillListingAttachments()` định dạng danh sách kỹ năng làm đính kèm, ngân sách là ~1% cửa sổ ngữ cảnh（tối đa 8000 ký tự）.
2. **Load（gọi）**: Mô hình gọi công cụ `Skill`（trường nhập `skill` + tuỳ chọn `args`, phiên bản giảng dạy dùng `name`）→ `getPromptForCommand()` mở rộng nội dung SKILL.md đầy đủ → `SkillTool` trả về tool_result chỉ hiển thị text là `"Launching skill: {name}"`，nội dung kỹ năng thực tế qua `newMessages` nhồi vào hội thoại. Phiên bản giảng dạy hợp nhất hai cái là một đơn giản hóa; sau tải SKILL.md vẫn có thể như hướng dẫn, giúp mô hình sau đó qua công cụ file/bash hiện có truy cập tài nguyên liên quan.

### Phiên bản giảng dạy đơn giản hóa là cố ý

- Nhiều file nhiều nguồn → 1 thư mục `skills/`: đủ để trình bày khái niệm cốt lõi tải hai tầng
- Nhiều trường frontmatter → chỉ phân tích name/description: giảm độ phức tạp phân tích
- forked skills（`context: 'fork'`）→ bỏ qua: phiên bản giảng dạy chỉ mở rộng tải kỹ năng inline
- Đầu vào công cụ `Skill` là `skill`+`args` → phiên bản giảng dạy dùng `name`: tránh độ phức tạp thêm của phân tích tham số

</details>

<!-- translation-sync: zh@v2, en@v2, ja@v2, vi@v1 -->
