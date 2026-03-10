# Thực hành Kiểm Tra Dockerfile-React (Test Checklist)

Dưới đây là các câu lệnh thực tế để kiểm tra trực tiếp (test) xem container có thật sự thỏa mãn các yêu cầu khắt khe trong bản đặc tả `Dockerfile-React.txt` hay không. 

Để chạy được các lệnh test này, đầu tiên bạn cần phải build và chạy container lên trước đã:

```bash
# Bước 1: Build image
docker build -t react-berry-app .

# Bước 2: Chạy container
docker run -d -p 3000:8080 --name berry-react-app react-berry-app
```

---

### Yêu cầu 1: Multi-stage build & Kích thước (Image Size < 40MB)
**Lệnh Test:**
```bash
docker images react-berry-app | awk '{print $NF}'
```
*Kỳ vọng:* Cột dung lượng (SIZE) xuất ra phải loanh quanh ở mức `~25MB - 35MB` (hoàn toàn thỏa mãn < 40MB).

### Yêu cầu 2: Non-root container (Chạy dưới user thường)
**Lệnh Test:**
```bash
docker exec berry-react-app whoami
```
*Kỳ vọng:* Kết quả xuất ra phải là `nginx` (chứng tỏ container đang chạy dưới user nginx chứ không phải root).

### Yêu cầu 3: Có Healthcheck & Check bằng lệnh
**Lệnh Test:**
```bash
docker inspect --format='{{json .State.Health.Status}}' berry-react-app
```
*Kỳ vọng:* Kết quả trả về phải là `"healthy"`. (Có thể mất khoảng 5-10 giây đầu lúc mới chạy container để lên trạng thái healthy).

### Yêu cầu 4 & 5: Cache Strategy & Conditional Request (ETag)

> [!IMPORTANT]
> **Lưu ý cho người dùng Windows (Git Bash):** Nếu bạn bị lỗi "No such file or directory" khi chạy lệnh `find`, hãy thêm một dấu gạch chéo nữa vào đầu đường dẫn (thành `//usr/...`) hoặc thêm `MSYS_NO_PATHCONV=1` vào trước lệnh.

**4.A - Test File Tĩnh đã băm chữ (Hashed static assets):**

> [!NOTE]
> Do chúng ta đã tối ưu xóa file gốc và giữ lại file nén, nên đuôi file sẽ là `.js.gz`.

```bash
# Lệnh cho Git Bash (Windows):
MSYS_NO_PATHCONV=1 docker exec berry-react-app find /usr/share/nginx/html/static/js -name "main.*.js.gz" | head -n 1 | awk -F'/' '{print $NF}'

# Lệnh cho CMD/PowerShell:
# docker exec berry-react-app find /usr/share/nginx/html/static/js -name "main.*.js.gz" | head -n 1 | awk -F'/' '{print $NF}'
```

Giả sử tên file ở trên trả về là `main.abcd123.js`, bạn dùng lệnh test:
```bash
curl -I http://localhost:8080/static/js/main.abcd123.js
```
*Kỳ vọng trong Header trả về:* 
- Có dòng `Cache-Control: public, max-age=31536000, immutable`
- KHÔNG CÓ dòng `ETag` nào cả.

**4.C - Test trang chủ index.html:**
```bash
curl -I http://localhost:3000/index.html
```
*Kỳ vọng trong Header trả về:*
- Có dòng `Cache-Control: no-cache, no-store, must-revalidate`
- Có dòng `ETag: "chữ ký mã file..."`

**5 - Test If-None-Match hoạt động (Mã 304):**
Lấy cái chuỗi Etag (Ví dụ là `W/"65b0c..."`) ở kết quả lệnh test 4.C ráp vào đây:
```bash
curl -I -H 'If-None-Match: W/"65b0c..."' http://localhost:3000/index.html
```
*Kỳ vọng:* Lệnh này phải trả về kết quả ngay dòng đầu tiên là `HTTP/1.1 304 Not Modified`.

### Yêu cầu 6: Gzip / Compression
**Lệnh Test (Gửi yêu cầu nhận file nén gzip):**
```bash
curl -I -H "Accept-Encoding: gzip" http://localhost:3000/index.html
```
*Kỳ vọng trong Header trả về:* Có dòng `Content-Encoding: gzip`.

### Yêu cầu 7 & 8: Nginx Config & SPA Routing (Điều khiển luồng 404)
**7 - Test Lộ Version Server:**
```bash
curl -I http://localhost:3000/
```
*Kỳ vọng trả về:* Có dòng `Server: nginx` (Tuyệt đối không có số phiên bản như `nginx/1.27.x`).

**8.A - Test Route SPA Ảo:**
```bash
curl -I http://localhost:3000/admin/settings
```
*Kỳ vọng:* Trả về mã `HTTP/1.1 200 OK` (Mặc dù thư mục /admin không hề tồn tại trên ổ cứng, Nginx đã đẩy về React Router).

**8.B - Test File không tồn tại (Fallback mù lòa):**
```bash
curl -I http://localhost:3000/file_rac.jpg
```
*Kỳ vọng:* Nginx phải trả về mã `HTTP/1.1 404 Not Found` (Vì đây là tệp tin thật, không phải ảo, nó không được đẩy về index.html).

### Yêu cầu 10: Image Security (Bảo mật)
**Lệnh Test 1: Có bỏ apk tool chưa?**
```bash
docker exec berry-react-app apk add htop
```
*Kỳ vọng:* Báo lỗi `executable file not found`. (Vì công cụ tải đã bị xóa, không ai leo quyền được nữa).

**Lệnh Test 2: Có lộ Source Map (.map) chưa?**
```bash
# Liệt kê tất cả file có đuôi .map
# Thêm // đầu đường dẫn cho Git Bash Windows
docker exec berry-react-app find //usr/share/nginx/html -name "*.map"
```
*Kỳ vọng:* Trống rỗng, không in ra bất kỳ dòng nào. (Code map đã bị thiêu hủy).
