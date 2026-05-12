# 📊 Thiết kế Cơ sở dữ liệu - Hệ thống Quản lý Cầm đồ

Phần này mô tả cấu trúc cơ sở dữ liệu cốt lõi, được thiết kế để xử lý các nghiệp vụ phức tạp của hệ thống cầm đồ như: quản lý nhiều tài sản trên một hợp đồng, tính lãi đơn/lãi kép theo hạn mức thời gian, trả nợ từng phần và quản lý trạng thái thanh lý tài sản.

## 1. Sơ đồ Thực thể Liên kết (ERD)

<img width="1662" height="1092" alt="BT3 SQL" src="https://github.com/user-attachments/assets/bd8cc0e4-92b4-48de-868a-960415e8874b" />

---

## 2. Cấu trúc các bảng dữ liệu (Tables)

<img width="1686" height="893" alt="Screenshot 2026-05-12 112309" src="https://github.com/user-attachments/assets/5da0e1a9-0f56-48e0-bc8e-4b9db1a527d8" />


### 👥 Nhóm thông tin người dùng & Phân quyền
* **`KHACHHANG` (Khách hàng):** Lưu trữ thông tin cá nhân của người đi vay (Họ tên, CCCD, SĐT, Địa chỉ). Một khách hàng có thể tạo nhiều hợp đồng cầm cố khác nhau.
* **`NHANVIEN` (Nhân viên):** Lưu thông tin định danh của nhân viên phụ trách tư vấn và làm hợp đồng.
* **`TAIKHOAN` (Tài khoản):** Liên kết 1-1 với Nhân viên, quản lý thông tin đăng nhập và trạng thái hoạt động của nhân viên trên hệ thống.

### 📄 Nhóm quản lý Hợp đồng & Tài sản (Nghiệp vụ Vay)
* **`HOPDONG` (Hợp đồng):** Bảng trung tâm của hệ thống, ghi nhận thông tin tổng quan của một lần giải ngân.
  * **Trường quan trọng:** `TienVayGoc` (số tiền giải ngân), `Deadline1` (Mốc thời gian bắt đầu tính lãi kép và ghi nhận nợ xấu), `Deadline2` (Mốc thời gian cho phép thanh lý tài sản).
  * **Trạng thái:** Quản lý vòng đời hợp đồng (Đang vay, Đang trả góp, Quá hạn, Đã thanh toán, Đã thanh lý).
* **`TAISAN` (Tài sản cầm cố):** Quản lý chi tiết từng món đồ khách hàng mang cầm. Thiết kế quan hệ 1-Nhiều với bảng `HOPDONG` giúp một hợp đồng có thể gộp chung nhiều tài sản.
  * **Trường quan trọng:** `DinhGia` (giá trị thực của tài sản, dùng để đối chiếu khi khách muốn chuộc lại một phần đồ), `IsSold` (Cờ đánh dấu tài sản đã bị bán thanh lý hay chưa).
* **`LOAITAISAN` (Danh mục tài sản):** Phân loại tài sản (Ví dụ: Xe máy, Điện thoại, Laptop) để dễ dàng thống kê và định giá.

### 💰 Nhóm quản lý Thu chi & Lịch sử biến động (Nghiệp vụ Trả/Log)
* **`KEHOACHTRAGOP` (Kế hoạch trả):** Lưu trữ các kỳ hạn thanh toán hoặc các "Giao dịch dự kiến" (Transaction). Bảng này là cơ sở để hệ thống tính toán số tiền gốc/lãi phải trả đến một mốc thời gian cụ thể.
* **`PHIEUTHU` (Phiếu thu):** Ghi nhận dòng tiền thực tế khách hàng đóng vào hệ thống.
  * **Chi tiết:** Tách biệt rõ `TienGocThu` (tiền trừ thẳng vào nợ gốc) và `TienLaiThu` (tiền đóng lãi). Việc giảm nợ gốc kịp thời sẽ làm giảm áp lực tính lãi cho các ngày tiếp theo.
* **`LICHSUBIENDONG` (Nhật ký - LOG):** Lưu lại toàn bộ "vết" thay đổi của hợp đồng theo thời gian thực.
  * **Chi tiết:** Ghi nhận lại các sự kiện như Giải ngân, Thu nợ, Rút tài sản, hoặc Chuyển trạng thái nợ xấu. Bảng này lưu cứng giá trị `SoTienDaTra` và `DuNoHienTai` tại đúng thời điểm xảy ra sự kiện để phục vụ việc kiểm toán và truy vấn lịch sử.

---

## 💡 Các Logic Nghiệp vụ Cốt lõi

1. **Cơ chế Lãi suất 2 tầng:** Hệ thống dựa vào `Deadline1` của Hợp đồng. Trước mốc này, lãi suất được tính theo mức lãi đơn cố định (VD: 5.000đ/1 triệu/ngày). Vượt quá mốc này, hệ thống áp dụng công thức Lãi kép dựa trên tổng dư nợ hiện tại (Gốc + Lãi cộng dồn).
2. **Rút tài sản từng phần:** Khách hàng có quyền chuộc lại một phần tài sản trước hạn. Hệ thống sẽ cấp phép dựa trên điều kiện: `Tổng định giá các tài sản còn lại đang cầm cố >= Tổng dư nợ hiện tại`.
3. **Thanh lý tài sản tự động:** * Quá `Deadline1`: Tự động đánh dấu hợp đồng là "Quá hạn (nợ xấu)".
   * Quá `Deadline2`: Chuyển trạng thái tài sản thành "Sẵn sàng thanh lý". Nếu tài sản bị gắn cờ `IsSold = True`, hệ thống sẽ chặn mọi giao dịch thu tiền / trả đồ liên quan đến tài sản đó.

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/c84e54c6-0dae-4c13-b0dc-4deda1bacd2f" />
