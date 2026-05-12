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

## 🧮 Event 2: Tính toán công nợ thời gian thực (Real-time Debt Calculation)

Tính toán chính xác dư nợ là "trái tim" của hệ thống quản lý cầm đồ. Ở Event 2, hệ thống được trang bị các Hàm tự tạo (User-Defined Functions) trong SQL để tự động nội suy số tiền khách hàng phải trả tại bất kỳ thời điểm nào trong tương lai (Target Date), hỗ trợ trừ lùi dư nợ và tự động chuyển đổi công thức tính lãi.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 200923" src="https://github.com/user-attachments/assets/71b1ee4b-ef94-44c5-8e64-c224f8d35198" />

### 1. Logic xử lý của các Hàm tính toán (Scripts)

Hệ thống cung cấp 2 hàm chính để phục vụ các mục đích truy vấn khác nhau:

* **Tính tiền theo kỳ trả góp (`fn_CalcMoneyTransaction`):**
  * **Mục đích:** Truy xuất nhanh số tiền cần đóng của một kỳ hạn cụ thể.
  * **Logic:** Hàm sẽ kiểm tra trạng thái của kỳ hạn đó. Nếu trạng thái là *"Chưa đóng"*, hệ thống sẽ cộng dồn Tiền gốc phải trả và Tiền lãi phải trả. Nếu khách đã thanh toán, hệ thống an toàn trả về `0đ` để chặn rủi ro thu tiền trùng lặp.

* **Tính Tổng nợ Hợp đồng tích hợp Lãi kép (`fn_CalcMoneyContract`):**
  * **Mục đích:** Tính tổng dư nợ cuối cùng (bao gồm Gốc + Lãi) của toàn bộ hợp đồng tính đến một ngày bất kỳ.
  * **Cơ chế Dư nợ giảm dần:** Trước khi tính lãi, hệ thống tự động tổng hợp số tiền gốc khách đã trả trong quá khứ để trừ vào nợ gốc ban đầu. Lãi suất chỉ được tính trên **Dư nợ gốc thực tế**.
  * **Cơ chế Lãi suất 2 tầng:**
    * **Tầng 1 (Trước Deadline 1):** Áp dụng công thức **Lãi đơn** cơ bản trên dư nợ hiện tại.
    * **Tầng 2 (Sau Deadline 1):** Đây là cơ chế phạt nợ xấu. Hệ thống "chốt sổ" số tiền tại mốc Deadline 1 (Gốc còn lại + Lãi đơn cộng dồn). Con số tổng này trở thành *Vốn mới* để đưa vào công thức **Lãi kép** (Lãi mẹ đẻ lãi con) thông qua hàm lũy thừa toán học của SQL.

---

### 2. Kịch bản Kiểm thử (Testing)

Để chứng minh tính chính xác tuyệt đối của thuật toán, hệ thống đã được chạy kiểm thử với 3 kịch bản thực tế dựa trên một hợp đồng giả định (Vay 20 triệu, ngày vay 01/10, Deadline 1 là 30 ngày sau):

1. **Test tính tiền 1 kỳ trả góp:** Giả lập khách có lịch hẹn trả 5 triệu gốc + 1.5 triệu lãi. Hệ thống truy vấn đúng trạng thái và trả về chính xác tổng khoản thu là `6.500.000 VNĐ`.
2. **Test tính Lãi đơn (Chưa quá hạn):** Truy vấn dư nợ vào ngày thứ 15 của hợp đồng (trước Deadline 1). Hệ thống chỉ áp dụng lãi đơn, trả về tổng nợ là `21.500.000 VNĐ` (Bao gồm 20 triệu gốc + 1.5 triệu tiền lãi 15 ngày).
3. **Test tính Lãi kép (Đã quá hạn):** Truy vấn dư nợ vào ngày thứ 35 (Trễ hạn 5 ngày so với Deadline 1). Hệ thống tự động gộp 30 ngày lãi đơn vào nợ gốc tạo thành vốn mới, sau đó đánh lãi kép cho 5 ngày trễ hẹn. Kết quả trả về tự động nhảy vọt lên `~23.580.916 VNĐ`, phản ánh đúng sức mạnh của "lãi mẹ đẻ lãi con".

<img width="2560" height="1440" alt="Screenshot 2026-05-12 200840" src="https://github.com/user-attachments/assets/c6629951-995e-4226-a456-774078915a8a" />

## 💰 Event 3: Xử lý Trả nợ và Hoàn trả tài sản (Payment & Asset Release)

Đây là module xử lý dòng tiền cốt lõi của hệ thống, giải quyết bài toán nghiệp vụ khi khách hàng đến thanh toán nợ (tất toán toàn bộ hoặc trả góp từng phần) và có nhu cầu chuộc lại tài sản.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 202214" src="https://github.com/user-attachments/assets/d22c4a08-4c31-496c-a8f1-f1e598278988" />

### 1. Logic xử lý của Store Procedure (Kịch bản Script)

Hệ thống không đơn thuần chỉ là "cộng trừ tiền" mà được thiết kế với một luồng kiểm tra chặt chẽ gồm 5 bước tự động:

1. **Chặn giao dịch rủi ro (Liquidation Check):** Ngay khi tiếp nhận yêu cầu, hệ thống quét cờ `IsSold` của toàn bộ tài sản trong hợp đồng. Nếu phát hiện tài sản đã bị đem đi thanh lý (do quá hạn mốc Deadline 2), thủ tục sẽ lập tức bị hủy bỏ kèm thông báo từ chối thu tiền và từ chối trả đồ để tránh rủi ro pháp lý.
2. **Chốt công nợ thời gian thực:** Gọi lại hàm tính lãi (`fn_CalcMoneyContract` ở Event 2) để xác định chính xác đến từng đồng số tiền Gốc + Lãi tính đến đúng ngày khách hàng cầm tiền đến quầy.
3. **Phân bổ dòng tiền & Chuyển đổi trạng thái:**
   * **Trường hợp tất toán (Trả đủ 100%):** Trạng thái hợp đồng tự động chuyển thành *"Đã thanh toán đủ"*. Hệ thống phát lệnh "giải phóng", đổi trạng thái toàn bộ tài sản sang *"Đã trả khách"*.
   * **Trường hợp trả góp (Trả một phần):** Tiền được thu vào để giảm trừ dư nợ. Trạng thái hợp đồng chuyển sang *"Đang trả góp"*.
4. **Minh bạch hóa dữ liệu (Audit Log):** Mọi giao dịch thu tiền đều được hệ thống tự động lưu vết vào nhật ký (`LICHSUBIENDONG`), ghi rõ *Số tiền khách vừa trả* và *Số dư nợ còn lại* tại khoảnh khắc đó.
5. **Thuật toán Gợi ý chuộc đồ thông minh:** Đây là tính năng nâng cao. Khi khách mới trả một phần nợ, hệ thống sẽ đánh giá: *"Nếu cho khách lấy món đồ A về, thì giá trị các món đồ còn lại tiệm đang giữ có đủ bù đắp số nợ còn lại không?"*. Nếu **Tổng định giá đồ còn giữ >= Dư nợ hiện tại**, hệ thống sẽ in ra danh sách gợi ý cho phép nhân viên trả món đồ đó cho khách.

---

### 2. Kịch bản Kiểm thử (Testing Scenarios)

Để nghiệm thu tính năng này, hệ thống được chạy qua một kịch bản giao dịch thực tế như sau:

* **Bối cảnh giả định:** Khách hàng có hợp đồng đang nợ tổng cộng 21.500.000 VNĐ. Tài sản đang cầm cố gồm 1 chiếc xe máy (định giá 60 triệu) và 1 chiếc điện thoại (định giá 25 triệu). Khách mang 10.000.000 VNĐ đến cửa hàng xin trả bớt nợ và muốn lấy lại chiếc điện thoại.
* **Tiến hành Test:** Đưa tham số số tiền trả là 10.000.000 VNĐ vào thủ tục xử lý giao dịch.
* **Kết quả nghiệm thu (Kỳ vọng & Thực tế khớp nhau):**
  1. Giao dịch thực hiện thành công, dư nợ hợp đồng giảm xuống còn 11.500.000 VNĐ.
  2. Trạng thái hợp đồng tự động nhảy từ *"Đang vay"* sang *"Đang trả góp"*.
  3. Bảng Lịch sử biến động ghi nhận thành công dòng log: Loại biến động là "Thu nợ", số tiền thu 10 triệu, dư nợ hiện tại 11.5 triệu.
  4. **Hệ thống in ra danh sách gợi ý hợp lệ:** Hệ thống gợi ý hoàn toàn CÓ THỂ trả lại chiếc điện thoại (25 triệu) cho khách. Vì chiếc xe máy giữ lại có giá trị 60 triệu, hoàn toàn vượt mức an toàn so với khoản nợ 11.5 triệu còn lại.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 202356" src="https://github.com/user-attachments/assets/2fb3b841-dfdc-4894-adb0-6a2d77e951fd" />

## 🚩 Event 4: Truy vấn Danh sách Nợ xấu & Dự báo Công nợ (Bad Debt Reporting)

Bên cạnh việc xử lý giao dịch, hệ thống còn đóng vai trò là một công cụ quản trị rủi ro mạnh mẽ. Module báo cáo này giúp chủ cửa hàng tự động xác định các khách hàng vi phạm hợp đồng (quá hạn) và dự báo gánh nặng nợ nần trong tương lai để có phương án thu hồi kịp thời.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 203855" src="https://github.com/user-attachments/assets/2552f2c5-4f59-40e3-ab1a-3448ea386447" />

### 1. Logic xử lý của hệ thống (Script Logic)

Báo cáo này không sử dụng các bảng lưu trữ tĩnh mà được thiết kế dựa trên logic truy vấn động (Dynamic Querying) kết hợp với các hàm tính toán thời gian thực:

* **Bộ lọc thông minh (Advanced Filtering):** Hệ thống quét toàn bộ cơ sở dữ liệu và chỉ trích xuất những hợp đồng thỏa mãn đồng thời hai điều kiện khắt khe: (1) Ngày hiện tại đã vượt quá mốc **Deadline 1** và (2) Trạng thái hợp đồng vẫn đang treo, tức là khác với *"Đã thanh toán đủ"* hoặc *"Đã thanh lý"*.
* **Tự động tính toán số ngày vi phạm:** Hệ thống sử dụng hàm tính toán thời gian để đo lường chính xác khoảng cách từ ngày khách hàng bắt đầu trễ hạn (Deadline 1) cho đến thời điểm bấm nút chạy báo cáo.
* **Tái sử dụng Module tính toán (Code Reusability):** Thay vì viết lại công thức tính lãi, thủ tục này gọi trực tiếp hàm `fn_CalcMoneyContract` (đã xây dựng ở Event 2). Điều này đảm bảo tính nhất quán tuyệt đối về số liệu tài chính trên toàn hệ thống.
* **Khả năng Dự báo tài chính (Forecasting):** Đây là "vũ khí" đắc lực nhất của báo cáo. Bằng cách truyền tham số thời gian là **1 tháng sau** vào hàm tính nợ, hệ thống tự động giả lập và đưa ra con số tổng nợ khách sẽ phải gánh chịu nếu tiếp tục chây ì. 

---

### 2. Kịch bản Kiểm thử (Testing Scenario)

Để nghiệm thu tính chính xác của báo cáo nợ xấu, hệ thống được kiểm thử bằng kịch bản sau:

* **Bối cảnh giả định:** Hệ thống có chứa các hợp đồng cũ (Ví dụ: Khách vay từ tháng 10/2023, chắc chắn đã vượt qua Deadline 1 rất lâu) và vẫn chưa thanh toán nợ.
* **Tiến hành Test:** Kích hoạt thủ tục xuất báo cáo nợ xấu.
* **Kết quả nghiệm thu:**
  1. Dữ liệu trả về hiển thị chính xác danh sách những khách hàng đang nợ quá hạn, bỏ qua các khách hàng trả đúng hạn.
  2. Cột **"Số ngày quá hạn"** hiển thị con số chính xác dựa trên lịch thực tế của hệ thống.
  3. Cột **"Tổng tiền hiện tại"** tự động áp dụng công thức Lãi kép, hiển thị số tiền nợ khổng lồ tính đến đúng giây phút truy vấn.
  4. Cột **"Tổng tiền sau 1 tháng nữa"** tự động nhảy số lớn hơn hẳn so với tiền hiện tại (phản ánh đúng sức mạnh của "lãi mẹ đẻ lãi con" sau 30 ngày tiếp theo).

<img width="2560" height="1440" alt="Screenshot 2026-05-12 203936" src="https://github.com/user-attachments/assets/f886aa37-2241-44b6-9319-8ba18c9e6083" />

## ⚖️ Event 5: Quản lý Thanh lý Tài sản (Automated Asset Liquidation)

Đây là module tự động hóa cốt lõi của hệ thống, giúp giảm thiểu tối đa thao tác thủ công của con người. Bằng cách sử dụng **Trigger** (Trình kích hoạt tự động), hệ thống có khả năng tự giám sát các mốc thời gian và tự động "nắn" lại trạng thái của hợp đồng cũng như tài sản cho đúng với quy định cầm đồ.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 205421" src="https://github.com/user-attachments/assets/344cc47d-b487-4d78-92c2-bc660951bed3" />

### 1. Logic xử lý của Trigger (Script Logic)

Trigger được thiết lập để bám sát vào các sự kiện cập nhật (`AFTER UPDATE`) trên bảng Hợp đồng. Điểm sáng trong tư duy thiết kế ở đây là **Thứ tự thực thi (Execution Order)**: Hệ thống luôn ưu tiên các quyết định chủ đích của con người trước, sau đó mới đến các rà soát tự động của máy móc.

* **Xử lý ưu tiên (Thao tác thủ công):** Ngay khi hệ thống phát hiện người quản lý chủ động đổi trạng thái hợp đồng thành *"Đã thanh lý"*, nó sẽ lập tức khóa sổ. Toàn bộ tài sản đi kèm đang chờ bán sẽ tự động chuyển sang *"Đã bán thanh lý"*, đồng thời cờ `IsSold` được bật lên để vĩnh viễn chặn mọi nỗ lực đóng tiền chuộc đồ (liên kết với chặn bảo mật ở Event 3).
* **Rà soát tự động Nợ xấu (Quá Deadline 1):** Nếu hệ thống bị "chạm" vào, nó sẽ tự động quét các hợp đồng đang ở trạng thái *"Đang vay"*. Nếu phát hiện ngày hiện tại đã vượt qua mốc Deadline 1, nó sẽ âm thầm chuyển trạng thái hợp đồng đó thành *"Quá hạn (nợ xấu)"*.
* **Rà soát tự động Thanh lý (Quá Deadline 2):** Tiếp nối bước trên, hệ thống kiểm tra các hợp đồng *"Quá hạn"*. Nếu phát hiện đã vượt qua mốc Deadline 2 (thời gian ân hạn cuối cùng), nó sẽ lập tức tước quyền bảo lưu tài sản, tự động đổi trạng thái tài sản thành *"Sẵn sàng thanh lý"* để báo hiệu cho chủ tiệm đem bán.

---

### 2. Kịch bản Kiểm thử (Testing Scenarios)

Để chứng minh hệ thống tự động hóa hoạt động chính xác và không bị xung đột, chúng ta tiến hành kiểm thử qua 2 giai đoạn:

#### Phần Test 1: Kiểm thử hệ thống tự động nhận diện "Quá hạn"
* **Bối cảnh:** Sử dụng một hợp đồng cũ (đã quá cả mốc Deadline 1 và Deadline 2 nhưng trạng thái vẫn đang hiện là "Đang vay"). Tạo một lệnh cập nhật (chạm nhẹ) vào hợp đồng để đánh thức Trigger.
* **Kết quả nghiệm thu:** 1. Trạng thái Hợp đồng tự động nhảy từ *"Đang vay"* sang *"Quá hạn (nợ xấu)"*.
    2. Trạng thái Tài sản lập tức chuyển thành *"Sẵn sàng thanh lý"*.
    3. Hệ thống quét tự động thành công mà không cần ai phải dò từng ngày.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 205443" src="https://github.com/user-attachments/assets/a603a825-323e-4660-a250-60495784ea10" />

#### Phần Test 2: Kiểm thử lệnh chốt "Thanh lý tài sản"
* **Bối cảnh:** Đối với hợp đồng đã ở trạng thái *"Quá hạn"* và tài sản đang *"Sẵn sàng thanh lý"* (kết quả từ phần Test 1), người quản lý ra quyết định bán đồ và cập nhật trạng thái hợp đồng thành *"Đã thanh lý"*.
* **Kết quả nghiệm thu:**
    1. Trạng thái Hợp đồng ghi nhận thành công là *"Đã thanh lý"*.
    2. Trigger tóm gọn hành động này, tự động chuyển trạng thái của các tài sản đi kèm thành *"Đã bán thanh lý"*.
    3. Cờ bảo mật `IsSold` tự động nhảy lên `1` (True).

<img width="2560" height="1440" alt="Screenshot 2026-05-12 205457" src="https://github.com/user-attachments/assets/a7b1b779-b22b-44e1-a102-d9c8f2775311" />

## 🔄 Các Sự kiện Bổ sung: Gia hạn Hợp đồng & Audit Log (Extension & Audit Trail)

Một hệ thống cầm đồ thực tế không chỉ cứng nhắc thu nợ và thanh lý tài sản, mà còn phải cung cấp các giải pháp linh hoạt cho khách hàng gặp khó khăn tài chính, đồng thời đảm bảo minh bạch tuyệt đối về dòng tiền cho chủ cửa hàng.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 220618" src="https://github.com/user-attachments/assets/c95acb0b-855e-4f2d-b6e1-ffa99b48f540" />

### 1. Logic xử lý của hệ thống (Script Logic)

Module này được chia làm hai mũi nhọn giải quyết hai vấn đề cốt lõi của quản trị cơ sở dữ liệu tài chính:

* **Nâng cấp Audit Log (Dấu vết dòng tiền):** Khắc phục triệt để rủi ro thất thoát dữ liệu bằng nguyên tắc **"Không bao giờ ghi đè (No Overwrite)"**. Thay vì chỉ cập nhật một con số tổng nợ duy nhất, hệ thống lưu lại mọi biến động thành từng dòng riêng biệt trong bảng Lịch sử (Bao gồm: Ngày giờ, Số tiền trả, Dư nợ còn lại, và đặc biệt là **Mã Nhân viên thu tiền**). Điều này giúp truy xuất chính xác ai là người chịu trách nhiệm cho từng khoản thu.
* **Xử lý nghiệp vụ Gia hạn Hợp đồng:**
  * **Chốt chặn tài chính:** Hệ thống tự động tính toán tổng số tiền lãi phát sinh tính đến đúng ngày khách xin gia hạn. Khách bắt buộc phải thanh toán toàn bộ khoản lãi này để được xét duyệt dời ngày.
  * **Phân bổ dòng tiền:** Số tiền khách đóng được ghi nhận vào Phiếu thu nhưng chỉ được trừ vào "Tiền Lãi", giữ nguyên "Dư nợ Gốc" để tiếp tục tính lãi cho kỳ hạn mới (đảm bảo chuẩn xác về toán học tài chính).
  * **Tái cơ cấu nợ (Restructuring):** Nếu đóng lãi thành công, hệ thống tự động cộng thêm số ngày gia hạn (ví dụ: 30 ngày) vào các mốc `Deadline 1` và `Deadline 2`. Hợp đồng được "giải cứu", chuyển trạng thái từ *"Quá hạn"* về lại *"Đang vay"*, giúp khách hàng thoát khỏi việc bị tính lãi kép.

---

### 2. Kịch bản Kiểm thử (Testing Scenarios)

Để chứng minh tính an toàn dữ liệu và logic dời ngày, chúng ta thực hiện kịch bản kiểm thử sau:

* **Bối cảnh giả định:** Khách hàng có hợp đồng đang rơi vào trạng thái nợ xấu do quá hạn Deadline 1. Tiền lãi đã cộng dồn lên một con số lớn. Khách mang tiền đến cửa hàng xin đóng trọn phần lãi này để dời kỳ hạn thêm 30 ngày.
* **Tiến hành Test:** Kích hoạt thủ tục Gia hạn hợp đồng, truyền vào Mã hợp đồng, Mã nhân viên thu tiền và số ngày muốn dời (30 ngày).
* **Kết quả nghiệm thu:**
  1. Giao dịch báo thành công, in ra đúng số tiền lãi khách vừa đóng và mốc Deadline mới.
  2. Truy vấn lại bảng Hợp đồng: Trạng thái đã được reset thành *"Đang vay"*. Mốc Deadline 1 và Deadline 2 đã được tịnh tiến thêm đúng 30 ngày so với trước đó.
  3. Truy vấn bảng Audit Log: Xuất hiện thêm **một dòng lịch sử mới nhất** ghi nhận hành động "Gia hạn hợp đồng", ghi rõ số tiền lãi đã thu và nhân viên thực hiện. Các dòng lịch sử thu tiền cũ của hợp đồng này vẫn được giữ nguyên vẹn, tạo thành một chuỗi minh bạch.

<img width="2560" height="1440" alt="Screenshot 2026-05-12 220649" src="https://github.com/user-attachments/assets/a02e332d-ba0c-4835-b2a2-f3a93874aa93" />
