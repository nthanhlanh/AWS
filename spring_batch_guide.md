# Spring Batch Guide

## 1. Concept Batch
- **Batch** = xử lý theo lô, gom nhiều dữ liệu/việc cần làm lại và xử lý cùng lúc.
- Thường chạy theo lịch (cron, schedule).
- Input: dữ liệu lớn (DB, file, API).
- Output: kết quả (DB, file, message...).

👉 Chỉ là **khái niệm** chứ không phải code thực thi.

---

## 2. Job / Step / Chunk
### Job
- Tiến trình batch lớn (ví dụ: Job tính lãi cuối ngày).
- Có thể có nhiều Step.

### Step
- Đơn vị xử lý nhỏ hơn trong Job.
- Gồm 2 loại:
  - **Tasklet**: xử lý 1 tác vụ nhỏ.
  - **Chunk-oriented**: chuyên để xử lý dữ liệu lớn với Reader/Processor/Writer.

### Chunk
- Đọc `n` bản ghi → xử lý → ghi → commit transaction.
- Giúp tối ưu hiệu năng và rollback dễ hơn.

---

## 3. Reader / Processor / Writer
- **Reader**: đọc dữ liệu (DB, file, API...).
- **Processor**: xử lý trung gian (validate, tính toán, transform).
- **Writer**: ghi dữ liệu (DB, file, API, MQ...).

👉 Luồng xử lý trong Step (chunk-oriented):
```
Reader → Processor → Writer
```

---

## 4. Job Flow
- Định nghĩa **dòng chảy** các Step trong Job.

### Các kiểu Job Flow:
- **Sequential**: Step1 → Step2 → Step3.
- **Conditional**: nếu Step1 FAILED → Step2; nếu COMPLETED → Step3.
- **Parallel**: chạy nhiều Flow song song.

👉 Giúp xây dựng "kịch bản" điều khiển Job.

---

## 5. Error Handling
- Không dùng `@ControllerAdvice` như REST API.
- Spring Batch có các cơ chế riêng:

### Cách xử lý lỗi:
- **Skip**: bỏ qua bản ghi lỗi (`skipLimit`, `skip`).
- **Retry**: thử lại nếu lỗi tạm thời (`retry`, `retryLimit`).
- **Flow**: nếu Step FAIL → chuyển sang Step xử lý lỗi.
- **Listener**: theo dõi Job/Step/Chunk để log, gửi cảnh báo.

---

## 6. Parallel / Partition
### Parallel Step
- Chạy nhiều Step/Flow độc lập song song bằng `split()`.

### Partition Step
- Chia dữ liệu lớn thành nhiều partition, mỗi partition xử lý song song bằng worker step.

👉 Parallel = nhiều công việc khác nhau chạy cùng lúc.  
👉 Partition = một công việc lớn chia nhỏ dữ liệu để nhiều thread xử lý song song.

---

## 7. Scheduler & Deploy
### Scheduler
- Dùng để lập lịch chạy Job batch định kỳ.
- Cách phổ biến:
  - **Spring @Scheduled**
  - **Quartz Scheduler**
  - **Cron job (Linux) / Jenkins pipeline / Kubernetes CronJob**

### Deploy
- Đưa batch app ra môi trường thực tế.
- Cách triển khai:
  - Build JAR (`java -jar batch.jar`).
  - Deploy Docker + Kubernetes CronJob.
  - CI/CD pipeline.

👉 Scheduler = hẹn giờ chạy.  
👉 Deploy = đưa batch lên môi trường để chạy.

---

# ✅ Tóm tắt
1. **Concept batch**: chỉ là khái niệm xử lý theo lô.
2. **Job/Step/Chunk**: cấu trúc cơ bản trong Spring Batch.
3. **Reader/Processor/Writer**: trái tim của Step chunk.
4. **Job Flow**: điều khiển trình tự Step.
5. **Error Handling**: skip, retry, flow, listener.
6. **Parallel/Partition**: song song & chia nhỏ dữ liệu.
7. **Scheduler & Deploy**: hẹn giờ và triển khai batch.

---

## Nhiệm vụ chính của Spring Batch

- **Quản lý batch job**: biết job nào chạy, chạy lúc nào, trạng thái ra sao.
- **Tự sinh DB schema** (nếu bạn cho phép) để lưu lại trạng thái job/step/execution.
- **Quản lý transaction**: rollback, commit theo chunk.
- **Hỗ trợ error handling**: skip, retry, restart.
- **Hỗ trợ parallel/partition** để xử lý dữ liệu lớn nhanh hơn.

