# Day 12 Lab - Mission Answers

> **Student Name:** _________________________
> **Student ID:** _________________________
> **Date:** 17/04/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in develop/app.py

1. **API key hardcode trong code** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL` chứa password, nếu push lên GitHub sẽ bị lộ ngay lập tức
2. **Secret bị log ra console** — `print(f"Using key: {OPENAI_API_KEY}")` in secret ra terminal
3. **Dùng `print()` thay vì proper logging** — không có log level, không có format chuẩn, không thể filter hay search
4. **Không có health check endpoint** — nếu agent crash, platform không biết để tự động restart container
5. **Port và host cố định** — `host="localhost"`, `port=8000` hardcode, không đọc từ env var; trên Railway/Render PORT được inject tự động qua environment variable

### Exercise 1.2: Chạy basic version

- Kết quả: App start thành công nhưng gọi `/ask` trả về **500 Internal Server Error**
- Nguyên nhân: `utils/mock_llm` import fail do thiếu module
- Quan sát: App chạy được trên local nhưng **không production-ready**

### Exercise 1.3: Comparison table

| Feature | Develop (Basic) | Production (Advanced) | Tại sao quan trọng? |
|---------|-----------------|----------------------|---------------------|
| Config | Hardcode trực tiếp trong code (`OPENAI_API_KEY = "sk-..."`) | Đọc từ environment variables qua `pydantic Settings` | Tránh lộ secret khi push lên GitHub; dễ thay đổi config theo môi trường |
| Health check | Không có | Có `/health` (liveness) + `/ready` (readiness) + `/metrics` | Platform biết khi nào container bị crash để tự động restart |
| Logging | `print()` — log cả secret ra màn hình, không có format | JSON structured logging, không log secret, có log level | Dễ parse và tìm kiếm trong log aggregator (Datadog, Loki) |
| Shutdown | Đột ngột — không xử lý SIGTERM | Graceful — có `lifespan()` context manager + `handle_sigterm()` signal handler | Request đang xử lý được hoàn thành trước khi tắt, không mất data |
| Host binding | `host="localhost"` — chỉ nhận kết nối local | `host="0.0.0.0"` — nhận kết nối từ bên ngoài container | Trong Docker/cloud, nếu bind localhost thì không thể access từ ngoài |
| Port | Hardcode `port=8000` | Đọc từ `PORT` env var | Railway/Render inject PORT tự động; hardcode sẽ conflict |
| Debug/reload | `reload=True` luôn bật | `reload=settings.debug` — chỉ bật khi DEBUG=true | Reload trong production gây chậm và security risk |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (develop/Dockerfile)

1. **Base image:** `python:3.11` — full Python distribution (~1GB)
2. **Working directory:** `/app`
3. **Tại sao COPY requirements.txt trước:** Docker build theo từng layer và cache lại. Nếu `requirements.txt` không thay đổi, bước `pip install` sẽ dùng cache → build nhanh hơn nhiều. Nếu copy code trước, mỗi lần sửa code sẽ invalidate cache và phải cài lại toàn bộ dependencies.
4. **CMD vs ENTRYPOINT:**
   - `CMD`: lệnh mặc định, có thể override khi `docker run image <other_command>`
   - `ENTRYPOINT`: lệnh cố định, không override được (chỉ append thêm args). Dùng ENTRYPOINT khi muốn container luôn chạy một executable cụ thể.

### Exercise 2.2: Build và run develop image

- Image size: **1.15 GB**
- Gọi `/ask` → 200 OK (mock LLM hoạt động trong container)

### Exercise 2.3: Multi-stage build — Image size comparison

| Image | Size | Ghi chú |
|-------|------|---------|
| `my-agent:develop` | 1.15 GB | Single-stage, full Python + build tools |
| `my-agent:advanced` | 160 MB | Multi-stage, chỉ runtime |
| **Giảm** | **~86%** | Loại bỏ gcc, pip, build tools khỏi final image |

**Stage 1 (builder):** Cài gcc, libpq-dev, pip install dependencies vào `/root/.local`
**Stage 2 (runtime):** Chỉ copy `/root/.local` từ builder + source code → image nhỏ, không có build tools, chạy với non-root user (security best practice)

### Exercise 2.4: Docker Compose stack

Services được start: **agent, redis, qdrant, nginx**
- **agent**: FastAPI app, không expose port trực tiếp
- **redis**: cache cho session và rate limiting
- **qdrant**: vector database cho RAG
- **nginx**: reverse proxy trên port 80/443, phân tán traffic đến agent instances

Communicate qua internal network `agent_net` (bridge driver) — các service gọi nhau bằng service name (e.g. `redis://redis:6379`).

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- **URL:** https://desirable-courage-production-dcfb.up.railway.app
- **Platform:** Railway
- **Deploy via:** GitHub (Khohk/khoi-day12, branch main, root /03-cloud-deployment/railway)

Health check result:
```json
{"status":"ok","uptime_seconds":1046.7,"platform":"Railway","timestamp":"2026-04-17T15:50:35.021301+00:00"}
```

### Exercise 3.2: So sánh render.yaml vs railway.toml

| | railway.toml | render.yaml |
|--|--|--|
| Builder | Nixpacks (auto-detect) | Docker hoặc native |
| Config | `[deploy]` section | `services:` YAML |
| Env vars | `railway variables set` hoặc dashboard | Dashboard hoặc trong render.yaml |
| Deploy | `railway up` hoặc GitHub push | GitHub push tự động |
| Free tier | $5 credit | 750h/month |

---

## Part 4: API Security

### Exercise 4.1: API Key authentication
- Không có key → `401 Unauthorized`
- Có key đúng → `200 OK`
- API key được check qua `X-API-Key` header
- Sai key → raise `HTTPException(401)`
- Rotate key: thay đổi env var `AGENT_API_KEY` và restart service

### Exercise 4.2: JWT authentication
- Lấy token: `POST /auth/token` với `{"username": "student", "password": "demo123"}` → `200 OK`
- Dùng token gọi `/ask`:
```json
{
  "question": "Explain jwt",
  "answer": "Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.",
  "usage": {
    "requests_remaining": 9,
    "budget_remaining_usd": 0.000019
  }
}
```

### Exercise 4.3: Rate limiting
- Mỗi user có giới hạn 10 requests/minute
- Sau khi dùng hết → response trả về `requests_remaining: 0`
- Admin bypass: dùng role khác với limit cao hơn (teacher: 100 req/min)

### Exercise 4.4: Cost guard implementation
- Mỗi user có budget $1/ngày (track trong memory/Redis theo ngày)
- Mỗi request trừ estimated cost từ budget
- Khi `budget_remaining_usd` về 0 → trả về `402 Payment Required`
- Reset đầu ngày mới theo key format `budget:{user_id}:{date}`

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

`/health` (liveness probe) — trả về 200 khi process còn sống:
```json
{
  "status": "ok",
  "uptime_seconds": 46.7,
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-04-17T10:16:48.607564+00:00",
  "checks": { "memory": { "status": "ok", "note": "psutil not installed" } }
}
```

`/ready` (readiness probe) — trả về 200 khi sẵn sàng nhận traffic:
```json
{ "ready": true, "in_flight_requests": 1 }
```

Trả về 503 khi đang startup hoặc shutdown — load balancer sẽ không route traffic vào instance này.

### Exercise 5.2: Graceful shutdown

- Signal handler `handle_sigterm()` bắt SIGTERM từ platform
- `lifespan()` context manager chờ `_in_flight_requests == 0` trước khi tắt (timeout 30s)
- Đảm bảo request đang xử lý được hoàn thành, không bị mất

### Exercise 5.3: Stateless design

**Anti-pattern:**
```python
conversation_history = {}  # state trong memory — mất khi scale

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
```

**Correct:**
```python
@app.post("/ask")
def ask(user_id: str, question: str):
    history = r.lrange(f"history:{user_id}", 0, -1)  # state trong Redis
```

Khi scale ra 3 instances, mỗi instance có memory riêng. Nếu dùng in-memory state, user gọi instance 1 sẽ mất history khi được route sang instance 2. Redis là shared storage cho tất cả instances.

### Exercise 5.4: Load balancing

Architecture của docker-compose stack:
- **nginx**: reverse proxy, phân tán traffic đến các agent instances
- **agent (x3)**: stateless FastAPI instances
- **redis**: shared state (conversation history, rate limiting)

Nginx dùng round-robin để phân tán requests. Nếu 1 instance fail health check → nginx tự động bỏ qua instance đó.

Test 10 requests qua `http://localhost:8080/ask` — 3 loại response khác nhau xác nhận requests được phân tán qua 3 instances.

### Exercise 5.5: Test stateless design

```
Session ID: f846134d-2ee8-4831-be37-db3d5544c129

Request 1: [instance-e7f583]  Q: What is Docker?
Request 2: [instance-3fd0df]  Q: Why do we need containers?
Request 3: [instance-1183e6]  Q: What is Kubernetes?
Request 4: [instance-e7f583]  Q: How does load balancing work?
Request 5: [instance-3fd0df]  Q: What is Redis used for?

Total requests: 5
Instances used: {instance-e7f583, instance-1183e6, instance-3fd0df}
✅ All requests served despite different instances!
✅ Session history preserved across all instances via Redis!
```

**Kết luận:** 5 requests được xử lý bởi 3 instances khác nhau nhưng conversation history vẫn liên tục nhờ Redis. Đây là proof of concept của stateless design — state lưu trong Redis thay vì memory của từng instance.
