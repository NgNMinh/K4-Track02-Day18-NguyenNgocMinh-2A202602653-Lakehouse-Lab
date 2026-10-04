# Kết quả thực thi Lakehouse Lab

Đường lightweight, Python 3.14.4 trên Windows. Các số dưới đây là output của lần chạy này; chi tiết từng cell được giữ trong tám notebook ở `submission/notebooks/`.

## NB1 — Delta basics

- Bảng đọc lại được; `_delta_log` có ít nhất hai commit JSON (ghi đầu và evolution). Lần ghi `age="thirty"` bị Delta chặn: `Cannot cast string 'thirty' to value of Int64 type`.
- `schema_mode="merge"` thêm `tier`; DuckDB trả hai nhóm: `premium` (1) và `NULL` (3).
- Enforcement từ chối dữ liệu không tương thích với schema hiện hữu; evolution thay đổi schema một cách có chủ ý. Cần opt-in để tránh dữ liệu đầu vào bất ngờ tự sửa hợp đồng schema. Transaction log lưu commit có thứ tự, metadata/schema và file add/remove cùng operation metrics, làm bằng chứng cho từng lần ghi.

## NB2 — Compaction và Z-order

- File: **200 → 55**; truy vấn median **279.6 → 39.8 ms**, speedup **7.0×**; pruning **55×** theo số file sau tối ưu so với trước.
- Compaction giảm overhead của nhiều file nhỏ. Z-order sắp xếp/co-locate giá trị để min/max mỗi file hẹp hơn, giúp skip file. Một file duy nhất không còn nhiều file để loại khỏi scan. Thời gian biến động theo cache, tải CPU/đĩa, hệ điều hành và nhiễu đo; pruning ratio thường ổn định hơn wall-clock.

## NB3 — MERGE, time travel, RESTORE

- MERGE **100,000** hàng: **50,000 update + 50,000 insert**, chạy 0.24 s. History có 5 version; RESTORE về v2 tạo v4 và score<0 còn **0**.
- Time travel chỉ đọc snapshot cũ, còn RESTORE đổi trạng thái hiện tại. Restore là transaction mới để lịch sử giữ được audit trail và có thể truy nguyên cả thao tác rollback, thay vì viết lại/xóa quá khứ.

## NB4 — Bronze → Silver → Gold

- Bronze **200,000**, Silver **190,052** (dedup bỏ **9,948**), Gold **24 hàng = 8 ngày × 3 model**. Đối chiếu trực tiếp: p50≤p95, cost_usd dương, error_rate nằm trong [0,1] cho tất cả hàng.
- Dedup ở Silver loại bản ghi trùng theo request ID để các phép tổng hợp không đếm lặp. Dashboard đọc Gold vì dữ liệu đã được chuẩn hóa và tổng hợp theo chiều truy vấn. Query tính error rate là tỷ lệ status khác `ok`; chi phí dùng tổng token nhân giá minh họa theo model, nên hợp lý cho dữ liệu lab nhưng không phải bảng giá production.

## NB5 — Iceberg và catalog

- Scan toàn bảng **10 file**, filter trên `ts` **1 file**, pruning **10×**. Metadata/data ratio của sample nhỏ: **287.1%**. Rename `latency_ms` thành `latency_millis` giữ field ID **4**; hai spec ID cùng tồn tại (**1, 2**) và **5,500** hàng vẫn đọc được.
- Hidden partitioning ánh xạ filter trên cột nguồn `ts` sang partition `day(ts)` trong planning. Field ID giữ danh tính cột dù tên đổi. Partition evolution tạo spec mới cho file append mới; giữ spec cũ cho file hiện hữu tránh rewrite toàn bộ dữ liệu ngay lập tức.

## NB6 — Maintenance

- Compaction **200 → 11 file** (**18×** ít hơn); clustering point query mở **1/10 file**, skip **90%**. Delta vacuum thu hồi **16.1 MB**; sweep tìm và xóa **3 orphan** (**21.2 KB**). Checkpoint `00000000000000000099.checkpoint.parquet` và `_last_checkpoint` có mặt.
- Iceberg expiry giảm snapshots **20 → 3** nhưng manifest lists trên đĩa ban đầu vẫn **40**; sweep xóa **17** file không còn tham chiếu (**37.0 KB**). Orphan chưa từng commit không có tombstone trong Delta log nên vacuum đường này không nhận ra. Snapshot expiry trong PyIceberg chỉ đổi metadata tham chiếu; xóa vật lý cần orphan sweep. Retention quá ngắn có thể khiến reader đang dùng snapshot cũ mất file cần đọc.

## NB7 — Multimodal và vectors

- Random read inline đọc row group **12.5 MB** cho frame **64 KB**, amplification **200×**. Int8 nhỏ hơn **5.8×** trên Parquet; recall@10 **0.904**, topic fidelity **1.000**.
- Quantization tiết kiệm dung lượng nhưng có thể đổi thứ hạng/ID gần nhau; recall đo trùng document ID, còn topic fidelity đo mức đúng chủ đề. Sau delete, bảng có **0 hit** nhưng external index cũ còn **8 hit**; CDF phát **8 delete events** để consumer evict các ID này.

## NB8 — Agents, version pin và provenance

- Silver có hai partition `agent_version` (policy-v2/v3); Gold có hai policy. Training pin version **0** với **1,578 bước**; sau append, replay version 0 vẫn đúng **1,578**. Năm lượt `list_tables` chỉ đọc catalog **1 lần**. Destructive call trả `input_required`; task kết thúc `completed` với 300 rows. Provenance có bốn bucket phân loại và `UNCLASSIFIED` bị loại khỏi tập trainable. Subject `user_007`: **8 → 0 dòng** ở bảng hiện tại.
- Pin version xác định đầu vào dữ liệu của run dù bảng tiếp tục append. Xóa current version không xóa snapshot cũ. Mô phỏng chưa phải production guardrail: không có server/auth boundary, cờ confirmed do caller cấp, task chạy giả lập, replay chỉ so số bước, provenance là nhãn minh họa chứ không chứng minh quyền sử dụng.
