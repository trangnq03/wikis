# Báo cáo số liệu Bộ Công Thương (MOIT)

## Tổng quan

App `src.gov_reporting` cung cấp 3 API:

1. **Statistics API**: Báo cáo chi tiết các chỉ tiêu theo khoảng thời gian tùy chỉnh
2. **Summary API**: Báo cáo tóm tắt so sánh số liệu cho dashboard
3. **Listing Charges API**: Danh sách chi tiết giao dịch thanh toán tin đăng (phân trang, có filter)

Cả ba API đều hỗ trợ xác thực bằng JWT hoặc UserName/PassWord theo mẫu BCT.

## Endpoints

| API | URL | Method | Mô tả |
|-----|-----|--------|-------|
| Statistics | `POST /api/v1/gov-reporting/moit/statistics` | POST | Báo cáo chi tiết số liệu giao dịch |
| Summary | `POST /api/v1/gov-reporting/moit/summary` | POST | Báo cáo tóm tắt so sánh |
| Listing Charges | `POST /api/v1/gov-reporting/moit/listing-charges` | POST | Danh sách giao dịch thanh toán tin đăng |

## Xác thực (Authentication)

### Cách 1: JWT Token (cho trang báo cáo nội bộ)

1. Lấy JWT token từ user `type=REPORT`:
   ```http
   POST /api/v1/token/
   Content-Type: application/json

   {
     "phone_number": "+84901234567",
     "password": "password_here"
   }
   ```

2. Sử dụng token trong header:
   ```http
   Authorization: Bearer <access_token>
   ```

### Cách 2: UserName/PassWord trong body (theo mẫu BCT)

```json
{
  "UserName": "BaocaoTMDT",
  "PassWord": "<mật_khẩu_user_REPORT>"
}
```

**Lưu ý:**
- `UserName` phải khớp với `MOIT_REPORT_BODY_USERNAME` (env, mặc định `BaocaoTMDT`)
- `PassWord` phải khớp mật khẩu của user `type=REPORT` đang active
- User không phải `REPORT` → 403 Forbidden
- Sai credentials → 401 Unauthorized

## API Details

### 1. Statistics API

**URL:** `POST /api/v1/gov-reporting/moit/statistics`

**Mô tả:** Báo cáo chi tiết các chỉ tiêu giao dịch theo khoảng thời gian tùy chỉnh.

**Caching:** Không cache - luôn query database mỗi request.

**Request Body:**
```json
{
  "UserName": "BaocaoTMDT",           // Optional nếu dùng JWT
  "PassWord": "password",              // Optional nếu dùng JWT
  "start_date": "2026-01-01",          // Optional: YYYY-MM-DD
  "end_date": "2026-05-15"             // Optional: YYYY-MM-DD
}
```

**Rules:**
- `start_date` và `end_date` phải có cả hai hoặc không có
- `end_date >= start_date`
- Nếu không gửi → mặc định: 01/01/năm hiện tại → hôm nay

**Response:**
```json
{
  "soLuongTruyCap": 0,
  "soNguoiBan": 150,
  "soNguoiBanMoi": 0,
  "tongSoSanPham": 0,
  "soSanPhamMoi": 0,
  "soLuongGiaoDich": 1250,
  "tongSoDonHangThanhCong": 1200,
  "tongSoDonHangKhongThanhCong": 50,
  "tongGiaTriGiaoDich": 250000000
}
```

### 2. Summary API

**URL:** `POST /api/v1/gov-reporting/moit/summary`

**Mô tả:** Báo cáo tóm tắt so sánh số liệu cho dashboard MOIT.

**Caching:** Có cache Django với TTL đến cuối ngày hiện tại.

**Khoảng thời gian cố định:**
- Current: 2020-01-01 → hôm qua
- Previous: 2020-01-01 → (hôm qua - 7 ngày)

**Request Body:**
```json
{
  "UserName": "BaocaoTMDT",           // Optional nếu dùng JWT
  "PassWord": "password"               // Optional nếu dùng JWT
}
```

**Response:**
```json
{
  "userCount": {
    "gia_tri": 1250,
    "don_vi": "",
    "ti_le_tang_truong": 5.26
  },
  "postingMembers": {
    "gia_tri": 150,
    "don_vi": "",
    "ti_le_tang_truong": 2.04
  },
  "transactionValue": {
    "gia_tri": 250000000,
    "don_vi": "VND",
    "ti_le_tang_truong": 8.75
  },
  "transactionCount": {
    "gia_tri": 1250,
    "don_vi": "",
    "ti_le_tang_truong": 6.38
  }
}
```

**Giải thích response:**
- `gia_tri`: Giá trị số liệu hiện tại
- `don_vi`: Đơn vị (VND cho tiền, rỗng cho count)
- `ti_le_tang_truong`: Tỷ lệ tăng trưởng % so với tuần trước

### 3. Listing Charges API

**URL:** `POST /api/v1/gov-reporting/moit/listing-charges`

**Mô tả:** Trả về danh sách chi tiết các giao dịch thanh toán tin đăng, hỗ trợ phân trang và nhiều bộ lọc. Bao gồm 3 loại order: `entitlement_market` (Đăng tin), `service_package` (mua gói dịch vụ), `listing_renewal` (gia hạn tin).

**Caching:** Không cache — luôn query database mỗi request.

---

#### Request Body

```json
{
  "UserName": "BaocaoTMDT",
  "PassWord": "password",
  "start_date": "2026-01-01",
  "end_date": "2026-05-15",
  "member_type": ["new", "old"],
  "categories": ["DENTAL", "PHARMACY"],
  "payment_status": ["completed", "pending", "failed"],
  "page": 1,
  "page_size": 30
}
```

| Field | Type | Required | Mô tả |
|-------|------|----------|-------|
| `UserName` | string | Không (nếu dùng JWT) | Xác thực theo mẫu BCT |
| `PassWord` | string | Không (nếu dùng JWT) | Xác thực theo mẫu BCT |
| `start_date` | string (YYYY-MM-DD hoặc DD/MM/YYYY) | Không | Ngày bắt đầu lọc. Phải đi kèm `end_date` |
| `end_date` | string (YYYY-MM-DD hoặc DD/MM/YYYY) | Không | Ngày kết thúc lọc. Phải đi kèm `start_date` |
| `member_type` | array of string | Không | Lọc theo loại thành viên: `"new"` (lần đầu), `"old"` (đã từng mua) |
| `categories` | array of string | Không | Lọc theo loại tin đăng (xem bảng categories bên dưới) |
| `payment_status` | array of string | Không | Lọc theo trạng thái thanh toán: `"completed"`, `"pending"`, `"failed"` |
| `page` | integer (≥ 1) | Không | Trang hiện tại. Mặc định: `1` |
| `page_size` | integer (≥ 1) | Không | Số item mỗi trang. Mặc định: `30` |

**Validation rules:**
- `start_date` và `end_date` phải có cả hai hoặc không có cả hai
- `end_date >= start_date`
- Nếu không gửi date range → mặc định: 01/01/năm hiện tại → hôm nay

---

#### Danh sách `categories`

| Giá trị | Mô tả |
|---------|-------|
| `CLINIC` | Phòng khám |
| `LAB_TEST` | Xét nghiệm |
| `DIAGNOSTIC` | Chẩn đoán |
| `DENTAL` | Nha khoa |
| `TRADITIONAL_MEDICINE` | Y học cổ truyền |
| `PHARMACY` | Nhà thuốc |
| `AMBULANCE` | Xe cứu thương |
| `DOCTOR` | Bác sĩ |
| `NURSE` | Điều dưỡng, y tá (tự động normalize → `DOCTOR`) |
| `HEALTHCARE_STAFF` | Nhân viên y tế (tự động normalize → `DOCTOR`) |
| `MEDICAL_MARKET` | Chợ dược phẩm |
| `EQUIPMENT_MARKET` | Chợ y tế |
| `JOB` | Việc làm |
| `SOCIAL` | Kết bạn, giao lưu |
| `HOUSESHARE` | Chia sẻ nhà |
| `PARTNERSHIP` | Hợp tác |
| `ACCOUNT` | Account |

> **Lưu ý:** `NURSE` và `HEALTHCARE_STAFF` được tự động normalize thành `DOCTOR` trước khi filter.

---

#### Response

```json
{
  "page": 1,
  "page_size": 30,
  "total_items": 125,
  "total_pages": 5,
  "items": [ ... ]
}
```

| Field | Type | Mô tả |
|-------|------|-------|
| `page` | integer | Trang hiện tại |
| `page_size` | integer | Số item mỗi trang |
| `total_items` | integer | Tổng số bản ghi khớp filter |
| `total_pages` | integer | Tổng số trang |
| `items` | array | Danh sách giao dịch (xem cấu trúc item bên dưới) |

---

#### Cấu trúc một item trong `items`

Cấu trúc của mỗi item phụ thuộc vào **loại order** (`payment_order_type`):

##### Loại `listing_renewal` (gia hạn tin)

`listing_name` là **array of objects**, mỗi object đại diện cho một tin đăng trong cùng order.

```json
{
  "full_name": "Trần Quan***",
  "phone_number": "+84356965***",
  "email": "t***1@teamsolutions.vn",
  "citizen_identification": "030*******80",
  "publish_at": "2026-04-15T08:31:37.348531+00:00",
  "listing_name": [
    {
      "listing_id": 123,
      "listing_name": "Nha khoa cơ sở",
      "line_amount": "56000.00",
      "target_type": "DENTAL"
    },
    {
      "listing_id": 456,
      "listing_name": "Nhà thuốc Long Châu",
      "line_amount": "70000.00",
      "target_type": "PHARMACY"
    }
  ],
  "payment_amount": 126000,
  "payment_status": "FAILED"
}
```

**Trường trong mỗi object của `listing_name`:**

| Field | Type | Mô tả |
|-------|------|-------|
| `listing_id` | integer | ID của tin đăng |
| `listing_name` | string | Tên tin đăng |
| `line_amount` | string (decimal) | Số tiền của riêng tin đăng này trong order |
| `target_type` | string | Loại tin đăng (ví dụ: `DENTAL`, `PHARMACY`, `CLINIC`, ...) |

**Ảnh hưởng của filter `categories` lên `listing_name`:**

| Filter `categories` | Kết quả `listing_name` |
|---------------------|------------------------|
| Không gửi (hoặc rỗng) | Toàn bộ listing trong order |
| `["DENTAL", "PHARMACY"]` | Chỉ listing có `target_type` là `DENTAL` hoặc `PHARMACY` |
| `["DENTAL"]` | Chỉ listing có `target_type` là `DENTAL` |

> `payment_amount` luôn là **tổng tiền của toàn bộ order**, không bị ảnh hưởng bởi filter categories.

---

##### Loại `entitlement_market` (Đăng tin) và `service_package` (mua gói dịch vụ)

`listing_name` là **string** — tên tin đăng hoặc tên gói dịch vụ.

```json
{
  "full_name": "Nguyễn Văn ***",
  "phone_number": "+84912345***",
  "email": "n***@example.com",
  "citizen_identification": "079*******12",
  "publish_at": "2026-03-10T07:00:00.000000+00:00",
  "listing_name": "Phòng khám đa khoa ABC",
  "payment_amount": 250000,
  "payment_status": "SUCCESS"
}
```

---

#### Trường chung của mọi item

| Field | Type | Mô tả |
|-------|------|-------|
| `full_name` | string | Họ tên người dùng (đã mask PII) |
| `phone_number` | string | Số điện thoại (đã mask PII) |
| `email` | string | Email (đã mask PII) |
| `citizen_identification` | string | CCCD/CMND (đã mask PII) |
| `publish_at` | string (ISO 8601) hoặc `null` | Thời điểm publish. Với `entitlement_market`: lấy từ `published_at` của listing. Với các loại khác: lấy từ `updated_at` của order |
| `listing_name` | string **hoặc** array of object | Xem chi tiết theo loại order ở trên |
| `payment_amount` | integer | Tổng số tiền của order (VND, không có phần thập phân) |
| `payment_status` | string | Trạng thái thanh toán: `SUCCESS`, `PENDING`, `FAILED` |

**Mapping `payment_status`:**

| Giá trị trả về | Trạng thái order gốc |
|----------------|----------------------|
| `SUCCESS` | `PAID` |
| `PENDING` | `PENDING_PAYMENT` |
| `FAILED` | `FAILED`, `CANCELED`, `EXPIRED` |

---

#### Ví dụ request/response đầy đủ

**Request — filter nhiều categories, chỉ lấy renew thất bại:**
```bash
curl -X POST http://localhost:8000/api/v1/gov-reporting/moit/listing-charges \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{
    "start_date": "2026-01-01",
    "end_date": "2026-05-15",
    "categories": ["DENTAL", "PHARMACY"],
    "payment_status": ["failed"],
    "page": 1,
    "page_size": 10
  }'
```

**Response:**
```json
{
  "page": 1,
  "page_size": 10,
  "total_items": 3,
  "total_pages": 1,
  "items": [
    {
      "full_name": "Trần Quan***",
      "phone_number": "+84356965***",
      "email": "t***1@teamsolutions.vn",
      "citizen_identification": "030*******80",
      "publish_at": "2026-04-15T08:31:37.348531+00:00",
      "listing_name": [
        {
          "listing_id": 123,
          "listing_name": "Nha khoa cơ sở",
          "line_amount": "56000.00",
          "target_type": "DENTAL"
        },
        {
          "listing_id": 456,
          "listing_name": "Nhà thuốc Long Châu",
          "line_amount": "70000.00",
          "target_type": "PHARMACY"
        }
      ],
      "payment_amount": 126000,
      "payment_status": "FAILED"
    }
  ]
}
```

**Request — filter chỉ một category:**
```bash
curl -X POST http://localhost:8000/api/v1/gov-reporting/moit/listing-charges \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{
    "start_date": "2026-01-01",
    "end_date": "2026-05-15",
    "categories": ["DENTAL"],
    "page": 1,
    "page_size": 10
  }'
```

**Response** — `listing_name` chỉ chứa item DENTAL, dù order có thể có thêm PHARMACY:
```json
{
  "page": 1,
  "page_size": 10,
  "total_items": 2,
  "total_pages": 1,
  "items": [
    {
      "full_name": "Trần Quan***",
      "phone_number": "+84356965***",
      "email": "t***1@teamsolutions.vn",
      "citizen_identification": "030*******80",
      "publish_at": "2026-04-15T08:31:37.348531+00:00",
      "listing_name": [
        {
          "listing_id": 123,
          "listing_name": "Nha khoa cơ sở",
          "line_amount": "56000.00",
          "target_type": "DENTAL"
        }
      ],
      "payment_amount": 126000,
      "payment_status": "FAILED"
    }
  ]
}
```

---

#### Lưu ý quan trọng cho FE

1. **`listing_name` có thể là string hoặc array** — FE cần kiểm tra kiểu dữ liệu trước khi render:
   - Nếu là `string` → hiển thị trực tiếp (order loại `entitlement_market` hoặc `service_package`)
   - Nếu là `array` → render danh sách listing (order loại `listing_renewal`)

2. **`payment_amount` không thay đổi theo filter categories** — luôn là tổng tiền của cả order, dù `listing_name` chỉ hiển thị một phần listing.

3. **`listing_name` array có thể rỗng `[]`** — nếu order `listing_renewal` không có item nào khớp với filter categories (hiếm gặp do queryset đã pre-filter, nhưng FE nên handle trường hợp này).

4. **Thứ tự item** — sắp xếp theo `created_at` giảm dần (mới nhất trước).

5. **PII masking** — tất cả thông tin cá nhân đã được che một phần theo quy định BCT. FE không cần xử lý thêm.

---

## Cấu hình

- **`MOIT_REPORT_BODY_USERNAME`**: Giá trị bắt buộc cho `UserName` (mặc định: `BaocaoTMDT`)
- **User REPORT**: Tạo ít nhất 1 user với `type=REPORT` trong admin/DB
- **INSTALLED_APPS**: Thêm `src.gov_reporting`
- **Whitelist**: Endpoint được whitelist trong `InternalAPIKeyMiddleware` (không cần `X-API-KEY`)

## Examples

### Statistics với JWT
```bash
curl -X POST http://localhost:8000/api/v1/gov-reporting/moit/statistics \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..." \
  -d '{
    "start_date": "2026-01-01",
    "end_date": "2026-05-15"
  }'
```

### Statistics với UserName/PassWord
```bash
curl -X POST http://localhost:8000/api/v1/gov-reporting/moit/statistics \
  -H "Content-Type: application/json" \
  -d '{
    "UserName": "BaocaoTMDT",
    "PassWord": "secure_password",
    "start_date": "2026-01-01",
    "end_date": "2026-05-15"
  }'
```

### Summary API
```bash
curl -X POST http://localhost:8000/api/v1/gov-reporting/moit/summary \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..." \
  -d '{}'
```

## Implementation Notes

### Caching Strategy
- **Statistics API**: `compute_details()` - luôn query fresh, không cache
- **Summary API**: `compute_all()` - cache Django với TTL đến 23:59:59 hôm nay
- **Listing Charges API**: `build_listing_charges_items_paginated()` - luôn query fresh, không cache
- Cache key: `moit_metrics_{start_date}_{end_date}`

### Listing Charges — Order Types
Có 3 loại order được xử lý, phân biệt qua `metadata.payment_order_type`:

| `payment_order_type` | Mô tả | `listing_name` type |
|----------------------|-------|---------------------|
| `entitlement_market` | Đăng tin đăng | `string` — tên listing |
| `service_package` | Mua gói dịch vụ | `string` — tên service |
| `listing_renewal` | Gia hạn tin đăng | `array of object` — danh sách listing trong order |
| `null` / `""` | Legacy orders | `string` — tên service |

### Listing Charges — Category Filter Logic
- Filter `categories` áp dụng ở **2 tầng**:
  1. **Queryset level**: Chỉ lấy các order có ít nhất một item khớp category (dùng `jsonb_array_elements` trên PostgreSQL)
  2. **Item level** (chỉ với `listing_renewal`): Trong mỗi order, chỉ trả về các listing item có `target_type` khớp với filter
- Với `entitlement_market` và `service_package`, filter categories chỉ áp dụng ở queryset level, không ảnh hưởng cấu trúc response

### Error Handling
- 400: Validation errors (invalid dates, missing fields)
- 401: Authentication failed
- 403: User không phải REPORT type
- 500: Server errors
