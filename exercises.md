# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hùng Phát  Mã học viên: 2A202601094

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Đúng tình huống tôi gặp khi deploy lên Railway: tôi tạo service mới nhưng
> quên set biến `API_TOKEN` trên dashboard trước khi deploy lần đầu. Vì
> `api_token` không có default, app crash ngay lúc khởi động với
> `ValidationError`, container không bao giờ mở port, healthcheck fail liên
> tục — Railway báo đỏ ngay trong vài chục giây và tôi biết chính xác phải
> sửa gì. Nếu để mặc định `"changeme"`, app sẽ chạy bình thường, healthz
> xanh, service "hoạt động" — nhưng bất kỳ ai đoán đúng chuỗi `"changeme"`
> cũng gọi được `/chat` miễn phí bằng tiền của tôi, và tôi chỉ phát hiện ra
> khi nhìn hóa đơn LLM cuối tháng, lúc đó thiệt hại đã xảy ra rồi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:27:32.362157+00:00", "client_id": "sv01", "prompt_tokens": 12, "completion_tokens": 37, "usd_cost": 2.265e-05}
> ```
>
> Hai việc làm được mà `print` không làm được:
> 1. **Truy vấn/tổng hợp bằng công cụ log**: vì mỗi dòng là JSON có field
>    riêng biệt (`client_id`, `usd_cost`), tôi có thể lọc theo
>    `event=chat_completed` rồi cộng dồn `usd_cost` theo `client_id` để trả
>    lời "client nào tốn tiền nhất hôm nay?" — chỉ cần một câu query, không
>    cần parse chuỗi bằng regex.
> 2. **Lọc theo mức độ nghiêm trọng**: khóa `severity` viết hoa là quy ước
>    Cloud Logging/nhiều hệ thống log hiểu sẵn, nên tôi bật cảnh báo "báo tôi
>    ngay khi có `severity=ERROR`" mà không cần sửa code — với `print` thì
>    không có khái niệm mức độ, muốn lọc phải tự quy ước và tự viết parser.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.46GB đến từ 3 nguồn chính:
> 1. **Base image**: bản 1 stage dùng `python:3.11` đầy đủ (có sẵn nhiều gói
>    hệ thống, công cụ build, dev headers...) trong khi bản multi-stage
>    dùng `python:3.11-slim` cho cả 2 stage — nhẹ hơn nhiều vì bỏ mọi thứ
>    không cần cho runtime.
> 2. **Không mang theo build tool sang stage cuối**: bản 1 stage chạy
>    `pip install` ngay trong image cuối cùng, nên toàn bộ cache pip,
>    compiler tạm thời (nếu package nào cần build từ source) đều nằm lại
>    trong layer của image. Bản multi-stage cài dependency ở stage
>    `builder` riêng, rồi chỉ `COPY --from=builder /install /usr/local`
>    đúng phần đã cài xong sang stage `runtime` — bỏ lại toàn bộ rác build.
> 3. **`COPY . .` copy cả thư mục project** (kể cả file test, cache,
>    `.git` nếu không có `.dockerignore` đủ tốt) trong bản 1 stage, còn bản
>    multi-stage chỉ `COPY app ./app` và `COPY utils ./utils` — đúng những
>    gì runtime cần.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một dòng comment vào cuối `app/main.py` rồi build lại, output
> thật:
> ```
> [runtime 3/6] COPY --from=builder /install /usr/local   CACHED
> [builder 4/4] RUN pip install --no-cache-dir ...        CACHED
> [builder 3/4] COPY requirements.txt .                   CACHED
> [runtime 4/6] RUN useradd --create-home ...              CACHED
> [runtime 5/6] COPY app ./app                              (chạy lại)
> [runtime 6/6] COPY utils ./utils                          (chạy lại)
> ```
> Toàn bộ stage `builder` (cài dependency) và layer `useradd` đều dùng lại
> cache — chỉ 2 layer `COPY app`/`COPY utils` phải chạy lại vì nội dung thư
> mục `app` đã đổi. Build gần như tức thì vì không phải `pip install` lại.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` (như bản Dockerfile gốc
> 1-stage), thì Docker sẽ hash layer `COPY . .` trước — sửa bất kỳ file nào
> trong project (kể cả 1 dấu phẩy trong code, không đụng đến
> `requirements.txt`) cũng làm layer đó đổi hash, làm hỏng cache của mọi
> layer phía sau nó, kể cả `RUN pip install`. Kết quả: mỗi lần sửa code là
> một lần cài lại toàn bộ thư viện từ đầu, build chậm hẳn dù dependency
> chẳng thay đổi gì.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của tôi có lỗ hổng (ví dụ deserialize dữ
> liệu không kiểm soát, hoặc dependency có RCE) → (2) kẻ tấn công khai thác
> lỗ hổng đó, chạy được lệnh tuỳ ý bên trong tiến trình uvicorn → (3) nếu
> tiến trình đó chạy với UID 0 (root) trong container, lệnh tuỳ ý đó cũng
> chạy với quyền root **bên trong** container → (4) container không phải
> một bức tường tuyệt đối: nếu có thêm một lỗ hổng ở tầng container runtime,
> hoặc container được mount volume/chạy privileged không đúng cách, quyền
> root bên trong dễ leo thang thành quyền cao trên host hơn nhiều so với
> khi tiến trình chỉ là user thường.
>
> Lệnh `USER appuser` cắt đứt chuỗi ngay ở bước (3): dù kẻ tấn công khai
> thác được lỗ hổng ở bước (1)-(2), lệnh tuỳ ý họ chạy được cũng chỉ có
> quyền của `appuser` (UID 10001, không phải root) — không tự ghi được vào
> file hệ thống, không cài được package, và nếu có lỗ hổng thoát container
> ở tầng dưới thì thiệt hại tối đa cũng chỉ tương đương một user thường,
> không phải root.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là bắt buộc theo chuẩn HTTP (RFC 7235/6750):
> response 401 phải nói cho client biết chính xác cơ chế xác thực nào được
> chấp nhận, để client (hoặc thư viện HTTP của họ) biết phải gửi lại request
> kèm header gì. Thiếu header này, client chỉ biết "bị từ chối" mà không
> biết phải sửa theo hướng nào một cách chuẩn hoá.
>
> Trả cùng một thông báo cho cả 3 trường hợp là để không "tặng" thông tin
> cho kẻ đang dò token. Nếu tôi trả lời khác nhau — ví dụ "thiếu header" vs
> "sai token" — thì kẻ tấn công dùng phản hồi đó để biết họ đã đi đúng
> hướng (ví dụ: header đã đúng định dạng, chỉ còn thiếu đoán đúng chuỗi
> token), giúp họ thu hẹp không gian dò từng bước. Trả đồng nhất một câu
> "invalid or missing bearer token" khiến mọi kiểu sai đều trông giống hệt
> nhau từ bên ngoài, không rò rỉ manh mối nào.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Giả sử client đã dùng hết xô (tokens=0) ngay trước khi im lặng 10 phút
> (giống kịch bản "dùng rồi nghỉ" trong guide). `refill_per_second =
> 10/60 ≈ 0.1667`. Sau 600 giây im lặng, lượng token nạp thêm =
> `600 × 0.1667 = 100`.
>
> **Có `min(capacity, ...)`**: token bị chặn ở `capacity=10`, nên client chỉ
> gửi được **10 request** liên tiếp trước khi xô cạn và bị 429.
>
> **Bỏ `min(capacity, ...)`**: token cứ cộng dồn không giới hạn, thành 100.
> Client sẽ gửi được **100 request** liên tiếp trước khi bị 429 — gấp 10 lần
> sức chứa xô đã thiết kế. Lý do: hàm `available()` không còn chặn trên,
> nên một client càng im lặng lâu càng "tích trữ" được vô hạn token, đúng
> cái token bucket được thiết kế để ngăn (cho phép bùng nổ có kiểm soát,
> không phải bùng nổ vô hạn theo thời gian chờ).

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **Hạn mức $30/tháng**: nếu sự cố bắt đầu lúc 2h sáng ngày đầu tháng, client
> có thể đốt hết $30 chỉ trong vài giờ (tuỳ tốc độ gọi), và vì ngân sách
> tính theo cả tháng, không có gì reset cho tới đầu tháng sau — nếu không
> ai để ý dashboard, client này có thể tiếp tục bị chặn (đã hết ngân sách)
> nhưng thiệt hại **tối đa đã là $30**, và có khi sự cố "im lặng" vì server
> chỉ trả 402 sau đó, không ai buồn kiểm tra tại sao. Không có cơ chế tự
> hồi phục cho tới khi có người chủ động refill/reset hoặc chờ hết tháng.
>
> **Hạn mức $1/ngày**: thiệt hại tối đa bị chặn lại ở **$1** — client dùng
> hết trong vài phút đến vài giờ rồi bị 402, không thể vượt quá con số đó
> trong ngày hôm đó dù sự cố có kéo dài bao lâu. Vì key Redis là
> `spend:<client>:<YYYY-MM-DD>`, sang 00:00 UTC là một ngày mới, key mới,
> ngân sách tự về $0 — **service tự hồi phục vào đầu ngày hôm sau mà không
> cần ai can thiệp**. So với hạn mức tháng, thiệt hại tối đa giảm xuống còn
> 1/30, đúng như lý do guide đưa ra.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp `/healthz` thành cũng kiểm tra Redis:
> 1. Redis mất kết nối. `store.ping()` trả `False` ở cả 3 container cùng
>    lúc (vì cả 3 dùng chung một Redis).
> 2. Endpoint gộp trả 503 ở cả 3 container — vì đây bây giờ đóng cả vai trò
>    liveness probe, orchestrator đọc 503 này như "process cần restart",
>    không phải "process tạm thời chưa sẵn sàng".
> 3. Orchestrator (Docker/Railway/K8s) restart **cả 3 container cùng lúc**,
>    vì cả 3 đều "unhealthy" theo liveness probe.
> 4. Trong lúc cả 3 container đang restart (khởi động lại tiến trình,
>    warm up), Redis vẫn chưa kết nối lại được trong 30 giây đó — nên
>    container mới cũng lập tức fail healthcheck ngay khi vừa lên, có thể
>    bị orchestrator restart lặp lại nhiều lần (crash loop).
> 5. Khi Redis quay lại sau 30 giây, **không còn container nào đang chạy ổn
>    định để phục vụ** — cả cụm vừa trải qua một đợt restart đồng loạt do
>    một sự cố lẽ ra chỉ cần "tạm ngừng nhận traffic mới" (readiness), chứ
>    không cần "giết tiến trình" (liveness). Một sự cố Redis 30 giây biến
>    thành downtime toàn cụm, đúng cái mà việc tách `/healthz`/`/readyz`
>    được thiết kế để tránh.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Deploy lên Railway lần đầu, build và deploy đều xanh nhưng bước
> **Healthcheck** liên tục fail. Vào Deploy Logs thấy dòng lỗi:
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
> Tôi tìm nguyên nhân bằng cách vào tab **Details** của deployment, thấy
> Railway tự hiển thị một **Start Command** riêng:
> `uvicorn app.main:app --host 0.0.0.0 --port $PORT`. Đây là cấu hình
> Railway tự sinh khi tạo service từ GitHub, và nó **ghi đè** lên `CMD`
> trong Dockerfile của tôi. Vấn đề là lệnh này chạy trực tiếp (không qua
> shell), nên `$PORT` không được shell expand thành số cổng thật — uvicorn
> nhận đúng chuỗi ký tự `"$PORT"` làm giá trị port và crash ngay khi parse
> argument.
>
> Cách sửa: vào **Settings → Deploy → Custom Start Command**, xoá trắng
> hoàn toàn giá trị đó. Khi để trống, Railway quay lại dùng `CMD` có sẵn
> trong Dockerfile — vốn đã viết đúng dạng
> `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`,
> có `sh -c` nên biến `${PORT:-8000}` được shell thay giá trị thật trước khi
> truyền cho uvicorn. Redeploy lại thì healthcheck pass ngay.
