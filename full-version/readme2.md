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
**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** Cần đóng gói (build) ứng dụng qua nhiều giai đoạn (Multi-stage). Giai đoạn 1 dùng image lớn chứa toàn bộ công cụ và thư viện (Node.js) để biên dịch code React. Giai đoạn 2 chỉ copy đúng thư mục tĩnh (`build` hoặc `dist`) sang một Nginx image siêu nhỏ gọn (alpine).
- **Tác hại trực tiếp:** Nếu không làm, image sẽ chứa toàn bộ mã nguồn chưa biên dịch, hàng nghìn thư viện Node không cần thiết ở runtime. Kích thước image có thể phình to lên đến hàng GB, khiến tốc độ deploy chậm, tốn dung lượng lưu trữ, và đặc biệt là gia tăng diện tấn công (attack surface) do chứa cả các công cụ build.

**2. Lệnh Test & Tác dụng:**
```bash
docker images react-berry-app | awk '{print $NF}'
```
*Tác dụng & Tại sao dùng lệnh này:* Lệnh này trích xuất trực tiếp giá trị `SIZE` (dung lượng) cuối cùng của image vừa build. Đây là cách nhanh và chính xác nhất để kiểm chứng xem chiến lược Multi-stage có thực sự hoạt động hay không (loại bỏ được các phần thừa thãi và đạt dung lượng cấp phép).
*Kỳ vọng:* Cột dung lượng xuất ra phải khoảng `~25MB - 35MB` (thỏa mãn < 40MB).

---

### Yêu cầu 2: Non-root container (Chạy dưới user thường)
**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** Tuân thủ nguyên tắc "đặc quyền tổi thiểu" (Least Privilege). Ứng dụng chỉ nên chạy dưới một user thường bị giới hạn quyền, thay vì user `root` toàn quyền của hệ thống.
- **Tác hại trực tiếp:** Nếu Nginx hoặc ứng dụng có lỗ hổng Zero-day mà đang chạy bằng user `root`, hacker lọt vào container sẽ lập tức có đặc quyền cao nhất. Từ đó chúng có thể dễ dàng "vượt ngục" (container breakout) để thâm nhập vào máy chủ gốc (Docker Host) bên ngoài.

**2. Lệnh Test & Tác dụng:**
```bash
docker exec berry-react-app whoami
```
*Tác dụng & Tại sao dùng lệnh này:* Dù bạn có cấu hình đổi user trong Dockerfile, vẫn có rủi ro container bị chạy bằng quyền root khi triển khai ngầm. Lệnh này truy cập thẳng vào container đang sống và xác nhận chính xác "thân phận" của tiến trình ở thời điểm chạy (runtime).
*Kỳ vọng:* Kết quả xuất ra phải là `nginx` (hoặc tên user thường do bạn chỉ định).

---

### Yêu cầu 3: Có Healthcheck & Check bằng lệnh
**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** Cho phép Docker/Kubernetes tự động "khám sức khỏe" định kỳ cho ứng dụng, đảm bảo web server thực sự đang phản hồi (hoạt động) thay vì chỉ kiểm tra xem tiến trình (process) còn bật hay không.
- **Tác hại trực tiếp:** Nhiều lúc tiến trình Nginx/Node vẫn báo "Running" nhưng bên trong bị treo (Crash ngầm hoặc tràn bộ nhớ). Nếu không có lệnh Healthcheck, hệ thống cân bằng tải (Load Balancer) sẽ không tự cách ly container hỏng, mà tiếp tục đẩy khách hàng vào đó gây gián đoạn dịch vụ liên tục.

**2. Lệnh Test & Tác dụng:**
```bash
docker inspect --format='{{json .State.Health.Status}}' berry-react-app
```
*Tác dụng & Tại sao dùng lệnh này:* Lệnh `docker ps` khó cho bạn thấy lịch sử check ổn định cụ thể. Trích xuất thuộc tính `Health.Status` qua `docker inspect` sẽ khẳng định chắc chắn rằng Docker Engine đã thực thi và lấy được trạng thái sức khỏe tốt của ứng dụng.
*Kỳ vọng:* Kết quả trả về phải là `"healthy"`. (Có thể mất 5-10 giây để lên trạng thái healthy).

---

### Yêu cầu 4 & 5: Cache Strategy & Conditional Request (ETag)

> [!IMPORTANT]
> **Lưu ý cho người dùng Windows (Git Bash):** Nếu bạn bị lỗi "No such file or directory" khi chạy lệnh `find`, hãy thêm một dấu gạch chéo nữa vào đầu đường dẫn (thành `//usr/...`) hoặc thêm `MSYS_NO_PATHCONV=1` vào trước lệnh.

**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** 
  - Khối tĩnh (file JS/CSS) đã băm mã hash (VD `main.123ab.js`): Ta ép trình duyệt cache VĨNH VIỄN (`max-age=31536000`).
  - File tĩnh sườn `index.html`: Ta ép hệ thống KHÔNG ĐƯỢC CACHE cứng (`no-cache`), luôn phải hỏi lại ETag của server mỗi khi load xem phiên bản UI cũ hay mới.
- **Tác hại trực tiếp:** 
  - Nếu cache nhầm `index.html`: Khách hàng dùng web suốt nhiều tháng không thấy bản cập nhật mới (do trình duyệt nhớ cứng bản cũ).
  - Hoặc hoàn toàn mất Cache: Mỗi lần truy cập khách đều phải tải lại hàng MB tài nguyên, gây chậm giao diện và lãng phí băng thông mạng lớn.

**2. Lệnh Test & Tác dụng:**

**4.A - File tĩnh đã băm chữ (Hashed static assets):**
> [!NOTE]
> Do chúng ta đã tối ưu xóa file gốc và giữ lại file nén, nên đuôi file sẽ là `.js.gz`.

```bash
# Lệnh cho Git Bash (Windows):
MSYS_NO_PATHCONV=1 docker exec berry-react-app find /usr/share/nginx/html/static/js -name "main.*.js.gz" | head -n 1 | awk -F'/' '{print $NF}'

# Lệnh cho CMD/PowerShell:
# docker exec berry-react-app find /usr/share/nginx/html/static/js -name "main.*.js.gz" | head -n 1 | awk -F'/' '{print $NF}'
```
Giả sử file trả về ở trên là `main.abcd123.js`, ta test HTTP Header của file này:
```bash
curl -I http://localhost:3000/static/js/main.abcd123.js
```
*Tác dụng:* Lệnh đọc Header (`-I`) nhằm xác định xem chỉ thị đánh Cache trọn đời đã được Nginx gửi trả kèm cái file đó chưa.
*Kỳ vọng:* 
- Có dòng `Cache-Control: public, max-age=31536000, immutable`
- KHÔNG CÓ dòng `ETag` nào cả.

**4.C - Test trang chủ index.html:**
```bash
curl -I http://localhost:3000/index.html
```
*Tác dụng:* Để verify xem luật chặn Cache cứng đã hoạt động trên index.html chưa (bắt buộc đánh version ETag).
*Kỳ vọng:*
- Có dòng `Cache-Control: no-cache, no-store, must-revalidate`
- Có dòng `ETag: "chữ ký mã file..."`

**5 - Test If-None-Match hoạt động (Mã chứng nhận 304):**
Lấy cái chuỗi Etag (Ví dụ là `W/"65b0c..."`) ở kết quả lệnh test 4.C ráp vào đây:
```bash
curl -I -H 'If-None-Match: W/"65b0c..."' http://localhost:3000/index.html
```
*Tác dụng & Tại sao dùng lệnh này:* Lệnh này giả mạo lại tình huống trình duyệt hỏi lại Server "Tài nguyên chứa Etag này cũ chưa?". Hành vi chuẩn của Server để tiết kiệm băng thông phải là gửi mã phản hồi 304 không đính kèm nội dung body.
*Kỳ vọng:* Trả lời trạng thái `HTTP/1.1 304 Not Modified`.

---

### Yêu cầu 6: Gzip / Compression
**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** Sử dụng nén Gzip/Brotli nhằm thu nhỏ dung lượng các tài nguyên (file text, HTML, css, js) trước khi gửi qua mạng.
- **Tác hại trực tiếp:** App React khi Build xong khối JS khá to. Nếu bỏ nén, ứng dụng sẽ truyền nguyên dữ liệu gốc, khiến website tải cực chậm, chực chờ gián đoạn trên các mạng yêú như 3G, phung phí tài nguyên Data Center.

**2. Lệnh Test & Tác dụng:**
```bash
curl -I -H "Accept-Encoding: gzip" http://localhost:3000/index.html
```
*Tác dụng & Tại sao dùng lệnh này:* Test chủ động khai báo "Accept-Encoding: gzip" để Nginx biết trình duyệt ta cho phép nén. Mục đích ép Nginx xuất xác nhận bật nén ngược lại nếu nó cấu hình Gzip hợp lệ.
*Kỳ vọng:* Báo lại dòng `Content-Encoding: gzip`.

---

### Yêu cầu 7 & 8: Nginx Config & SPA Routing (Điều khiển luồng 404)
**1. Tại sao & Tác hại nếu không làm:**
- **Yêu cầu 7 - Ẩn Version:** Kẻ tấn công chỉ cần nhìn vào Version lộ ra của Server để tra cứu tìm lỗi bảo mật (CVE) đã biết rồi khai thác. (Ví dụ lộ bản nginx 1.18.0 cũ mèm là chết).
- **Yêu cầu 8 - Route SPA:** Bản chất web React là ứng dụng 1 trang (Single Page Application). Nếu không định tuyến các đường dẫn ảo (Ví dụ: `/admin/settings`) trỏ ngược về `index.html`, Nginx sẽ tìm các thư mục /admin thật sự trên ổ đĩa. Khi tìm không thấy, nó sẽ đập lỗi 404 vào mặt khách.

**2. Lệnh Test & Tác dụng:**

**7 - Test Ẩn Version Server:**
```bash
curl -I http://localhost:3000/
```
*Tác dụng:* Đảm bảo HTTP Header trả lời "vô thưởng vô phạt", không rò rỉ thông số lõi.
*Kỳ vọng:* Mục tiêu chỉ thấy `Server: nginx` (Không hiện số lượng 1.x.x).

**8.A - Test Route SPA Ảo:**
```bash
curl -I http://localhost:3000/admin/settings
```
*Tác dụng & Tại sao dùng lệnh này:* Việc gọi cái link "ảo" không có thật này là minh chứng test cơ chế định tuyến tĩnh (`try_files`) của Nginx. Xem xem nó có biết đường bao bọc link ảo ném lại nội dung React index.html trơn tru không.
*Kỳ vọng:* Nhận được mã phản hồi `HTTP/1.1 200 OK`.

**8.B - Test File không tồn tại (Lỗi Fallback):**
```bash
curl -I http://localhost:3000/file_rac.jpg
```
*Tác dụng & Tại sao dùng lệnh này:* Đây là bước test tính chặt chẽ. Đề phòng lúc config chặn 404 ta để lỏng, nếu load tĩnh 1 file .jpg hỏng mà Server vứt lại cái mã HTML của ứng dụng (200 OK) thì web sẽ nổ tung loạn các thẻ img. Nên phải xác minh Nginx đủ xịn trả khước khước từ đúng mã 404 đối diện với lỗi từ thư mục file tĩnh.
*Kỳ vọng:* Bắt buộc trả mã `HTTP/1.1 404 Not Found`.

---

### Yêu cầu 10: Image Security (Bảo mật nâng cao)
**1. Tại sao & Tác hại nếu không làm:**
- **Tại sao:** Nginx Image phải được tước sạch cờ thiết lập ứng dụng và xóa triệt để file Source map `.map`.
- **Tác hại trực tiếp:**
  - Nếu quên xóa Source map: Đối thủ bật trình soi code F12 của Chrome lên sẽ "tải được 100% bản nháp chưa thu gọn" source code gốc (ES6), rò rỉ logic mật khẩu, bí mật tài sản phần mềm.
  - Nếu quên xóa tool `apk`: Máy chủ có thể bị lợi dụng cài cắm Botnet, các tool backdoor trinh sát như nmap, htop nếu hacker lọt được thông qua lỗi React Upload/RFI.

**2. Lệnh Test & Tác dụng:**

**Lệnh Test 1: Bỏ Apk?**
```bash
docker exec berry-react-app apk add htop
```
*Tác dụng & Tại sao dùng lệnh này:* Thử nhập vai kẻ gian vào phá hoại. Nếu lệnh vô hiệu thì kẻ gian cũng chào thua. Đứng từ quan điểm Dev đây gọi là "Chặn đứng mọi phương pháp tái tổ chức hệ thống".
*Kỳ vọng:* Docker báo `executable file not found`.

**Lệnh Test 2: Xóa lọt Source map?**
```bash
docker exec berry-react-app find //usr/share/nginx/html -name "*.map"
```
*Tác dụng & Tại sao dùng lệnh này:* Lệnh này rà đệ quy quét sạch trong lõi ổ đĩa web nhằm triệt tiêu tất cả "bản đồ phân tích code". Xác nhận rằng việc vệ sinh build được diễn ra thành công không sót rác thải.
*Kỳ vọng:* Output trắng trơn hoàn toàn (không thấy file map nào).
