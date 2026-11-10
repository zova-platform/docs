# รายละเอียดแต่ละ repo

แต่ละหัวข้อคือ spec ของ repo หนึ่งตัว ใช้เป็น README ตั้งต้นของ repo นั้นได้เลย เรียงตามลำดับที่ควรสร้าง

## กติกากลางของทุก service

ทุก service ที่สร้างจาก `service-template` ทำเหมือนกันตามนี้ ส่วนหัวข้อของแต่ละ service ด้านล่างจะบอกแค่สิ่งที่ต่าง

| เรื่อง | ค่ามาตรฐาน |
| --- | --- |
| port gRPC | `9090` |
| port HTTP ภายใน | `8081` สำหรับ `/healthz`, `/readyz`, `/metrics` |
| port HTTP ภายนอก | `8080` (เฉพาะ gateway) |
| proto package | `zova.<service>.v1` เช่น `zova.trip.v1` |
| Go module | `github.com/zova-platform/<repo>` |
| image | `ghcr.io/zova-platform/<repo>:<sha>` |
| database | แยก database ต่อ service ชื่อเดียวกับ service เช่น `trip` |

**Environment variables ที่ทุก service มี**

```env
SERVICE_NAME=trip-service
GRPC_ADDR=:9090
HTTP_ADDR=:8081
LOG_LEVEL=info
NATS_URL=nats://nats:4222
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

service ที่ใช้ database หรือ Redis เพิ่ม `DATABASE_URL` และ `REDIS_URL` ส่วนการเรียก service อื่นใช้รูปแบบ `<NAME>_SERVICE_ADDR` เช่น `LOCATION_SERVICE_ADDR=location-service:9090`

**ทุก service ต้องมีใน README:** วิธีรัน, env ทั้งหมด, gRPC method, event ที่ส่งและรับ, ตารางที่เป็นเจ้าของ

## Repo กลางและ infra (Phase 0)

7 repo นี้ไม่มี business logic แต่ทุก service พึ่งมัน จึงต้องสร้างก่อน

### `.github`

หน้าแรกของ org และไฟล์กลางที่ทุก repo ใช้ถ้าไม่มีของตัวเอง

- `profile/README.md`: โลโก้ ZOVA, ZOVA คืออะไร, architecture diagram, สถานะแต่ละ phase, ลิงก์ demo
- `ISSUE_TEMPLATE/`, `pull_request_template.md`, `CONTRIBUTING.md`

### `docs`

ความรู้ทั้งหมดที่ไม่ผูกกับ service ใด service หนึ่ง

- `architecture/`: C4 diagram, sequence ของ flow หลัก (Mermaid)
- `adr/`: Architecture Decision Record เช่น `0001-polyrepo.md`, `0002-nats-jetstream.md`
- `branding/`: โลโก้ ชุดสี
- `runbooks/`: ทำอย่างไรเมื่อ service ล่ม, outbox ค้าง
- `benchmarks.md`: ผล load test

### `proto`

สัญญาระหว่าง service ทั้งหมด เป็น source of truth ของ API ภายใน

- โครง: `zova/<service>/v1/<service>.proto` และ `gen/go/` ที่ generate แล้ว commit ไว้
- เครื่องมือ: Buf (`buf.yaml`, `buf.gen.yaml`)
- CI: `buf lint`, `buf breaking` เทียบกับ `main`, generate แล้วเช็คว่าไม่มีไฟล์เปลี่ยน
- ออก tag SemVer ทุกครั้งที่เปลี่ยน service อื่น `go get github.com/zova-platform/proto@vX.Y.Z`

### `platform-go`

library infrastructure ที่ทุก service ใช้ **ห้ามมี business logic**

| package | ทำอะไร |
| --- | --- |
| `config` | โหลด env เข้า struct |
| `logx` | ตั้งค่า slog JSON พร้อม trace\_id |
| `otelx` | เริ่ม tracer และ meter ส่งไป Collector |
| `grpcx` | สร้าง server/client พร้อม interceptor (log, trace, recover, timeout) |
| `natsx` | ต่อ JetStream, publish/subscribe แบบมี trace context, ตรวจ event\_id ซ้ำ |
| `outbox` | ตาราง outbox + worker ที่ publish |
| `health` | `/healthz`, `/readyz`, `/metrics` |
| `run` | graceful shutdown รวมทุกอย่าง |

### `ci-workflows`

reusable workflows ติด tag `v1`

- `go-ci.yml`: lint → test (race + testcontainers) → govulncheck
- `docker-publish.yml`: build → Trivy → push ghcr
- `web-ci.yml`: pnpm install → lint → typecheck → build
- `deploy-bump.yml`: แก้ image tag ใน `deploy` แล้ว commit

### `deploy`

ทุกอย่างที่ต้องใช้รันระบบทั้งชุด

- `compose/infra.yml`: Postgres, Redis, NATS, OSRM, observability stack
- `compose/services.yml`: ทุก service ระบุ image tag
- `compose/.env.example`: แม่แบบค่าตั้งต้น `task up` คัดลอกเป็น `compose/.env` (ไม่ commit) ให้เมื่อยังไม่มี บน server แก้รหัสผ่านใน `.env` เอง
- `osrm/`: script เตรียมข้อมูลกรุงเทพฯ (ตัดจากไฟล์ประเทศไทย)
- `observability/`: config ของ Collector, Prometheus, Grafana dashboard
- `k8s/` (phase 4): Helm chart กลาง, values ต่อ service, Argo CD app
- `Taskfile.yml`: `task up`, `task down`, `task clone-all`, `task osrm:prepare`

### `service-template`

แม่แบบ Go service ตั้งเป็น template repository มีโครง `cmd/`, `internal/`, `migrations/`, `queries/`, Dockerfile, Taskfile, `.golangci.yml`, workflow ที่เรียก `ci-workflows@v1` และ README ตามกติกากลาง

## `gateway` (Phase 1)

ทางเข้าเดียวของทุก client: รับ REST และ WebSocket, ตรวจ JWT, แปลงเป็น gRPC ไป service ข้างหลัง และ push event กลับหาผู้ใช้แบบ real-time ไม่เก็บข้อมูลเอง

**ไม่ทำ:** business logic ทุกชนิด เช่น ไม่ตัดสินว่าทริปยกเลิกได้ไหม แค่ส่งต่อให้ trip-service

**REST API (`/v1`)**

| Method | Path | ส่งต่อไป | ใช้โดย |
| --- | --- | --- | --- |
| POST | `/auth/register`, `/auth/login`, `/auth/refresh` | auth | ทุกคน |
| GET | `/me` | auth | ทุกคน |
| POST | `/quotes` | pricing | ผู้โดยสาร |
| POST | `/trips` | trip | ผู้โดยสาร |
| GET | `/trips/{id}`, `/trips` | trip | ผู้โดยสาร, คนขับ |
| POST | `/trips/{id}/cancel` | trip | ผู้โดยสาร, คนขับ |
| POST | `/trips/{id}/arrived`, `/start`, `/complete` | trip | คนขับ |
| POST | `/driver/online`, `/driver/offline` | location | คนขับ |
| POST | `/offers/{id}/accept`, `/offers/{id}/reject` | dispatch | คนขับ |

**WebSocket `/v1/ws`**

- ต่อพร้อม access token ใน query หรือ subprotocol
- client → server: `location.update` (คนขับส่งพิกัด), `ping`
- server → client: `trip.status_changed`, `driver.location`, `offer.new`, `offer.expired`
- gateway subscribe NATS subject `push.user.<user_id>` ของผู้ใช้ที่ต่ออยู่ แล้วส่งต่อเข้า WebSocket

**Events:** subscribe `push.user.*` เท่านั้น ไม่ publish

**ข้อมูล:** ไม่มี database ใช้ Redis แค่ทำ rate limit

**Env เพิ่มเติม**

```env
PUBLIC_HTTP_ADDR=:8080
JWT_PUBLIC_KEY_PATH=/secrets/jwt.pub
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174
AUTH_SERVICE_ADDR=auth-service:9090
LOCATION_SERVICE_ADDR=location-service:9090
PRICING_SERVICE_ADDR=pricing-service:9090
DISPATCH_SERVICE_ADDR=dispatch-service:9090
TRIP_SERVICE_ADDR=trip-service:9090
REDIS_URL=redis://redis:6379/0
```

**เสร็จเมื่อ**

- [ ] REST ทุกเส้นด้านบนส่งต่อได้ และแปลง gRPC error เป็น HTTP status ถูกต้อง
- [ ] WebSocket รับพิกัดคนขับส่งต่อ location-service ได้ และ push event กลับถึง client
- [ ] rate limit ต่อผู้ใช้ และปิด connection ที่ token หมดอายุ
- [ ] metric `gateway_ws_connections` และ RED ของทุก route

## `auth-service` (Phase 1)

เจ้าของข้อมูลผู้ใช้ คนขับ และรถ ออก token ให้ทั้งระบบ เซ็น JWT ด้วย private key (EdDSA หรือ RS256) แล้ว gateway ตรวจด้วย public key โดยไม่ต้องเรียก auth ทุก request

**gRPC (`zova.auth.v1.AuthService`)**

| Method | ทำอะไร |
| --- | --- |
| `Register` | สมัครด้วยเบอร์หรืออีเมล + รหัสผ่าน ระบุบทบาท passenger/driver |
| `Login` | คืน access token (15 นาที) + refresh token (30 วัน) |
| `Refresh` | หมุน refresh token ใหม่ ตัวเก่าใช้ไม่ได้อีก |
| `Logout` | เพิกถอน refresh token |
| `GetUser` | ข้อมูลผู้ใช้ตาม id |
| `GetDriverProfile` | ข้อมูลคนขับ + รถ (dispatch และ trip ใช้แสดงชื่อ ป้ายทะเบียน) |
| `UpsertVehicle` | คนขับเพิ่มหรือแก้รถ (car/moto) |

**Events:** publish `auth.user.registered` (ไว้ให้ notification ส่งข้อความต้อนรับใน phase 4)

**ข้อมูล (Postgres `auth`):** `users`, `drivers`, `vehicles`, `refresh_tokens` (เก็บ hash ของ token, expires\_at, revoked\_at)

**Env เพิ่มเติม**

```env
DATABASE_URL=postgres://auth:***@postgres:5432/auth
JWT_PRIVATE_KEY_PATH=/secrets/jwt.key
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL=720h
```

**เสร็จเมื่อ**

- [ ] รหัสผ่าน hash ด้วย argon2id ไม่มี plaintext ใน log
- [ ] refresh token rotation: ใช้ token เก่าซ้ำแล้วโดนเพิกถอนทั้งชุด
- [ ] seed script สร้างผู้โดยสารและคนขับทดสอบได้ (simulator ใช้)

## `location-service` (Phase 1)

รู้ว่าคนขับแต่ละคนอยู่ที่ไหนและว่างไหม service นี้รับโหลดหนักสุดของระบบ (พิกัดทุก 3 วินาทีจากคนขับทุกคน) จึงเก็บใน Redis ล้วน ไม่เขียน Postgres

**gRPC (`zova.location.v1.LocationService`)**

| Method | ทำอะไร |
| --- | --- |
| `UpdateLocation` (client stream) | รับพิกัดต่อเนื่องจาก gateway: lat, lng, heading, speed, timestamp |
| `SetDriverStatus` | `online` / `offline` / `busy` (dispatch ตั้ง busy ตอนได้งาน) |
| `NearbyDrivers` | คนขับว่างรอบจุดหนึ่ง: center, radius\_m, vehicle\_type, limit → เรียงจากใกล้สุด |
| `GetDriverLocation` | พิกัดล่าสุดของคนขับหนึ่งคน |

**Events:** publish `location.driver.moved` และ `push.user.<passenger_id>` เฉพาะคนขับที่มี `active_job_id` ไม่เกิน 1 ครั้งต่อวินาที

**ข้อมูล (Redis)**

- `drivers:geo:car`, `drivers:geo:moto`: GEO set เฉพาะคนขับที่ว่าง
- `driver:<id>`: hash ของ status, vehicle\_type, last\_seen, active\_job\_id มี TTL 30 วินาที
- คนขับหยุดส่ง ping เกิน TTL → ถือว่าออฟไลน์ ต้องมี job เล็ก ๆ ลบออกจาก GEO set ด้วย (GEO set ไม่มี TTL ต่อสมาชิก)

**Env เพิ่มเติม**

```env
REDIS_URL=redis://redis:6379/1
DRIVER_TTL=30s
MOVED_EVENT_INTERVAL=1s
```

**เสร็จเมื่อ**

- [ ] `NearbyDrivers` ตอบภายในหลักมิลลิวินาทีเมื่อมีคนขับ 1,000 คน (บันทึกตัวเลขจริงใน benchmarks)
- [ ] คนขับที่หลุดหายไปจากผลค้นหาภายใน TTL
- [ ] คนขับ `busy` ไม่โผล่ใน `NearbyDrivers`

## `pricing-service` (Phase 1)

คิดราคาจากระยะทางและเวลาจริงบนถนนที่ได้จาก OSRM ราคาที่ประเมินถูกล็อกไว้ด้วย `quote_id` 5 นาที ผู้โดยสารจึงจ่ายตามที่เห็นตอนกดเรียก ถ้าใช้แนว MVP ลดขนาด ให้ยุบ service นี้เป็น package ใน `trip-service` ก่อนแล้วแยกทีหลัง

**gRPC (`zova.pricing.v1.PricingService`)**

| Method | ทำอะไร |
| --- | --- |
| `CreateQuote` | pickup, dropoff, vehicle\_type → ถาม OSRM → คืน quote\_id, fare, distance\_m, duration\_s, route polyline |
| `GetQuote` | ตรวจว่า quote ยังไม่หมดอายุ (trip เรียกก่อนสร้างทริป) |
| `CalculateFinalFare` | ระยะและเวลาจริงตอนจบทริป → ราคาสุดท้าย |

**สูตรเริ่มต้น:** `max(min_fare, base_fare + per_km × km + per_min × นาที)` ปัดเป็นบาทเต็ม ค่าแต่ละตัวแยกตาม car/moto อยู่ในตาราง `fare_rules` (phase หลังเพิ่ม surge multiplier)

**Events:** ไม่มี

**ข้อมูล:** Postgres `pricing` เก็บ `fare_rules` · Redis เก็บ `quote:<id>` TTL 5 นาที

**Env เพิ่มเติม**

```env
DATABASE_URL=postgres://pricing:***@postgres:5432/pricing
REDIS_URL=redis://redis:6379/2
OSRM_URL=http://osrm:5000
QUOTE_TTL=5m
```

**เสร็จเมื่อ**

- [ ] unit test ของสูตรครอบกรณี min\_fare, ระยะ 0, moto กับ car
- [ ] OSRM ล่มหรือช้า → ตอบ error ชัดเจนภายใน timeout ไม่ค้าง
- [ ] quote หมดอายุแล้วสร้างทริปไม่ได้

## `dispatch-service` (Phase 1)

จับคู่ "งาน" กับคนขับ รู้จักแค่งานที่มีจุดรับ จุดส่ง ประเภท (`ride`, `delivery`, `parcel`) และ vehicle\_type ไม่รู้จักผู้โดยสารหรือออเดอร์ นี่คือ service ที่ยากและน่าเล่าที่สุดของโปรเจกต์

**วิธีจับคู่ (ทีละคน)**

1. รับ `dispatch.job.requested` → บันทึก job สถานะ `searching`
2. ถาม location `NearbyDrivers` (รัศมี 3 กม. สูงสุด 10 คน) แล้วตัดคนที่เคยปฏิเสธงานนี้ออก
3. ล็อกคนแรกด้วย Redis `SET lock:driver:<id> <job_id> NX EX 20` ล็อกไม่ได้แปลว่าโดนงานอื่นจองอยู่ ข้ามไปคนถัดไป
4. บันทึก offer แล้ว push `offer.new` ไปที่ `push.user.<driver_id>` ให้เวลาตอบ 15 วินาที
5. รับ → ตั้งคนขับ `busy` ที่ location → publish `dispatch.job.assigned` · ปฏิเสธหรือหมดเวลา → ปลดล็อก กลับข้อ 2
6. รวมเกิน 2 นาทีหรือไม่มีคนเหลือ → publish `dispatch.job.unfulfilled`

**gRPC (`zova.dispatch.v1.DispatchService`)**

| Method | ทำอะไร |
| --- | --- |
| `RespondOffer` | คนขับรับ/ปฏิเสธ คืนผลทันทีว่าได้งานหรือ offer หมดอายุแล้ว |
| `CancelJob` | เจ้าของงาน (trip/order) ยกเลิกระหว่างค้นหา |
| `GetJob` | สถานะงาน (ใช้ debug และ admin-web) |

**Events:** subscribe `dispatch.job.requested` · publish `dispatch.job.assigned`, `dispatch.job.unfulfilled`, `push.user.<driver_id>` (offer.new, offer.expired)

**ข้อมูล:** Postgres `dispatch` เก็บ `jobs`, `job_offers`, `outbox` · Redis เก็บ `lock:driver:<id>` และ timer ของ offer

**Env เพิ่มเติม**

```env
DATABASE_URL=postgres://dispatch:***@postgres:5432/dispatch
REDIS_URL=redis://redis:6379/3
LOCATION_SERVICE_ADDR=location-service:9090
SEARCH_RADIUS_M=3000
MAX_CANDIDATES=10
OFFER_TIMEOUT=15s
JOB_TIMEOUT=2m
```

**เสร็จเมื่อ**

- [ ] test ยิงงาน 200 งานพร้อมกันในจุดเดียว ไม่มีคนขับคนไหนได้สองงาน
- [ ] restart service ระหว่างมีงานค้าง แล้วงานเดินต่อหรือจบอย่างถูกต้อง (timer ต้องกู้คืนได้จาก DB)
- [ ] metric `dispatch_match_duration_seconds`, `dispatch_offers_total`, `dispatch_unfulfilled_total`

## `trip-service` (Phase 1)

เจ้าของวงจรชีวิตทริปรับส่งคน (ZOVA Car และ Moto) ทุกการเปลี่ยนสถานะเกิดที่นี่ที่เดียว ผ่าน state machine ในแท็บแรก และบันทึกประวัติทุกครั้ง

**gRPC (`zova.trip.v1.TripService`)**

| Method | ใครเรียก | ทำอะไร |
| --- | --- | --- |
| `RequestTrip` | ผู้โดยสาร | ตรวจ quote กับ pricing → สร้างทริป `requested` → ส่งงานเข้า dispatch |
| `CancelTrip` | ผู้โดยสาร, คนขับ | ผู้โดยสารยกเลิกได้ก่อน `in_progress` · คนขับยกเลิกแล้วทริปกลับเป็น `requested` |
| `MarkArrived` | คนขับ | `driver_assigned → driver_arrived` |
| `StartTrip` | คนขับ | `driver_arrived → in_progress` |
| `CompleteTrip` | คนขับ | `in_progress → completed` + ถาม pricing ราคาสุดท้าย |
| `GetTrip`, `ListTrips` | ทุกคน | ดูทริปของตัวเอง (ตรวจสิทธิ์ว่าเป็นผู้โดยสารหรือคนขับของทริปนั้น) |

ทุก method ที่เปลี่ยนสถานะรับ `idempotency_key` กดซ้ำจากเน็ตกระตุกต้องไม่สร้างทริปซ้ำ

**Events:** subscribe `dispatch.job.assigned`, `dispatch.job.unfulfilled` · publish `dispatch.job.requested`, `ride.trip.status_changed`, `ride.trip.completed`, `push.user.<passenger_id>` และ `push.user.<driver_id>`

**ข้อมูล (Postgres `trip`):** `trips` (มี `version` สำหรับ optimistic locking), `trip_events`, `idempotency_keys`, `outbox`

**Env เพิ่มเติม**

```env
DATABASE_URL=postgres://trip:***@postgres:5432/trip
PRICING_SERVICE_ADDR=pricing-service:9090
DISPATCH_SERVICE_ADDR=dispatch-service:9090
```

**เสร็จเมื่อ**

- [ ] table-driven test ครอบทุกการเปลี่ยนสถานะ ทั้งที่อนุญาตและไม่อนุญาต
- [ ] เปลี่ยนสถานะพร้อมกันสองทาง (เช่น ผู้โดยสารยกเลิกขณะคนขับกดถึงแล้ว) ได้ผลเดียวที่ถูกต้อง
- [ ] ทุกการเปลี่ยนสถานะออก event ผ่าน outbox ไม่มีหาย
- [ ] metric `trips_by_status`

## เว็บ: `passenger-web`, `driver-web`, `admin-web`

ทั้งสามเป็น Vite + React + TypeScript ใช้ stack เดียวกัน (React Router, TanStack Query, Zustand, Tailwind ชุดสี ZOVA, react-map-gl + MapLibre) build เป็นไฟล์ static เสิร์ฟด้วย Caddy คุยกับ gateway อย่างเดียว

**โครงร่วม**

```text
src/
├── api/        # client REST + WebSocket (reconnect อัตโนมัติ)
├── map/        # component แผนที่, หมุดรถ, เส้นทาง
├── stores/     # Zustand: session, trip ปัจจุบัน
├── pages/
└── theme.ts    # design token ZOVA
```

**Env (ตอน build):** `VITE_API_URL`, `VITE_WS_URL`, `VITE_MAP_STYLE_URL`

### `passenger-web` (Phase 1)

| หน้า | ทำอะไร |
| --- | --- |
| `/login`, `/register` | เข้าสู่ระบบ |
| `/` | ปักหมุดต้นทาง/ปลายทาง เลือก Car/Moto เห็นราคาและเส้นทาง กดเรียก |
| `/trip/:id` | สถานะทริป รถวิ่งมาบนแผนที่ ข้อมูลคนขับและป้ายทะเบียน ปุ่มยกเลิก |
| `/history` | ทริปที่ผ่านมา |

### `driver-web` (Phase 1, PWA)

| หน้า | ทำอะไร |
| --- | --- |
| `/` | สวิตช์ออนไลน์/ออฟไลน์ ส่งพิกัดจาก GPS ทุก 3 วินาทีผ่าน WebSocket |
| (popup) | งานใหม่: จุดรับ ระยะ ราคา นับถอยหลัง 15 วินาที ปุ่มรับ/ปฏิเสธ |
| `/job` | นำทางไปจุดรับ ปุ่ม ถึงแล้ว → เริ่มเดินทาง → จบทริป |
| `/earnings` | รายได้ (phase 4) |

ต้องเปิดผ่าน HTTPS ถึงจะใช้ GPS บนมือถือได้ ตอน dev ใช้ Cloudflare Tunnel หรือ `vite --host` กับ certificate local

### `admin-web` (Phase 2)

| หน้า | ทำอะไร |
| --- | --- |
| `/map` | แผนที่รวมเห็นรถทุกคันแบบ real-time แยกสีตามสถานะ |
| `/trips` | ค้นหาทริป ดูประวัติสถานะจาก `trip_events` |
| `/drivers` | รายชื่อคนขับ อนุมัติ ระงับ |
| `/metrics` | ฝัง Grafana dashboard |

**เสร็จเมื่อ (ทั้งสามตัว)**

- [ ] WebSocket หลุดแล้วต่อใหม่เอง และดึงสถานะล่าสุดผ่าน REST หลังต่อได้
- [ ] รถบนแผนที่ขยับลื่น (interpolate ระหว่างพิกัด) ไม่กระโดด
- [ ] มี attribution © OpenStreetMap contributors บนแผนที่

## `simulator` (Phase 2)

โปรแกรม Go ตัวเดียวที่ปล่อยคนขับและผู้โดยสารปลอมจำนวนมาก ทำตัวเหมือน client จริงทุกอย่าง (ล็อกอิน, REST, WebSocket ผ่าน gateway) ห้ามแตะ database หรือ NATS ตรง ๆ เพื่อให้ตัวเลขที่วัดได้สะท้อนระบบจริง

**คำสั่ง**

```bash
simulator drivers   --count 500 --area bangkok-inner --accept-rate 0.8
simulator riders    --rate 2/s --area bangkok-inner
simulator scenario  concert-exit   # 200 คนขอรถพร้อมกันจุดเดียว
simulator seed      --drivers 1000 --riders 1000   # สร้างบัญชีผ่าน auth
```

**Config:** จำนวนคนขับ, สัดส่วน car/moto, อัตราขอรถ, อัตรารับงาน, หน่วงเวลาตอบ, ตัวคูณความเร็ว (เร่งเวลาสำหรับ demo), กรอบพื้นที่

**ข้างใน**

- คนขับแต่ละคันเป็น goroutine มี state machine ของตัวเอง: idle → cruising → offered → to\_pickup → to\_dropoff
- ขอเส้นทางจาก OSRM แล้วขยับตาม polyline ตามความเร็วที่ตั้ง
- `loadtest/`: script k6 สำหรับยิง REST และ WebSocket แยกจาก simulator
- export metric ของตัวเอง เช่น จำนวนคนขับจำลองที่ทำงานอยู่ ทริปสำเร็จ/ล้มเหลว

**Env:** `GATEWAY_URL`, `GATEWAY_WS_URL`, `OSRM_URL`

**เสร็จเมื่อ**

- [ ] รัน 500 คันบนเครื่อง dev ได้ไม่กิน CPU ของ simulator เองจนบิดตัวเลข
- [ ] scenario `concert-exit` รันซ้ำได้ ผลออกมาเป็นตัวเลขสรุป (match สำเร็จ, p95, unfulfilled)

## `payment-service` (Phase 4)

ZOVA Wallet และการคิดเงินตอนจบทริป เริ่มจากเงินจำลองทั้งหมด ไม่แตะเงินจริง ทุกการเคลื่อนไหวของเงินเก็บแบบ ledger (บันทึกเพิ่มอย่างเดียว ไม่แก้ยอดตรง ๆ)

**gRPC (`zova.payment.v1.PaymentService`)**

| Method | ทำอะไร |
| --- | --- |
| `GetWallet` | ยอดคงเหลือและรายการล่าสุด |
| `TopUp` | เติมเงินจำลอง (ภายหลังต่อ PromptPay QR) |
| `GetPayment` | สถานะการจ่ายของทริปหนึ่ง |

**Events:** subscribe `ride.trip.completed` → ตัดเงินผู้โดยสาร เข้ากระเป๋าคนขับ หักค่าธรรมเนียม · publish `payment.charge.succeeded`, `payment.charge.failed`

**ข้อมูล (Postgres `payment`):** `wallets`, `ledger_entries` (debit/credit คู่กันเสมอ), `payments`, `outbox`

**เสร็จเมื่อ**

- [ ] event `ride.trip.completed` ซ้ำไม่ทำให้ตัดเงินซ้ำ
- [ ] ผลรวม ledger ของทั้งระบบเป็นศูนย์เสมอ (มี test ตรวจ)

## `notification-service` (Phase 4)

ส่งข้อความออกนอกแอปเมื่อผู้ใช้ไม่ได้เปิดเว็บอยู่ เช่น คนขับใกล้ถึง ทริปจบพร้อมใบเสร็จ

**Events:** subscribe `ride.trip.status_changed`, `payment.charge.succeeded`, `auth.user.registered` · ไม่ publish

**ช่องทาง:** เริ่มจาก email (SMTP ทดสอบด้วย Mailpit) แล้วค่อยเพิ่ม LINE Messaging API หรือ Web Push

**ข้อมูล (Postgres `notification`):** `notification_logs`, `user_preferences` (ปิดเปิดแต่ละช่องทาง)

**เสร็จเมื่อ**

- [ ] ส่งไม่สำเร็จแล้ว retry แบบ backoff ไม่ทำให้ consumer ค้าง
- [ ] event เดียวกันไม่ส่งข้อความซ้ำ

## ZOVA Food (Phase 5)

4 repo ใหม่ ใช้ dispatch, location, payment เดิมโดยไม่แก้ เขียนละเอียดตอนถึง phase 5 ตอนนี้กำหนดขอบเขตไว้ก่อน

| Repo | หน้าที่ | gRPC หลัก | Events |
| --- | --- | --- | --- |
| `restaurant-service` | ร้าน ที่อยู่ เวลาเปิดปิด พื้นที่ส่ง สถานะเปิดรับออเดอร์ | `GetRestaurant`, `ListNearby`, `SetOpen` | ไม่มี |
| `menu-service` | เมนู หมวดหมู่ ราคา ตัวเลือกเสริม ของหมด | `GetMenu`, `UpsertItem`, `SetAvailability` | ไม่มี |
| `order-service` | ตะกร้า ออเดอร์ state machine `placed → accepted → preparing → ready → picked_up → delivered` | `PlaceOrder`, `AcceptOrder`, `MarkReady`, `CancelOrder` | publish `dispatch.job.requested` (`job_type=delivery`), `food.order.status_changed` · subscribe `dispatch.job.assigned` |
| `merchant-web` | เว็บร้านค้า: รับออเดอร์ เสียงแจ้งเตือน กดใกล้เสร็จ จัดการเมนู | (ผ่าน gateway) | ผ่าน WebSocket |

สิ่งที่ต้องแก้ในของเดิม: gateway เพิ่ม route ของ food, auth เพิ่มบทบาท `merchant`, pricing เพิ่มสูตรค่าส่ง, passenger-web เพิ่มหน้าสั่งอาหาร
