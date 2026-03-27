# Gói dịch vụ (Service): Entitlement mode, Purchase policy, Stackable, Credit pool group

Tài liệu mô tả **ý nghĩa từng trường** trên model `Service`, **luồng đang chạy trong code**, và **gợi ý cấu hình**.  
(Tra cứu thêm: `entitlement.py` — reserve/commit tin; `userservice.create_user_service_after_payment` — cấp quota sau thanh toán.)

---

## 1. `entitlement_mode`

**Vai trò:** Quyết định **cách tính số lượt (quota) cấp cho user** khi đơn hàng gói được thanh toán và tạo `UserServiceFeature` / `UserCreditBalance`.

| Giá trị | Ý nghĩa nghiệp vụ | Hành vi hiện tại trong code |
|--------|-------------------|-----------------------------|
| `credit_pool` | Gói dùng **kho lượt** (tin / ảnh / video / …) cộng dồn trong thời hạn gói — pattern phổ biến. | Công thức cấp phát: `allocated = quota_mỗi_dòng_feature × số_lượng_trong_order` (không nhân thêm `duration`). |
| `aggregate` | **Gộp theo cả kỳ**: lượt từ cấu hình feature được nhân theo độ dài gói (vd. mua 3 tháng → quota × 3). | Giống trên **và** nhân thêm `max(order_detail.duration, 1)` khi tạo allocation (xem `userservice.py`). |
| `monthly_bucket` | **Quota theo tháng** (reset hoặc bucket từng tháng). | Cấp phát giống `credit_pool` (không nhân `duration`). Metadata `UserServiceFeature` lưu `monthly_bucket`, `per_period_quantity`, `bucket_period_index`. Refill mỗi **kỳ 30 ngày** kể từ `user_service.start_date` (`monthly_bucket.sync_monthly_buckets_for_user` gọi khi xem tổng credit và khi `check_and_reserve_entitlements`). |

**Khi nào chọn gì**

- Hầu hết gói tin / VIP: **`credit_pool`**.
- Gói bán theo “đơn vị thời gian” mà **mỗi đơn vị thời gian phải nhân full bộ quota** (vd. 12 tháng = 12 × lượt tháng trong spec nghiệp vụ): **`aggregate`**.
- **`monthly_bucket`**: kỳ refill **theo block 30 ngày** từ `start_date` gói (khớp `resolve_end_date`). Nếu cần “tháng dương lịch” hoặc prorata ngày đầu kỳ, cần chỉnh riêng.

---

## 2. `purchase_policy`

**Vai trò:** Mô tả **chính sách được phép mua gói** (song song, khi hết quota, hay phải hủy gói cũ).

| Giá trị | Ý nghĩa |
|--------|---------|
| `allow_parallel` | Cho phép user **mua thêm** gói (thường song song với gói đang hiệu lực, tùy `is_stackable` và luồng thanh toán). |
| `allow_when_exhausted` | Chỉ nên mua / gia hạn khi **đã dùng hết** quota hoặc hết “room” theo quy định. |
| `allow_cancel_then_buy` | **Hủy** gói cũ rồi mới mua gói mới (tránh chồng hai gói cùng loại). |

**Lưu ý triển khai:**  
Enforce trên server: `assert_can_purchase_service_package` — gọi từ `_create_service_package_order` (`payment_orders`) và `create_order` (`views`). `allow_parallel` chỉ chịu ràng `is_stackable`; `allow_when_exhausted`/`allow_cancel_then_buy` chặn theo mô tả bảng trên.

---

## 3. `is_stackable`

**Vai trò:** Gợi ý **có cho phép nhiều gói cùng catalog “chồng” lên nhau** (cộng dồn hiệu lực hoặc quota) hay chỉ một gói active tại một thời điểm.

| Giá trị | Ý nghĩa |
|--------|---------|
| `True` | Cho phép **stack** theo nghiệp vụ (nhiều lần mua cùng gói / nhiều instance). |
| `False` | Hạn chế chồng gói (vd. admin tạo service phụ đăng tin với `is_stackable=False` trong `market/admin.py` cho một số checkout). |

**Luồng phụ thuộc:**  
`create_user_service_after_payment` có xếp `start_date` sau `end_date` gói cũ **cùng service** nếu đã có `UserService` active/scheduled — tức vẫn có thể có nhiều kỳ nối tiếp. **`is_stackable` chưa được đọc trực tiếp trong `userservice`**; cần đồng bộ với product/FE hoặc thêm kiểm tra khi tạo order.

---

## 4. `credit_pool_group`

**Vai trò:** Khi user **đăng tin / reserve entitlement**, backend lọc **dòng `UserCreditBalance` nào được dùng** theo nhóm gói gắn trên `Service` (`source_user_service_feature → … → service.credit_pool_group`) **và** (tùy nhóm) **`target_type` của service** — xem `entitlement._resolve_group`.

| Giá trị | Khi đăng tin thuộc khu vực | Filter credit (khái niệm) |
|--------|----------------------------|---------------------------|
| `COMMUNITY` | Loại tin **cộng đồng / mua bán** (partnership, houseshare, social, job, equipment_market, medical_market, …). | `credit_pool_group = COMMUNITY` + service đang active. Mọi gói gán `COMMUNITY` **dùng chung** một “pool” lượt cho user. |
| `SPECIAL` | Tin **đặc thù theo ngành** (clinic, lab, dental, …). | `credit_pool_group = SPECIAL` **và** `service.target_type` = đúng loại tin đó → nhiều gói SPECIAL cùng ngành vẫn **dùng chung pool**. |
| `HEALTHCARE_STAFF` | Luồng **NVYT** (bác sĩ / điều dưỡng). | `credit_pool_group = HEALTHCARE_STAFF` + `target_type` khớp tập HEALTHCARE_STAFF + role (DOCTOR/NURSE…). |
| `GENERAL` | Các trường hợp **default** không rơi vào ba nhóm trên. | `credit_pool_group = GENERAL` + khớp `target_type` pricing. |

**Cách cấu hình thực tế**

1. **Cùng một “kho lượt dùng chung”** cho nhiều gói (vd. Standard + Pro + VIP toàn là tin mua bán ACCOUNT): đặt **cùng `credit_pool_group`** (thường `COMMUNITY`) và `target_type` phù hợp (thường `ACCOUNT` cho gói cộng đồng).
2. **Tách pool theo ngành đặc thù**: gói phòng khám A và B khác mã nhưng cùng `SPECIAL` + `target_type=CLINIC` → user mua cả hai vẫn **cộng lượt** khi đăng tin CLINIC.
3. **Không lẫn COMMUNITY với SPECIAL**: đăng tin CLINIC không dùng credit của gói COMMUNITY — đúng theo `_resolve_group`.

---

## 5. Tóm tắt nhanh

| Trường | Chỗ “chạy thật” trong BE | Ghi chú |
|--------|-------------------------|--------|
| `entitlement_mode` | Cấp quota + refill bucket | `aggregate` nhân `duration`; `monthly_bucket` cấp theo kỳ + refill 30 ngày từ `start_date`. |
| `purchase_policy` | Lưu + API + validate đơn | `payment_orders` / `views.create_order`. |
| `is_stackable` | Lưu + API + validate đơn | Gói non-stackable chặn khi còn entitlement chồng. |
| `credit_pool_group` | Reserve/commit đăng tin | **Bắt buộc** cấu hình đúng để lượt tin khớp kỳ vọng nghiệp vụ. |

---

*Cập nhật theo mã nguồn trong `src/service` (entitlement, userservice, models). Khi bổ sung enforce `purchase_policy` / `monthly_bucket`, nên chỉnh lại mục tương ứng trong file này.*
