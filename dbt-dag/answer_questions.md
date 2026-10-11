# Một số câu hỏi về Data Engineering

## 1. ETL vs ELT khác gì nhau?

- 2 cái này sẽ khác nhau ở các bước thực hiện.
- **ETL:** Extract → Transform → Load.
- **ELT:** Extract → Load → Transform.

→ ELT sẽ thường tiện lợi hơn do khác biệt ở bước thứ 2, việc load trước data lên datawarehouse sẽ giúp việc transform data trở lên nhanh chóng hợp, tối ưu thời gian phân tích và chi phí.

## 2. Vai trò AE sẽ là gì trên thực tế?

- AE sẽ là một người làm việc với raw data, từ những bước như nạp data, transform data, tạo các bài test, cài đặt workflow để tự động làm việc → Và làm thành 1 dataset hoàn chỉnh để đưa cho bên DA.

## 3. Single source of truth là gì?

- Trong mọi dữ liệu hoặc thông tin nào đó sẽ chỉ có duy nhất một nguồn uy tín và chính xác nhất để mọi người tra cứu.

## 4. Medallion architecture là gì?

- Đây sẽ là 3 tầng dữ liệu trong một mô hình thiết kế:

  - **Bronze:** Đây là tầng raw data, là những dữ liệu đc import từ nhiều nguồn khác nhau.
  - **Silver:** Đây là tầng clean data, những data đã được transform, làm sạch, join lại với nhau.
  - **Gold:** Đây giống như final data, là tầng để phục vụ cho các nghiệp vụ của doanh nghiệp.

## 5. Docker là gì, tại sao dùng Docker? Airflow là gì, nhiệm vụ ra sao trong project?

- Docker là một công cụ dùng để chạy môi trường trong 1 project cụ thể.
- Airflow được dùng trong bước orchestration, đây là một công cụ điều phối quy trình dữ liệu, giúp chúng ta set up lịch trình theo một workflow cụ thể (VD: Vào 1 khoảng tgian nhất định trong ngày, data sẽ tự động nạp → dbt run sẽ chạy để update các bảng → Sau đó sẽ test xem các data mới đó có hợp lệ hay ko).

→ Docker sẽ là công cụ để chạy trong môi trường Airflow, còn Airflow sẽ là công cụ điều phối các quy trình đó.

## 6. OLTP vs OLAP? Có những đại diện nào cho từng loại và Snowflake thuộc loại nào?

- **OLTP (Online Transaction Processing):** Là quy trình xử lý những giao dịch trực tuyến, phục vụ cho các giao dịch trực tuyến, dùng để quản lý và thực hiện những giao dịch hằng ngày của tổ chức, doanh nghiệp - Như việc các ngân hàng sẽ thường dùng hệ thống này do cần phải cập nhật dữ liệu về cách giao dịch khách hàng 1 cách nhanh chóng và chính xác nhất. (Những dữ liệu chi tiết và cập nhật thường xuyên). VD: PostgreSQL, MySQL, SQL Sever.
- **OLAP (Online Analytical Processing):** Là hệ thống dùng cho việc phân tích trực tuyến, được tạo ra để hỗ trợ việc phân tích chiều sâu và đa chiều của dữ liệu - Như việc tính các bài toán doanh thu của 1 doanh nghiệp. VD: Snowflake, Databrick, Bigquery.
