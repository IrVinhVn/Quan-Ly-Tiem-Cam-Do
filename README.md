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


## ⚙️ Giải thích Logic Cài đặt Cơ sở dữ liệu (Database Setup)
Tạo Database:
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/c84e54c6-0dae-4c13-b0dc-4deda1bacd2f" />

Để đảm bảo hệ thống hoạt động trơn tru và dữ liệu được toàn vẹn, quá trình khởi tạo cơ sở dữ liệu trên SQL Server được thiết kế với các quy tắc và logic cụ thể như sau:

<img width="2560" height="1440" alt="Screenshot 2026-05-12 113917" src="https://github.com/user-attachments/assets/bb19936d-92d5-4992-b97b-ea3008552aa4" />

### 1. Trình tự khởi tạo các bảng (Execution Order)
Để tránh lỗi ràng buộc khóa ngoại (Foreign Key Constraint Error), các bảng được tạo theo trình tự phân cấp từ độc lập đến phụ thuộc:
* **Nhóm bảng Độc lập (Tạo đầu tiên):** `KHACHHANG`, `NHANVIEN`, `LOAITAISAN`. Đây là các danh mục gốc, không phụ thuộc vào bất kỳ dữ liệu nào khác.
* **Nhóm bảng Cấp 1:** `HOPDONG`, `TAIKHOAN`. Được tạo sau khi có danh mục gốc vì cần tham chiếu đến thông tin Khách hàng và Nhân viên.
* **Nhóm bảng Cấp 2 (Tạo cuối cùng):** `TAISAN`, `KEHOACHTRAGOP`, `PHIEUTHU`, `LICHSUBIENDONG`. Nhóm này chứa các chi tiết phát sinh dựa trên một `HOPDONG` cụ thể.

### 2. Lựa chọn Kiểu dữ liệu tối ưu (Data Type Decisions)
* **Xử lý Tiền tệ (VNĐ):** Thay vì sử dụng kiểu `FLOAT` (dễ gây sai số khi tính toán lãi kép) hoặc `MONEY`, hệ thống sử dụng kiểu `DECIMAL(18, 0)`. Kiểu dữ liệu này đảm bảo độ chính xác tuyệt đối cho các phép cộng trừ nhân chia và phù hợp với đặc thù tiền Việt Nam (không có số thập phân sau dấu phẩy).
* **Cờ trạng thái (Boolean/Flag):** Các trường kiểm tra điều kiện True/False (ví dụ: cờ `IsSold` trong bảng Tài sản để phục vụ Event 3) được sử dụng kiểu `BIT` (0 hoặc 1) giúp tiết kiệm không gian lưu trữ và tối ưu hóa tốc độ truy vấn `IF...ELSE` trong Store Procedure.

### 3. Tự động hóa và Ràng buộc (Constraints & Automation)
* **Cột tính toán tự động (Computed Column):** Trong bảng `PHIEUTHU`, trường `TongTienThu` được thiết lập tự động tính toán dựa trên công thức `(TienGocThu + TienLaiThu)`. Điều này giúp giảm thiểu thao tác nhập liệu và ngăn ngừa lỗi sai lệch số liệu từ phía ứng dụng (Backend).
* **Giá trị mặc định (Default Values):** * Các trường lưu thời gian giao dịch (như ngày đóng tiền, ngày ghi LOG) được gắn hàm lấy thời gian hệ thống tự động (`GETDATE()`).
  * Các hợp đồng và tài sản khi mới tạo sẽ được gắn ngay trạng thái mặc định như *"Đang vay"* hay *"Đang cầm cố"* để chuẩn hóa quy trình đầu vào.

## 📝 Event 1: Đăng ký Hợp đồng mới (Nghiệp vụ Vay tiền)

Nghiệp vụ này đóng vai trò "cửa ngõ" của hệ thống, xử lý quá trình tiếp nhận khách hàng mang tài sản đến cầm cố và giải ngân khoản vay. Do một hợp đồng có thể bao gồm nhiều tài sản, logic xử lý được thiết kế chặt chẽ để đảm bảo tính toàn vẹn dữ liệu.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 115307" src="https://github.com/user-attachments/assets/44b7cf99-fb41-437c-b3cd-2734fab1f0f5" />

### 🔄 Luồng xử lý nghiệp vụ (Workflow)

Quá trình khởi tạo hợp đồng được thực hiện thông qua một giao dịch (Transaction) duy nhất với 4 bước cốt lõi:

**1. Xử lý thông tin Khách hàng:**
* Hệ thống sẽ kiểm tra số Căn cước công dân (CCCD) của người vay.
* Nếu là khách hàng mới: Tự động lưu thông tin cá nhân vào danh mục `KHACHHANG`.
* Nếu là khách hàng cũ: Tái sử dụng mã Khách hàng hiện tại để gắn vào hợp đồng, giúp hệ thống không bị rác dữ liệu và dễ dàng thống kê "khách quen".

**2. Tự động tính toán các mốc Deadline:**
Thay vì bắt người dùng tự nhập tay từng ngày, hệ thống chỉ cần nhận "Ngày vay" và tự động nội suy ra 2 mốc quan trọng:
* **Deadline 1 (Mốc nợ xấu & Lãi kép):** Ví dụ sau 30 ngày kể từ ngày vay.
* **Deadline 2 (Mốc thanh lý):** Ví dụ cộng thêm 15 ngày kể từ Deadline 1.

**3. Lưu Hợp đồng và Danh sách Tài sản (Xử lý hàng loạt):**
* Khởi tạo thông tin chung của `HOPDONG` với trạng thái mặc định là *"Đang vay"*, ghi nhận số tiền giải ngân gốc và mức lãi suất thỏa thuận.
* Đối với danh sách tài sản (xe máy, điện thoại, laptop...), hệ thống tiếp nhận một mảng dữ liệu (được truyền dưới dạng chuỗi JSON). Sau đó, tự động bóc tách và lưu từng món đồ vào bảng `TAISAN` kèm theo **Giá trị định giá** và trạng thái *"Đang cầm cố"*.

**4. Ghi nhận Lịch sử Biến động (Audit Log):**
* Ngay khi hợp đồng và tài sản được lưu thành công, hệ thống tự động sinh ra một bản ghi trong bảng `LICHSUBIENDONG`.
* Hành động này lưu lại vết: *Ngày giờ tạo, Loại biến động ("Giải ngân"), và Dư nợ gốc ban đầu*, làm cơ sở vững chắc cho việc đối soát và tính lãi sau này.

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/fe6ed909-0f50-4e32-9c91-665cd826997f" />
