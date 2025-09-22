# PostgreSQL Query Plan – Cách Planner và Executor hoạt động


## Ghi chú: Khi nào PostgreSQL dùng index?

- ✅ **Dùng index khi**:
    - Lọc ra **ít dòng** (thường < 10–20% tổng bảng).
    - Index **bao phủ đủ cột** (index-only scan).
    - Query có **ORDER BY / LIMIT** trùng thứ tự index.

- ❌ **Không dùng index khi**:
    - Lọc ra **nhiều dòng** (>20% → quét tuần tự nhanh hơn).
    - Statistics sai → planner ước lượng sai.
    - Điều kiện không match index:
        - Dùng hàm/ép kiểu trên cột (`lower(col)` mà không có index chức năng).
        - LIKE với wildcard đầu (`'%abc'`).
    - Index quá to hoặc nhiều cấp → cost cao hơn seq scan.

## 1. Các bước chính khi PostgreSQL xử lý một query
1. **Parse**  
   - Chuyển SQL → cây cú pháp.

2. **Rewrite**  
   - Áp dụng rule, view, security policy…

3. **Plan/Optimize (Query Planner)**  
   - PostgreSQL dựa vào **statistics** (được cập nhật bằng `ANALYZE`) để ước lượng:
     - Bao nhiêu row thỏa `WHERE`.
     - Chi phí I/O đọc index vs đọc tuần tự heap.
     - Bộ nhớ khả dụng cho sort/hash.
   - Planner so sánh chi phí → chọn **execution plan tối ưu**.

4. **Execute (Executor)**  
   - Thực hiện theo plan: đọc dữ liệu từ disk (heap/index), đưa vào RAM, sort, aggregate, limit…

---

## 2. Planner chọn Index hay Seq Scan dựa vào đâu?
### Các yếu tố:
- **Selectivity (tỷ lệ row khớp)**  
  - Nếu chỉ 0.1% row match → index scan gần như chắc chắn.  
  - Nếu 30% row match → seq scan có thể rẻ hơn.

- **Chi phí I/O**  
  - Index Scan: nhiều random I/O → đắt nếu số row nhiều.  
  - Seq Scan: đọc tuần tự → nhanh nếu phải quét >10–15% bảng.

- **Cấu hình cost parameters**  
  - `seq_page_cost` (mặc định 1.0).  
  - `random_page_cost` (mặc định 4.0, nghĩa là random đắt hơn seq).  

- **Index type**  
  - B-Tree phù hợp `=` hoặc range.  
  - GIN/GiST cho full-text, jsonb, trgm.  

👉 Planner sẽ tính **cost (startup + total)** cho từng phương án, rồi chọn plan có cost nhỏ nhất.

---

## 3. Dòng chảy khi query chạy (Executor)
Ví dụ query:

```sql
SELECT *
FROM policyholders
WHERE name_full LIKE '%保険1%'
ORDER BY name_full
LIMIT 200;
```
## 4. Ảnh hưởng của STATISTICS đến Join Type

- PostgreSQL dựa vào **STATISTICS** (thống kê cột, phân phối giá trị) để ước lượng số row thỏa mãn điều kiện.
- Nếu STATISTICS không chính xác (ví dụ, dữ liệu phân bố lệch, hoặc chưa ANALYZE sau khi insert/update nhiều), planner có thể **dự đoán số row thấp hơn thực tế**.
- Khi đó, planner có xu hướng chọn **Nested Loop Join** (tốt cho ít row) thay vì **Hash Join** (tốt cho nhiều row).
- Nếu thực tế số row lớn, Nested Loop Join sẽ rất chậm so với Hash Join.

**Ví dụ:**
- Điều kiện search trả về nhiều row hơn dự đoán → Nested Loop Join chạy lâu.
- Kiểm tra bằng `EXPLAIN ANALYZE` sẽ thấy `actual rows` lớn hơn nhiều so với `rows` planner dự đoán.

**Giải pháp:**
- Chạy `ANALYZE` thường xuyên để cập nhật STATISTICS.
- Tăng `default_statistics_target` cho các cột có phân phối giá trị lệch.
- Xem xét dùng `SET enable_nestloop = off;` để buộc planner chọn Hash Join (chỉ dùng để kiểm tra).

---
**Ví dụ tăng STATISTICS cho cột có phân phối lệch:**

```sql
ALTER TABLE claim.claimant_consumers 
  ALTER COLUMN yomi SET STATISTICS 1000;

ANALYZE claim.claimant_consumers;
```

## 5. Tăng hiệu năng với `work_mem` và Parallel Query

- **`work_mem`**: Dung lượng RAM cho mỗi thao tác sort, hash, aggregate. Nếu query có `ORDER BY`, tăng `work_mem` giúp sort trên RAM thay vì ghi ra disk.
    - Ví dụ: Tăng tạm thời cho session
      ```sql
      SET work_mem = '128MB';
      ```
    - Chú ý: Mỗi sort/hash dùng riêng, nên tổng RAM tiêu thụ có thể lớn nếu nhiều query chạy song song.

- **Parallel Query**: PostgreSQL có thể chạy song song nhiều worker cho các thao tác scan, join, aggregate.
    - Các tham số cần quan tâm:
        - `max_parallel_workers_per_gather`: Số worker tối đa cho mỗi query.
        - `max_parallel_workers`: Tổng số worker cho toàn hệ thống.
        - `parallel_setup_cost`, `parallel_tuple_cost`: Điều chỉnh ngưỡng planner chọn parallel.

    - Ví dụ: Tăng số worker cho session
      ```sql
      SET max_parallel_workers_per_gather = 4;
      ```

- **Lưu ý**: Chỉ hiệu quả với query lớn, bảng lớn, và hardware đủ RAM/CPU.

---


