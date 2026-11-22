# Silver Transfer

Từ thư mục workspace, cài toàn bộ dependency Python và CLI bằng `python -m pip install -r .\requirements.txt` trong venv Python 3.11 x64. File tổng lấy dependency của Silver Transfer từ `pyproject.toml` và thêm CairoSVG cho các script SVG cũ. ODA File Converter, 7-Zip CLI và Cairo native cho script cũ cài riêng.

Một CLI cho các bước điền Silver JSONL, phục hồi định dạng khó, tạo index, QA và xuất Parquet. Nhận raw và template qua tham số. Kết quả của mỗi lượt nằm trong một thư mục mới.

## Chạy trên Windows

Từ thư mục workspace. Các lệnh dưới dùng Python 3.11 đã có trong workspace này. Trên máy khác, thay dòng đầu bằng `$python = 'python'` nếu Python đã có trên PATH.

```powershell
$python = '.\.uv-python\cpython-3.11.15-windows-x86_64-none\python.exe'
& $python -m venv .\silver_transfer_tổng\.venv-transfer
.\silver_transfer_tổng\.venv-transfer\Scripts\python.exe -m pip install '.\silver_transfer_tổng[recovery]'
.\silver_transfer_tổng\.venv-transfer\Scripts\silver-transfer.exe run `
  --raw-root .\03_PROJECTS\03_PROJECTS `
  --template .\silver-template-a0b9b530-55b4-4cdc-8b68-ca92d4491001.jsonl `
  --out .\outputs\transfer-test-01 `
  --recover --index
```

Có thể dùng `python -m silver_transfer` sau khi cài. Chạy mặc định tạo `silver_filled.jsonl` và `handoff/`, có xác minh SHA-256 khi template cung cấp hash hợp lệ. `--recover` thêm `unsupported/` và `exotic/`, `--index` thêm `search/`. Khi bật recovery hoặc converter, `silver_base.jsonl` giữ đầu ra filler ban đầu. `silver_filled.jsonl` nhận thêm các đoạn chữ trích được và tham chiếu metadata, preview, DXF trong `derived_extraction` trước khi tạo index. Trạng thái gốc trong `status` mô tả filler và trạng thái xử lý bổ sung nằm trong `derived_extraction`. Với extra `large-images`, ảnh vượt ngưỡng Pillow dùng libvips để lấy kích thước và tạo thumbnail cho embedding ảnh. Parser hiện chưa chạy OCR.

| Lệnh | Chức năng |
|---|---|
| `fill`, `finalize` | Điền template bằng raw, QA và xuất Parquet |
| `recover`, `recover-exotic` | Phân loại file chưa hỗ trợ, đọc AI/SVG/JFIF và kiểm kê archive. Dùng `--expand-zip` riêng khi cần trích file con |
| `auto-convert` | Thử SVG → PNG và DWG → DXF từ `unsupported_enrichment.jsonl`. DWG cần ODA và có thể chỉ định `--oda-path` |
| `index build`, `index search`, `audit-index` | Xây, tìm kiếm và kiểm tra keyword/image index |
| `extend-images` | Bổ sung ảnh lớn vào bản sao của image index. Chỉ định file bằng `--names` |
| `index-previews build/search` | Tạo và truy vấn index SigLIP2 từ preview manifest |
| `text-semantic build/search` | Tạo và truy vấn text embeddings từ `search.sqlite` |
| `bronze-demo ingest/search`, `legacy-fill` | Hai luồng thử cũ cho đối chiếu |

Dùng `silver-transfer <lệnh> --help` để xem tham số. `run --auto-convert` bật converter thử nghiệm. Native tools như 7-Zip và ODA cài riêng. Cài thêm extras `vision` cho SigLIP2, `semantic` cho text embedding, `large-images` cho libvips và `demo` cho Bronze demo. Encoder có thể tải model ở lần chạy đầu.

SVG thiếu riêng thẻ đóng `</svg>` được thử phục hồi trong bộ nhớ. `auto-convert` ghi `RECOVERED` khi render thành công và lưu cách phục hồi trong metadata. Preview đã phục hồi vẫn cần đối chiếu nếu dùng để bàn giao. Raw SVG giữ nguyên.

Silver chia văn bản đã trích thành đoạn tối đa 1.000 ký tự, ưu tiên ranh giới đoạn, dòng, câu rồi đến khoảng trắng. Các đoạn giữ số trang hoặc slide và nhãn mục của phần trích xuất. `chunk_no` đánh lại liên tục trong từng record. Để text search dùng cách chia mới, tạo JSONL và index mới. Index cũ vẫn đọc được.

### Tạo lại Silver và text index, dùng tiếp vector ảnh

```powershell
$silver = '.\silver_transfer_tổng\.venv-transfer\Scripts\silver-transfer.exe'
& $silver run `
  --template .\silver-template-a0b9b530-55b4-4cdc-8b68-ca92d4491001.jsonl `
  --raw-root .\03_PROJECTS\03_PROJECTS `
  --out .\outputs\silver-chunked-full-02 `
  --recover --auto-convert `
  --oda-path 'C:\Program Files\ODA\ODAFileConverter 27.10.0\ODAFileConverter.exe' `
  --semantic-text --reuse-image-index .\outputs\transfer-full-vector-01\search `
  --device cpu
```

Lệnh tạo `silver_filled.jsonl`, `search/search.sqlite` và `semantic_text/text_vectors.npy` từ đầu ra cùng lượt. Vector ảnh được sao chép sau khi đối chiếu ID và SHA-256 của nguồn. Mức phủ ảnh giữ theo index cũ. Để tạo lại vector ảnh, thay `--reuse-image-index <đường-dẫn>` bằng `--images`. Chọn một thư mục `--out` mới cho mỗi lượt.

## OCR tiếng Việt và tiếng Anh

Từ workspace, thêm dependency local vào môi trường Silver Transfer hiện tại:

```powershell
uv pip install --python '.\silver_transfer_tổng\.venv-transfer\Scripts\python.exe' '.\silver_transfer_tổng[ocr-local]'
```

Thêm `--ocr paddle --semantic-text` vào lệnh `silver-transfer run` và chọn thư mục `--out` mới. OCR bật luôn keyword index. Text semantic dùng multilingual E5 để tìm theo ý nghĩa bằng tiếng Việt hoặc tiếng Anh, kể cả khi ngôn ngữ truy vấn khác ngôn ngữ tài liệu. Keyword tìm theo chữ xuất hiện trong tài liệu. Lần đầu chạy PaddleOCR và text semantic có thể cần tải model.

Các chế độ OCR:

- `--ocr paddle` chạy local với PP-OCRv5 server detector và Latin recognizer.
- `--ocr google` dùng Cloud Vision `DOCUMENT_TEXT_DETECTION`. Cài extra `ocr-cloud`, bật Vision API trong dự án Google Cloud và cấu hình Application Default Credentials.
- `--ocr fallback` dùng Paddle trước và gửi crop sang Google khi crop đó không có kết quả local đạt ngưỡng. Cài cả hai extras và chọn chế độ này khi dữ liệu được phép gửi lên Google Cloud. Crop gửi có thể tính phí.

`run` và `fill` áp dụng OCR cho ảnh (khung đầu với ảnh nhiều khung) và trang PDF không có lớp chữ trích được. OCR chia vùng trước khi đọc chữ nhỏ và giữ model, confidence, bbox cùng số trang trong JSONL. `--ocr-min-confidence` mặc định là `0.85`. Chữ đạt ngưỡng có thể tìm kiếm, nhưng cần rà lại trước khi bàn giao.

Log `OCR ảnh x/tổng` đếm số record ảnh đã xử lý trên tổng record ảnh trong template, kèm trạng thái và số đoạn OCR của ảnh đó. Bộ đếm gồm cả ảnh không có chữ hoặc bị bỏ qua và không tính trang PDF. JSONL được ghi ra ngay sau mỗi record khi bật OCR.

`--ocr-device gpu` chạy PaddleOCR trên GPU NVIDIA số 0. Cờ này dùng được với `--ocr paddle` và `--ocr fallback`. Môi trường GPU cần `paddlepaddle-gpu` phù hợp với driver và hệ điều hành. Dùng venv riêng cho GPU để tránh hai gói PaddlePaddle cùng ghi vào module `paddle`. Giữ `--device cpu` nếu chỉ muốn OCR chạy GPU, vì `--device` điều khiển embedding. CLI kiểm tra phép tính CUDA lúc khởi tạo và báo lỗi ngay nếu GPU chưa dùng được.

```powershell
$silver = '.\silver_transfer_tổng\.venv-transfer\Scripts\silver-transfer.exe'
$out = '.\outputs\silver-ocr-01'
& $silver run `
  --template .\silver-template-a0b9b530-55b4-4cdc-8b68-ca92d4491001.jsonl `
  --raw-root .\03_PROJECTS\03_PROJECTS `
  --out $out --ocr paddle --semantic-text `
  --reuse-image-index .\outputs\silver-chunked-full-01\search
& $silver index search --index "$out\search" --mode keyword --query 'khối nhà chính'
& $silver text-semantic search --index "$out\semantic_text" --query 'main building'
```

## Test bằng dữ liệu tạm

Sau khi cài với extra `recovery`, chạy từ thư mục này:

```powershell
python -m unittest discover -s tests -v
```

Test tạo raw và template giả lập trong thư mục tạm, kiểm tra JSONL, hash, Parquet, keyword index, preview và bảo toàn file nguồn.
