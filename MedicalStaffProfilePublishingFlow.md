**Đối với hồ sơ cá nhân:**

Việc public hồ sơ sẽ phát sinh **phí đăng tin** và chỉ được tính **một lần duy nhất** cho mỗi profile, tại thời điểm người dùng đủ điều kiện đăng hồ sơ.

**Điều kiện được phép đăng hồ sơ:**

* Đối với NVYT:

  * Sau khi hoàn tất bước duyệt thông tin lần 1, NVYT cần nhập đầy đủ 4 mục thông tin hồ sơ.
  * Cả 4 mục này phải được approve.
  * Có thể hiểu đơn giản là phải tồn tại **4 record tương ứng với 4 section đã được approve** trong bảng `updatedoctorprofile`.

**Luồng xử lý:**

1. Khi đủ điều kiện, nút **Đăng hồ sơ** sẽ được kích hoạt (active).
2. Khi người dùng click:

   * Hệ thống tổng hợp toàn bộ chi phí cần thanh toán của hồ sơ.
   * Kiểm tra số token hiện có của user:

     * Nếu không đủ → chuyển sang trang thanh toán, hiển thị các khoản phí tương ứng.
3. Sau khi thanh toán thành công:

   * Server nhận callback từ VietQR.
   * Thực hiện fulfill order.
   * Xác định profile tương ứng và tiến hành **public profile**.
4. Sau khi profile đã được public:

   * Nút **Đăng hồ sơ** sẽ chuyển thành **Gia hạn tin**.
   * Các chi phí phát sinh sau đó (service, ảnh bổ sung,...) sẽ:

     * Được kiểm tra token ngay tại thời điểm người dùng lưu cập nhật.
     * Nếu không đủ token → thực hiện luồng thanh toán tương ứng cho từng section.
```mermaid
flowchart TD
    %% ===== ĐĂNG HỒ SƠ LẦN ĐẦU =====
    Start([Bắt đầu]) --> CheckCondition{Đã hoàn tất\n4 mục approve?}
    
    CheckCondition -->|Chưa| NotReady[Chưa active nút]
    NotReady --> CheckCondition
    
    CheckCondition -->|Đủ điều kiện| EnableBtn[Active nút Đăng hồ sơ]
    
    EnableBtn --> UserClick[User click Đăng hồ sơ]
    UserClick --> CalcFee[Tính tổng chi phí đăng tin\n1 lần duy nhất]
    
    CalcFee --> CheckToken1{Token đủ?}
    
    CheckToken1 -->|Đủ| DoPublic[Public profile]
    CheckToken1 -->|Không đủ| GotoPayment[Chuyển sang trang thanh toán]
    
    GotoPayment --> ShowFee[Hiển thị các khoản phí]
    ShowFee --> PayViaQR[User thanh toán qua VietQR]
    PayViaQR --> Callback[Server nhận callback]
    Callback --> Fulfill[Fulfill order]
    Fulfill --> DoPublic
    
    DoPublic --> ChangeBtn[Đổi nút thành\nGia hạn tin]
    ChangeBtn --> EndPublic([Kết thúc đăng lần đầu])
    
    %% ===== LUỒNG CẬP NHẬT SAU PUBLIC =====
    EndPublic --> UpdateStart[User cập nhật hồ sơ\nthêm service / ảnh]
    
    UpdateStart --> CalcExtra[Tính chi phí phát sinh\ntheo từng section]
    CalcExtra --> CheckToken2{Token đủ?}
    
    CheckToken2 -->|Đủ| SaveSuccess[Lưu cập nhật thành công]
    SaveSuccess --> MoreUpdate{Còn cập nhật khác?}
    
    MoreUpdate -->|Có| UpdateStart
    MoreUpdate -->|Không| EndUpdate([Kết thúc])
    
    CheckToken2 -->|Không đủ| TriggerPayment[Trigger thanh toán\ncho section đó]
    TriggerPayment --> PayExtra[Thanh toán qua VietQR]
    PayExtra --> Callback2[Callback + Fulfill order]
    Callback2 --> SaveSuccess
