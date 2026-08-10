# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phùng Đình Đạt  Mã học viên: 2A202601540

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống em gặp thật: deploy lên Render mà quên nhập `API_TOKEN`. Nếu
> `api_token` có mặc định `"changeme"`, container vẫn khởi động, `/healthz`
> vẫn 200, Render báo deploy thành công và service nằm công khai trên
> Internet với một token ai cũng đoán được. Không có tín hiệu nào báo sai —
> em chỉ phát hiện khi thấy log lạ hoặc hóa đơn LLM tăng.
>
> Với `api_token` không mặc định, pydantic ném ValidationError ngay lúc
> `Settings()` chạy, process chết trước khi uvicorn kịp mở cổng, health check
> đỏ và Render không đưa bản deploy đó lên live. Em biết mình thiếu biến môi
> trường trong vòng một phút, và quan trọng hơn: không tồn tại khoảnh khắc nào
> service chạy thật với token yếu. Đổi một lỗi ồn ào lúc deploy lấy việc không
> có lỗi im lặng ở production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log em lấy từ `docker compose logs chat` sau khi gọi `/chat`:
>
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:48:47.425910+00:00", "client_id": "hv-2A202601540", "prompt_tokens": 3, "completion_tokens": 35, "usd_cost": 2.145e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Tổng hợp bằng máy.** `client_id` và `usd_cost` là field có cấu trúc nên
>    em cộng chi phí theo từng client bằng một câu lệnh
>    (`jq -s 'group_by(.client_id) | map({client: .[0].client_id, usd: map(.usd_cost) | add})'`),
>    rồi dựng cảnh báo "client nào vượt $0.8/ngày". Chuỗi text tự do không có
>    con số nào để cộng.
> 2. **Lọc và tương quan theo thời gian.** `ts` chuẩn ISO-8601 kèm timezone và
>    `severity` cho phép em lọc đúng cửa sổ 5 phút quanh sự cố, và đối chiếu với
>    event `service_started` để biết container có vừa restart giữa chừng hay
>    không. Với `print`, em không có mốc thời gian, không có mức độ, và phải
>    viết regex dễ vỡ mỗi lần đổi câu chữ.

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng **1.46GB**, và đó là những thứ chỉ cần lúc *build* chứ
> không cần lúc *chạy*:
>
> - Bản 1 stage dùng `python:3.11` đầy đủ: kèm gcc/g++, make, header của
>   Python và các thư viện hệ thống, git, tài liệu — riêng base image này đã
>   ~1GB, trong khi `python:3.11-slim` chỉ ~120MB.
> - Cache và file trung gian của pip: bản đầu chạy `pip install` không có
>   `--no-cache-dir` nên wheel tải về nằm lại trong layer.
> - `COPY . .` mang cả `tests/`, `LAB_GUIDE.md`, screenshot… vào image.
>
> Bản multi-stage chỉ chuyển đúng kết quả `pip install` (`/install`) từ stage
> `builder` sang `python:3.11-slim`, còn compiler và cache bị bỏ lại ở stage
> builder, không có mặt trong image cuối. Image nhỏ hơn không chỉ để tiết kiệm
> ổ cứng: pull nhanh hơn khi scale, và bề mặt tấn công nhỏ hơn vì trong
> container không có sẵn compiler cho kẻ tấn công dùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Em thêm một dòng vào `app/main.py` rồi `docker build` lại. Kết quả thật:
>
> - **Dùng lại cache:** `[builder 2/4] WORKDIR /build`,
>   `[builder 3/4] COPY requirements.txt .`,
>   `[builder 4/4] RUN pip install ...`,
>   `[runtime 3/6] COPY --from=builder /install /usr/local`.
> - **Phải chạy lại:** `[runtime 4/6] COPY app ./app`,
>   `[runtime 5/6] COPY utils ./utils`, `[runtime 6/6] RUN useradd ... && chown`.
>
> Tức là bước cài thư viện — bước tốn thời gian nhất — không chạy lại. Build
> lần hai chỉ mất vài giây.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, hash của layer COPY đổi mỗi lần
> em sửa bất kỳ ký tự nào trong repo, và Docker phải làm lại **mọi layer đứng
> sau nó**, gồm cả `pip install`. Sửa một dấu chấm cũng phải tải và cài lại
> toàn bộ fastapi/uvicorn/redis. Nguyên tắc rút ra: xếp layer theo tần suất
> thay đổi — thứ ít đổi (requirements.txt) lên trước, thứ đổi liên tục (source
> code) xuống dưới.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code Python có lỗ hổng cho phép chạy lệnh (RCE qua deserialization, path
>    traversal, hoặc một thư viện dính CVE). Kẻ tấn công thực thi lệnh với
>    đúng uid của tiến trình uvicorn — là **uid 0**.
> 2. Là root trong container, nó ghi được vào `/usr`, cài thêm công cụ bằng
>    `apt`/`pip`, đọc `/proc/1/environ` để lấy `API_TOKEN` và `REDIS_URL`.
> 3. Nếu có bind-mount hay volume gắn từ host, nó ghi file vào host với quyền
>    root — ví dụ cắm thêm khóa vào `authorized_keys`.
> 4. Nếu container chạy `--privileged`, hoặc mount `/var/run/docker.sock`, hoặc
>    gặp lỗi thoát container (CVE của runc/kernel), thì **uid 0 trong container
>    ánh xạ thẳng thành uid 0 trên host** — mất luôn máy chủ.
>
> `USER appuser` (uid 10001) cắt chuỗi ở **bước 2**. Lỗ hổng ở bước 1 vẫn còn,
> nhưng kẻ tấn công thừa hưởng một uid vô danh: không ghi được `/usr`, không
> cài được package, không đọc được file của root, và khi lọt ra host thì cũng
> chỉ là uid 10001 không sở hữu gì. `USER` không ngăn được vụ xâm nhập, nó ngăn
> vụ xâm nhập biến thành chiếm quyền toàn hệ thống.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> **Vì sao 401 phải kèm `WWW-Authenticate: Bearer`:** chuẩn HTTP (RFC 9110)
> quy định response 401 *bắt buộc* có header này — nó là phần trả lời cho câu
> hỏi "tôi phải xác thực bằng cách nào để thử lại?". Thiếu nó thì 401 chỉ nói
> "không được vào" mà không nói cửa nào, và các client tự động (SDK, trình
> duyệt, `curl`, API gateway) không biết nên gắn Bearer token, Basic auth hay
> chạy luồng OAuth. Đây là điểm khác nhau giữa 401 (chưa xác thực, thử lại
> được) và 403 (đã biết anh là ai, vẫn không cho).
>
> **Vì sao cùng một message cho cả ba trường hợp:** trong `app/auth.py` cả ba
> nhánh đều trả `"invalid or missing bearer token"`. Nếu phân biệt, endpoint
> trở thành một cái *oracle* cho kẻ dò token: "sai scheme" xác nhận header đã
> tới được chỗ kiểm tra, "sai token" xác nhận định dạng đã đúng — từ đó dò
> được độ dài, prefix, và biết token nào tồn tại nhưng hết hạn. Người dùng hợp
> lệ không mất gì vì họ có tài liệu API để biết cách gắn header; chỉ kẻ dò tìm
> mới cần thông tin chi tiết đó, nên không cho là đúng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> `capacity=10`, `refill_per_minute=10` nghĩa là 1 token mỗi 6 giây.
>
> **Có `min(capacity, ...)`:** sau 10 phút im lặng, `available()` tính
> `tokens + 600 × (10/60) = tokens + 100`, nhưng bị chặn lại còn `min(10, …) = 10`.
> Client gửi được đúng **10 request** liên tiếp; tới request thứ 11 thì
> `tokens < 1` nên nhận **429** kèm `Retry-After: 6`.
>
> **Bỏ `min(capacity, ...)`:** xô tích lũy không giới hạn — im 10 phút từ trạng
> thái cạn thì có ~100 token, cộng phần còn dư lúc dừng thì tới ~110. Client
> bắn được **khoảng 100–110 request trong một giây** rồi mới bị 429.
>
> Chỗ nguy hiểm là con số đó tỉ lệ thuận với thời gian nghỉ: im một ngày thì
> tích 14.400 token. Rate limit sinh ra để bảo vệ hệ thống khỏi burst, mà bỏ
> `min()` thì đúng lúc burst lớn nhất nó lại cho qua hết. `min()` chính là thứ
> biến "tổng số request" thành "tốc độ request".
>
> (Trong code còn `BUCKET_TTL_SECONDS = 3600`: nghỉ quá 1 giờ thì key Redis hết
> hạn, client được coi là mới và xô đầy lại đúng bằng `capacity` — vẫn là 10.)

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **$30/tháng:** một client lỗi gọi liên tục từ 2h sáng có thể đốt sạch $30
> trong vài giờ. Thiệt hại tiền tối đa là $30, nhưng sau đó service từ chối
> client đó cho tới **ngày đầu tháng sau** — nếu sự cố xảy ra ngày 2 thì gần
> 29 ngày chết. Em cũng chỉ phát hiện khi nhìn hóa đơn cuối tháng.
>
> **$1/ngày** (`CostGuard` đặt key theo `%Y-%m-%d` giờ UTC): client đốt tối đa
> **$1** rồi nhận 402 `daily budget exceeded`. Hạn mức tự hồi phục lúc
> **00:00 UTC hôm sau** — sự cố lúc 2h sáng thì client bị chặn đến hết ngày
> hôm đó, tối đa khoảng 22 giờ, không cần ai can thiệp thủ công.
>
> Xấu nhất cả tháng thì hai cách vẫn ra ~$30, nhưng cách theo ngày chia rủi ro
> thành 30 lát mỏng: không bao giờ có cú "cháy sạch ngân sách trong một đêm rồi
> chết cả tháng". Thêm nữa, mốc theo ngày biến sự cố thành cảnh báo sớm — em
> thấy 402 ngay sáng hôm đó, còn mốc theo tháng thì im lặng cho tới khi quá
> muộn. Nếu muốn chặt hơn nữa thì đặt cả hai tầng: trần ngày để giới hạn thiệt
> hại, trần tháng để giới hạn tổng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Gộp làm một endpoint có kiểm tra Redis, cụm 3 container, Redis mất 30 giây:
>
> 1. **t = 0s** — Redis mất kết nối. Cả 3 container vẫn chạy bình thường, chỉ
>    phần lưu history/rate limit là không dùng được.
> 2. **t ≈ 0–10s** — cả 3 container cùng trả 503 trên endpoint gộp đó, vì cả 3
>    nói chuyện với đúng một Redis.
> 3. **Load balancer** đọc endpoint đó như readiness → rút cả 3 khỏi pool.
>    Traffic từ ngoài vào nhận 502/503: **downtime 100%**, kể cả những request
>    không cần Redis.
> 4. **Orchestrator** đọc *cùng* endpoint đó như liveness → sau đủ số lần fail
>    (mặc định 3 lần × 10s) thì kết luận cả 3 container đã hỏng và **kill rồi
>    restart đồng loạt cả 3**.
> 5. Container mới khởi động lại vẫn không thấy Redis → lại 503 → lại bị kill.
>    Vòng lặp CrashLoopBackOff, kèm backoff tăng dần; mọi kết nối đang mở và
>    state trong process bị mất.
> 6. **t = 30s** — Redis sống lại, nhưng cụm đang trong backoff và khởi động
>    lạnh, nên dịch vụ trở lại muộn hơn 30 giây khá nhiều.
>
> Tách hai endpoint thì: `/healthz` không đụng Redis nên **không container nào
> bị restart** (bước 4, 5 biến mất hoàn toàn); `/readyz` trả 503 nên LB tạm
> ngừng đẩy traffic. Tới t = 30s Redis về, `/readyz` xanh lại ngay và traffic
> vào lại tức thì — tổng downtime đúng bằng 30 giây của sự cố gốc. Một câu tóm
> lại: **liveness trả lời "có cần giết tôi không", readiness trả lời "có nên
> gửi traffic cho tôi lúc này không"** — trộn hai câu hỏi đó là biến một sự cố
> phụ thuộc bên ngoài thành một vụ restart toàn cụm.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** Render từ chối Blueprint ngay ở bước Apply, báo cấu hình không hợp
> lệ, chưa tạo được service nào.
>
> **Tìm nguyên nhân:** thay vì đoán, em parse chính file đó bằng đúng thư viện
> YAML:
>
> ```bash
> python3 -c "import yaml, json; print(json.dumps(yaml.safe_load(open('render.yaml')), indent=1))"
> ```
>
> Kết quả in ra `"persistenceMode": false` — trong khi em viết `persistenceMode: off`
> và nghĩ nó là chuỗi `"off"`. YAML 1.1 coi `off/on/yes/no` là **boolean**, nên
> Render nhận được kiểu dữ liệu sai so với spec (trường này chỉ nhận chuỗi
> `journal-snapshot`, `snapshot` hoặc `off`).
>
> Để chắc chắn lỗi không nằm ở app, em build và chạy thử đúng image mà Render
> sẽ dùng, với biến môi trường giống Render:
>
> ```
> docker build -t day12-chat:render .      → OK
> PORT=10000 → GET /healthz  200
>              GET /readyz   503 {"redis": false}   (đúng, chưa có Redis)
>              POST /chat không token  401
> ```
>
> **Cách sửa:** bỏ hẳn `persistenceMode` khỏi `render.yaml` — gói free của
> Render vốn đã ép persistence là off, nên khai báo thừa chỉ tạo thêm chỗ để
> sai.
>
> **Bài học:** trong YAML, mọi giá trị `off/on/yes/no/y/n` phải đặt trong ngoặc
> kép nếu muốn nó là chuỗi. Và khi một platform báo "file cấu hình sai", parse
> file đó tại máy trước — nhìn kiểu dữ liệu thật nhanh hơn đọc lại YAML bằng
> mắt rất nhiều.
