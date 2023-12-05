# CVE-2021-44790

## Mô tả CVE-2021-44790

- CVE-2021-44790 mô tả một lỗi bảo mật cấp độ hệ thống nằm trong mod_lua của Apache HTTP Server, đặc biệt là trong phiên bản 2.4.51 và các phiên bản trước đó. Lỗ hổng này được kích hoạt khi hàm r:parsebody() xử lý các yêu cầu HTTP multipart/form-data không đúng cách, thường là do việc gửi dữ liệu form không chuẩn mực.
- Sự cố này không chỉ làm tăng khả năng xảy ra sự cố từ chối dịch vụ mà còn mở ra cơ hội cho việc thực thi mã độc từ xa qua các yêu cầu được cấu trúc kỹ lưỡng.

## **Thực nghiệm CVE-2023-25690**

- Sử dụng máy Ubuntu 22.04 làm máy tấn công
- Sử dụng Windows 10 64bit  làm server
    - Cài đặt phần mềm apache server VC11 2.4.38

## Mô tả hình thực nghiệm

![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/8ca6bdf3-128d-4746-9494-58c9c114ebed)


- Máy tấn công: Ubuntu 22.04 - IP: 10.6.10.130
- Máy nạn nhân: Windows 10 64bit -IP: 10.6.10.131

## **Yêu cầu hệ thống**

- Máy tấn công:
    - Hệ điều hành Ubuntu 22.04
    - Cài đặt python, Curl.
- Yêu cầu máy nạn nhân:
    - Hệ điều hành Windows 10 64bit
    - Cài đặt Apache 2.4.38
    - Cấu hình httpd.conf để có thể sử dụng modul Lua ( xem phần phụ lục)

**Mục tiêu khai thác: Lợi dụng lỗ hổng BufferOver flow để tiến hành DOS mục tiêu.**

## Triển khai thực nghiệm

- B1: Ở phía máy nạn nhân:
    - Tải apache 2.4.38 về máy.
    - Tiến hành cấu hình httpd.conf ( phụ lục I ).
    - Tạo file test.lua ở trong thư mục htdocs ( phụ lục I).
    - Trước khi sử dụng máy tấn công, ta cần tắt tường lửa của máy nạn nhân.
    - Sử dụng cmd  di chuyển đến thư mục :
    
    ```bash
    **Apache24/bin**  để tiến hành chạy **httpd.exe.**
    ```
    
- B2: Ở máy tấn công:
    - Thử kết nối tới trang web của nạn nhân ( Vì ở thực nghiệm là mạng local ⇒ kết nối tới IP của máy nạn nhân: 10.6.10.131 ).
    
    ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/0421ae62-e096-4c63-814b-50aa97b831de)

    
    - Khi thấy có thể kết nối được, tìm vị trí của file Lua ( Ở thực nghiệm sẽ là : 10.6.10.131/test.lua ).
    
    ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/1be0dd8a-11d7-4a2d-b27f-874d9ead1061)

    
    - Tiến hành gửi request khai thác đến vị trí đó  và quan sát kết quả:
        1. **Lỗi thiết lập lại kết nối ban đầu:**
            - Phần đầu tiên của nhật ký cho thấy **`ConnectionResetError104`**
                
                đã xảy ra sự cố với số lỗi cụ thể
                
                , có nghĩa là "Thiết lập lại kết nối ngang hàng". Điều này thường xảy ra khi máy chủ từ xa ("ngang hàng") nhận được yêu cầu kết nối của bạn nhưng sau đó đột ngột đóng kết nối.
                
        2. **Thư viện Urllib3 và yêu cầu:**
            - Tập lệnh dường như đang sử dụng **`requestsurllib3`**
                
                thư viện, do đó dựa vào đó
                
                để tạo kết nối HTTP.
                
            - Thư **`urllib3`**
                
                viện đang cố gắng mở kết nối và đưa ra yêu cầu nhưng không thành công do đã đặt lại kết nối.
                
        3. **Xử lý ngoại lệ trong thư viện:**
            - Cả hai **`urllib3requestsurllib3`**
                
                đều
                
                đang nắm bắt và nêu lại các ngoại lệ. Thư
                
                viện đang cố gắng tăng số lần thử lại cho yêu cầu, cho biết nó có thể được thiết lập để thử lại khi không thành công, nhưng cuối cùng lại bỏ cuộc sau khi hết số lần thử lại.
                
        4. **Lỗi kết nối cuối cùng trong yêu cầu:**
            - Ngoại lệ cuối cùng được **`requestsConnectionError`**
                
                thư viện đưa ra, đó là tệp
                
                . Đây là một ngoại lệ chung được đưa ra khi không thể thực hiện kết nối đến máy chủ, điều này có thể xảy ra nếu máy chủ đặt lại kết nối.
                

![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/31a360d1-0c11-4ed7-86a1-3426d23eb64a)


![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/70f3b4e0-6866-49d3-be0e-cf0d7de62905)


- B3: Ở máy nạn nhân:
    - Kiểm  tra log:
    - **`[Wed Nov 29 15:11:58.320403 2023] [mpm_winnt:notice] [pid 6224:tid 732] AH00428: Parent: child process 7220 exited with status 3221225477 -- Restarting.`**
    - Quá trình con 7220 đã thoát với mã lỗi **`3221225477`**. Đây có thể là lỗi do truy cập vùng nhớ không hợp lệ hoặc lỗi tương tự. Apache sau đó cố gắng khởi động lại quá trình.
    
    ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/9c0d00c7-5d9a-4195-85bf-4d4cca5b0f65)

    
    - Kiểm tra service:
        - Chi tiết lỗi đề cập rằng tên ứng dụng bị lỗi là **`httpd.exe`**, là tệp thực thi của máy chủ HTTP Apache. Tên mô-đun bị lỗi là **`MSVCR110.dll11.0.51106.1`**Mã ngoại lệ là **`0xc0000005`**, thường chỉ ra lỗi vi phạm quyền truy cập. Điều này có nghĩa là chương trình đã cố đọc hoặc ghi vào một vị trí bộ nhớ mà lẽ ra nó không nên làm, đây là nguyên nhân phổ biến gây ra sự cố ứng dụng.
    
    ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/e832e742-0191-4b02-8646-7b59d3e5629a)

    

## 

## Phụ lục I

- Cấu hình httpd.conf :
    - Sửa  lại đường dẫn
    
        ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/19a87f5a-f7ca-4126-93ea-8399df9179cf)

    
    - Khởi chạy module Lua
        
        ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/72658fa0-941b-4a38-8ed7-ba3e59180f33)

        
    - Thêm Handler cho module Lua
        
        ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/358611a2-149f-4fe7-98d4-159bbbf86aa2)

        
- Nội dung test.lua và đường dẫn:
    
    ![image](https://github.com/nuPacaChi/-CVE-2021-44790/assets/127914517/48eb6419-b472-4b0e-b627-b49fb3aebcd6)

