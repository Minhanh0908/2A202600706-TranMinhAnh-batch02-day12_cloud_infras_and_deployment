# Solution — Code Lab: Deploy Your AI Agent to Production

> **2A202600706 — Tran Minh Anh · AI20K Batch 02**  

---

## Part 1: Localhost vs Production

### Exercise 1.1: Phát hiện anti-patterns

Đọc file `01-localhost-vs-production/develop/app.py`, tìm được **8 vấn đề**:

| # | Vị trí | Vấn đề | Giải thích |
|---|--------|--------|------------|
| 1 | `OPENAI_API_KEY = "sk-hardcoded..."` | **API key hardcode** | Push lên GitHub → key bị lộ ngay lập tức |
| 2 | `DATABASE_URL = "postgresql://admin:password123@..."` | **Credentials hardcode** | Password DB trong source code |
| 3 | `DEBUG = True`, `MAX_TOKENS = 500` | **Không có config management** | Config cứng trong code, không đọc từ env vars |
| 4 | `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` | **Log ra secret** | In API key ra stdout — lộ trong log hệ thống |
| 5 | Không có `/health` endpoint | **Không có health check** | Platform (Railway/Render/K8s) không biết khi nào restart |
| 6 | `host="localhost"` trong `uvicorn.run()` | **Bind sai host** | Chỉ nhận request từ trong máy, không deploy được lên cloud |
| 7 | `port=8000` cố định | **Port cứng** | Trên Railway/Render, PORT được inject qua env var |
| 8 | `reload=True` trong production | **Debug reload** | Gây memory leak + instability trong production |

> Lưu ý thêm: `question: str` trong `ask_agent()` được FastAPI đọc là **query parameter**, không phải request body — cần Pydantic model để nhận JSON body đúng cách.

---

### Exercise 1.2: Chạy basic version

```bash
cd 01-localhost-vs-production/develop
pip install -r requirements.txt
python app.py
```

**Kết quả quan sát:**

App chạy được trên local. Tuy nhiên, gọi API bằng JSON body sẽ gặp lỗi 422:

```bash
# ❌ Lỗi — FastAPI đọc là query param, không phải JSON body
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# → {"detail":[{"type":"missing","loc":["query","question"],"msg":"Field required"}]}

# ✅ Đúng — dùng query param
curl -X POST "http://localhost:8000/ask?question=Hello"
```

**Kết luận:** Chạy được nhưng **không production-ready** vì tất cả 8 vấn đề ở Exercise 1.1.

---

### Exercise 1.3: So sánh basic vs advanced

| Feature | Basic | Advanced | Tại sao quan trọng? |
|---------|-------|----------|---------------------|
| **Config** | Hardcode trong code | Đọc từ env vars qua `os.getenv()` hoặc `pydantic_settings` | Bảo mật secrets, dễ thay đổi theo môi trường |
| **Health check** | Không có | Có `/health` endpoint | Platform biết container còn sống để restart khi cần |
| **Logging** | `print()` | Structured JSON logging | Dễ search/filter trong log aggregation tools (Datadog, CloudWatch) |
| **Shutdown** | Đột ngột (SIGKILL) | Graceful — hoàn thành request trước khi tắt | Không drop request đang xử lý khi deploy version mới |
| **Host binding** | `host="localhost"` | `host="0.0.0.0"` | Nhận request từ bên ngoài container/server |
| **Port** | Cứng `8000` | `int(os.getenv("PORT", 8000))` | Tương thích với cloud platforms inject PORT qua env |
| **Request body** | Query param (`str`) | Pydantic model | Nhận JSON body đúng cách, có validation tự động |

---

## Part 2: Docker Containerization

### Exercise 2.1: Đọc Dockerfile cơ bản

File: `02-docker/develop/Dockerfile`

**1. Base image là gì?**

```dockerfile
FROM python:3.11
```

`python:3.11` — full Python distribution, bao gồm toàn bộ standard library, pip, build tools. Kích thước ~1GB.

**2. Working directory là gì?**

```dockerfile
WORKDIR /app
```

Mọi lệnh `RUN`, `COPY`, `CMD` phía sau đều chạy trong `/app` bên trong container.

**3. Tại sao COPY requirements.txt trước?**

```dockerfile
COPY requirements.txt .          # Layer 1
RUN pip install -r requirements  # Layer 2  ← được cache
COPY app.py .                    # Layer 3
```

Docker build theo từng layer và cache lại. Nếu `requirements.txt` không thay đổi, Docker dùng cache cho bước `pip install` — tiết kiệm thời gian build đáng kể. Nếu COPY code trước, mỗi lần sửa code sẽ invalidate cache và phải pip install lại từ đầu.

**4. CMD vs ENTRYPOINT khác nhau thế nào?**

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| Mục đích | Default command, **có thể override** | Command cố định, **không override được** |
| Override | `docker run image <lệnh_khác>` | Cần `--entrypoint` flag |
| Dùng khi | App có thể chạy nhiều mode | Container chỉ làm 1 việc duy nhất |
| Ví dụ | `CMD ["python", "app.py"]` → có thể override thành `python shell.py` | `ENTRYPOINT ["python", "app.py"]` → luôn chạy app.py |

Dockerfile cơ bản dùng `CMD ["python", "app.py"]` — phù hợp cho lab vì dễ override khi debug.

---

### Exercise 2.2: Build và run (develop)

**Build image:**

```bash
# Đứng tại root project
cd /d/AI20K/Lab/2A202600706-TranMinhAnh-batch02-day12_cloud_infras_and_deployment

docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
```

> **Lưu ý quan trọng:** Phải chạy từ **root project**, không phải từ `02-docker/develop/`. Build context (`.`) phải chứa cả `utils/mock_llm.py` lẫn `02-docker/develop/app.py`.

**Run container:**

```bash
docker run -d -p 8080:8000 my-agent:develop
```

> Dùng port `8080` thay vì `8000` vì trên Windows, `localhost` resolve sang IPv6 `[::1]` trong khi Docker bind IPv4. Dùng `127.0.0.1` để tránh conflict:

```bash
curl http://127.0.0.1:8080/
# {"message":"Agent is running in a Docker container!"}

curl -X POST "http://127.0.0.1:8080/ask?question=What+is+Docker"
# {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

**Kích thước image:**

```bash
docker images my-agent:develop
# my-agent   develop   c96f3ef709cd   ...   1.66GB
```

→ **1.66 GB** — lớn vì dùng `python:3.11` full image.

**Lỗi hay gặp và cách fix:**

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `Cannot find file 02-docker` | Chạy build từ sai thư mục | Về root project |
| `curl: (52) Empty reply` | curl dùng IPv6 `[::1]`, Docker bind IPv4 | Dùng `127.0.0.1` thay `localhost` |
| `422 Field required` | `question: str` là query param | Dùng `?question=...` hoặc sửa code thêm Pydantic |

---

### Exercise 2.3: Multi-stage build (production)

**Đọc Dockerfile production — 2 stages:**

**Stage 1 — Builder:**
```dockerfile
FROM python:3.11-slim AS builder
RUN apt-get install -y gcc libpq-dev   # Build tools
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt  # Compile packages
```
Mục đích: Cài đặt và compile tất cả dependencies (một số package Python cần gcc để compile C extensions như `psycopg2`).

**Stage 2 — Runtime:**
```dockerfile
FROM python:3.11-slim AS runtime
RUN groupadd -r appuser && useradd -r -g appuser appuser  # Non-root user
COPY --from=builder /root/.local /home/appuser/.local     # Chỉ copy kết quả
COPY main.py .
COPY utils/mock_llm.py utils/
RUN chown -R appuser:appuser /app
```
Mục đích: Chỉ lấy compiled packages từ Stage 1, **bỏ hoàn toàn** gcc, libpq-dev, source cache.

**Tại sao image nhỏ hơn?**

```
Stage 1 (builder):  python:3.11-slim + gcc + libpq + build cache ≈ 1.5GB
                                              ↓ chỉ copy /root/.local
Stage 2 (runtime):  python:3.11-slim + compiled packages only    ≈ 236MB
```

**Build và so sánh:**

```bash
# Build từ root project
docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .

# So sánh kích thước
docker images | grep my-agent
# my-agent   advanced   12f782a92594   ...   236MB    ✅
# my-agent   develop    c96f3ef709cd   ...   1.66GB
```

**Kết quả: Giảm 86%** — từ 1.66GB xuống còn 236MB.

**Lợi ích thực tế của multi-stage build:**

| | Single-stage | Multi-stage |
|---|---|---|
| Image size | 1.66 GB | 236 MB |
| Pull time (deploy) | ~3 phút | ~30 giây |
| Attack surface | Có gcc, build tools | Chỉ có runtime |
| Security | Thấp hơn | Cao hơn (non-root user) |
| Build cache | Kém hiệu quả | Tối ưu theo layer |

---

---

### Exercise 2.4: Docker Compose stack

**Architecture diagram:**

```
┌─────────────────────────────────────────────────┐
│                  Host Machine                   │
│                                                 │
│   curl http://127.0.0.1:80                      │
│          │                                      │
│          ▼                                      │
│   ┌─────────────┐                               │
│   │    Nginx    │  port 80/443 (exposed)        │
│   │  (reverse   │                               │
│   │   proxy)    │                               │
│   └──────┬──────┘                               │
│          │  proxy_pass http://agent:8000        │
│          │  (internal network)                  │
│          ▼                                      │
│   ┌─────────────┐                               │
│   │    Agent    │  port 8000 (NOT exposed)      │
│   │  (FastAPI   │  2 workers (pid 8, 9)         │
│   │  + uvicorn) │                               │
│   └──────┬──────┘                               │
│          │                                      │
│     ┌────┴────┐                                 │
│     ▼         ▼                                 │
│  ┌──────┐  ┌────────┐                           │
│  │Redis │  │ Qdrant │  (internal only)          │
│  │:6379 │  │ :6333  │                           │
│  └──────┘  └────────┘                           │
│                                                 │
│  Network: production_internal (bridge)          │
└─────────────────────────────────────────────────┘
```

**Services được start:**

| Service | Image | Role | Port |
|---------|-------|------|------|
| `agent` | `production-agent:latest` | FastAPI AI agent | 8000 (internal) |
| `redis` | `redis:7-alpine` | Session cache + rate limiting | 6379 (internal) |
| `qdrant` | `qdrant/qdrant:v1.9.0` | Vector database cho RAG | 6333 (internal) |
| `nginx` | `nginx:alpine` | Reverse proxy + load balancer | **80, 443 (exposed)** |

**Cách các services communicate:**

Tất cả nằm trong network `production_internal` (bridge). Docker DNS tự động resolve tên service thành IP container — ví dụ `agent` có thể gọi `redis://redis:6379` mà không cần biết IP thực.

```
Client → Nginx (port 80) → Agent (port 8000) → Redis / Qdrant
                                    ↑
                         Chỉ Nginx mới expose ra ngoài
                         Agent/Redis/Qdrant hoàn toàn internal
```

**Lỗi gặp phải và cách fix:**

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `env file .env.local not found` | Chưa tạo file secrets | `echo "AGENT_API_KEY=secret-key-123" > .env.local` |
| `qdrant unhealthy` | Image không có `curl` để chạy healthcheck | Đổi healthcheck dùng `bash TCP check` hoặc `depends_on: condition: service_started` |
| `build context sai` | `docker compose up` chạy từ `02-docker/production/`, Dockerfile COPY path từ root | Sửa `context: ../..` trong `docker-compose.yml` |
| `curl: (52) Empty reply` | `localhost` resolve IPv6 `[::1]` | Dùng `127.0.0.1` |

**Kết quả test:**

```bash
curl http://127.0.0.1/health
# {"status":"ok","uptime_seconds":369.3,"version":"2.0.0","timestamp":"2026-06-12T09:49:10.870143"}

curl http://127.0.0.1/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
# {"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

**Quan sát thêm từ log:**

```
agent-1 | INFO: Started parent process [1]
agent-1 | INFO: Started server process [9]   ← worker 1
agent-1 | INFO: Started server process [8]   ← worker 2
```

Production mode chạy **multiple workers** (2 processes) thay vì 1 như develop — tăng throughput, tận dụng multi-core CPU.

---

### Checkpoint 2 ✅

- [x] Hiểu cấu trúc Dockerfile — base image, WORKDIR, layer cache, CMD vs ENTRYPOINT
- [x] Biết lợi ích của multi-stage builds — giảm 86% image size (1.66GB → 236MB)
- [x] Hiểu Docker Compose orchestration — 4 services, internal network, depends_on
- [x] Biết cách debug container — `docker logs`, `docker compose ps`, `docker exec`

---

---

## Part 3: Cloud Deployment

### Concepts

**Vấn đề:** Laptop không thể chạy 24/7, không có public IP.

**Giải pháp:** Cloud platforms — Railway, Render, GCP Cloud Run.

| Platform | Độ khó | Free tier | Best for |
|----------|--------|-----------|----------|
| Railway | ⭐ | $5 credit | Prototypes |
| Render | ⭐⭐ | 750h/month | Side projects |
| Cloud Run | ⭐⭐⭐ | 2M requests | Production |

---

### Exercise 3.1: Deploy Railway

**Các bước thực hiện:**

```bash
cd 03-cloud-deployment/railway

# 1. Cài Railway CLI
npm i -g @railway/cli

# 2. Login
railway login
# → Mở browser, đăng nhập → ✓ Signed in as Tran Minh Anh

# 3. Khởi tạo project
railway init
# → Chọn workspace, đặt tên project: code_lab_12

# 4. Set environment variables
railway variables set PORT=8000
railway variables set AGENT_API_KEY=secret-key-123

# 5. Deploy
railway up
# → Chọn service → upload + build trên Railway

# 6. Tạo public domain
railway domain
# 🚀 https://2a202600706-tranminhanh-batch02-day12cloudinfr-production.up.railway.app
```

**Lỗi gặp phải và cách fix:**

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `Project has no services` | Chạy `railway variables set` trước khi có service | Tạo service qua dashboard (`railway open`) trước |
| `Service "create" not found` | Railway CLI v5 đổi syntax, không còn `railway service create` | Dùng `railway open` → tạo service trên web UI |
| `operation timed out` khi `railway up` | Mạng không ổn định | Build vẫn chạy trên server — kiểm tra Build Logs URL trong output |
| `Not Found` trên browser ngay sau deploy | Build chưa hoàn thành | Chờ thêm ~1-2 phút rồi reload |

**Kết quả test:**

```bash
# Health check
curl https://2a202600706-tranminhanh-batch02-day12cloudinfr-production.up.railway.app/health
# {"status":"ok","uptime_seconds":66.8,"platform":"Railway","timestamp":"2026-06-12T10:03:05.640344+00:00"}

# Agent endpoint
curl https://2a202600706-tranminhanh-batch02-day12cloudinfr-production.up.railway.app/ask \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello from Railway!"}'
# {"question":"Hello from Railway!","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","platform":"Railway"}
```

**Quan sát:** Response có thêm field `"platform":"Railway"` — app detect được môi trường đang chạy qua env var `ENVIRONMENT`.

---

### Checkpoint 3 ✅

- [x] Deploy thành công lên Railway
- [x] Có public URL hoạt động: `https://2a202600706-tranminhanh-batch02-day12cloudinfr-production.up.railway.app`
- [x] Hiểu cách set environment variables trên cloud (`railway variables set`)
- [x] Biết cách xem logs (`railway logs`)

---

---

## Part 4: API Security

### Concepts

**Vấn đề:** Public URL = ai cũng gọi được = hết tiền OpenAI.

**Giải pháp:**
1. **Authentication** — Chỉ user hợp lệ mới gọi được
2. **Rate Limiting** — Giới hạn số request/phút
3. **Cost Guard** — Dừng khi vượt budget

---

### Exercise 4.1: API Key Authentication

**API key được check ở đâu?**

Dùng `APIKeyHeader` của FastAPI làm Security scheme, inject qua `Depends`:

```python
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_api_key(api_key: str = Security(api_key_header)) -> str:
    if not api_key:
        raise HTTPException(status_code=401, detail="Missing API key...")
    if api_key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API key.")
    return api_key

@app.post("/ask")
async def ask_agent(
    question: str,
    _key: str = Depends(verify_api_key),  # ← inject vào đây
):
    ...
```

Endpoint `/health` và `/` không có `Depends(verify_api_key)` → public, không cần auth.

**Điều gì xảy ra nếu sai/thiếu key?**

| Trường hợp | HTTP Status | Response |
|------------|-------------|----------|
| Không có header `X-API-Key` | `401 Unauthorized` | `{"detail":"Missing API key..."}` |
| Sai key | `403 Forbidden` | `{"detail":"Invalid API key."}` |
| Đúng key | `200 OK` | `{"question":..., "answer":...}` |

**Làm sao rotate key?**

API key đọc từ env var `AGENT_API_KEY` — không hardcode trong code:

```python
API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")
```

Để rotate: đổi giá trị env var rồi restart service — không cần sửa code hay redeploy image.

**Kết quả test:**

```bash
# Không có key → 401
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Sai key → 403
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: secret-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"detail":"Invalid API key."}

# Đúng key → 200
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: demo-key-change-in-production" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"question":"Hello","answer":"..."}
```

**Quan sát từ server log:**

```
127.0.0.1 - "POST /ask HTTP/1.1" 401 Unauthorized   ← không có key
127.0.0.1 - "POST /ask HTTP/1.1" 403 Forbidden       ← sai key
```

---

---

### Exercise 4.2: JWT Authentication (Advanced)

**JWT flow hoạt động thế nào (`auth.py`):**

```
Client                          Server
  │                               │
  │  POST /auth/token             │
  │  {username, password}  ──────►│ authenticate_user()
  │                               │ create_token() → jwt.encode()
  │◄────── {access_token: ...} ───│
  │                               │
  │  POST /ask                    │
  │  Authorization: Bearer <token>│
  │                        ──────►│ verify_token() → jwt.decode()
  │                               │ extract: sub, role, exp
  │◄────── {answer: ...} ─────────│
```

Token JWT gồm 3 phần (base64): `header.payload.signature`
- **Header:** algorithm (HS256)
- **Payload:** `sub` (username), `role`, `iat` (issued at), `exp` (expiry)
- **Signature:** HMAC-SHA256 của header+payload with SECRET_KEY — không thể giả mạo

**Lỗi gặp phải và cách fix:**

| Lỗi | Nguyên nhân | Fix |
|-----|-------------|-----|
| `AttributeError: module 'jwt' has no attribute 'encode'` | Cài nhầm package `jwt` v1.4.0 thay vì `PyJWT` | `pip uninstall jwt -y && pip install PyJWT` |
| `AttributeError: 'MutableHeaders' has no attribute 'pop'` | Starlette mới không support `.pop()` trên headers | Đổi thành `del response.headers["server"]` |
| `/token` → 404 | Endpoint thực tế là `/auth/token` | Dùng đúng path `/auth/token` |

**Kết quả test:**

```bash
# Bước 1: Lấy token
curl http://localhost:8000/auth/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'
# {"access_token":"eyJhbGci...","token_type":"bearer","expires_in_minutes":60}

# Bước 2: Dùng token gọi API
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW..."

curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'
# {"question":"Explain JWT","answer":"...","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
```

**Quan sát:** Response trả về thêm `usage` — tracking `requests_remaining` và `budget_remaining_usd` realtime.

---

### Exercise 4.3: Rate Limiting

**Đọc `rate_limiter.py` — trả lời các câu hỏi:**

**Algorithm nào được dùng?**

Fixed-window counter lưu trong memory (dict), key theo từng user. Mỗi request tăng counter lên 1; counter reset khi sang phút mới (so sánh timestamp window hiện tại với window đã lưu).

**Limit là bao nhiêu requests/minute?**

Limit theo role, đọc từ thông tin user trong JWT payload:

| Role | Limit |
|------|-------|
| `student` | **10 requests/phút** |
| `teacher` | **100 requests/phút** |

**Làm sao bypass limit cho admin?**

Role `teacher` (hoặc admin) có limit cao hơn (100 req/min) thay vì bị chặn hoàn toàn — bypass thực hiện bằng cách cấp limit riêng theo `role` lấy từ JWT, không hardcode chung 1 mức cho mọi user.

**Kết quả test (gọi liên tục bằng user `student`, limit 10 req/min):**

```bash
for i in {1..20}; do
  curl http://localhost:8000/ask -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done
```

**Quan sát từ server log:**

```
INFO:cost_guard:Usage: user=student req=1  cost=$0.0000/1.0  → 200 OK
...
INFO:cost_guard:Usage: user=student req=12 cost=$0.0002/1.0  → 200 OK
INFO:     "POST /ask HTTP/1.1" 429 Too Many Requests   ← request 13
INFO:     "POST /ask HTTP/1.1" 429 Too Many Requests   ← request 14
... (tiếp tục 429 cho đến request 20)
```

→ Sau khi vượt limit (10 req/min đối với `student`, thực tế cho phép đến 12 do cách tính window), server trả về **`429 Too Many Requests`** cho tất cả request tiếp theo trong cùng cửa sổ phút đó. Khi sang phút mới, counter reset và request lại được chấp nhận.

---

---

### Exercise 4.4: Cost Guard

**Đọc `cost_guard.py` — logic `check_budget()`:**

```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    """
    Return True nếu còn budget, False nếu vượt.

    - Mỗi user có budget riêng (demo: $1/day cho student và teacher)
    - Track spending trong memory, key theo user + ngày hiện tại
    - Reset tự động khi sang ngày mới (đổi date_key)
    """
    today_key = f"{user_id}:{date.today()}"
    current = usage_db.get(today_key, 0.0)

    if current + estimated_cost > DAILY_BUDGET[user_id]:
        return False

    usage_db[today_key] = current + estimated_cost
    return True
```

**Cách hoạt động:**

1. Trước mỗi request, tính `estimated_cost` dựa trên độ dài input/output (ước lượng theo token).
2. Cộng `estimated_cost` vào tổng đã dùng trong ngày (`usage_db[today_key]`).
3. Nếu tổng vượt `DAILY_BUDGET` của user → raise `HTTPException(402)`.
4. Nếu còn budget → cho request đi qua và cập nhật usage.

**Budget demo:**

| User | Budget/ngày | Rate limit |
|------|-------------|------------|
| `student` | $1.00 | 10 req/min |
| `teacher` | $1.00 | 100 req/min |

**Kết quả test:**

```
INFO:cost_guard:Usage: user=student req=1  cost=$0.0000/1.0
INFO:cost_guard:Usage: user=student req=2  cost=$0.0000/1.0
INFO:cost_guard:Usage: user=student req=3  cost=$0.0001/1.0
...
INFO:cost_guard:Usage: user=student req=12 cost=$0.0002/1.0
```

→ Mỗi request có cost rất nhỏ (mock LLM), tổng cost tăng dần và được log realtime dạng `cost=$X/BUDGET`. Trong demo này, **rate limit (10-12 req/min)** chạm ngưỡng trước khi **cost guard ($1/day)** kịp kích hoạt — với mock LLM giá ~$0.0002/request, cần ~5000 requests mới chạm budget $1, trong khi rate limit đã chặn lại ở request 13.

**Kết luận:** 2 lớp bảo vệ hoạt động độc lập và bổ trợ nhau:
- **Rate limiter** chặn spike traffic ngắn hạn (per-minute).
- **Cost guard** chặn tổng chi phí dài hạn (per-day/month), bảo vệ khi traffic dàn trải đều nhưng tổng vẫn vượt budget.

---

### Checkpoint 4 ✅

- [x] Implement API key authentication
- [x] Hiểu JWT flow
- [x] Implement rate limiting (fixed-window, theo role)
- [x] Implement cost guard (daily budget, theo user)

---

## Part 5: Scaling & Reliability

### Concepts

**Vấn đề:** 1 instance không đủ khi có nhiều users.

**Giải pháp:**
1. **Stateless design** — Không lưu state trong memory
2. **Health checks** — Platform biết khi nào restart
3. **Graceful shutdown** — Hoàn thành requests trước khi tắt
4. **Load balancing** — Phân tán traffic

---

### Exercise 5.1: Health checks

**File:** `05-scaling-reliability/develop/app.py`

So với template TODO trong đề bài (chỉ trả `{"status": "ok"}` và `{"status": "ready"}`), bản implement ở đây nâng cao hơn:

```python
@app.get("/health")
def health():
    """LIVENESS PROBE — Agent có còn sống không?"""
    uptime = round(time.time() - START_TIME, 1)
    checks = {}
    try:
        import psutil
        mem = psutil.virtual_memory()
        checks["memory"] = {
            "status": "ok" if mem.percent < 90 else "degraded",
            "used_percent": mem.percent,
        }
    except ImportError:
        checks["memory"] = {"status": "ok", "note": "psutil not installed"}

    overall_status = "ok" if all(v.get("status") == "ok" for v in checks.values()) else "degraded"
    return {
        "status": overall_status,
        "uptime_seconds": uptime,
        "version": "1.0.0",
        "environment": os.getenv("ENVIRONMENT", "development"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "checks": checks,
    }


@app.get("/ready")
def ready():
    """READINESS PROBE — Agent có sẵn sàng nhận traffic không?"""
    if not _is_ready:
        raise HTTPException(503, "Agent not ready. Check back in a few seconds.")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Khác biệt so với solution mẫu trong đề:**

| | Solution mẫu (đề bài) | Bản implement |
|---|---|---|
| `/health` | `{"status": "ok"}` | Thêm `uptime_seconds`, `version`, `environment`, `timestamp`, `checks.memory` |
| `/ready` | Check Redis + DB bằng `try/except` | Check biến `_is_ready` (set `True` sau khi lifespan startup hoàn tất, set `False` khi shutdown) + trả `in_flight_requests` |

→ Vì bản develop chưa có Redis/DB, `/ready` dùng flag `_is_ready` toggled trong `lifespan()` — tương đương "đã load model/dependencies xong chưa". Ở bản production (5.3) sẽ cần thêm check Redis thật như solution mẫu.

**Kết quả test:**

```bash
curl http://localhost:8000/health
# {"status":"ok","uptime_seconds":16.6,"version":"1.0.0","environment":"development",
#  "timestamp":"2026-06-12T14:44:06.018692+00:00","checks":{"memory":{"status":"ok","used_percent":83.0}}}

curl http://localhost:8000/ready
# {"ready":true,"in_flight_requests":1}
```

**Quan sát:** `in_flight_requests:1` vì middleware `track_requests` đếm cả request `/ready` đang được xử lý ngay lúc gọi — giá trị này tăng/giảm realtime theo số request đồng thời. Lần test này, package `psutil` đã được cài nên `checks.memory` trả về `used_percent` thực tế (83.0%) thay vì fallback `"note":"psutil not installed"`.

---

### Exercise 5.2: Graceful shutdown

**Cơ chế trong code (`app.py`):**

```python
@app.middleware("http")
async def track_requests(request, call_next):
    global _in_flight_requests
    _in_flight_requests += 1
    try:
        response = await call_next(request)
        return response
    finally:
        _in_flight_requests -= 1
```

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _is_ready
    # startup
    _is_ready = True
    yield
    # shutdown
    _is_ready = False
    logger.info("🔄 Graceful shutdown initiated...")

    timeout = 30
    elapsed = 0
    while _in_flight_requests > 0 and elapsed < timeout:
        logger.info(f"Waiting for {_in_flight_requests} in-flight requests...")
        time.sleep(1)
        elapsed += 1

    logger.info("✅ Shutdown complete")
```

```python
def handle_sigterm(signum, frame):
    logger.info(f"Received signal {signum} — uvicorn will handle graceful shutdown")

signal.signal(signal.SIGTERM, handle_sigterm)
signal.signal(signal.SIGINT, handle_sigterm)
```

**Cách hoạt động:**

1. Mỗi request đi qua middleware `track_requests` → tăng `_in_flight_requests` khi bắt đầu, giảm khi xong (kể cả lỗi, nhờ `finally`).
2. Khi nhận `SIGTERM`/`SIGINT`, Uvicorn (với `timeout_graceful_shutdown=30`) ngừng nhận connection mới và gọi `lifespan` shutdown.
3. `lifespan` shutdown set `_is_ready = False` (→ `/ready` trả 503, load balancer ngừng route traffic vào instance này) rồi **chờ tối đa 30s** cho `_in_flight_requests` về 0 trước khi log "Shutdown complete" và exit.

**Kết quả test:**

```bash
python app.py
```
```
2026-06-12 21:43:49,386 INFO Starting agent on port 8000
INFO:     Started server process [7624]
INFO:     Waiting for application startup.
2026-06-12 21:43:49,493 INFO Agent starting up...
2026-06-12 21:43:49,493 INFO Loading model and checking dependencies...
2026-06-12 21:43:49,693 INFO ✅ Agent is ready!
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Gọi `/health`, `/ready`, `/ask` (terminal khác):
```bash
curl http://localhost:8000/health
curl http://localhost:8000/ready
curl -X POST "http://localhost:8000/ask?question=Hello"
# {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```
```
INFO:     127.0.0.1:49681 - "GET /health HTTP/1.1" 200 OK
INFO:     127.0.0.1:49683 - "GET /ready HTTP/1.1" 200 OK
INFO:     127.0.0.1:49685 - "POST /ask?question=Hello HTTP/1.1" 200 OK
```

Nhấn `Ctrl+C` (SIGINT) ở terminal chạy app:
```
INFO:     Shutting down
INFO:     Waiting for application shutdown.
2026-06-12 21:44:34,331 INFO 🔄 Graceful shutdown initiated...
2026-06-12 21:44:34,331 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [7624]
2026-06-12 21:44:34,333 INFO Received signal 2 — uvicorn will handle graceful shutdown
```

**Quan sát:**

1. Uvicorn nhận `Ctrl+C` → log "Shutting down" → gọi `lifespan` shutdown phase.
2. Lifespan shutdown chạy ngay: vì lúc này `_in_flight_requests = 0` (request `/ask` trước đó đã hoàn thành từ lâu), nên while loop không lặp, log "🔄 Graceful shutdown initiated..." rồi ngay "✅ Shutdown complete" — không có dòng "Waiting for N in-flight requests...".
3. **Thứ tự log đáng chú ý:** `handle_sigterm` (dòng cuối, "Received signal 2") được log **sau** khi lifespan shutdown đã hoàn tất và process gần kết thúc. Lý do: signal handler của Python (`signal.signal(signal.SIGINT, handle_sigterm)`) và signal handler nội bộ của Uvicorn cùng nhận SIGINT; Uvicorn xử lý graceful shutdown trước (qua lifespan), còn handler tùy chỉnh của ta chỉ log thông tin và được flush/in ra cuối cùng — không ảnh hưởng đến thứ tự shutdown thực tế, chỉ là buffering/scheduling của log.
4. "Signal 2" = `SIGINT` (Ctrl+C), đúng như `signal.signal(signal.SIGINT, handle_sigterm)` đã đăng ký.

**Để thấy log "Waiting for N in-flight requests..."**, cần có request còn đang xử lý (`_in_flight_requests > 0`) tại đúng thời điểm Ctrl+C được nhấn — ví dụ thêm `time.sleep(3)` giả lập xử lý lâu trong mock LLM, rồi gửi request và Ctrl+C ngay sau đó trong vài trăm ms.

---

### Checkpoint 5 (phần 5.1–5.2)

- [x] Implement health check (`/health`) — liveness probe với uptime, version, memory check
- [x] Implement readiness check (`/ready`) — readiness dựa trên `_is_ready` flag + `in_flight_requests`
- [x] Implement graceful shutdown — signal handler + lifespan shutdown chờ in-flight requests
- [ ] Refactor code thành stateless (5.3 — cần bản `production` với Redis)
- [ ] Load balancing with Nginx + test stateless (5.4, 5.5)

---

---

### Exercise 5.3: Stateless design

**File:** `05-scaling-reliability/production/app.py`

**Anti-pattern (theo đề bài) vs cách implement thực tế:**

```python
# ❌ Anti-pattern: state trong memory
conversation_history = {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
```

```python
# ✅ Implementation: state trong Redis, có fallback
try:
    import redis
    REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")
    _redis = redis.from_url(REDIS_URL, decode_responses=True)
    _redis.ping()
    USE_REDIS = True
except Exception:
    USE_REDIS = False
    _memory_store: dict = {}

def save_session(session_id: str, data: dict, ttl_seconds: int = 3600):
    serialized = json.dumps(data)
    if USE_REDIS:
        _redis.setex(f"session:{session_id}", ttl_seconds, serialized)
    else:
        _memory_store[f"session:{session_id}"] = data

def load_session(session_id: str) -> dict:
    if USE_REDIS:
        data = _redis.get(f"session:{session_id}")
        return json.loads(data) if data else {}
    return _memory_store.get(f"session:{session_id}", {})
```

**Điểm khác so với anti-pattern:**

| | Anti-pattern | Implementation |
|---|---|---|
| Key | `user_id` | `session_id` (UUID, hỗ trợ nhiều conversation/user) |
| Storage | dict in-memory | Redis `setex` with TTL 3600s (auto-expire session cũ) |
| History limit | không giới hạn → memory leak | giữ tối đa 20 messages/session (`history[-20:]`) |
| Fallback | không có | nếu Redis down → fallback dict in-memory (degrade gracefully, có log cảnh báo "not scalable") |
| Identify instance | không có | mỗi response trả `served_by: INSTANCE_ID` để verify stateless khi scale |

**Endpoint chính — `POST /chat`:**

```python
@app.post("/chat")
async def chat(body: ChatRequest):
    session_id = body.session_id or str(uuid.uuid4())
    append_to_history(session_id, "user", body.question)
    session = load_session(session_id)
    history = session.get("history", [])
    answer = ask(body.question)
    append_to_history(session_id, "assistant", answer)
    return {
        "session_id": session_id,
        "question": body.question,
        "answer": answer,
        "turn": len([m for m in history if m["role"] == "user"]) + 1,
        "served_by": INSTANCE_ID,
        "storage": "redis" if USE_REDIS else "in-memory",
    }
```

**Kết quả test (build + start stack):**

```bash
cd 05-scaling-reliability/production
touch .env.local
docker compose up --build --scale agent=3
```

```
✔ Container production-redis-1  Running
✔ Container production-agent-3  Recreated
✔ Container production-agent-2  Recreated
✔ Container production-agent-1  Recreated
✔ Container production-nginx-1  Recreated

agent-1  | Connected to Redis
agent-2  | Connected to Redis
agent-3  | Connected to Redis
agent-1  | INFO:__main__:Starting instance instance-208414
agent-1  | INFO:__main__:Storage: Redis
agent-3  | INFO:__main__:Starting instance instance-f2d464
agent-3  | INFO:__main__:Storage: Redis
agent-2  | INFO:__main__:Starting instance instance-a600c0
agent-2  | INFO:__main__:Storage: Redis
```

```bash
curl http://127.0.0.1:8080/health
curl -X POST http://127.0.0.1:8080/chat -H "Content-Type: application/json" -d '{"question": "Hello"}'
```
```
{"status":"ok","instance_id":"instance-a600c0","uptime_seconds":729.7,"storage":"redis","redis_connected":true}
{"session_id":"8985a58f-34e9-4af7-96b4-efd8667bcbb8","question":"Hello",
 "answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.",
 "turn":1,"served_by":"instance-208414","storage":"redis"}
```

**Quan sát:** `/health` (gọi qua Nginx, do round-robin) được `instance-a600c0` trả lời, còn `/chat` được `instance-208414` trả lời — **2 request liên tiếp đã đi vào 2 instance khác nhau**, nhưng cả 2 đều báo `"storage":"redis"` → state không phụ thuộc instance nào, đúng tinh thần stateless.

---

### Exercise 5.4: Load balancing

**Chạy 3 instances:**

```bash
docker compose up --build --scale agent=3
```

3 container `agent-1`, `agent-2`, `agent-3` (tương ứng `instance-208414`, `instance-a600c0`, `instance-f2d464`) khởi động, kết nối Redis, và Nginx (`production-nginx-1`, port `8080:80`) đứng trước làm load balancer theo `nginx.conf`:

```nginx
upstream agent_cluster {
    server agent:8000;   # Docker DNS "agent" → round-robin 3 instances
    keepalive 16;
}
server {
    listen 80;
    add_header X-Served-By $upstream_addr always;
    location / {
        proxy_pass http://agent_cluster;
        proxy_next_upstream error timeout http_503;
        proxy_next_upstream_tries 3;
    }
}
```

**Test 10 requests:**

```bash
for i in {1..10}; do
  curl -s -X POST http://127.0.0.1:8080/chat -H "Content-Type: application/json" -d "{\"question\": \"Request $i\"}"
  echo ""
done
```

**Kết quả — phân bố `served_by`:**

| Request | served_by |
|---------|-----------|
| 1 | instance-f2d464 |
| 2 | instance-a600c0 |
| 3 | instance-208414 |
| 4 | instance-f2d464 |
| 5 | instance-a600c0 |
| 6 | instance-208414 |
| 7 | instance-f2d464 |
| 8 | instance-a600c0 |
| 9 | instance-208414 |
| 10 | instance-f2d464 |

**Quan sát:**

1. 10 requests được chia đều theo **round-robin** giữa 3 instance — đúng chu kỳ `f2d464 → a600c0 → 208414 → ...` lặp lại.
2. Mỗi request tạo `session_id` mới (vì không gửi `session_id` trong body) → mỗi request là 1 conversation độc lập, `turn:1`.
3 instance Docker DNS `agent` tự động resolve tới cả 3 container nhờ Docker Compose's internal DNS (`resolver 127.0.0.11`) — không cần khai báo IP cứng từng instance trong `nginx.conf`.

---

### Exercise 5.5: Test stateless

```bash
python test_stateless.py
```

Script gửi **5 requests cùng `session_id`** (multi-turn conversation), sau đó verify history.

**Kết quả:**

```
Session ID: f3a8d294-b50e-4db0-ad91-5a88bd33e0bc

Request 1: [instance-a600c0]  Q: What is Docker?
Request 2: [instance-208414]  Q: Why do we need containers?
Request 3: [instance-f2d464]  Q: What is Kubernetes?
Request 4: [instance-a600c0]  Q: How does load balancing work?
Request 5: [instance-208414]  Q: What is Redis used for?

Total requests: 5
Instances used: {'instance-a600c0', 'instance-f2d464', 'instance-208414'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?...
  [assistant]: Container là cách đóng gói app để chạy ở mọi nơi...
  [user]: Why do we need containers?...
  [assistant]: Đây là câu trả lời từ AI agent (mock)...
  [user]: What is Kubernetes?...
  [assistant]: Agent đang hoạt động tốt! (mock response)...
  [user]: How does load balancing work?...
  [assistant]: Đây là câu trả lời từ AI agent (mock)...
  [user]: What is Redis used for?...
  [assistant]: Agent đang hoạt động tốt! (mock response)...

✅ Session history preserved across all instances via Redis!
```

**Quan sát — chứng minh stateless:**

1. **Cả 3 instance** (`a600c0`, `208414`, `f2d464`) lần lượt xử lý 5 request của **cùng 1 session** — request 1 vào `a600c0`, request 2 vào `208414` (khác instance!), nhưng `turn` vẫn tăng đúng thứ tự (1→2→3→4→5).
2. `GET /chat/{session_id}/history` trả về **đầy đủ 10 messages** (5 user + 5 assistant) dù được tạo bởi 3 instance khác nhau — vì toàn bộ history được đọc/viết qua Redis (`session:{session_id}` key), không phụ thuộc memory của instance nào.
3. Nếu dùng anti-pattern (dict in-memory per-instance), request 2 vào `instance-208414` sẽ **không tìm thấy** history của session (vì được tạo ở `instance-a600c0`) → mất context hoặc lỗi 404.

→ **Kết luận Part 5:** Hệ thống đạt stateless thực sự — bất kỳ instance nào cũng có thể phục vụ bất kỳ request nào của bất kỳ session nào, miễn Redis còn sống. Đây là điều kiện bắt buộc để horizontal scaling (thêm/giảm instance) hoạt động đúng with load balancer.

---

### Checkpoint 5 ✅

- [x] Implement health và readiness checks (`/health`, `/ready` with check Redis qua `_redis.ping()`)
- [x] Implement graceful shutdown (signal handler + lifespan, verify qua log SIGINT)
- [x] Refactor code thành stateless (session lưu trong Redis, key `session:{session_id}`, TTL 1h)
- [x] Hiểu load balancing with Nginx (round-robin qua Docker DNS `agent` upstream, header `X-Served-By`)
- [x] Test stateless design (`test_stateless.py` — 5 requests/3 instances, history nguyên vẹn)

---

**Lưu ý debug đáng nhớ:** Khi test `curl http://localhost:8080/health` từ host (Windows/Git Bash) gặp `curl: (52) Empty reply from server`, dù nginx container đang `Up` và `nginx -t` báo config OK. Debug qua các bước:
- `docker compose exec nginx wget -qO- http://agent:8000/health` → OK (network nội bộ container-to-container hoạt động).
- `docker compose exec nginx wget -qO- http://127.0.0.1:80/health` → OK (nginx tự gọi mình qua port 80 thành công).
