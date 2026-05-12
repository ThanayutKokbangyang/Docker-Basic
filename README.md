# 🐳 Docker: Zero to Senior

บทเรียน Docker ฉบับสมบูรณ์ — เน้นปัญหาจริงที่เจอในงาน พร้อมวิธีแก้

---

## สารบัญ

1. [Level 0: ทำความเข้าใจ Docker ก่อนแตะคำสั่ง](#level-0)
2. [Level 1: คำสั่งพื้นฐานที่ต้องคล่อง](#level-1)
3. [Level 2: Image และ Dockerfile](#level-2)
4. [Level 3: Volume และ Network](#level-3)
5. [Level 4: Docker Compose](#level-4)
6. [Level 5: Multi-stage Build และ Optimization](#level-5)
7. [Level 6: Production-Ready Patterns](#level-6)
8. [Level 7: Security และ Best Practices](#level-7)
9. [Level 8: Debugging และ Troubleshooting ระดับ Senior](#level-8)
10. [Cheat Sheet](#cheat-sheet)

---

<a id="level-0"></a>
## Level 0: ทำความเข้าใจ Docker ก่อนแตะคำสั่ง

### Docker คืออะไร (แบบเข้าใจจริงๆ)

Docker ไม่ใช่ VM — Docker ใช้ feature ของ Linux Kernel ที่ชื่อว่า **namespaces** (แยก process, network, filesystem) และ **cgroups** (จำกัด CPU/RAM) ทำให้รัน process แยกกันได้แบบเบามาก

**เปรียบเทียบ:**

| | VM | Docker Container |
|---|---|---|
| OS | มี Guest OS เต็มๆ | ใช้ Kernel ของ Host |
| ขนาด | GB | MB |
| Boot time | นาที | วินาที |
| Isolation | สูงมาก | ปานกลาง |

### คำศัพท์ที่ต้องแยกให้ออก

- **Image** = template (เหมือน class) — read-only
- **Container** = instance ที่รันอยู่ (เหมือน object ของ class)
- **Dockerfile** = สูตรการสร้าง image
- **Registry** = ที่เก็บ image (เช่น Docker Hub, ECR, GCR)
- **Volume** = ที่เก็บข้อมูลแบบถาวร (อยู่นอก container)

### 🔥 ปัญหายอดฮิตของมือใหม่ #1: "Docker กับ VM ต่างกันยังไง?"

**สถานการณ์:** เพื่อนถามว่าทำไมต้องใช้ Docker ในเมื่อมี VirtualBox อยู่แล้ว

**คำตอบ:** Container แชร์ kernel กับ host ทำให้:
- Start ใน <1 วินาที (VM ใช้นาที)
- รัน 100 containers บนเครื่องเดียวได้ (VM รันได้ 5-10)
- Image ขนาดเล็ก (Alpine แค่ 5MB, Ubuntu VM ขั้นต่ำ 1GB+)

**แต่!** Container ไม่เหมาะกับงานที่ต้องการ kernel ต่างจาก host เช่น รัน Windows app บน Linux host

---

<a id="level-1"></a>
## Level 1: คำสั่งพื้นฐานที่ต้องคล่อง

### Lifecycle ของ Container

```bash
# Pull image จาก registry
docker pull nginx:alpine

# Run container (สร้าง + start)
docker run -d --name web -p 8080:80 nginx:alpine
#         -d = detached (background)
#         --name = ตั้งชื่อ
#         -p host:container = map port

# ดู container ที่รัน
docker ps              # เฉพาะที่รัน
docker ps -a           # ทั้งหมดรวมที่หยุดแล้ว

# หยุด/เริ่ม/รีสตาร์ท
docker stop web
docker start web
docker restart web

# ลบ
docker rm web          # ต้องหยุดก่อน
docker rm -f web       # บังคับลบทั้งที่ยังรัน

# เข้าไปใน container
docker exec -it web sh
#           -i = interactive
#           -t = pseudo-TTY

# ดู log
docker logs web
docker logs -f web     # follow แบบ tail -f
docker logs --tail 100 web
```

### 🔥 ปัญหายอดฮิต #2: Port already in use

**สถานการณ์:**
```bash
$ docker run -p 8080:80 nginx
docker: Error response from daemon: driver failed programming external connectivity
on endpoint: Bind for 0.0.0.0:8080 failed: port is already allocated
```

**สาเหตุที่เป็นไปได้:**
1. มี container อื่นใช้ port 8080 อยู่
2. มี process อื่นบนเครื่อง (เช่น Node.js dev server)

**วิธีหา:**
```bash
# หา container ที่ใช้ port นี้
docker ps | grep 8080

# หา process บนเครื่อง (Linux/Mac)
lsof -i :8080
# หรือ
sudo netstat -tulpn | grep 8080

# Windows
netstat -ano | findstr :8080
```

**วิธีแก้:** เปลี่ยน port host หรือฆ่า process เก่า
```bash
docker run -p 8081:80 nginx       # เปลี่ยนเป็น 8081
```

### 🔥 ปัญหายอดฮิต #3: Container start แล้วดับทันที

**สถานการณ์:**
```bash
$ docker run -d --name app ubuntu
$ docker ps
(ไม่เห็น container)
$ docker ps -a
STATUS: Exited (0) 2 seconds ago
```

**สาเหตุ:** Container จะรันอยู่ตราบเท่าที่ process หลักยังทำงาน Ubuntu image ไม่ได้มี service ที่รันค้าง พอ entrypoint จบ container ก็จบ

**วิธีแก้:**
```bash
# ให้รัน command ที่ค้าง
docker run -d --name app ubuntu sleep infinity

# หรือเข้า interactive mode
docker run -it --name app ubuntu bash
```

**ตรวจสอบสาเหตุที่แท้จริง:**
```bash
docker logs app                    # ดู error
docker inspect app | grep -i exit  # ดู exit code
```

**Exit code ที่ควรรู้:**
- `0` = จบปกติ
- `1` = error ทั่วไป
- `125` = docker daemon error
- `126` = command ไม่มี execute permission
- `127` = command ไม่พบ
- `137` = ถูก kill (มักเพราะ OOM — Out of Memory)
- `139` = segmentation fault

---

<a id="level-2"></a>
## Level 2: Image และ Dockerfile

### Dockerfile แรกของคุณ

```dockerfile
# ระบุ base image
FROM node:20-alpine

# ตั้ง working directory ใน container
WORKDIR /app

# คัดลอกไฟล์จาก host เข้า image
COPY package*.json ./

# รันคำสั่งตอน build image
RUN npm install

# คัดลอกที่เหลือ
COPY . .

# บอก Docker ว่า container จะใช้ port อะไร (documentation เท่านั้น)
EXPOSE 3000

# คำสั่งที่รันตอน start container
CMD ["node", "server.js"]
```

### Build และ Run

```bash
docker build -t myapp:1.0 .
#            -t = tag
#            . = build context (โฟลเดอร์ปัจจุบัน)

docker run -d -p 3000:3000 myapp:1.0
```

### คำสั่ง Dockerfile สำคัญและกับดัก

| คำสั่ง | ทำอะไร | กับดัก |
|---|---|---|
| `FROM` | กำหนด base image | อย่าใช้ `latest` ใน production |
| `RUN` | รันคำสั่งตอน build | แต่ละ RUN สร้าง layer ใหม่ — ควรรวมกัน |
| `COPY` | คัดลอกไฟล์ | คัดลอกตรงๆ เร็วกว่า ADD |
| `ADD` | คัดลอก + แตก tar + ดาวน์โหลด URL | ใช้เมื่อจำเป็นเท่านั้น |
| `CMD` | default command ตอน run | override ได้ตอน `docker run` |
| `ENTRYPOINT` | command หลัก ลบไม่ได้ | argument จะต่อท้าย |
| `ENV` | กำหนด environment variable | จะติดไปใน image |
| `ARG` | ตัวแปรตอน build | หายไปหลัง build |
| `WORKDIR` | เปลี่ยน directory | ใช้แทน `RUN cd` |
| `EXPOSE` | บอกว่าจะใช้ port อะไร | แค่ documentation — ไม่ได้ publish จริง |

### CMD vs ENTRYPOINT — สับสนตลอด

```dockerfile
# แบบที่ 1: ใช้ CMD อย่างเดียว
CMD ["node", "server.js"]
# → docker run myapp           → รัน node server.js
# → docker run myapp bash      → รัน bash (override CMD)

# แบบที่ 2: ใช้ ENTRYPOINT อย่างเดียว
ENTRYPOINT ["node", "server.js"]
# → docker run myapp           → รัน node server.js
# → docker run myapp --port=8000 → รัน node server.js --port=8000

# แบบที่ 3: ผสม (แนะนำ)
ENTRYPOINT ["node"]
CMD ["server.js"]
# → docker run myapp           → รัน node server.js
# → docker run myapp app.js    → รัน node app.js
```

### 🔥 ปัญหายอดฮิต #4: Build ช้ามาก (ทุกครั้งที่แก้โค้ด)

**สถานการณ์:** แก้โค้ดนิดเดียวแต่ build ใหม่ใช้เวลา 5 นาที เพราะ `npm install` ทำงานทุกครั้ง

**สาเหตุ:** Docker ใช้ **layer caching** — ถ้าไฟล์ที่ COPY เปลี่ยน layer นั้นกับ layers ที่อยู่ "หลังจากนั้น" จะถูก rebuild หมด

**Dockerfile ที่แย่:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .              # ❌ แก้ไฟล์อะไรก็ตาม cache พัง
RUN npm install        # ❌ ทำงานทุกครั้ง
CMD ["node", "server.js"]
```

**Dockerfile ที่ดี:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./  # ✅ คัดลอกแค่ manifest ก่อน
RUN npm ci             # ✅ ใช้ cache ตราบที่ package.json ไม่เปลี่ยน
COPY . .               # ✅ คัดลอกโค้ดทีหลัง
CMD ["node", "server.js"]
```

**หลักการ:** สิ่งที่เปลี่ยนน้อย → วางด้านบน, สิ่งที่เปลี่ยนบ่อย → วางด้านล่าง

### 🔥 ปัญหายอดฮิต #5: Image ขนาดใหญ่มหาศาล

**สถานการณ์:** Build image แล้วได้ 1.5GB ทั้งที่เป็นแค่ Node.js app เล็กๆ

**ตรวจสอบ:**
```bash
docker images
docker history myapp:1.0    # ดูว่า layer ไหนใหญ่
docker image inspect myapp:1.0
```

**สาเหตุที่พบบ่อย:**

1. **ใช้ base image ที่ใหญ่เกินไป**
   ```dockerfile
   FROM node:20            # ~1GB (Debian-based เต็ม)
   FROM node:20-slim       # ~250MB
   FROM node:20-alpine     # ~150MB ✅
   ```

2. **ไม่ลบ cache ของ package manager**
   ```dockerfile
   # ❌ ทิ้ง cache ไว้
   RUN apt-get update && apt-get install -y curl

   # ✅ ลบ cache
   RUN apt-get update && apt-get install -y curl \
       && rm -rf /var/lib/apt/lists/*
   ```

3. **คัดลอกไฟล์ที่ไม่จำเป็น** — ใช้ `.dockerignore`:
   ```
   node_modules
   .git
   .env
   *.log
   coverage
   .vscode
   README.md
   Dockerfile
   .dockerignore
   ```

4. **ไม่ใช้ multi-stage build** (เดี๋ยวเจอใน Level 5)

### 🔥 ปัญหายอดฮิต #6: "ที่เครื่องผมรันได้ปกตินะ" แต่ใน container พัง

**สถานการณ์คลาสสิก:** เครื่อง dev เป็น Mac M1 (ARM) แต่ deploy บน server x86

**สาเหตุ:** Image ที่ build บน ARM ใช้บน x86 ไม่ได้

**วิธีแก้:**
```bash
# Build แบบระบุ platform
docker build --platform linux/amd64 -t myapp:1.0 .

# หรือ build หลาย platform พร้อมกัน (ต้องใช้ buildx)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .
```

---

<a id="level-3"></a>
## Level 3: Volume และ Network

### Volume — เก็บข้อมูลไม่ให้หาย

**ปัญหา:** Container ถูกลบ → ข้อมูลหายหมด

**3 วิธีเก็บข้อมูล:**

#### 1. Named Volume (แนะนำ)
```bash
# สร้าง volume
docker volume create mydata

# ใช้
docker run -d -v mydata:/var/lib/postgresql/data postgres

# Docker จัดเก็บที่ /var/lib/docker/volumes/mydata/_data
docker volume ls
docker volume inspect mydata
```

#### 2. Bind Mount (สำหรับ dev)
```bash
# Map folder ของ host เข้า container
docker run -d -v $(pwd)/code:/app node:alpine

# แก้โค้ดบน host → เห็นใน container ทันที (hot reload)
```

#### 3. tmpfs (ใน RAM, หายเมื่อปิด)
```bash
docker run --tmpfs /tmp nginx
```

### 🔥 ปัญหายอดฮิต #7: ข้อมูล database หายหลังลบ container

**สถานการณ์:**
```bash
$ docker run -d --name db postgres
$ # ใส่ข้อมูลเข้าไป...
$ docker rm -f db
$ docker run -d --name db postgres
$ # ข้อมูลหายหมด!
```

**สาเหตุ:** ไม่ได้ mount volume — ข้อมูลอยู่ใน writable layer ของ container ซึ่งหายไปกับ container

**วิธีแก้:**
```bash
docker run -d \
  --name db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres
```

**ทำไมต้อง mount path นี้?** ดูใน Docker Hub ของแต่ละ image จะบอก path ที่เก็บข้อมูล:
- PostgreSQL: `/var/lib/postgresql/data`
- MySQL: `/var/lib/mysql`
- MongoDB: `/data/db`
- Redis: `/data`

### 🔥 ปัญหายอดฮิต #8: Permission denied ใน bind mount

**สถานการณ์:**
```bash
$ docker run -v $(pwd):/app node:alpine npm install
npm ERR! EACCES: permission denied
```

**สาเหตุ:** User ใน container (UID/GID) ไม่ตรงกับเจ้าของไฟล์บน host

**วิธีแก้:**
```bash
# วิธีที่ 1: รันด้วย user ของ host
docker run --user $(id -u):$(id -g) -v $(pwd):/app node:alpine npm install

# วิธีที่ 2: ใน Dockerfile กำหนด user ให้ตรง
# (สำหรับ production)
```

### Network ใน Docker

```bash
# ดู networks ที่มี
docker network ls

# Network drivers หลัก:
# - bridge: default, แยกแต่ละ container
# - host: ใช้ network ของ host (ไม่มี isolation)
# - none: ไม่มี network เลย
# - overlay: สำหรับ Swarm/multi-host

# สร้าง custom bridge network (แนะนำ)
docker network create mynet

# Run container บน network นี้
docker run -d --name db --network mynet postgres
docker run -d --name api --network mynet myapi
```

### 🔥 ปัญหายอดฮิต #9: Container คุยกันไม่ได้

**สถานการณ์:** Run 2 containers (api + db) แต่ api ติดต่อ db ไม่ได้

**สาเหตุ:** ใช้ `localhost` เพื่อเชื่อมกัน
```javascript
// ❌ ผิด — localhost = ตัว container เอง
const db = new Client({ host: 'localhost' })

// ✅ ถูก — ใช้ชื่อ container เป็น hostname
const db = new Client({ host: 'db' })
```

**เงื่อนไข:** Container ทั้งสองต้องอยู่ใน **custom network เดียวกัน** (default bridge ไม่มี DNS resolution)

```bash
docker network create mynet
docker run -d --name db --network mynet -e POSTGRES_PASSWORD=x postgres
docker run -d --name api --network mynet -e DB_HOST=db myapi
```

### 🔥 ปัญหายอดฮิต #10: Container ติดต่อ service บน host ไม่ได้

**สถานการณ์:** มี DB รันบน host (ไม่ใช่ใน container) — Container ติดต่อไม่ได้ด้วย `localhost`

**วิธีแก้ตาม OS:**

```bash
# Mac/Windows (Docker Desktop)
host.docker.internal

# Linux (Docker 20.10+)
docker run --add-host=host.docker.internal:host-gateway ...

# หรือใช้ IP ของ docker0 bridge
ip addr show docker0    # มักจะเป็น 172.17.0.1
```

---

<a id="level-4"></a>
## Level 4: Docker Compose

### ทำไมต้องใช้ Compose

ถ้ามี 5 services (api, db, redis, nginx, worker) การ `docker run` ทีละตัวพร้อม flags ต่างๆ มันเหนื่อยและ error เยอะ — Compose ให้เขียน YAML ครั้งเดียว

### docker-compose.yml พื้นฐาน

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### คำสั่งหลัก

```bash
docker compose up -d            # start ทุก services (background)
docker compose down             # stop และลบ
docker compose down -v          # ลบ volume ด้วย (ระวัง! ข้อมูลหาย)
docker compose ps               # ดูสถานะ
docker compose logs -f api      # ดู log ของ service
docker compose exec api sh      # เข้าไปใน container
docker compose restart api      # restart service เดียว
docker compose build api        # build ใหม่
docker compose up -d --build    # rebuild + restart
```

### 🔥 ปัญหายอดฮิต #11: API start ก่อน DB พร้อม

**สถานการณ์:** Compose start `api` พร้อมกับ `db` — แต่ db ยังไม่พร้อมรับ connection → api crash

**คำเตือน:** `depends_on` แบบธรรมดารอแค่ "container start" ไม่ใช่ "service ready"

```yaml
# ❌ ไม่พอ
depends_on:
  - db

# ✅ รอจน healthcheck ผ่าน
depends_on:
  db:
    condition: service_healthy

# และใน db service ต้องมี healthcheck:
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    retries: 5
    start_period: 10s
```

**ทางเลือกอื่น:** ใช้ retry logic ในโค้ด (แนะนำสำหรับ production)
```javascript
async function connectWithRetry(maxRetries = 10) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await db.connect()
    } catch (err) {
      console.log(`Retry ${i+1}/${maxRetries}...`)
      await sleep(2000)
    }
  }
  throw new Error('Cannot connect to DB')
}
```

### 🔥 ปัญหายอดฮิต #12: เปลี่ยน .env แล้วไม่มีผล

**สถานการณ์:** แก้ค่าใน `.env` แล้ว `docker compose up` แต่ค่าไม่เปลี่ยน

**สาเหตุ:** Container เดิมยังรันอยู่ — Compose จะ recreate เฉพาะที่ config เปลี่ยน

**วิธีแก้:**
```bash
docker compose up -d --force-recreate
# หรือ
docker compose down && docker compose up -d
```

### Override สำหรับ Dev/Prod

```yaml
# docker-compose.yml (base)
services:
  api:
    image: myapp:latest
    environment:
      NODE_ENV: production

# docker-compose.override.yml (auto-loaded สำหรับ dev)
services:
  api:
    build: .
    volumes:
      - ./src:/app/src    # hot reload
    environment:
      NODE_ENV: development
    command: npm run dev
```

```bash
# Dev (โหลด override อัตโนมัติ)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

<a id="level-5"></a>
## Level 5: Multi-stage Build และ Optimization

### ปัญหา: Image ใหญ่เพราะมีของไม่จำเป็นใน production

ลองดู Dockerfile ปกติของ Node.js:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install            # ติดตั้ง devDependencies ด้วย!
COPY . .
RUN npm run build
CMD ["node", "dist/server.js"]
```
**ปัญหา:** `node_modules` มี devDependencies, source code TS, test files — ของพวกนี้ไม่จำเป็นตอนรัน

### Multi-stage Build — สร้าง stage แล้วเอาเฉพาะ output

```dockerfile
# ========== Stage 1: Builder ==========
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # รวม devDependencies
COPY . .
RUN npm run build             # compile TS → JS ใน dist/
RUN npm prune --production    # ลบ devDependencies

# ========== Stage 2: Runtime ==========
FROM node:20-alpine
WORKDIR /app
# เอาเฉพาะที่จำเป็นจาก builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
USER node                     # ไม่รันด้วย root
CMD ["node", "dist/server.js"]
```

**ผลลัพธ์:** Image เล็กลง 60-80% และปลอดภัยขึ้น (ไม่มี build tools)

### Multi-stage สำหรับ Go (compile แล้วเอา binary)

```dockerfile
# ========== Stage 1: Build ==========
FROM golang:1.22-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# ========== Stage 2: Minimal runtime ==========
FROM scratch                  # image เปล่าๆ ไม่มีอะไรเลย
COPY --from=builder /build/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```
**ขนาด image:** ~10MB (เทียบกับ 800MB+ ถ้าใช้ golang image เต็ม)

### Cache Mount (BuildKit) — เร็วขึ้นอีก

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
# Mount cache directory — npm cache จะอยู่ระหว่าง build
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
CMD ["node", "server.js"]
```

### 🔥 ปัญหายอดฮิต #13: Build ช้าใน CI/CD

**วิธีเร่งใน CI:**

```bash
# 1. ใช้ BuildKit (เร็วกว่า, parallel layers)
DOCKER_BUILDKIT=1 docker build .

# 2. Use cache จาก registry
docker buildx build \
  --cache-from=type=registry,ref=myapp:cache \
  --cache-to=type=registry,ref=myapp:cache,mode=max \
  -t myapp:latest .

# 3. ใช้ --platform เฉพาะที่ต้องการ
```

---

<a id="level-6"></a>
## Level 6: Production-Ready Patterns

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1
```

หรือใน Compose:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**ใน Application ต้องมี endpoint `/health`:**
```javascript
app.get('/health', async (req, res) => {
  try {
    await db.query('SELECT 1')        // เช็ค DB connection
    await redis.ping()                 // เช็ค Redis
    res.json({ status: 'ok' })
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message })
  }
})
```

### Graceful Shutdown

```javascript
const server = app.listen(3000)

// Docker ส่ง SIGTERM ก่อน SIGKILL (default 10 วินาที)
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing server...')

  server.close(() => {
    console.log('HTTP server closed')
  })

  await db.end()
  await redis.quit()
  process.exit(0)
})
```

**ใน Dockerfile:** ใช้ exec form ของ CMD เพื่อให้ signal ส่งถึง process จริง
```dockerfile
# ❌ shell form — node เป็นลูกของ /bin/sh ไม่ได้รับ SIGTERM
CMD node server.js

# ✅ exec form — node เป็น PID 1 รับ signal โดยตรง
CMD ["node", "server.js"]
```

### Resource Limits

```yaml
services:
  api:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

หรือผ่าน docker run:
```bash
docker run -d \
  --memory=1g \
  --memory-reservation=512m \
  --cpus=2 \
  --restart=unless-stopped \
  myapp
```

### 🔥 ปัญหายอดฮิต #14: Container ถูก kill ด้วย OOM (Out of Memory)

**สถานการณ์:** Container ดับเองตลอด — `docker inspect` แสดง `OOMKilled: true`, exit code 137

**ตรวจสอบ:**
```bash
docker stats              # ดูการใช้ memory แบบ real-time
docker inspect <container> | grep -i oom
```

**สาเหตุที่พบบ่อย:**

1. **Memory leak ในแอป** — ใช้ profiler หา
2. **JVM/Node ไม่รู้ limit ของ container**

   ```bash
   # Java
   -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0

   # Node.js
   NODE_OPTIONS="--max-old-space-size=512"  # ถ้า limit 1GB ตั้ง heap 512MB
   ```

3. **Limit ต่ำเกินไป** — เพิ่ม memory limit

### Logging

```bash
# จำกัดขนาด log file (สำคัญมาก! ไม่งั้น disk เต็ม)
docker run --log-opt max-size=10m --log-opt max-file=3 myapp
```

ใน `/etc/docker/daemon.json` (default ทั้งระบบ):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Best practice:** Log เป็น stdout/stderr (ไม่เขียน file ใน container) — Docker จัดการเอง

---

<a id="level-7"></a>
## Level 7: Security และ Best Practices

### Checklist สำหรับ Production Image

#### 1. อย่ารันด้วย root
```dockerfile
FROM node:20-alpine
# สร้าง user
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --chown=app:app . .
USER app
CMD ["node", "server.js"]
```

#### 2. Pin version ของ base image
```dockerfile
# ❌ อาจเปลี่ยนทุกครั้งที่ build
FROM node:alpine

# ✅ pin version
FROM node:20.11.0-alpine3.19

# ✅✅ pin digest (ป้องกัน supply chain attack)
FROM node:20.11.0-alpine3.19@sha256:abc123...
```

#### 3. อย่าใส่ secret ใน image
```dockerfile
# ❌ secret อยู่ใน layer ตลอดไป (ดูได้จาก docker history)
ENV API_KEY=sk-abc123

# ❌ COPY .env ติดไปด้วย
COPY .env .

# ✅ ส่งตอน runtime
# docker run -e API_KEY=$API_KEY myapp
```

**ระวัง:** แม้จะ `RUN rm secret.txt` ใน layer ถัดไป — secret ยังอยู่ใน layer เดิม! ตรวจดูด้วย:
```bash
docker history --no-trunc myapp:1.0
```

**สำหรับ build-time secret ใช้ BuildKit:**
```dockerfile
# syntax=docker/dockerfile:1.4
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```
```bash
docker build --secret id=npm_token,src=$HOME/.npm_token -t myapp .
```

#### 4. Scan vulnerabilities
```bash
# Docker built-in
docker scout cves myapp:1.0

# Trivy (popular open-source)
trivy image myapp:1.0

# Snyk
snyk container test myapp:1.0
```

#### 5. ใช้ read-only filesystem ถ้าได้
```bash
docker run --read-only --tmpfs /tmp myapp
```

#### 6. Drop capabilities
```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

### 🔥 ปัญหายอดฮิต #15: Image มี vulnerability CVE สูง

**สถานการณ์:** Security scan รายงาน CVE 100+ ในแอปง่ายๆ

**สาเหตุ:** ส่วนใหญ่มาจาก base image — เก่าหรือมี package ที่ไม่ใช้

**วิธีแก้:**

1. **Update base image เป็นล่าสุด** — `node:20-alpine` → version ใหม่กว่า
2. **ใช้ distroless** (Google) — มีแต่ runtime ไม่มี shell, package manager
   ```dockerfile
   FROM node:20-alpine AS builder
   # ... build ...

   FROM gcr.io/distroless/nodejs20-debian12
   COPY --from=builder /app /app
   CMD ["/app/server.js"]
   ```
3. **ใช้ Chainguard images** (มี CVE น้อยมาก)
4. **อัปเดต dependencies** ด้วย `npm audit fix`, `pip-audit`, etc.

---

<a id="level-8"></a>
## Level 8: Debugging และ Troubleshooting ระดับ Senior

### เครื่องมือที่ต้องคล่อง

```bash
# ดูทุกอย่างเกี่ยวกับ container
docker inspect <container>

# ดู resource usage real-time
docker stats

# ดู process ใน container
docker top <container>

# ดู events ของ docker daemon
docker events

# ดู disk usage
docker system df
docker system df -v        # ละเอียด

# ทำความสะอาด
docker system prune        # ลบ container, network, image ที่ไม่ใช้
docker system prune -a --volumes   # ลบทุกอย่าง (ระวัง!)
```

### 🔥 ปัญหายอดฮิต #16: Disk เต็มเพราะ Docker

**สถานการณ์:** Disk เต็ม 100% — Docker กิน 50GB

**ตรวจสอบ:**
```bash
docker system df

# ผลลัพธ์อาจประมาณนี้:
# TYPE          TOTAL   ACTIVE   SIZE     RECLAIMABLE
# Images        45      5        30GB     25GB
# Containers    20      3        500MB    400MB
# Volumes       15      4        15GB     10GB
# Build Cache   200     -        5GB      5GB
```

**ของที่กินที่:**
- **Dangling images** = image ที่ไม่มี tag (จาก rebuild)
- **Stopped containers** = container เก่าที่ลืมลบ
- **Unused volumes** = volume ที่ไม่มี container ใช้
- **Build cache** = สะสมจากการ build

**ทำความสะอาด:**
```bash
# ลบเฉพาะ dangling images
docker image prune

# ลบ image ที่ไม่มี container ใช้
docker image prune -a

# ลบ stopped containers
docker container prune

# ลบ volume ที่ไม่ใช้ (ระวังข้อมูลหาย!)
docker volume prune

# ลบ build cache
docker builder prune

# ลบทุกอย่างที่ไม่ใช้
docker system prune -a --volumes
```

### 🔥 ปัญหายอดฮิต #17: Network ช้าระหว่าง containers

**ตรวจสอบ:**
```bash
# Test latency ระหว่าง containers
docker exec api ping -c 5 db
docker exec api curl -w "@curl-format.txt" -o /dev/null -s http://db:5432

# ดู network ที่ container ใช้
docker inspect <container> | jq '.[0].NetworkSettings.Networks'
```

**สาเหตุที่พบบ่อย:**
- Container อยู่คนละ network ต้อง route ผ่าน iptables
- DNS resolution ช้า — ใช้ IP โดยตรงเร็วกว่า แต่ไม่ flexible
- ใช้ `host` network mode บน Mac/Windows — ช้ากว่า bridge (เพราะมี VM ขั้น)

### 🔥 ปัญหายอดฮิต #18: Container CPU 100% ตลอด

**Debug:**
```bash
# 1. ดู process ใน container
docker top <container>

# 2. เข้าไป profile
docker exec -it <container> sh
top
# หรือ
ps aux --sort=-%cpu

# 3. ถ้าเป็น Node.js
docker exec <container> kill -USR1 1   # เปิด debugger
# ใช้ Chrome DevTools connect ผ่าน port

# 4. ดู syscalls
docker exec <container> strace -p <pid>
```

### 🔥 ปัญหายอดฮิต #19: Container restart loop

**สถานการณ์:** Container restart ทุก 10 วินาที

**ตรวจสอบ:**
```bash
docker ps -a
# STATUS: Restarting (1) 5 seconds ago

docker logs --tail 50 <container>

# ดู restart policy
docker inspect <container> | grep -A 5 RestartPolicy

# ดู exit code ล่าสุด
docker inspect <container> | grep -A 5 State
```

**สาเหตุที่พบบ่อย:**
1. Configuration ผิด — env variable หาย
2. Dependency ยังไม่พร้อม (DB ยังไม่ start)
3. Permission ปัญหา
4. Health check fail ตลอด

**วิธี debug:** ลบ restart policy ชั่วคราวเพื่อดู error
```bash
docker update --restart=no <container>
docker start <container>
docker logs -f <container>
```

### Tools ที่ Senior ใช้

**1. dive — วิเคราะห์ layer ของ image**
```bash
dive myapp:1.0
# ดูได้ว่าแต่ละ layer ใส่ไฟล์อะไร ขนาดเท่าไหร่
```

**2. ctop — htop สำหรับ container**
```bash
ctop
```

**3. lazydocker — TUI สำหรับจัดการ docker**
```bash
lazydocker
```

**4. docker scout — analyze + scan**
```bash
docker scout quickview myapp:1.0
docker scout cves myapp:1.0
docker scout recommendations myapp:1.0
```

---

<a id="cheat-sheet"></a>
## 📋 Cheat Sheet — คำสั่งที่ใช้บ่อย

### Container
```bash
docker run -d -p HOST:CONT --name NAME --restart unless-stopped IMAGE
docker ps -a
docker logs -f --tail 100 NAME
docker exec -it NAME sh
docker stop NAME && docker rm NAME
docker rm -f $(docker ps -aq)        # ลบทุก container (ระวัง!)
```

### Image
```bash
docker build -t NAME:TAG .
docker build --no-cache -t NAME:TAG .
docker images
docker rmi NAME:TAG
docker image prune -a
docker tag local:1.0 registry.com/repo:1.0
docker push registry.com/repo:1.0
docker pull registry.com/repo:1.0
docker history NAME:TAG
```

### Volume & Network
```bash
docker volume create NAME
docker volume ls
docker volume inspect NAME
docker volume rm NAME
docker network create NAME
docker network connect NETWORK CONTAINER
docker network inspect NAME
```

### Compose
```bash
docker compose up -d
docker compose down [-v]
docker compose ps
docker compose logs -f [SERVICE]
docker compose exec SERVICE sh
docker compose restart [SERVICE]
docker compose build [--no-cache] [SERVICE]
docker compose up -d --build --force-recreate
docker compose pull
docker compose config        # validate และดู merged config
```

### Debug
```bash
docker inspect NAME
docker stats
docker top NAME
docker events
docker system df
docker system prune -a
docker scout cves IMAGE
```

### Build แบบโปร
```bash
DOCKER_BUILDKIT=1 docker build .
docker buildx build --platform linux/amd64,linux/arm64 -t NAME --push .
docker build --secret id=mytoken,src=./token.txt .
docker build --build-arg VERSION=1.0 .
```

---

## 🎓 เส้นทางต่อจากนี้

ถ้าเข้าใจทุกอย่างในนี้แล้ว ต่อไปคือ:

1. **Kubernetes** — orchestrate container ระดับ cluster
2. **Container security** — Falco, OPA, runtime security
3. **Service Mesh** — Istio, Linkerd
4. **CI/CD pipelines** — สร้าง pipeline build → test → scan → deploy
5. **Observability** — Prometheus, Grafana, Loki, Jaeger สำหรับ containerized apps
6. **Docker internals** — ลึกถึง namespaces, cgroups, OCI runtime (runc, containerd)

---

## 📚 Resources ที่แนะนำ

- [Docker Official Docs](https://docs.docker.com/) — เอกสารทางการ
- [Play with Docker](https://labs.play-with-docker.com/) — sandbox ฟรี
- [Docker Hub](https://hub.docker.com/) — image registry
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker) — รวม resources

---

*สร้างขึ้นเพื่อให้คุณ pass interview Docker ได้และรอด production จริง* 🚀
