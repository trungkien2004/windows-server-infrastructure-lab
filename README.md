# Triển khai Active Directory Domain Services trên Windows Server 2019
### Xây dựng hệ thống quản lý tập trung cho mô hình doanh nghiệp: từ Domain Controller đến phân quyền tài nguyên theo phòng ban

---

## 1. Mục tiêu dự án

Dự án cá nhân mô phỏng toàn bộ vòng đời triển khai một hệ thống **Domain** cho doanh nghiệp vừa và nhỏ, sử dụng **Windows Server 2019** và **Active Directory Domain Services (AD DS)**. Mục tiêu là thực hành đúng quy trình chuẩn mà một **IT Support / Helpdesk / System Administrator** cần nắm vững:

- Cấu hình hạ tầng mạng và cài đặt Domain Controller đầu tiên cho forest.
- Thiết kế cấu trúc tổ chức (OU) và nhóm bảo mật theo mô hình phòng ban thực tế.
- Thiết lập chính sách bảo mật tài khoản và kiểm soát truy cập.
- Join máy trạm vào domain, xử lý sự cố phát sinh ngoài kịch bản chuẩn.
- Chia sẻ tài nguyên và phân quyền NTFS theo nguyên tắc *least privilege*.
- Kiểm thử toàn diện đầu-cuối (end-to-end) trên máy trạm thực tế.

## 2. Môi trường triển khai

| Thành phần | Thông tin |
|---|---|
| Hệ điều hành Server | Windows Server 2019 Standard Evaluation |
| Ảo hóa | VMware |
| Cấu hình máy chủ | 4 GB RAM, 59.4 GB ổ đĩa |
| Máy trạm Client | Windows 10 Education |
| Domain | `hactech.vn` |
| NetBIOS name | `HACTECH` |
| Vai trò cài đặt | AD DS, DNS Server |

## 3. Sơ đồ cấu trúc Active Directory hoàn chỉnh

```
hactech.vn (HACTECH)
└── Hanoi (OU)
    ├── Lanh dao          (Security Group - Global) → Lanhdao1, Lanhdao2
    ├── Nhan vien         (Security Group - Global) → Nhan vien 1/2, Nhanvien 3
    ├── Nhan Su (OU)
    │     └── Nhan su     (Security Group - Global) → Nhansu 1, Nhansu 2
    ├── Ke Toan (OU)
    │     └── Ke Toan     (Security Group - Global) → Ke toan 1, Ke toan 2
    ├── Kinh doanh (OU)
    │     └── Kinh Doanh  (Security Group - Global) → Kinhdoanh 1, Kinh doanh 2
    └── Administrators (Built-in) ← chứa "Lanh dao" (nested group)
```

**Quy mô:** 4 OU · 5 Security Group · 11 tài khoản người dùng · 2 lớp phân quyền NTFS (Public/Private)

---

## 4. Quy trình thực hiện

### PHẦN A — Triển khai Domain Controller

#### Bước 1 — Cấu hình địa chỉ IP tĩnh cho máy chủ

Trước khi triển khai Domain Controller, máy chủ cần địa chỉ IP tĩnh cố định để đảm bảo tính ổn định cho các dịch vụ mạng (DNS, DHCP...). Tại **Network Connections → Ethernet0 Properties → TCP/IPv4**, thiết lập IP `192.168.10.10`, Subnet mask `255.255.255.0`, Preferred DNS server `127.0.0.1` (trỏ về chính máy chủ, chuẩn bị cho vai trò DNS Server).

![Mở cấu hình Ethernet](winserver2019%20img/img1.png)
![Chọn TCP/IPv4](winserver2019%20img/img2.png)
![Thiết lập IP tĩnh](winserver2019%20img/img3.png)

#### Bước 2 — Cài đặt vai trò Active Directory Domain Services

Vào **Server Manager → Add Roles and Features Wizard**, chọn vai trò **Active Directory Domain Services**. Sau khi cài đặt, hệ thống thông báo cần bước **"Promote this server to a domain controller"** để hoàn tất kích hoạt.

![Server Manager Dashboard trước khi cấu hình](winserver2019%20img/img4.png)
![Add Roles and Features Wizard](winserver2019%20img/img5.png)
![Cài đặt AD DS thành công](winserver2019%20img/img6.png)

#### Bước 3 — Nâng cấp máy chủ thành Domain Controller

Sử dụng **Active Directory Domain Services Configuration Wizard**:
- **Deployment Configuration:** chọn *Add a new forest*, Root domain `hactech.vn`
- **Domain Controller Options:** Forest/Domain functional level Windows Server 2016, bật **DNS Server** và **Global Catalog**, thiết lập mật khẩu **DSRM** (Directory Services Restore Mode)
- **Additional Options:** xác nhận NetBIOS domain name `HACTECH`
- **Paths:** giữ đường dẫn mặc định cho AD DS database, log files, SYSVOL
- Ghi nhận và xử lý cảnh báo bảo mật liên quan cryptography compatibility (NTLM) theo khuyến nghị Microsoft

![Deployment Configuration - Add a new forest](winserver2019%20img/img7.png)
![Domain Controller Options](winserver2019%20img/img8.png)
![Additional Options - NetBIOS name](winserver2019%20img/img9.png)
![Paths - đường dẫn AD DS](winserver2019%20img/img10.png)
![Installation - cảnh báo bảo mật](winserver2019%20img/img11.png)

#### Bước 4 — Xác nhận Domain Controller hoạt động

Sau khi cài đặt, máy chủ tự khởi động lại. Đăng nhập thành công với `HACTECH\Administrator`, xác nhận domain đã kích hoạt và máy chủ chính thức là Domain Controller của forest `hactech.vn`.

![Đăng nhập HACTECH\Administrator](winserver2019%20img/img12.png)
![Local Server - Services cần kiểm tra](winserver2019%20img/img13.png)
![All Servers - Manageability](winserver2019%20img/img15.png)

#### Bước 5 — Xử lý sự cố mạng phát sinh sau khi promote

Sau khi promote thành Domain Controller, hệ thống báo lỗi **"Unidentified network – No Internet"** kèm **"Windows cannot access the specified device, path, or file"** khi mở lại cấu hình Ethernet qua Settings.

**Cách xử lý:**
- Xác định nguyên nhân: quyền truy cập bị giới hạn do Windows Defender Firewall chuyển từ *Public* sang *Domain: On* sau khi promote
- Chuyển sang thao tác qua **Control Panel → Network Connections** (thay vì Settings app) để có đủ quyền chỉnh sửa
- Cấu hình lại **Preferred DNS server** trỏ về chính IP máy chủ (`192.168.10.10`) thay vì `127.0.0.1`, đảm bảo phân giải tên miền nội bộ chính xác sau khi DNS Server role được kích hoạt

> Đây là bước thể hiện khả năng **chẩn đoán và xử lý sự cố thực tế** — không chỉ làm theo wizard mà còn phát hiện và khắc phục lỗi phát sinh ngoài kịch bản.

![Lỗi truy cập Settings sau khi promote domain](winserver2019%20img/img16.png)
![Cấu hình lại DNS trỏ về chính máy chủ](winserver2019%20img/img18.png)

#### Bước 6 — Kiểm tra cấu hình Local Server

Xác nhận qua **Server Manager → Local Server**: Computer name `WIN-E63VGIQ297S`, Domain `hactech.vn`, Windows Defender Firewall **Domain: On**, Ethernet0 `192.168.10.10`, hệ điều hành Windows Server 2019 Standard Evaluation.

![Local Server Properties - xác nhận domain và IP](winserver2019%20img/img17.png)

#### Bước 7 — Khảo sát Active Directory Users and Computers

Mở **ADUC**, khảo sát các nhóm bảo mật mặc định trong domain (`Domain Admins`, `Domain Users`, `DnsAdmins`, `Enterprise Admins`, `Group Policy Creator Owners`...) và các thao tác **New → Organizational Unit / User / Group / Computer** để chuẩn bị phân chia cấu trúc tổ chức.

![Danh sách nhóm bảo mật mặc định trong AD](winserver2019%20img/img19.png)
![Tạo mới OU / User / Group](winserver2019%20img/img20.png)

---

### PHẦN B — Xây dựng cấu trúc OU và tài khoản người dùng

#### Bước 8 — Tạo Organizational Unit (OU) theo chi nhánh

Chuột phải vào domain `hactech.vn` → **New → Organizational Unit**, đặt tên `Hanoi`, giữ tùy chọn **"Protect container from accidental deletion"** (thực hành chuẩn của Microsoft, ngăn xóa nhầm OU chứa dữ liệu quan trọng).

> OU là đơn vị tổ chức logic trong AD, cho phép nhóm user/computer theo phòng ban hoặc khu vực địa lý, từ đó áp dụng Group Policy và phân quyền quản trị (delegation) riêng biệt mà không ảnh hưởng toàn domain.

![Tạo OU Hanoi](winserver2019%20img/img21.png)
![Menu New Object trong OU](winserver2019%20img/img22.png)

#### Bước 9 — Tạo tài khoản người dùng theo vai trò

Trong OU `Hanoi`, tạo tài khoản đại diện 2 nhóm vai trò: **Nhân viên** và **Lãnh đạo**.

| User logon name | Full Name | Vai trò |
|---|---|---|
| nhanvien1 | Nhan vien 1 | Nhân viên |
| nhanvien2 | Nhanvien 2 | Nhân viên |
| nhanvien3 | Nhanvien 3 | Nhân viên |
| lanhdao1 | Lanhdao1 | Lãnh đạo |
| lanhdao2 | Lanhdao2 | Lãnh đạo |

Với mỗi user, thiết lập mật khẩu và tick **"User must change password at next logon"** — chính sách bảo mật chuẩn, buộc người dùng đổi mật khẩu cá nhân trong lần đăng nhập đầu tiên thay vì dùng chung mật khẩu do quản trị viên cấp.

> Việc tách riêng tài khoản Nhân viên và Lãnh đạo ngay từ đầu là bước chuẩn bị cần thiết để tạo **Security Group** theo vai trò, từ đó áp dụng Group Policy và phân quyền tài nguyên (NTFS Permission) khác nhau cho từng nhóm.

![Tạo user Nhan vien 1](winserver2019%20img/img23.png)
![Tạo user Nhan vien 2](winserver2019%20img/img24.png)
![Tạo user Nhan vien 3](winserver2019%20img/img25.png)
![Thiết lập mật khẩu Nhân viên](winserver2019%20img/img26.png)
![Xác nhận 3 tài khoản Nhân viên](winserver2019%20img/img27.png)
![Tạo user Lanhdao1](winserver2019%20img/img28.png)
![Tạo user Lanhdao2](winserver2019%20img/img29.png)
![Xác nhận tài khoản Lanhdao2](winserver2019%20img/img30.png)
![Thiết lập mật khẩu Lãnh đạo](winserver2019%20img/img31.png)

---

### PHẦN C — Nhóm bảo mật, phân quyền quản trị và chính sách tài khoản

#### Bước 10 — Kiểm tra cấu trúc OU hiện có

Xác định OU `Hanoi` đã có sẵn 5 tài khoản người dùng: `Lanhdao1`, `Lanhdao2`, `Nhan vien 1`, `Nhan vien 2`, `Nhanvien 3`.

![OU Hanoi và danh sách User](winserver2019%20img/img32.png)

#### Bước 11 — Tạo Security Group "Nhan vien"

Chuột phải OU `Hanoi` → **New → Group**, đặt tên `Nhan vien`, **Group scope: Global**, **Group type: Security** — phù hợp để cấp quyền truy cập tài nguyên trong cùng domain và có thể lồng vào nhóm khác.

![Tạo Security Group Nhan vien](winserver2019%20img/img33.png)

#### Bước 12 — Bổ sung người dùng vào nhóm "Nhan vien"

Chọn đồng thời 3 tài khoản `Nhan vien 1`, `Nhan vien 2`, `Nhanvien 3` → **Add to a group…** — gán nhiều user vào nhóm cùng lúc, tiết kiệm thời gian so với chỉnh sửa từng tài khoản.

![Add to a group](winserver2019%20img/img34.png)
![Chọn nhóm Nhan vien](winserver2019%20img/img35.png)
![Nhập tên nhóm - Check Names](winserver2019%20img/img36.png)

#### Bước 13 — Tạo Security Group "Lanh dao" và bổ sung thành viên

Tương tự Bước 11–12, tạo nhóm `Lanh dao` (Global Security Group), thêm `Lanhdao1`, `Lanhdao2`. Kết quả: OU `Hanoi` có đầy đủ 2 Security Group và 5 tài khoản người dùng.

![Chọn nhóm Lanh dao](winserver2019%20img/img37.png)
![Check Names - Lanh dao](winserver2019%20img/img38.png)
![Kết quả tạo nhóm](winserver2019%20img/img39.png)

#### Bước 14 — Lồng nhóm "Lanh dao" vào "Administrators" (Nested Group)

Để cấp quyền quản trị cục bộ cho phòng Lãnh đạo, thêm nhóm `Lanh dao` làm thành viên của nhóm built-in `Administrators`. Kiểm tra qua tab **Member Of** để xác nhận.

![Chọn nhóm Administrators](winserver2019%20img/img40.png)
![Nhập Administrators - Check Names](winserver2019%20img/img41.png)
![Xác nhận Member Of](winserver2019%20img/img42.png)

> **Lưu ý bảo mật:** Gán nhóm vào `Administrators` chỉ nên áp dụng khi thực sự cần quyền quản trị cục bộ. Trong môi trường thực tế nên tuân thủ nguyên tắc *least privilege*.

#### Bước 15 — Cấu hình hàng loạt Logon Hours

Chọn đồng thời `Nhan vien 1/2`, `Nhanvien 3` → **Properties → Account**, tick **Logon hours**, thiết lập khung giờ được phép đăng nhập — biện pháp hạn chế đăng nhập ngoài giờ hành chính.

![Properties for Multiple Items](winserver2019%20img/img43.png)
![Cấu hình Logon Hours](winserver2019%20img/img44.png)

#### Bước 16 — Thiết lập ngày hết hạn tài khoản (Account Expires)

Với tài khoản `Nhanvien3`, thiết lập **Account expires → End of <ngày>**, kết hợp bắt buộc đổi mật khẩu lần đầu — biện pháp phù hợp cho tài khoản thử việc/hợp đồng ngắn hạn.

![Thiết lập Account Expires cho Nhanvien3](winserver2019%20img/img45.png)

---

### PHẦN D — Cấu hình máy trạm (Client) và kiểm thử gia nhập miền

#### Bước 17 — Kiểm tra thông tin máy trạm

**Settings → System → About**: tên máy `DESKTOP-G513SOF`, hệ điều hành Windows 10 Education.

![Thông tin máy trạm Windows 10](winserver2019%20img/img46.png)

#### Bước 18 — Cấu hình IP tĩnh trỏ về Server

Cấu hình IP tĩnh cùng dải mạng với Domain Controller, **Preferred DNS server** trỏ về `192.168.10.10` — điều kiện bắt buộc để phân giải tên miền và join domain thành công.

![Cấu hình IP tĩnh và DNS trỏ về Server](winserver2019%20img/img47.png)

#### Bước 19 — Kiểm tra kết nối tới Domain Controller

`ping` tới IP Server để xác nhận đường truyền thông suốt trước khi join domain.

![Ping kiểm tra kết nối tới Domain Controller](winserver2019%20img/img48.png)

#### Bước 20 — Join máy trạm vào Domain

Tại **System Properties → Computer Name → Change…**, chọn Domain, nhập `hactech.vn`.

![Computer Name/Domain Changes](winserver2019%20img/img49.png)
![Nhập tài khoản để join domain](winserver2019%20img/img50.png)

**Lưu ý quan trọng:** Dùng tài khoản `lanhdao1` (chỉ thuộc nhóm `Lanh dao`, không có quyền join computer) → báo lỗi xác thực:

![Lỗi sai tài khoản/mật khẩu khi join domain](winserver2019%20img/img51.png)

> **Giải thích:** Chỉ tài khoản thuộc **Domain Admins** hoặc được delegate quyền **"Add workstations to domain"** mới join được máy trạm vào domain — minh họa thực tế nguyên tắc phân quyền trong AD.

Thực hiện lại với `administrator` (đủ quyền domain) → join thành công.

![Join domain thành công với tài khoản Administrator](winserver2019%20img/img52.png)

#### Bước 21 — Kiểm thử đăng nhập bằng tài khoản domain

Khởi động lại máy trạm, chọn **Other user**, đăng nhập theo cú pháp `hactech\<username>`. Đăng nhập thành công với `Lanhdao2`.

![Đăng nhập domain với tài khoản Lanhdao2](winserver2019%20img/img53.png)
![Xác thực mật khẩu thành công](winserver2019%20img/img54.png)

#### Bước 22 — Kiểm chứng chính sách Account Expires

Thử đăng nhập bằng `nhanvien3` (đã cấu hình Account Expires ở Bước 16) — hệ thống từ chối với thông báo **"The user's account has expired."**, xác nhận chính sách được AD thực thi chính xác trên máy trạm.

![Đăng nhập bằng tài khoản nhanvien3](winserver2019%20img/img55.png)
![Tài khoản đã hết hạn - đăng nhập bị từ chối](winserver2019%20img/img56.png)

---

### PHẦN E — Mở rộng cấu trúc OU theo phòng ban

#### Bước 23 — Tạo các OU phòng ban: Nhân Sự, Kế Toán, Kinh Doanh

Chuột phải OU `Hanoi` → **New → Organizational Unit**, tạo lần lượt `Nhan Su`, `Ke Toan`, `Kinh doanh`.

![Tạo mới Organizational Unit](winserver2019%20img/img57.png)
![Đặt tên OU Nhan Su](winserver2019%20img/img59.png)
![3 OU phòng ban vừa tạo](winserver2019%20img/img60.png)
![Kiểm tra OU Kinh doanh trống](winserver2019%20img/img61.png)

#### Bước 24 — Tạo User và Security Group cho từng phòng ban

Áp dụng đúng quy trình ở Bước 11–13 cho từng OU phòng ban:

- **Phòng Nhân Sự** — nhóm `Nhan su` + `Nhansu 1`, `Nhansu 2`
- **Phòng Kế Toán** — nhóm `Ke Toan` + `Ke toan 1`, `Ke toan 2`
- **Phòng Kinh Doanh** — nhóm `Kinh Doanh` + `Kinhdoanh 1`, `Kinh doanh 2`

![OU Nhan Su với User và Group](winserver2019%20img/img62.png)
![OU Ke Toan với User và Group](winserver2019%20img/img63.png)
![OU Kinh doanh với User và Group](winserver2019%20img/img64.png)

> Tách riêng OU theo phòng ban giúp áp dụng **GPO** độc lập cho từng bộ phận (chính sách máy in, mapping ổ đĩa mạng, hạn chế Control Panel…) mà không ảnh hưởng phòng ban khác — mô hình chuẩn trong hạ tầng doanh nghiệp thực tế.

---

### PHẦN F — Chia sẻ tài nguyên và phân quyền NTFS nâng cao

#### Bước 25 — Đăng nhập quản trị để cấu hình chia sẻ dữ liệu

Đăng nhập máy trạm bằng `HACTECH\administrator`.

![Đăng nhập bằng tài khoản Administrator](winserver2019%20img/img65.png)

#### Bước 26 — Khôi phục cấu hình mạng về DHCP

Chuyển TCP/IPv4 về **Obtain an IP address automatically / Obtain DNS server address automatically** — áp dụng khi vận hành thực tế với DHCP Server.

![Chuyển về Obtain IP tự động](winserver2019%20img/img66.png)

#### Bước 27 — Kiểm tra thư mục chia sẻ dữ liệu chung

Kiểm tra thư mục **Public** (`E:\HaNoi\Ketoan\Public`) chứa `Tailieuchung` — cấu hình **File Sharing** kết hợp **NTFS Permissions** theo nhóm bảo mật.

![Thư mục chia sẻ Public/Tailieuchung](winserver2019%20img/img67.png)

#### Bước 28 — Kiểm thử đăng nhập với tài khoản phòng ban mới

Đăng nhập `hactech\nhansu1` — thành công, hiển thị đúng tên **"Nhan Su 1"**.

![Đăng nhập bằng tài khoản nhansu1](winserver2019%20img/img68.png)
![Đăng nhập thành công tài khoản Nhan Su 1](winserver2019%20img/img69.png)

#### Bước 29 — Tổng quan cấu trúc Active Directory hoàn chỉnh

OU `Hanoi` bao gồm đầy đủ các OU phòng ban (`Ke Toan`, `Kinh Doanh`, `Nhan Su`) cùng Security Group và tài khoản tương ứng, sẵn sàng triển khai GPO và phân quyền tài nguyên.

![Tổng quan cấu trúc Active Directory hoàn chỉnh](winserver2019%20img/img70.png)

#### Bước 30 — Gỡ kế thừa quyền (Disable Inheritance) trên thư mục Public

Tại **Advanced Security Settings for Public**, nhấn **Disable inheritance** → chọn **Convert inherited permissions into explicit permissions on this object** — giữ nguyên quyền đang kế thừa nhưng chuyển thành quyền tường minh, làm nền tảng tùy chỉnh phân quyền riêng cho từng nhóm.

![Block Inheritance - Convert inherited permissions](winserver2019%20img/img74.png)

#### Bước 31 — Cấu hình quyền truy cập cho từng nhóm trên Public

Cấp quyền **Modify** cho `G_Ketoan` (Read & execute, List folder contents, Read, Write). Kết quả: `Administrators` → **Full control**; `G_Ketoan` → **Modify**; `G_Kinhdoanh`, `G_Nhansu` → **Read & execute** — mọi phòng ban xem được tài liệu chung nhưng chỉ Kế toán được chỉnh sửa.

![Cấu hình Permission Entry cho nhóm G_Ketoan](winserver2019%20img/img75.png)
![Kết quả phân quyền NTFS trên Public](winserver2019%20img/img76.png)

#### Bước 32 — Giới hạn quyền riêng tư trên thư mục Private

Thư mục `Private` (dữ liệu nội bộ Kế toán) chỉ cấp **Read & execute** cho `G_Ketoan`, không cấp cho nhóm phòng ban nào khác — đảm bảo dữ liệu nhạy cảm không bị truy cập ngoài phạm vi cho phép.

![Phân quyền giới hạn trên thư mục Private](winserver2019%20img/img77.png)

#### Bước 33 — Kết nối thư mục mạng bằng Map Network Drive

Đăng nhập lại `nhansu1` để xác nhận trạng thái ổn định sau thay đổi cấu hình. Trên máy trạm, dùng **Map Network Drive** trỏ tới `\\DC\HaNoi`, gán ổ đĩa `Z:` — giúp truy cập thư mục dùng chung thuận tiện như ổ đĩa cục bộ.

![Đăng nhập lại tài khoản Nhan Su 1](winserver2019%20img/img78.png)
![Map Network Drive tới \\DC\HaNoi](winserver2019%20img/img79.png)

#### Bước 34 — Kiểm thử truy cập thực tế với tài khoản phòng Kế toán

Đăng nhập `hactech\ketoan1`, truy cập ổ đĩa mạng `Z:` → mở `Ketoan\Public` → thấy `Tailieuchung`, đúng theo quyền **Modify** đã cấp ở Bước 31. Đối chiếu với thư mục vật lý trên Server cho kết quả nhất quán, xác nhận **File Sharing + NTFS Permissions** hoạt động chính xác.

![Đăng nhập bằng tài khoản ketoan1](winserver2019%20img/img80.png)
![Truy cập Public qua ổ đĩa mạng đã map](winserver2019%20img/img81.png)
![Đối chiếu thư mục Public trên Server](winserver2019%20img/img82.png)

---

## 5. Kết quả đạt được

- ✅ Triển khai thành công **Domain Controller** đầu tiên cho forest `hactech.vn` trên Windows Server 2019, tích hợp vai trò **DNS Server**, với hệ thống mạng nội bộ ổn định (IP tĩnh + DNS nội bộ).
- ✅ Tự phát hiện và xử lý **2 sự cố thực tế** phát sinh ngoài kịch bản chuẩn: lỗi mất kết nối mạng sau khi promote Domain Controller, và lỗi xác thực khi join domain do tài khoản không đủ quyền.
- ✅ Xây dựng cấu trúc **4 OU** (`Hanoi`, `Nhan Su`, `Ke Toan`, `Kinh doanh`) và **5 Security Group** theo mô hình phòng ban, áp dụng nested group để phân cấp quyền quản trị.
- ✅ Cấu hình chính sách bảo mật tài khoản: **Logon Hours**, **Account Expires**, bắt buộc đổi mật khẩu lần đầu — kiểm thử và xác nhận hoạt động chính xác trên máy trạm thực tế.
- ✅ Join thành công máy trạm Windows 10 vào domain, xử lý đúng lỗi phân quyền phát sinh khi join bằng tài khoản không hợp lệ.
- ✅ Triển khai mô hình phân quyền **NTFS 2 lớp (Public/Private)** theo nguyên tắc *least privilege*, kết hợp **Mapped Network Drive**, kiểm thử xác nhận đúng theo thiết kế cho toàn bộ **11 tài khoản người dùng**.

## 6. Kỹ năng thể hiện

| Kỹ năng | Mô tả |
|---|---|
| Quản trị hệ thống Windows Server | Cài đặt, cấu hình vai trò máy chủ, promote Domain Controller |
| Active Directory | Triển khai forest/domain, thiết kế OU, quản lý user & Security Group |
| Networking cơ bản | Cấu hình IP tĩnh, DNS nội bộ, cơ chế Domain Firewall |
| Access Control | Logon Hours, Account Expires, NTFS Permissions, least privilege |
| File Sharing | Cấu hình share folder, phân quyền theo nhóm, Mapped Network Drive |
| Domain Join & Troubleshooting | Join máy trạm, xử lý lỗi mạng và lỗi phân quyền thực tế |
| Tư duy quy trình | Tuân thủ best practice Microsoft (DSRM, functional level, NetBIOS...) |

## 7. Tổng kết dự án

Dự án mô phỏng đầy đủ vòng đời quản trị **Active Directory** trong doanh nghiệp vừa và nhỏ:

1. **Triển khai hạ tầng** — cài đặt AD DS, promote Domain Controller, cấu hình DNS nội bộ.
2. **Thiết kế cấu trúc tổ chức** — 4 OU theo khu vực/phòng ban, 5 Security Group theo vai trò.
3. **Chính sách bảo mật tài khoản** — Logon Hours, Account Expires, nested group.
4. **Triển khai & xử lý sự cố client** — join domain, khắc phục lỗi mạng và lỗi phân quyền thực tế.
5. **Chia sẻ tài nguyên & phân quyền NTFS nâng cao** — mô hình Public/Private, Mapped Network Drive.
6. **Kiểm thử toàn diện** — xác nhận mọi chính sách hoạt động đúng trên máy trạm thực tế.

Toàn bộ quy trình được thực hiện và kiểm chứng bằng hình ảnh thực tế trên **Windows Server 2019 + Windows 10 Client**, thể hiện năng lực quản trị hệ thống toàn diện — từ hoạch định, triển khai, bảo mật đến kiểm thử.

---

## 8. Gợi ý mô tả trong CV

> Có thể sử dụng trực tiếp hoặc điều chỉnh 4 gạch đầu dòng sau cho mục **Dự án cá nhân / Kinh nghiệm thực hành** trong CV:

- Triển khai hạ tầng **Active Directory Domain Services** trên Windows Server 2019 cho mô hình doanh nghiệp mô phỏng, xây dựng cấu trúc **4 OU và 5 Security Group** theo phòng ban, giúp áp dụng Group Policy và phân quyền tài nguyên độc lập cho từng bộ phận mà không ảnh hưởng chéo.
- Tự phát hiện và xử lý thành công **2 sự cố kỹ thuật thực tế** phát sinh ngoài tài liệu hướng dẫn (mất kết nối mạng sau khi promote Domain Controller, lỗi xác thực khi join domain do sai phân quyền tài khoản), thể hiện năng lực troubleshooting độc lập không phụ thuộc quy trình mẫu.
- Thiết kế và triển khai mô hình phân quyền **NTFS 2 lớp (Public/Private)** theo nguyên tắc *least-privilege* cho **11 tài khoản** thuộc 5 phòng ban, kết hợp Mapped Network Drive — đảm bảo dữ liệu nội bộ chỉ truy cập đúng phạm vi được cấp quyền.
- Cấu hình và kiểm thử đầu-cuối các chính sách bảo mật tài khoản (**Logon Hours, Account Expires**, bắt buộc đổi mật khẩu lần đầu) trên máy trạm Windows 10 đã join domain, xác nhận **100% chính sách được Active Directory thực thi chính xác** trong môi trường thực tế.

---

*Dự án cá nhân thực hiện nhằm mục đích học tập và minh họa năng lực triển khai, vận hành hạ tầng CNTT doanh nghiệp — phù hợp với vị trí IT Support / Helpdesk / System Administrator.*
