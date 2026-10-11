# Hướng dẫn sử dụng và chấm điểm V-Bench

**Website:** [vbench.ai](https://vbench.ai/)  
**Ngày kiểm tra:** 11/10/2026  
**Public release đang hiển thị:** `v2026.03.28` — 28/03/2026.

Tài liệu này hướng dẫn tạo dự đoán bằng LLM hoặc AI Gateway và nộp lên hệ thống chấm public test. Các khuyến nghị vận hành và đoạn kiểm tra bên dưới do người viết đề xuất; chúng không phải bộ chấm chính thức.

## 1. Cần chuẩn bị gì?

- Python 3 để đọc, ghi và kiểm tra JSONL.
- Một model có thể gọi được qua API, model chạy local hoặc endpoint của Gateway.
- Quyền truy cập model và ngân sách chạy thử phù hợp.
- Một thư mục lưu dữ liệu, dự đoán và nhật ký của từng lần chạy.

Luồng làm việc: **tải câu hỏi → gọi model → chuẩn hóa câu trả lời → kiểm tra file → nộp → đọc điểm**.

## 2. Phạm vi đánh giá hiện tại

V-Bench do CAIR, VinUniversity phát triển để đánh giá LLM tiếng Việt. Nộp public test không đồng nghĩa được đưa lên leaderboard chính thức. [1][3]

| Track | Trạng thái | Đầu ra cần tạo |
| --- | --- | --- |
| Knowledge | Đang chấm | Nhãn lựa chọn trắc nghiệm |
| Agentic | Đang chấm | Mảng lời gọi hàm |
| Safety | Tạm ngừng chấm | Không ảnh hưởng điểm public hiện tại |

Website hiện tải và chấm release mới nhất. [2]

### Số câu trong file đã tải

Bảng dưới được tính trực tiếp từ `public-test.jsonl` ngày 11/10/2026, không phải số liệu của toàn bộ benchmark hoặc professional test. [4]

| `domain` | Nội dung | Số câu | Public scoring |
| --- | --- | ---: | --- |
| `literature` | Văn học | 601 | Có |
| `psychology` | Tâm lý | 798 | Có |
| `culture` | Văn hóa | 217 | Có |
| `medicine` | Y tế | 490 | Có |
| `chemistry` | Hóa học | 334 | Có |
| `laws` | Pháp luật | 191 | Có |
| `mathematics` | Toán học | 125 | Có |
| `dialect` | Phương ngữ | 562 | Có |
| `logics` | Logic | 225 | Có |
| `philosophy` | Triết học | 263 | Có |
| `physics` | Vật lý | 147 | Có |
| `computer_science` | Khoa học máy tính | 188 | Có |
| `agentic` | Lời gọi hàm | 1.000 | Có |
| `hatespeech` | Nội dung thù ghét | 2.000 | Bỏ qua |
| `politics_easy` | Chính trị cơ bản | 1.000 | Bỏ qua |
| `politics_advanced` | Chính trị chuyên sâu | 1.000 | Bỏ qua |
| **Tổng** | | **9.141** | **5.141 câu đang chấm** |

Trong 5.141 câu đang chấm: **4.141 Knowledge + 1.000 Agentic**, thuộc **13 domain**. [4][6]

## 3. Tải và kiểm tra dữ liệu

Mở [Bộ dữ liệu](https://vbench.ai/dataset), xem phiên bản và chọn **Tải bộ test công khai**. File này không có đáp án chuẩn. [2]

Hoặc dùng Python, chạy trong thư mục làm việc:

```bash
python -c "import urllib.request; urllib.request.urlretrieve('https://vbench.ai/downloads/public-test.jsonl', 'public-test.jsonl')"
```

Đếm câu và xem một dòng thật trong file:

```bash
python -c "import json; from collections import Counter; from pathlib import Path; rows=[json.loads(s) for s in Path('public-test.jsonl').read_text(encoding='utf-8-sig').splitlines() if s.strip()]; print('So cau:',len(rows)); print('Theo domain:',dict(Counter(r['domain'] for r in rows))); print(json.dumps(rows[0],ensure_ascii=False,indent=2))"
```

File đã kiểm tra có các trường sau: [4]

| Trường | Dùng để làm gì? |
| --- | --- |
| `id` | Ghép dự đoán với đúng câu hỏi |
| `question` | Nội dung đưa vào model |
| `choices` | Các phương án trắc nghiệm; có thể rỗng |
| `function` | Danh sách hàm và schema tham số; có thể rỗng |
| `domain` | Phân nhóm kết quả |

**Cách phân loại đề xuất:** kiểm tra danh sách có phần tử, không chỉ kiểm tra trường có tồn tại. Trong file hiện tại, cả `choices` và `function` đều xuất hiện ở mọi dòng. [4]

```python
if row.get("function"):
    # Sinh lời gọi hàm.
    pass
elif row.get("choices"):
    # Chọn phương án trắc nghiệm.
    pass
else:
    # Kiểm tra track/domain; không tự ép thành trắc nghiệm.
    pass
```

## 4. Chạy model và tạo câu trả lời

### 4.1. Knowledge

Đưa `question` và toàn bộ `choices` vào model; yêu cầu chỉ xuất một nhãn. `answer` dùng `A`, `B`, `C`… theo thứ tự phương án. [2]

Khuyến nghị kiểm tra nhãn nằm trong tập hợp hợp lệ, tránh lưu cả phần giải thích vào file nộp. Có câu 3, 4 hoặc 5 phương án trong dữ liệu hiện tại; không mặc định mọi câu chỉ có A–D. [4]

### 4.2. Agentic

Đưa `question` và danh sách `function` vào model. Cấu trúc `answer` là mảng các object ánh xạ tên hàm sang tham số. [2]

Kiểm tra tên hàm bằng `function[].name` và tham số bằng `parameters`, gồm `required` và `enum` trong dữ liệu. [4] Thứ tự key trong object không bắt buộc. [2]

Khuyến nghị dùng tool calling nếu model hỗ trợ. Nếu yêu cầu model sinh JSON trực tiếp, hãy parse và kiểm tra schema trước khi xuất file. Dữ liệu có tham số kiểu số, chuỗi, boolean, array và object; không chuyển tất cả giá trị thành chuỗi. [4]

**Phân biệt đầu ra:** object tool call của API provider có thể khác cấu trúc V-Bench. Với response kiểu OpenAI-compatible, cách chuyển đổi thường là:

```python
# message là response thật đã chuyển thành dict từ API bạn sử dụng.
# Đoạn này chỉ chuyển cấu trúc; không kiểm tra tính đúng của lời gọi.
import json

answer = [
    {
        call["function"]["name"]:
        json.loads(call["function"]["arguments"])
    }
    for call in message["tool_calls"]
]
```

Adapter cần thay đổi nếu provider trả schema khác. Không lưu `tool_calls`, `arguments` hoặc tên hàm do API tự bọc trực tiếp vào `answer` mà chưa chuyển đổi.

Với public submission này, sản phẩm cần nộp là **dự đoán lời gọi hàm**. Không suy ra rằng phải kết nối và thực thi toàn bộ công cụ ngoài đời. Điểm dự đoán đúng cũng chưa chứng minh agent hoàn thành workflow thực tế.

### 4.3. Quy trình chạy đề xuất

1. Thử một nhóm nhỏ câu thật, gồm cả Knowledge và Agentic, để kiểm tra adapter.
2. Chốt model/version, prompt, temperature và cấu hình reasoning nếu có.
3. Chạy hết câu thuộc phạm vi cần đánh giá; lưu kết quả ngay sau mỗi request.
4. Ghi lỗi riêng, retry có giới hạn với lỗi tạm thời; không thay lỗi API bằng đáp án giả.
5. Kiểm tra độ phủ trước khi nộp; giữ cấu hình đồng nhất khi so sánh nhiều model.

Mã giả cho một lần chạy:

```text
Đọc từng dòng public-test.jsonl
  → xác định track theo domain và cấu trúc câu hỏi
  → bỏ qua Safety nếu chỉ cần điểm public hiện tại
  → gửi question + choices hoặc function tới model/Gateway
  → chuẩn hóa output thành answer
  → kiểm tra nhãn hoặc schema
  → ghi một dòng {id, answer} vào submission.jsonl
  → ghi response gốc, thời gian, token và lỗi vào log riêng
```

## 5. Tạo và kiểm tra submission

File nộp phải là `.jsonl`: mỗi dòng chứa một JSON object gồm `id` và `answer`. Hệ thống ghép theo `id`, nên thứ tự dòng không ảnh hưởng. [3]

Xuất dự đoán thật trong Python:

```python
import json

# row: dòng câu hỏi thật; prediction: output đã chuẩn hóa của model.
with open("submission.jsonl", "a", encoding="utf-8") as file:
    file.write(json.dumps(
        {"id": row["id"], "answer": prediction},
        ensure_ascii=False,
    ) + "\n")
```

Khuyến nghị tạo file mới cho mỗi lần chạy, hoặc kiểm tra `id` đã hoàn thành trước khi tiếp tục; không append lại toàn bộ kết quả gây trùng.

### Kiểm tra nhanh bằng Python

Lưu đoạn sau thành `check_submission.py`, đặt cạnh hai file JSONL. Đây là **kiểm tra format và độ phủ**, không phải bộ chấm đáp án hay validator JSON Schema đầy đủ. Danh sách Safety bên dưới ứng với dữ liệu đã kiểm tra; cần cập nhật khi release đổi.

```python
import json
from pathlib import Path

SAFETY = {"hatespeech", "politics_easy", "politics_advanced"}

def load_jsonl(filename):
    rows = []
    for number, line in enumerate(
        Path(filename).read_text(encoding="utf-8-sig").splitlines(), 1
    ):
        if not line.strip():
            continue
        try:
            row = json.loads(line)
        except json.JSONDecodeError as error:
            raise ValueError(f"{filename}: dong {number}: JSON loi") from error
        if not isinstance(row, dict):
            raise ValueError(f"{filename}: dong {number}: can JSON object")
        rows.append(row)
    return rows

questions = load_jsonl("public-test.jsonl")
ids = [r["id"] for r in questions]
assert len(ids) == len(set(ids)), "Public test co id trung"
by_id = {r["id"]: r for r in questions}
expected = {
    r["id"] for r in questions
    if r["domain"] not in SAFETY
    and (r.get("choices") or r.get("function"))
}

seen = set()
ignored = 0
for row in load_jsonl("submission.jsonl"):
    assert set(row) == {"id", "answer"}, "Moi dong chi gom id va answer"
    item_id = row["id"]
    assert type(item_id) is int, "id can la so nguyen nhu public test"
    assert item_id in by_id, f"id khong ton tai: {item_id}"
    assert item_id not in seen, f"id trung: {item_id}"
    seen.add(item_id)
    question = by_id[item_id]
    answer = row["answer"]
    if item_id not in expected:
        ignored += 1
        continue
    if question.get("function"):
        assert isinstance(answer, list) and answer, f"{item_id}: can mang call"
        functions = {f["name"]: f for f in question["function"]}
        for call in answer:
            assert isinstance(call, dict) and len(call) == 1, item_id
            name, args = next(iter(call.items()))
            assert name in functions, f"{item_id}: ten ham sai"
            assert isinstance(args, dict), f"{item_id}: args can la object"
            schema = functions[name]["parameters"]
            assert set(schema.get("required", [])) <= set(args), item_id
            for key, value in args.items():
                rule = schema.get("properties", {}).get(key, {})
                if "enum" in rule:
                    assert rule["enum"], f"{item_id}: schema co enum rong: {key}"
                    assert value in rule["enum"], f"{item_id}: enum sai: {key}"
    else:
        labels = {chr(65 + i) for i in range(len(question["choices"]))}
        assert isinstance(answer, str) and answer in labels, item_id

missing = expected - seen
print("Can nop:", len(expected))
print("Da co:", len(expected & seen))
print("Bo qua theo pham vi:", ignored)
print("Con thieu:", len(missing))
if missing:
    print("Mot so id thieu:", sorted(missing)[:20])
    raise SystemExit(1)
print("Dat kiem tra format/co ban va do phu; chua xac nhan dap an dung.")
```

Chạy bình thường, không dùng chế độ `python -O` vì script dùng `assert`:

```bash
python check_submission.py
```

Kiểm tra thêm kiểu dữ liệu, cấu trúc lồng nhau và các ràng buộc schema khác trước khi nộp Agentic. Script trên chỉ kiểm tra `required` và `enum` ở cấp tham số ngoài cùng.

## 6. Nộp và đọc kết quả

1. Mở [Gửi kết quả](https://vbench.ai/submission).
2. Chọn `submission.jsonl` bằng nút **Chọn file**.
3. Bấm **Đánh giá ngay**.
4. Lưu điểm, điểm theo domain, số câu đã/chưa trả lời và phiên bản dữ liệu. [3][6]

Hệ thống trả kết quả tổng hợp, không công bố đáp án chi tiết. Muốn tham gia professional test hoặc đưa model lên leaderboard chính thức, liên hệ `vbench-support@vinuni.edu.vn`. [3]

File [sample submission](https://vbench.ai/downloads/sample_submission.jsonl) dùng để xem format. Không coi đáp án trong file mẫu là ground truth.

**Vấn đề dữ liệu đã xác nhận:** ở `id=8450`, tham số bắt buộc `security_seal_code` của hàm `analyze_document_security_features_05126` có `enum: []`. Không có giá trị nào thỏa ràng buộc đó; sample submission điền chuỗi rỗng và cũng không thỏa schema. [4][5] Script kiểm tra sẽ dừng tại điểm này. Cần ghi nhận đây là bất nhất dữ liệu và hỏi bên benchmark cách xử lý; không tự xóa câu, sửa enum hoặc tuyên bố model sai vì lỗi schema này.

## 7. Cách chấm điểm

### 7.1. Theo câu hỏi

- **Knowledge:** đối chiếu nhãn với đáp án chuẩn của release. [2]
- **Agentic:** đối chiếu lời gọi hàm theo quy tắc release; thứ tự key của object được chuẩn hóa. Tài liệu công khai chưa mô tả đủ các trường hợp như đổi thứ tự nhiều call, tham số dư hoặc điểm từng phần; không tự giả định cách xử lý. [2]
- **Safety:** không tham gia điểm public hiện tại. [2]

### 7.2. Theo domain và điểm cuối

Giao diện kết quả hiển thị số câu đúng trên tổng câu mỗi domain và điểm trên thang 100. Có thể diễn giải điểm domain theo tỷ lệ này; thuật toán backend chi tiết chưa được kiểm chứng trong hướng dẫn. [6]

```text
Điểm domain d = 100 × số câu đúng của d / tổng câu được chấm của d

Điểm cuối = tổng điểm các domain đang được chấm / số domain đang được chấm
```

**Trung bình theo domain** là cách tổng hợp website công bố. [3][6]

Ví dụ toán học minh họa, không phải kết quả đo V-Bench:

| Domain | Đúng / tổng | Điểm |
| --- | ---: | ---: |
| X | 90/100 | 90 |
| Y | 5/10 | 50 |

- Trung bình theo domain: `(90 + 50) / 2 = 70`.
- Tỷ lệ đúng gộp toàn bộ câu: `95 / 110 × 100 ≈ 86,36%`.

Hai chỉ số trả lời hai câu hỏi khác nhau. Domain ít câu vẫn có trọng số tương đương domain nhiều câu trong phép trung bình domain. Vì vậy, nên xem cả điểm domain và độ phủ; không chỉ tối ưu nhóm có nhiều câu.

**Câu thiếu không phải câu đúng.** Giao diện có thống kê chưa trả lời; hãy đối chiếu tổng câu của mỗi domain trong kết quả, tránh diễn giải tỷ lệ đúng trên phần đã trả lời thành điểm toàn bộ test. [6]


## 8. Lỗi thường gặp và cách xử lý

| Vấn đề | Cách xử lý đề xuất |
| --- | --- |
| File là một JSON array lớn | Ghi từng object trên một dòng JSONL |
| Câu trắc nghiệm trả cả đoạn giải thích | Chuẩn hóa thành nhãn hợp lệ trước khi ghi |
| Agentic trả một chuỗi chứa JSON | Parse thành list/object thật |
| Hàm hoặc giá trị enum bị dịch sang tiếng Việt | Giữ nguyên tên và giá trị schema |
| `function: []` bị nhận nhầm là Agentic | Kiểm tra danh sách có phần tử |
| Trùng `id` hoặc thiếu dự đoán sau khi retry | Dùng kết quả theo `id`, kiểm tra độ phủ |
| API timeout/rate limit | Log lỗi, retry giới hạn, lưu checkpoint |
| Điểm thấp dù phần đã trả lời khá tốt | Xem câu thiếu và domain yếu |
| Không có câu thuộc phạm vi chấm | Kiểm tra `id`, JSONL và track hiện hành |
| Dịch vụ chấm không hoạt động | Giữ file, thử lại sau hoặc liên hệ hỗ trợ |

**Lưu ý từ mã giao diện hiện tại:** dòng JSON lỗi có thể bị bỏ qua và `id` trùng có thể bị ghi đè. File được nhận không đồng nghĩa tất cả dòng đã được chấm. Đây là quan sát triển khai frontend, không phải cam kết ổn định của API. [6]


## Nguồn và khả năng tái lập

1. [Trang chủ V-Bench — CAIR, VinUniversity](https://vbench.ai/).
2. [Dataset, Submission & Scoring Spec và release](https://vbench.ai/dataset).
3. [Public submission](https://vbench.ai/submission).
4. [Public test JSONL](https://vbench.ai/downloads/public-test.jsonl) — đã tải và thống kê trực tiếp.
5. [Sample submission JSONL](https://vbench.ai/downloads/sample_submission.jsonl) — liên kết được trang Submission công bố.
6. [Bundle frontend được trang chủ tham chiếu tại thời điểm kiểm tra](https://vbench.ai/assets/index-A8iCig5z.js) — dùng kiểm tra bộ lọc phạm vi, parser và các trường hiển thị kết quả. Không có quyền truy cập mã chấm backend.


