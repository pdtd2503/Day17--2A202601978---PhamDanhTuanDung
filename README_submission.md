# Báo cáo Nộp bài Lab 17: Multi-Memory Agent với Zep

## 1. Trả lời câu hỏi trọng tâm (Section 5.2)

### Câu 1: Layer quan trọng nhất trong bộ test & Case minh họa
Trong bộ test 11 ca thực hành (E01-E11), **Long-term / Declarative Memory** là tầng quan trọng nhất (chiếm 4 case độc lập E02, E03, E08, E09 và tham gia trong E07 mixed). Layer này giải quyết các thách thức then chốt:
- Duy trì preferences cá nhân xuyên suốt các session độc lập (case `E02`: Minh ưu tiên `Python`).
- Ghi nhớ open loops và deadlines mà không bị mất dấu (`E03`: `benchmark report` trước `16:00`).
- Cập nhật thông tin mới theo phạm vi dự án (`E08`: `BLUEBIRD-42` dùng `TypeScript`/`NestJS`).
- Đảm bảo tính cô lập dữ liệu người dùng (`E09`: Lan chỉ truy xuất `LOTUS-88` / `Java` / `Spring Boot` và tuyệt đối không rò rỉ `ORCHID-27` của Minh).

### Câu 2: Trade-off giữa Zep Context Block vs Tự xây dựng Redis + Qdrant
- **Zep Cloud (Managed Memory)**: Tự động trích xuất graph, facts, episodes, quan hệ thực thể, validity time ranges, tự sinh Context Block dựa trên relevance; tuy nhiên độ trễ phụ thuộc vào network/cloud và ingestion xử lý bất đồng bộ (cần polling).
- **Tự build Redis + Qdrant (Self-hosted)**: Chủ động độ trễ cực thấp (<5ms), kiểm soát schema và dữ liệu 100%, chi phí hạ tầng thấp khi scale local; nhưng tốn công xây dựng pipeline phân đoạn, trích xuất thực thể, deduplication, conflict resolution và lifecycle maintenance.

### Câu 3: Guardrail chống Memory Poisoning
1. **Kiểm duyệt & Xác thực đầu vào (Input Guard)**: Áp dụng PII redaction (`minimize_pii`), phát hiện Prompt Injection trước khi đưa vào bộ nhớ.
2. **Phân quyền & Nguồn tin cậy (Provenance)**: Chỉ ghi nhận facts từ nguồn được xác thực hoặc khi người dùng đã chấp thuận (`require_memory_consent`), lưu kèm metadata nguồn gốc.
3. **Bảo trì & Thu hồi định kỳ (Maintenance & Right to be Forgotten)**: Chạy Heartbeat kiểm tra mâu thuẫn, áp dụng cơ chế xóa/quên triệt để (`forget`) khi phát hiện fact độc hại.

---

## 2. Phân tích kết quả Benchmark (Section 5.2)

1. **Layer có hit rate thấp nhất**: Ở baseline `no_memory`, toàn bộ các tầng lưu trữ dài hạn (`long_term`, `episodic`, `semantic`, `mixed`) có hit rate 0% do không có cơ chế recall xuyên thread. Ở bản `student`, toàn bộ các tầng đều đạt 100% (11/11 case PASS).
2. **Query retrieve nhiều token nhất**: Các query tầng episodic (`E04`) và semantic (`E06`, `E11`) retrieve nhiều token nhất do chứa tài liệu quy tắc và trajectory debug chi tiết.
3. **Case Mixed (E07)**: Yêu cầu kết hợp giữa **Long-term Memory** (lấy preference ngôn ngữ `Python` của Minh) và **Semantic Memory** (lấy quy tắc `Idempotency-Key` từ domain KB). Hai marker bắt buộc xuất hiện là `Python` và `Idempotency-Key`.
4. **Token reduction vs Hit rate**: Token reduction đo tỷ lệ thu gọn context so với ném toàn bộ transcript. `no_memory` có reduction cao nhưng hit rate = 0 do thiếu hoàn toàn context cần thiết. Hệ thống tốt cần cân đối giữa tiết kiệm token và độ chính xác retrieval.

---

## 3. Phân tích Recency (E08) & Compaction (E10)

- **E08 (Recency & Scoped Update)**: Minh cập nhật rằng dự án công ty `BLUEBIRD-42` bắt buộc dùng `TypeScript`/`NestJS`. Zep ghi nhận fact mới kèm temporal validity range, ưu tiên thông tin mới nhất đúng theo scope dự án mà không ghi đè mất preference `Python` cho dự án cá nhân `ORCHID-27`.
- **E10 (Compaction)**: Khi số lượng message vượt ngưỡng trượt (`max_recent_messages`), cơ chế sliding window nén các turn cũ nhưng vẫn trích xuất và bảo toàn ràng buộc bất biến `REVIEW-DEADLINE-1600` (`Friday`, `16:00`) vào durable notes, đảm bảo agent không quên deadline dù tin nhắn gốc đã bị evict khỏi buffer.
