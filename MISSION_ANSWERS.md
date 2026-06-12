# Day 12 Lab - Mission Answers

> **Student Name:** Hoàng Phúc Quân
> **Student ID:** 2A202600560
> **Date:** 12/06/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

1. API key hardcode trong code (`OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`)
2. Không có health check endpoint — platform không biết khi container crash
3. Port cố định `port=8000` — không đọc từ `PORT` env var
4. `host="localhost"` — container không nhận kết nối từ bên ngoài
5. `reload=True` cứng — debug mode bật trong production
6. `print()` thay vì proper logging — còn log cả secret ra output
7. Không xử lý SIGTERM — tắt đột ngột, không graceful shutdown

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---------|-------------|----------------|---------------------|
| Config | Hardcode trong code (`DEBUG = True`, `port=8000`) | Đọc từ env vars qua `pydantic-settings` | Thay đổi config không cần sửa code, không lộ secrets |
| Health check | Không có | `GET /health` (liveness) + `GET /ready` (readiness) | Platform biết khi nào restart container |
| Logging | `print()` — log cả secret ra stdout | Structured JSON logging, không log secrets | Dễ parse bởi log aggregator (Datadog, Loki), an toàn hơn |
| Shutdown | Tắt đột ngột (không handle SIGTERM) | `handle_sigterm()` + `lifespan()` context manager | Request đang xử lý được hoàn thành trước khi tắt |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

1. **Base image là gì?** `python:3.11` — full Python distribution (~1 GB)
2. **Working directory là gì?** `/app` — tất cả file trong container được đặt tại đây
3. **Tại sao COPY requirements.txt trước?** Docker build theo từng layer và cache lại. Nếu `requirements.txt` không đổi, Docker tái dùng layer `pip install` đã cache → build nhanh hơn nhiều. Nếu copy toàn bộ code trước, mỗi lần sửa code dù nhỏ cũng phải chạy lại `pip install`.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**
   - `CMD ["python", "app.py"]`: lệnh mặc định, có thể override khi `docker run image python other.py`
   - `ENTRYPOINT`: lệnh cố định, không thể override bình thường (phải dùng `--entrypoint`)
   - File này dùng `CMD` → linh hoạt hơn cho dev/testing


### Exercise 2.2: Build và run kết quả

```bash
# Build
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
# → Build thành công, image size: 1.66 GB

# Run
docker run -p 8000:8000 my-agent:develop

# Test (question là query param, không phải JSON body)
curl "http://localhost:8000/ask?question=What+is+Docker" -X POST
# Response: {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Image size comparison

- Develop image (`python:3.11` single-stage): 1660 MB
- Production image (`python:3.11-slim` multi-stage): 236 MB
- Difference: ~86% nhỏ hơn (giảm 7x)
- **Lý do image nhỏ hơn:**
  - Dùng base image `python:3.11-slim` thay vì `python:3.11` (bỏ nhiều tool không cần)
  - Multi-stage build: Stage 1 (builder) cài gcc + build tools để compile deps, Stage 2 (runtime) chỉ copy `/root/.local` (packages đã build) — không mang theo compiler hay build tools
  - Kết quả: image runtime sạch, không chứa công cụ build

### Exercise 2.4: Docker Compose architecture

```
Client (browser / curl)
        │
        ▼ :80 / :443
  ┌──────────────┐
  │    Nginx     │  ← Reverse proxy + Load balancer
  └──────┬───────┘
         │ (internal network)
         ▼
  ┌──────────────┐
  │    Agent     │  ← FastAPI app (có thể scale nhiều replicas)
  └──────┬───────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐  ┌────────┐
│ Redis │  │ Qdrant │
│(cache)│  │(vector │
│ :6379 │  │   DB)  │
└───────┘  └────────┘
```

- **Agent** không expose port trực tiếp ra ngoài, chỉ giao tiếp qua Nginx
- **Redis** lưu session history và rate limiting data
- **Qdrant** là vector database cho RAG (semantic search)
- Tất cả services chạy trong cùng network `internal` (bridge), isolated với bên ngoài

**Test kết quả:**
```bash
curl http://localhost/health
# {"status":"ok","uptime_seconds":91.2,"version":"2.0.0","timestamp":"2026-06-12T09:15:11.732107"}

curl http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
# {"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- Public URL: `https://hoangphucquan-2a202600560-production.up.railway.app`

```bash
# Health check
curl https://hoangphucquan-2a202600560-production.up.railway.app/health
# {"status":"ok","uptime_seconds":190.7,"platform":"Railway","timestamp":"2026-06-12T09:38:16.779317+00:00"}

# Agent endpoint
curl https://hoangphucquan-2a202600560-production.up.railway.app/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello from Railway"}'
# {"question":"Hello from Railway","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","platform":"Railway"}
```

### Exercise 3.2: So sánh render.yaml vs railway.toml

| | railway.toml | render.yaml |
|---|---|---|
| **Format** | TOML | YAML |
| **Trigger deploy** | `railway up` (CLI) | Push lên GitHub → auto deploy |
| **Environment variables** | Set qua CLI hoặc Dashboard, không khai báo trong file | Khai báo key ngay trong file (`envVars`), giá trị secret set trên Dashboard |
| **Auto-generate secret** | Không có | Có (`generateValue: true`) |
| **Thêm services** | Thêm add-on qua Dashboard | Khai báo luôn trong file (Redis, PostgreSQL...) |
| **Region** | Chọn khi tạo project | Khai báo trong file (`region: singapore`) |
| **Ưu điểm** | Đơn giản, nhanh, không cần GitHub | Infrastructure as Code — toàn bộ stack trong 1 file, reproducible |

---

## Part 4: API Security

### Exercise 4.1: API key authentication

- **API key được check ở đâu?** Hàm `verify_api_key()` (dòng 39) — dùng `Security(APIKeyHeader)` để đọc header `X-API-Key`, sau đó inject vào endpoint `/ask` qua `Depends(verify_api_key)`
- **Điều gì xảy ra nếu sai key?** Trả về `403 Forbidden` với message `"Invalid API key."`; nếu không có key trả về `401 Unauthorized`
- **Làm sao rotate key?** Đổi giá trị env var `AGENT_API_KEY` và restart service — không cần sửa code

```bash
# Không có key → 401
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Có key đúng → 200
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: demo-key-change-in-production" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock)..."}
```

### Exercise 4.2: JWT authentication

```bash
# Bước 1: Lấy token
curl http://localhost:8000/auth/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'
# {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...","token_type":"bearer","expires_in_minutes":60}

# Bước 2: Dùng token gọi API
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODEyNTg4NDAsImV4cCI6MTc4MTI2MjQ0MH0.NshD6ZiE6cj-IZ-BKYgD0qUmGVycA13xpMPd4afexoU"
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'
# {"question":"Explain JWT","answer":"Agent đang hoạt động tốt!...","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
```

**JWT flow:** Client login → Server trả JWT token (có chữ ký) → Client gửi token trong header `Authorization: Bearer` mỗi request → Server verify chữ ký, không cần query DB.

### Exercise 4.3: Rate limiting

- **Algorithm được dùng?** Sliding Window Counter — đếm số request trong cửa sổ thời gian trượt (60 giây)
- **Limit là bao nhiêu requests/minute?** User: 10 req/phút, Admin: 100 req/phút
- **Làm sao bypass limit cho admin?** Dùng `rate_limiter_admin` riêng cho role `admin` — giới hạn cao hơn (100 req/phút)

**Test kết quả:** Request 1-10 thành công, request 11-20 bị block với `429 Rate limit exceeded, retry_after_seconds: 59`

### Exercise 4.4: Cost guard implementation

```python
class CostGuard:
    def __init__(self, daily_budget_usd=1.0, global_daily_budget_usd=10.0, warn_at_pct=0.8):
        self.daily_budget_usd = daily_budget_usd
        self.global_daily_budget_usd = global_daily_budget_usd
        self._records: dict[str, UsageRecord] = {}
        self._global_cost = 0.0

    def check_budget(self, user_id: str) -> None:
        record = self._get_record(user_id)

        # Global budget check — toàn service vượt $10/ngày → 503
        if self._global_cost >= self.global_daily_budget_usd:
            raise HTTPException(status_code=503, detail="Service unavailable due to budget limits.")

        # Per-user check — user vượt $1/ngày → 402 Payment Required
        if record.total_cost_usd >= self.daily_budget_usd:
            raise HTTPException(status_code=402, detail={
                "error": "Daily budget exceeded",
                "used_usd": record.total_cost_usd,
                "budget_usd": self.daily_budget_usd,
                "resets_at": "midnight UTC",
            })

    def record_usage(self, user_id: str, input_tokens: int, output_tokens: int):
        # Ghi nhận sau khi LLM trả về, cộng dồn cost
        record = self._get_record(user_id)
        record.input_tokens += input_tokens
        record.output_tokens += output_tokens
        record.request_count += 1
        cost = (input_tokens / 1000 * PRICE_PER_1K_INPUT_TOKENS +
                output_tokens / 1000 * PRICE_PER_1K_OUTPUT_TOKENS)
        self._global_cost += cost
```

**Giải thích:**
- Budget $1/ngày per user, $10/ngày global
- `check_budget()` gọi **trước** khi call LLM — nếu vượt thì block ngay (trả 402)
- `record_usage()` gọi **sau** khi LLM trả về — cộng dồn tokens thực tế đã dùng
- Trong production thật: lưu vào Redis thay vì in-memory để tồn tại qua restart và scale nhiều instance

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks implementation

```python
@app.get("/health")
def health():
    """Liveness probe — platform restart container nếu non-200"""
    uptime = round(time.time() - START_TIME, 1)
    return {
        "status": "ok",
        "uptime_seconds": uptime,
        "version": "1.0.0",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }

@app.get("/ready")
def ready():
    """Readiness probe — load balancer không route traffic vào nếu 503"""
    if not _is_ready:
        raise HTTPException(status_code=503, detail="Agent not ready yet.")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Sự khác nhau:**
- `/health` (liveness): process còn sống không? → platform dùng để quyết định có restart không
- `/ready` (readiness): có sẵn sàng nhận traffic không? → load balancer dùng để route request

### Exercise 5.2: Graceful shutdown implementation

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    _is_ready = True

    yield  # app đang chạy

    # Shutdown — chờ request đang xử lý hoàn thành
    _is_ready = False
    timeout, elapsed = 30, 0
    while _in_flight_requests > 0 and elapsed < timeout:
        time.sleep(1)
        elapsed += 1

def handle_sigterm(signum, frame):
    logger.info(f"Received SIGTERM — graceful shutdown starting")

signal.signal(signal.SIGTERM, handle_sigterm)
```

**Cơ chế:** Khi nhận SIGTERM, app dừng nhận request mới, chờ tối đa 30 giây cho request đang xử lý xong, rồi mới tắt.

### Exercise 5.3: Stateless refactor

**Anti-pattern (stateful):**
```python
conversation_history = {}  # state trong memory — mất khi restart

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
```

**Correct (stateless):**
```python
@app.post("/ask")
def ask(user_id: str, question: str):
    history = r.lrange(f"history:{user_id}", 0, -1)  # state trong Redis
```

**Lý do:** Khi scale ra 3 instances, mỗi instance có memory riêng biệt. Request 1 vào Instance A lưu history, request 2 vào Instance B sẽ không thấy history đó. Redis là shared storage, tất cả instances đều đọc/ghi chung một nơi.

### Exercise 5.4: Load balancing observation

Chạy `docker compose up --scale agent=3` — Nginx phân tán requests round-robin qua 3 instances. Nếu 1 instance bị kill, Nginx tự động route traffic sang 2 instances còn lại nhờ health check.

### Exercise 5.5: Stateless test results

Stateless design đảm bảo conversation history được lưu trong Redis (shared), không phải trong memory của từng instance. Khi kill random instance, conversation vẫn tiếp tục được vì instance mới đọc lại history từ Redis.

---

## Part 6: Final Project

### Public URL

`https://hoangphucquan-2a202600560-production-2228.up.railway.app`

### Test commands

```bash
BASE="https://hoangphucquan-2a202600560-production-2228.up.railway.app"

# Health check
curl $BASE/health
# {"status":"ok","version":"1.0.0","environment":"development","uptime_seconds":174.1,"total_requests":3,"checks":{"llm":"mock"},"timestamp":"2026-06-12T14:33:09.670395+00:00"}

# Readiness probe
curl $BASE/ready
# {"ready":true}

# Auth required (expect 401)
curl -X POST $BASE/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}

# With API key (expect 200)
curl -X POST $BASE/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
# {"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","model":"gpt-4o-mini","timestamp":"2026-06-12T14:33:25.917795+00:00"}
```

### Production readiness checklist

- [x] Dockerfile (multi-stage, < 500 MB) — python:3.11-slim, image ~236 MB
- [x] docker-compose.yml (agent + redis + nginx + qdrant)
- [x] .dockerignore
- [x] Health check endpoint (`GET /health`) — liveness probe
- [x] Readiness endpoint (`GET /ready`) — readiness probe
- [x] API Key authentication — header `X-API-Key`, trả 401 nếu thiếu/sai
- [x] Rate limiting — Sliding Window Counter, 20 req/min
- [x] Cost guard — $5/ngày per user, reset hàng ngày
- [x] Config từ environment variables — 12-Factor App, dùng `os.getenv()`
- [x] Structured logging — JSON format (`{"ts":...,"lvl":...,"msg":...}`)
- [x] Graceful shutdown — SIGTERM handler + lifespan context manager
- [x] Public URL hoạt động — `https://hoangphucquan-2a202600560-production-2228.up.railway.app`
