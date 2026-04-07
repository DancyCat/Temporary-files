# PIN Authenticator – ImmortalWrt + openNDS

Hệ thống xác thực mạng WiFi qua mã PIN, tích hợp với openNDS v10.3.1 và LuCI.

---

## Kiến trúc hệ thống

```
Thiết bị kết nối WiFi
       │
       ▼ (HTTP redirect)
openNDS (chặn traffic)
       │
       ▼ FAS Level 1 redirect → http://192.168.1.1:2050/
uhttpd port 2050  ─── /www/pin_fas/index.html (trang nhập PIN)
       │
       ▼ POST /cgi-bin/auth
  pin_mgr validate
       │  ├── Kiểm tra UCI /etc/config/pin_auth
       │  ├── Gọi `ndsctl auth <mac> <session_secs>`
       │  └── Ghi thiết bị vào UCI
       │
       ▼ Chuyển hướng tới URL gốc

LuCI Admin  →  /admin/services/pin_auth
  Controller  : /usr/lib/lua/luci/controller/pin_auth.lua
  View        : /usr/lib/lua/luci/view/pin_auth/index.htm
  API (AJAX)  : /api_pins, /api_devices, /api_settings ...
```

---

## Yêu cầu

| Phần mềm | Phiên bản |
|---|---|
| ImmortalWrt | v24.10.0 |
| openNDS | v10.3.1 |
| LuCI | mặc định theo firmware |
| uhttpd | mặc định theo firmware |

---

## Cài đặt

### 1. Upload lên router
```bash
scp -r pin_auth_system root@192.168.1.1:/tmp/
```

### 2. Chạy script cài đặt
```bash
ssh root@192.168.1.1
sh /tmp/pin_auth_system/install.sh
```

Script tự động:
- Copy tất cả files vào đúng vị trí
- Cấu hình uhttpd port 2050
- Cấu hình openNDS FAS
- Thêm cron job dọn dẹp mỗi 15 phút
- Khởi động lại dịch vụ

---

## Cấu trúc file sau cài đặt

```
/etc/config/pin_auth              ← UCI config (PINs, devices, settings)
/www/pin_fas/
  index.html                      ← Trang captive portal (nhập PIN)
  cgi-bin/auth                    ← CGI xử lý xác thực
/usr/bin/pin_mgr                  ← Tiện ích quản lý PIN
/usr/lib/lua/luci/controller/pin_auth.lua
/usr/lib/lua/luci/view/pin_auth/index.htm
```

---

## Cấu hình openNDS thủ công (nếu install.sh không tự động)

Sửa `/etc/config/opennds`:
```
config opennds
    option enabled          1
    option fwhook_enabled   1
    option gatewayname      'WiFi Hotspot'
    option fasport          '2050'
    option faspath          '/'
    option fasremoteip      '192.168.1.1'    # ← IP LAN của router
    option fasssl           '0'
    option fas_secure_enabled '1'
    option checkinterval    '30'
    option maxclients       '250'
    option preauth_idle_timeout '20'
    option preauthenticated_users 'allow tcp port 2050'
```

---

## Giao diện LuCI Admin

Truy cập: **Services → PIN Authenticator**

### Nhóm 1 – Tạo mã PIN
- Nhấn **"Tạo mã PIN mới"** → tạo PIN ngẫu nhiên
- Bảng danh sách PIN với trạng thái:
  - 🟢 **Vẫn còn** – có thể dùng
  - 🟠 **Đã sử dụng** – đã có thiết bị đăng nhập
  - ⚫ **Hết hiệu lực** – quá thời gian
- Nút **Xoá** chỉ hiện với PIN còn hiệu lực

### Nhóm 2 – Danh sách thiết bị
- Tên thiết bị + MAC address
- Thời gian sử dụng còn lại (vd: `1 tiếng 30 phút`)
- Nút **⭐ Quản trị**: popup xác nhận → lưu và tải lại trang
- Thiết bị admin được đánh dấu màu vàng

### Nhóm 3 – Cấu hình
| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| Thời gian hiệu lực mã PIN | Sau khi tạo, mã PIN tồn tại bao lâu | 120 phút |
| Thời gian sử dụng thiết bị | Phiên kết nối kéo dài bao lâu | 120 phút |
| Độ dài mã PIN | Số chữ số | 6 |
| Tên mạng | Hiển thị trên trang đăng nhập | WiFi Hotspot |
| Tự động tạo PIN | Tạo PIN mới tự động khi cần | Tắt |

---

## Lệnh CLI hữu ích

```bash
# Tạo PIN mới (từ terminal)
pin_mgr generate

# Xem danh sách PIN (JSON)
pin_mgr list_pins

# Xem thiết bị đã kết nối
pin_mgr list_devices

# Xác thực thủ công một thiết bị
pin_mgr validate 123456 aa:bb:cc:dd:ee:ff 192.168.1.100

# Dọn dẹp PIN/device hết hạn
pin_mgr cleanup

# Xem cấu hình hiện tại
pin_mgr get_settings
```

---

## UCI config mẫu

```
# /etc/config/pin_auth

config settings 'settings'
    option pin_validity   '120'     # phút
    option device_session '120'     # phút
    option auto_generate  '0'
    option pin_length     '6'
    option gateway_ip     '192.168.1.1'
    option gateway_name   'WiFi Hotspot'

config pin 'pin_123456'
    option code       '123456'
    option status     'valid'       # valid | used | expired
    option created_ts '1744272000'
    option expires_ts '1744279200'

config device 'dev_aa_bb_cc_dd_ee_ff'
    option mac             'aa:bb:cc:dd:ee:ff'
    option name            'iPhone của Minh'
    option ip              '192.168.1.50'
    option is_admin        '0'
    option pin_used        '123456'
    option session_start   '1744272000'
    option session_expires '1744279200'
```

---

## Gỡ lỗi

```bash
# Xem log openNDS
logread | grep opennds

# Test FAS thủ công
curl "http://192.168.1.1:2050/?tok=test&clientmac=aa:bb:cc:dd:ee:ff&clientip=192.168.1.50"

# Kiểm tra trạng thái openNDS
ndsctl status

# Xem danh sách client openNDS
ndsctl json

# Khởi động lại openNDS
/etc/init.d/opennds restart
```

---

## Gỡ cài đặt

```bash
uci delete uhttpd.pin_fas && uci commit uhttpd
rm -rf /www/pin_fas /usr/bin/pin_mgr
rm -f /usr/lib/lua/luci/controller/pin_auth.lua
rm -rf /usr/lib/lua/luci/view/pin_auth
rm -f /etc/config/pin_auth
/etc/init.d/uhttpd restart
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache
```
