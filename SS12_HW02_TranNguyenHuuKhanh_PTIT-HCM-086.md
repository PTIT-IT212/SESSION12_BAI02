# SOFTWARE REQUIREMENTS SPECIFICATION (SRS)
## HỆ THỐNG ĐỊNH DANH ĐIỆN TỬ KHÁCH HÀNG (eKYC) - ABC BANK
### Tiêu chuẩn thiết kế: IEEE Std 830-1998
### Định hướng vận hành: Zero Manual Operation (Straight-Through Processing - STP 100%)

---

## PHẦN 1: INTRODUCTION (GIỚI THIỆU CHUNG)

### 1.1 Purpose (Mục đích)
Tài liệu Đặc tả Yêu cầu Phần mềm (SRS) này được xây dựng nhằm mô tả chi tiết các yêu cầu chức năng và phi chức năng cho tính năng định danh điện tử khách hàng (eKYC) trực tuyến trên ứng dụng di động của ABC Bank. 
Tài liệu này là cơ sở kỹ thuật chính thức phục vụ cho:
* **Đội ngũ Phát triển (Development Team)**: Sử dụng làm tài liệu đặc tả thiết kế kiến trúc hệ thống, xây dựng cơ sở dữ liệu, viết mã nguồn (Backend & Frontend Mobile) và cấu hình các tích hợp hệ thống bên thứ ba.
* **Đội ngũ Kiểm thử (QA/QC Team)**: Sử dụng để xây dựng tài liệu Kế hoạch kiểm thử (Test Plan), thiết kế các kịch bản kiểm thử (Test Cases) chi tiết cho luồng nghiệp vụ chính, các luồng ngoại lệ và xác thực hiệu năng hệ thống.

### 1.2 Scope (Phạm vi)
Phạm vi của hệ thống eKYC bao gồm các thành phần:
* **Ứng dụng Mobile Client**: Cung cấp giao diện tương tác thu thập dữ liệu sinh trắc học và giấy tờ tùy thân của khách hàng trên hai nền tảng iOS và Android.
* **Hệ thống điều phối eKYC Orchestrator**: Tiếp nhận, điều phối và xử lý toàn bộ các yêu cầu từ Mobile Client, kết nối trực tiếp với các Engine phân tích (NFC, OCR, Liveness, Face Match, AML/PEP).
* **Quy trình Phê duyệt Tự động (STP Engine)**: Đưa ra quyết định phê duyệt hoặc từ chối tạo CIF/Tài khoản ngay lập tức mà không có sự tham gia của con người ở luồng xử lý thời gian thực.
* **Tích hợp Core Banking**: Tự động hóa việc tạo CIF và số tài khoản thanh toán cho các khách hàng được eKYC phê duyệt thành công.

*Nằm ngoài phạm vi*: Các dịch vụ tại quầy giao dịch, các quy trình phát hành thẻ vật lý ngoài luồng eKYC tự động, hoặc việc cập nhật lại thông tin định danh eKYC định kỳ của khách hàng hiện tại.

### 1.3 Definitions, Acronyms, and Abbreviations (Định nghĩa và Thuật ngữ)
Dưới đây là các định nghĩa và chữ viết tắt được sử dụng xuyên suốt tài liệu:

| Thuật ngữ / Viết tắt | Tên đầy đủ | Định nghĩa |
| :--- | :--- | :--- |
| **eKYC** | Electronic Know Your Customer | Định danh điện tử khách hàng dựa trên công nghệ xác thực trực tuyến. |
| **OCR** | Optical Character Recognition | Nhận dạng ký tự quang học, chuyển đổi hình ảnh chữ viết trên giấy tờ tùy thân thành văn bản số. |
| **Liveness Check** | Biometric Liveness Detection | Công nghệ kiểm tra thực thể sống, đảm bảo hình ảnh khuôn mặt thu thập được là từ người thật đang thực hiện trực tiếp, không phải ảnh chụp lại hay video giả mạo. |
| **Face Matching** | Facial Recognition Comparison | Công nghệ so sánh đối chiếu hai khuôn mặt để tính toán độ tương đồng (%) và xác minh danh tính. |
| **SDK** | Software Development Kit | Bộ công cụ phát triển phần mềm được tích hợp vào ứng dụng Mobile Banking để hỗ trợ chụp ảnh, quét NFC và liveness. |
| **API** | Application Programming Interface | Giao diện lập trình ứng dụng, phương thức giao tiếp dữ liệu giữa các dịch vụ Backend của hệ thống. |
| **NFC** | Near-Field Communication | Công nghệ giao tiếp trường gần, dùng để đọc dữ liệu mã hóa từ chip tích hợp trên CCCD. |
| **STP** | Straight-Through Processing | Xử lý xuyên suốt tự động, không yêu cầu sự can thiệp thủ công của con người. |
| **CIF** | Customer Information File | Mã định danh thông tin khách hàng duy nhất trên hệ thống Core Banking của ngân hàng. |
| **AML/PEP** | Anti-Money Laundering / Politically Exposed Persons | Chống rửa tiền và người có tầm ảnh hưởng chính trị. |

---

## PHẦN 2: OVERALL DESCRIPTION (MÔ TẢ TỔNG QUAN)

### 2.1 Product Perspective (Góc nhìn sản phẩm)
Hệ thống eKYC là một phân hệ cốt lõi thuộc Hệ sinh thái Ngân hàng số của ABC Bank. Phân hệ này đóng vai trò là "cổng vào" tự động, tương tác trực tiếp với các hệ thống phân tích sinh trắc học và hệ thống nghiệp vụ nội bộ của ngân hàng để xử lý yêu cầu trực tuyến theo thời gian thực.

```
+-----------------------------------------------------------------------+
|                       ABC Bank Mobile App                             |
+----------------------------------+------------------------------------+
                                   | API Requests (HTTPS)
                                   v
+-----------------------------------------------------------------------+
|                        eKYC Orchestrator                              |
+-----+-------------------+--------+-------------------+----------+-----+
      |                   |        |                   |          |
      v (NFC/OCR APIs)    v (Bio)  v (CA Verification) v (AML/PEP)v (STP)
+-----------+       +-----------+  +-----------+  +-----------+  +----------+
| OCR/NFC   |       | Liveness/ |  | Bộ Công An|  | AML/PEP   |  | Core     |
| Engine    |       | FaceMatch |  | C06 API   |  | Screening |  | Banking  |
+-----------+       +-----------+  +-----------+  +-----------+  +----------+
```

### 2.2 User Classes and Characteristics (Đặc điểm nhóm người dùng)
Hệ thống eKYC tương tác với ba nhóm đối tượng (gồm người dùng và hệ thống tự động):

1. **Khách hàng vãng lai (End-User / Customer)**:
   - Là những cá nhân chưa có tài khoản tại ABC Bank, có nhu cầu mở tài khoản trực tuyến qua thiết bị di động.
   - Có trình độ công nghệ khác nhau, sử dụng nhiều loại thiết bị điện thoại thông minh với camera và cấu hình phần cứng khác nhau. Yêu cầu giao diện UI/UX trực quan và hướng dẫn đơn giản.

2. **Hệ thống Core Banking (System Actor)**:
   - Hệ thống tự động nhận yêu cầu khởi tạo mã khách hàng (CIF) và số tài khoản thanh toán từ eKYC Orchestrator. 
   - Đòi hỏi dữ liệu đầu vào phải sạch, đồng bộ và hoàn thành xác thực eKYC 100%.

3. **Nhân viên Kiểm toán/Hậu kiểm (Compliance & Post-Audit Staff)**:
   - Không tham gia phê duyệt thời gian thực (Real-time).
   - Truy cập hệ thống quản trị sau khi tài khoản đã được mở để rà soát chất lượng hình ảnh, đối soát tỷ lệ chính xác của OCR/NFC và kiểm tra ngẫu nhiên nhằm phục vụ mục đích kiểm soát rủi ro định kỳ.

### 2.3 Constraints (Các ràng buộc hệ thống)
* **Ràng buộc về Luật pháp**:
  - Tuân thủ Luật Phòng chống rửa tiền số 14/2022/QH15 và các thông tư hướng dẫn của Ngân hàng Nhà nước Việt Nam về eKYC (ví dụ: Thông tư 16/2020/TT-NHNN).
  - Tuân thủ Nghị định 13/2023/NĐ-CP về Bảo vệ dữ liệu cá nhân (PII), bắt buộc lưu trữ dữ liệu mã hóa và xin ý kiến chấp thuận xử lý dữ liệu của người dùng trước khi thực hiện eKYC.
* **Ràng buộc về Công nghệ**:
  - **Camera**: Thiết bị di động của khách hàng phải có độ phân giải camera trước và sau tối thiểu từ 5 Megapixels trở lên.
  - **Kết nối mạng**: Băng thông mạng tối thiểu yêu cầu là 3G/4G/5G hoặc kết nối Wi-Fi có tốc độ download/upload ổn định tối thiểu 2 Mbps.
  - **Hỗ trợ NFC**: Thiết bị di động phải tích hợp chip NFC và hỗ trợ hệ điều hành iOS (từ iOS 13 trở lên) hoặc Android (từ Android 8.0 trở lên) để thực hiện quét chip CCCD.

---

## PHẦN 3: SPECIFIC FUNCTIONAL REQUIREMENTS (YÊU CẦU CHỨC NĂNG CHI TIẾT)

### 3.1 Module 1: Đăng ký tài khoản (Nhập SĐT, OTP, Email)

* **Mô tả chức năng**:
  Hệ thống tiếp nhận thông tin khởi tạo của khách hàng gồm số điện thoại và email, gửi mã OTP để xác thực quyền sở hữu kênh liên lạc của khách hàng.

* **Luồng xử lý chính (Main flow)**:
  1. Khách hàng nhập Số điện thoại và Email vào giao diện đăng ký tài khoản mới của Mobile App.
  2. Mobile App kiểm tra định dạng cú pháp SĐT (10 chữ số, đúng đầu số nhà mạng Việt Nam) và định dạng Email.
  3. Hệ thống gửi yêu cầu khởi tạo OTP tới SMS Gateway; gửi OTP qua tin nhắn SMS tới số điện thoại đăng ký.
  4. Khách hàng nhận OTP và nhập vào ô xác thực trên Mobile App trong thời hạn hiệu lực 180 giây.
  5. eKYC Orchestrator xác minh tính hợp lệ của mã OTP.
  6. Nếu OTP chính xác, hệ thống chuyển khách hàng sang Module 2 và ghi nhận Số điện thoại này làm kênh liên lạc chính thức.

* **Luồng ngoại lệ (Exception flow)**:
  * **Luồng ngoại lệ 1.1**: Số điện thoại nhập sai định dạng.
    * *Hành vi*: Hệ thống chặn không cho nhấn "Tiếp tục" và hiển thị thông báo lỗi màu đỏ dưới trường nhập liệu: *"Số điện thoại không đúng định dạng. Vui lòng kiểm tra lại"*.
  * **Luồng ngoại lệ 1.2**: Nhập sai OTP.
    * *Hành vi*: Hệ thống hiển thị thông báo lỗi: *"Mã OTP không chính xác. Số lần thử còn lại: N/3"*. Nếu nhập sai quá 3 lần liên tiếp, hệ thống khóa đăng ký đối với Số điện thoại này trong vòng 24 giờ.
  * **Luồng ngoại lệ 1.3**: OTP hết hạn sử dụng.
    * *Hành vi*: Đồng hồ đếm ngược trên màn hình về 0, hệ thống vô hiệu hóa mã OTP cũ. Hiển thị nút "Gửi lại OTP". Khách hàng nhấn nút để gửi lại OTP mới (tối đa 3 lần/phiên).

---

### 3.2 Module 2: Upload & Đọc CCCD (OCR & NFC Verification)

* **Mô tả chức năng**:
  Thu thập hình ảnh giấy tờ tùy thân CCCD gắn chip, thực hiện quét dữ liệu NFC mã hóa của chip và đối chiếu tính hợp lệ của chữ ký số CA đối với Bộ Công an. Nếu thiết bị không hỗ trợ NFC, cho phép chụp ảnh và dùng OCR dự phòng có hạn mức thấp.

* **Luồng xử lý chính (Main flow)**:
  1. Mobile App kiểm tra phần cứng NFC trên thiết bị di động của khách hàng.
  2. Hệ thống hướng dẫn khách hàng chụp ảnh mặt trước/sau của CCCD gắn chip và quét mã QR trên CCCD bằng camera.
  3. Ứng dụng hiển thị hướng dẫn người dùng áp thẻ CCCD gắn chip vào khu vực cảm biến NFC của điện thoại.
  4. Chip NFC được giải mã thành công, trích xuất dữ liệu gốc bao gồm: Số định danh, Họ tên, Ngày sinh, Ngày hết hạn, Đặc điểm nhận dạng và Ảnh chân dung gốc chất lượng cao lưu trong chip (DG2).
  5. eKYC Orchestrator gửi dữ liệu chữ ký số kèm nội dung chip lên C06 (Bộ Công an) để xác thực tính toàn vẹn.
  6. Hệ thống nhận kết quả xác thực là **VALID**, điền tự động toàn bộ dữ liệu văn bản lên màn hình và cho phép khách hàng nhấn xác nhận chuyển sang Module 3.

* **Luồng ngoại lệ (Exception flow)**:
  * **Luồng ngoại lệ 2.1**: Thiết bị không có NFC hoặc Quét NFC thất bại 2 lần.
    * *Hành vi*: Hệ thống chuyển hướng sang luồng OCR chụp ảnh. Hệ thống OCR trích xuất thông tin văn bản từ ảnh chụp CCCD. Tài khoản mở qua luồng này sau khi hoàn tất sẽ tự động bị phân loại nhóm rủi ro cao và áp dụng hạn mức giao dịch thấp (dưới 10 triệu VND/tháng) cho đến khi xác thực sinh trắc học vật lý.
  * **Luồng ngoại lệ 2.2**: Giấy tờ tùy thân đã hết hạn.
    * *Hành vi*: Hệ thống kiểm tra Ngày hết hạn trên chip CCCD, nếu nhỏ hơn ngày hiện tại, hệ thống tự động từ chối giao dịch và thông báo: *"Giấy tờ tùy thân của quý khách đã hết hạn sử dụng. Vui lòng sử dụng giấy tờ còn hiệu lực"*.
  * **Luồng ngoại lệ 2.3**: Xác thực chữ ký số CA từ Bộ Công An báo thất bại (CA INVALID).
    * *Hành vi*: Hệ thống đánh dấu phiên giao dịch là có dấu hiệu giả mạo nghiêm trọng, tự động chấm dứt eKYC, lưu log cảnh báo rủi ro cao lên Fraud Detection System và hiển thị thông báo từ chối: *"Không thể xác thực thông tin giấy tờ trực tuyến. Vui lòng liên hệ quầy giao dịch gần nhất"*.

---

### 3.3 Module 3: Xác thực sinh trắc học khuôn mặt (Liveness & Face Match)

* **Mô tả chức năng**:
  Xác thực thực thể sống của người dùng bằng camera trước và so khớp khuôn mặt đó với ảnh gốc chất lượng cao được lưu trong chip CCCD.

* **Luồng xử lý chính (Main flow)**:
  1. Mobile App hiển thị khung tròn hướng dẫn khuôn mặt và yêu cầu người dùng nhìn thẳng vào camera trước.
  2. Thuật toán Passive Liveness Check ghi nhận luồng ảnh sinh trắc học và phân tích các dấu hiệu ánh sáng, độ sâu để xác định người thật đang thao tác trực tiếp.
  3. Liveness Check trả kết quả **PASS**.
  4. Face Matching Engine tiến hành so sánh cấu trúc khuôn mặt vừa thu thập với ảnh chân dung trích xuất từ chip CCCD gốc (DG2).
  5. Tỷ lệ tương đồng đạt kết quả $\ge 85.00\%$.
  6. Hệ thống lưu trữ thông tin sinh trắc học thành công và chuyển sang Module 4.

* **Luồng ngoại lệ (Exception flow)**:
  * **Luồng ngoại lệ 3.1**: Phát hiện giả mạo khuôn mặt (Liveness Check FAIL).
    * *Hành vi*: Hệ thống phát hiện tấn công giả mạo (sử dụng màn hình LCD, ảnh in hoặc mặt nạ). Hệ thống tự động từ chối ngay lập tức, lưu log IP/Device ID vào Fraud Blacklist và chặn đăng ký mới từ thiết bị này trong 7 ngày.
  * **Luồng ngoại lệ 3.2**: Tỷ lệ tương đồng khuôn mặt dưới 85.00%.
    * *Hành vi*: Hệ thống thông báo lỗi: *"Khuôn mặt chụp thực tế không khớp với giấy tờ tùy thân"*. Hệ thống cho phép người dùng chụp lại tối đa 3 lần. Nếu lần thứ 3 vẫn thất bại, hệ thống tự động từ chối mở tài khoản trực tuyến.
  * **Luồng ngoại lệ 3.3**: Điều kiện ánh sáng yếu hoặc camera mờ.
    * *Hành vi*: Ứng dụng phát hiện độ phơi sáng thấp hoặc ảnh bị nhòe thông qua AI cục bộ trên Mobile App, hiển thị cảnh báo thời gian thực: *"Vui lòng di chuyển đến khu vực có đủ ánh sáng và giữ vững điện thoại"*.

---

### 3.4 Module 4: Kích hoạt tài khoản (Phê duyệt STP & Tích hợp Core Banking)

* **Mô tả chức năng**:
  Hệ thống chạy Rule Engine phê duyệt tự động toàn bộ hồ sơ eKYC, đối chiếu AML/PEP, gửi yêu cầu tạo CIF và tài khoản trực tuyến tới Core Banking, sau đó thông báo kết quả cho khách hàng.

* **Luồng xử lý chính (Main flow)**:
  1. eKYC Orchestrator chạy Rule Engine kiểm tra các điều kiện: 
     $$\text{NFC CA} = \text{VALID} \;\land\; \text{Liveness} = \text{PASS} \;\land\; \text{Face Match} \ge 85\% \;\land\; \text{Tuổi} \ge 18$$
  2. eKYC Orchestrator gọi API kiểm tra hệ thống AML/PEP. Hệ thống trả về trạng thái **CLEAR** (không trùng danh sách đen).
  3. Hệ thống đưa ra trạng thái phê duyệt tự động **APPROVED** (STP Rate 100%).
  4. Orchestrator gửi bản tin API tạo CIF và Số Tài khoản thanh toán mới sang Core Banking.
  5. Core Banking xử lý thành công, trả về Số CIF và Số Tài khoản mới.
  6. Hệ thống gửi thông tin SMS kích hoạt kèm mật khẩu tạm thời cho khách hàng.
  7. Mobile App hiển thị màn hình chúc mừng mở tài khoản thành công kèm theo số tài khoản mới.

* **Luồng ngoại lệ (Exception flow)**:
  * **Luồng ngoại lệ 4.1**: Trùng danh sách đen AML hoặc danh sách PEP.
    * *Hành vi*: 
      - Nếu trùng danh sách đen (Blacklist/Sanctions): Hệ thống tự động từ chối mở tài khoản trực tuyến và ghi nhận log cảnh báo nội bộ.
      - Nếu trùng danh sách PEP: Hệ thống tự động từ chối duyệt trực tuyến và thông báo: *"Do quy định phòng chống rửa tiền, yêu cầu mở tài khoản trực tuyến của bạn chưa thể thực hiện. Vui lòng mang giấy tờ tùy thân ra quầy giao dịch ABC Bank gần nhất để được phục vụ"*.
  * **Luồng ngoại lệ 4.2**: Core Banking gặp lỗi kết nối hoặc phản hồi chậm (Timeout).
    * *Hành vi*: Hệ thống tự động đưa yêu cầu tạo CIF/Tài khoản vào hàng đợi hàng đợi nghiệp vụ (Message Queue) để thực hiện thử lại tự động (Auto-retry) tối đa 5 lần. Nếu vẫn thất bại sau 5 lần, hệ thống thực hiện Rollback phiên giao dịch eKYC, thông báo cho khách hàng qua Email/SMS về sự cố kỹ thuật tạm thời và hẹn liên hệ lại để cấp số tài khoản tự động sau khi hệ thống hoạt động bình thường.

---

## PHẦN 4: NON-FUNCTIONAL REQUIREMENTS (YÊU CẦU PHI CHỨC NĂNG)

### 4.1 Security (Bảo mật thông tin)
* **Mã hóa đường truyền**: Toàn bộ dữ liệu trao đổi giữa Client Mobile App, API Gateway và các máy chủ nội bộ phải sử dụng giao thức HTTPS tối thiểu phiên bản TLS 1.3 và triển khai cơ chế SSL Pinning trên Client để ngăn chặn tấn công nghe lén dữ liệu (Man-in-the-Middle).
* **Mã hóa lưu trữ**: Dữ liệu định danh cá nhân nhạy cảm (PII) gồm: Họ tên, số định danh, ngày sinh, ảnh khuôn mặt và ảnh CCCD phải được mã hóa ở mức cơ sở dữ liệu bằng thuật toán mã hóa đối xứng AES-256. Khóa mã hóa phải được quản lý bởi hệ thống quản lý khóa tập trung (Key Management System - KMS).
* **Tokenization**: Sử dụng định dạng JWT (JSON Web Token) cho mỗi phiên giao dịch eKYC với thời gian sống của token (TTL - Time to Live) tối đa là 15 phút.

### 4.2 Performance (Hiệu năng vận hành)
* **Thời gian phản hồi (Response Time)**:
  - API nhận diện và xác thực NFC: $\le 1.5$ giây/giao dịch.
  - API OCR trích xuất dữ liệu: $\le 1.0$ giây/giao dịch.
  - API Liveness Check và Face Matching: $\le 2.0$ giây/giao dịch.
* **Thời gian hoàn thành quy trình (Customer Journey Time)**: Tổng thời gian hoàn thành 1 phiên eKYC tính từ thời điểm nhập số điện thoại đến lúc nhận số tài khoản thành công trên app phải $\le 120$ giây.

### 4.3 Availability (Tính sẵn sàng & Chịu tải)
* **Chỉ số SLA Uptime**: Hệ thống Backend eKYC Gateway và Orchestrator phải đảm bảo hoạt động liên tục 24/7/365 với tỷ lệ sẵn sàng tối thiểu là **99.9%** (tổng thời gian gián đoạn hệ thống không được vượt quá 8.76 giờ/năm).
* **Khả năng chịu tải (Scalability)**:
  - Hệ thống hỗ trợ xử lý tải đồng thời tối thiểu **1.000 TPS** (Transactions Per Second) tại thời điểm cao điểm.
  - Tích hợp tính năng Auto-scaling trên nền tảng Kubernetes để tăng số lượng pod xử lý khi tải của CPU vượt mức 70%.

---

## PHẦN 5: VISUAL DIAGRAM (SƠ ĐỒ TRỰC QUAN)

Dưới đây là mã Mermaid cho sơ đồ Use Case Diagram của Hệ thống eKYC ABC Bank mô tả đầy đủ các mối tương tác của các Tác nhân (Actors) và các module chức năng chính:

```mermaid
usecaseDiagram
    actor Customer as "Khách hàng"
    actor CoreSystem as "Hệ thống Core Banking"
    actor ThirdPartyC06 as "API Xác thực C06 (Bộ Công An)"
    actor AMLPEP as "Hệ thống AML/PEP Screening"
    actor SMSGateway as "SMS/Email Gateway"

    subgraph ABC_Bank_eKYC_System ["Hệ thống eKYC ABC Bank"]
        usecase UC01 as "UC-01: Xác thực Số điện thoại & OTP"
        usecase UC02 as "UC-02: Quét NFC & Xác thực chữ ký CCCD"
        usecase UC03 as "UC-03: Chụp ảnh & Nhận dạng OCR (Backup)"
        usecase UC04 as "UC-04: Xác thực Liveness & So khớp khuôn mặt"
        usecase UC05 as "UC-05: Kiểm tra AML/PEP & Danh sách đen"
        usecase UC06 as "UC-06: Phê duyệt tự động & Mở tài khoản (STP)"
    end

    Customer --> UC01
    Customer --> UC02
    Customer --> UC03
    Customer --> UC04

    UC01 --> SMSGateway : "Gửi/Xác thực mã OTP"
    UC02 --> ThirdPartyC06 : "Xác thực chữ ký số CA chip"
    UC05 --> AMLPEP : "Tra cứu thông tin cấm vận/PEP"
    UC06 --> CoreSystem : "Đồng bộ tạo CIF & Tài khoản"
    UC06 --> SMSGateway : "Gửi mật khẩu đăng nhập tạm thời"

    UC02 ..> UC03 : "<<extend>> (Nếu lỗi NFC)"
    UC06 ..> UC05 : "<<include>> (Đánh giá rủi ro khách hàng)"
```

---
**Tài liệu này được biên soạn bởi Senior Business Analyst - ABC Bank Digital Transformation Team.**
