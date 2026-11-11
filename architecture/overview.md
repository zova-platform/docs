# ZOVA — เอกสารโปรเจกต์ระบบเรียกรถ

Nov 10, 2026 · @nae

## ภาพรวมโปรเจกต์

สร้างระบบเรียกรถแบบ Grab ย่อส่วนด้วย microservices (Go) เก็บแบบ polyrepo ใน GitHub org เดียว เริ่มจากรับส่งคนก่อน และออกแบบให้ขยายไปส่งอาหารได้โดยไม่ต้องรื้อของเดิม

ชื่อโปรเจกต์คือ **ZOVA** (tagline: MOVE YOUR WAY.) และ GitHub org คือ [`zova-platform`](https://github.com/zova-platform) (ชื่อ `zova` มีคนใช้แล้ว)

**เป้าหมาย**

- เป็น portfolio สาย Backend/DevOps ที่โชว์ distributed system จริง: real-time, race condition, event-driven, observability
- demo ได้บนแผนที่จริงของกรุงเทพฯ ด้วยคนขับจำลองหลายร้อยคัน
- มีตัวเลขเอาไปเล่าได้ เช่น match สำเร็จภายในกี่วินาที รับพิกัดได้กี่ ping ต่อวินาที

**ทำใน phase แรก (ride MVP)**

- สมัคร/ล็อกอิน แยกบทบาทผู้โดยสาร คนขับ แอดมิน
- คนขับออนไลน์และส่งพิกัดแบบ real-time
- ประเมินราคาจากระยะทางและเวลาจริงบนถนน
- จับคู่คนขับที่ใกล้ที่สุด เสนองาน รับ/ปฏิเสธ/หมดเวลา แล้วส่งต่อคนถัดไป
- ติดตามรถบนแผนที่และจัดการสถานะทริปจนจบ
- simulator คนขับและผู้โดยสาร

**ไม่ทำ (อย่างน้อยช่วงแรก)**

- แอปมือถือ native ใช้เว็บแทน
- เงินจริง ใช้การจ่ายจำลองก่อน
- KYC คนขับ การให้คะแนน โปรโมชัน แชท
- multi-region หรือ scale ระดับ production จริง

## Branding

โลโก้เป็นตัว Z สองท่อนสีกรมท่ากับส้มประกบกัน สื่อถึงเส้นทางรับและส่งที่มาบรรจบกัน ใช้ชุดสีเดียวกันทั้งโลโก้และ UI

&#91;image: โลโก้ ZOVA: ตัว Z สองท่อนสีกรมท่าและส้ม\]

**ชุดสี**

| สี | Hex | บทบาท | ใช้กับ |
| --- | --- | --- | --- |
| Navy | #172B3D | Primary | header, navigation, ตัวอักษรหลัก |
| Orange | #F4A340 | Accent | ปุ่ม CTA, ไฮไลต์, องค์ประกอบแบรนด์ |
| Mint | #54B8A1 | Secondary | สถานะสำเร็จ, ไฮไลต์รอง |
| White | #F5F6F7 | Background | พื้นหลัง, card, surface |
| Red (เสนอเพิ่ม) | #D64545 | Status | ยกเลิก, หาคนขับไม่ได้, error |
| Yellow (เสนอเพิ่ม) | #F2C94C | Status | คำเตือน, รอคนขับตอบรับ |

- ตัวอักษรบนปุ่มส้มให้ใช้ Navy ไม่ใช่สีขาว เพราะขาวบน #F4A340 contrast ต่ำ (ราว 2:1) อ่านยาก
- ส้มในไฟล์โลโก้ตอนนี้สดกว่า #F4A340 และกรมท่าก็เข้มกว่า #172B3D ให้ปรับโลโก้ให้ใช้ hex ชุดนี้
- ทดสอบโลโก้ที่ 16–32px (favicon, หมุดบนแผนที่) ถ้าช่องระหว่างสองท่อนหาย ให้ทำเวอร์ชันย่อที่ช่องกว้างขึ้น

**Brand architecture → ส่วนของระบบ**

| บริการ | ความหมาย | ในระบบคือ | Phase |
| --- | --- | --- | --- |
| ZOVA Car | เรียกรถยนต์ | งาน `ride` + `vehicle_type=car` | 1 |
| ZOVA Moto | เรียกมอเตอร์ไซค์ | งาน `ride` + `vehicle_type=moto` | 1 |
| ZOVA Wallet | กระเป๋าเงิน | `payment-service` | 4 |
| ZOVA Food (เสนอเพิ่ม) | สั่งอาหาร | งาน `delivery` + `restaurant`, `menu`, `order` service | 5 |
| ZOVA Send | ส่งพัสดุ | งาน `parcel` | หลัง 5 |

## Flow ผู้ใช้หลักและ trip state machine

ทริปหนึ่งผ่าน 5 สถานะหลักตามลำดับ และจบแบบผิดปกติได้ 2 ทาง ทุกการเปลี่ยนสถานะทำที่ `trip-service` ที่เดียว

1. คนขับเปิดออนไลน์ → ส่งพิกัดทุก 3 วินาทีผ่าน WebSocket → `location-service` อัปเดต Redis GEO
2. ผู้โดยสารปักหมุดต้นทาง/ปลายทาง → `pricing-service` ถาม OSRM แล้วตอบราคาพร้อม `quote_id` (ล็อกราคา 5 นาที)
3. ผู้โดยสารกดเรียก → `trip-service` สร้างทริปสถานะ `requested` แล้วส่ง `dispatch.job.requested`
4. `dispatch-service` ขอคนขับว่างใกล้สุด 10 คนในรัศมี 3 กม. → ล็อกคนแรก → เสนองาน ให้เวลาตอบ 15 วินาที
5. ปฏิเสธหรือไม่ตอบ → ปลดล็อก เสนอคนถัดไป · ครบรายชื่อหรือเกิน 2 นาที → `no_driver_found`
6. คนขับรับ → `dispatch.job.assigned` → ทริปเป็น `driver_assigned` → ผู้โดยสารเห็นรถวิ่งมาบนแผนที่
7. คนขับกด "ถึงแล้ว" → "รับผู้โดยสารแล้ว" → "ถึงปลายทาง" ตามลำดับ
8. จบทริป → `pricing-service` คิดค่าโดยสารจริง → `ride.trip.completed` → `payment-service`

ตัวเลข (10 คน, 3 กม., 15 วินาที, 2 นาที) เป็นค่าเริ่มต้นที่ตั้งผ่าน config ได้

&#91;embedded content: trip state machine · 7 สถานะ\]

ผู้โดยสารยกเลิกได้ถึงก่อน `in_progress` และถ้าคนขับยกเลิกหลังรับงาน ทริปจะกลับไปเป็น `requested` เพื่อหาคนขับใหม่

## Architecture

client ทุกตัวคุยกับ `gateway` ที่เดียว gateway เรียก service ด้วย gRPC และ service แจ้งกันเองผ่าน event บน NATS JetStream

&#91;embedded content: architecture phase 1 · 4 client, gateway, 5 service, event bus\]

เส้นประจาก NATS ขึ้นไป gateway คือการ push event (เช่น รถขยับ, ได้คนขับแล้ว) กลับไปหา WebSocket ของผู้ใช้

**เลือก sync หรือ async อย่างไร**

- **gRPC (sync)** เมื่อผู้เรียกต้องได้คำตอบทันทีจึงทำต่อได้: ขอราคา, คนขับกดรับงาน, `dispatch` ถาม `location` หาคนขับใกล้สุด, `trip` ถาม `pricing` ตอนคิดเงินจบทริป
- **Event (async)** เมื่อเป็นการบอกว่า "มีอะไรเกิดขึ้นแล้ว" และผู้รับอาจมีหลายตัว: ทริปเปลี่ยนสถานะ, งานถูก assign, ทริปจบ
- ทุก gRPC call ตั้ง timeout และ retry เฉพาะ call ที่ idempotent
- ไม่มี service ไหนเรียกกันเป็นวงกลม ถ้าเริ่มเป็นวงให้เปลี่ยนเส้นใดเส้นหนึ่งเป็น event

## รายละเอียดแต่ละ service

phase แรกมี 6 service + simulator และอีก 2 service เพิ่มทีหลัง แต่ละ service เป็นเจ้าของข้อมูลของตัวเอง ห้ามอ่าน database ของ service อื่นตรง ๆ

| Service | หน้าที่ | API ที่เปิด | ข้อมูลที่เป็นเจ้าของ | Phase |
| --- | --- | --- | --- | --- |
| `gateway` | ประตูเดียวของ client: ตรวจ JWT, แปลง REST เป็น gRPC, ถือ WebSocket ของผู้ใช้ทุกคน, push event กลับหา client | REST `/v1/*`, WebSocket `/v1/ws` | ไม่มี (stateless) | 1 |
| `auth-service` | สมัคร ล็อกอิน ออก access/refresh token, ข้อมูลคนขับและรถ | gRPC `Register`, `Login`, `Refresh`, `GetUser`, `GetDriverProfile` | users, drivers, vehicles | 1 |
| `location-service` | รับพิกัดคนขับ เก็บสถานะ online/busy, หาคนขับใกล้สุด | gRPC `UpdateLocation` (stream), `NearbyDrivers`, `SetDriverStatus` | Redis GEO + สถานะคนขับ (มี TTL) | 1 |
| `pricing-service` | ประเมินราคาจากเส้นทาง OSRM แล้วล็อกราคาไว้ด้วย quote\_id | gRPC `CreateQuote`, `GetQuote`, `CalculateFinalFare` | fare\_rules, quotes (Redis, TTL) | 1 |
| `dispatch-service` | จับคู่ "งาน" ทั่วไปกับคนขับ: เสนองานทีละคน, timeout, ล็อกคนขับกันโดนจับซ้อน | gRPC `RespondOffer`; event `job.requested` เข้า | jobs, job\_offers | 1 |
| `trip-service` | วงจรชีวิตทริปรับส่งคน (state machine), ประวัติทริป | gRPC `RequestTrip`, `CancelTrip`, `UpdateTripStatus`, `GetTrip`, `ListTrips` | trips, trip\_events, outbox | 1 |
| `payment-service` | คิดเงินตอนจบทริป เริ่มจากเงินสด/wallet จำลอง ต่อยอดเป็น PromptPay QR | gRPC `GetPayment`; event `trip.completed` เข้า | payments, wallets | 4 |
| `notification-service` | แจ้งเตือนนอกแอป เช่น LINE, email | event เข้าอย่างเดียว | notification\_logs | 4 |
| `simulator` | ปล่อยคนขับและผู้โดยสารปลอม วิ่งตามถนนจริง | CLI + config | ไม่มี | 2 |

**ทำไม `dispatch` แยกจาก `trip`:** dispatch รู้จักแค่ "งาน" ที่มีจุดรับ จุดส่ง และประเภท (`ride`, `delivery`, `parcel`) ไม่รู้จักผู้โดยสาร พอทำส่งอาหาร `order-service` แค่สร้างงานประเภท `delivery` ส่งเข้ามา

**ทำไม realtime อยู่ใน `gateway`:** client ต่อ WebSocket ที่เดียว gateway subscribe NATS subject ของผู้ใช้แต่ละคน (`push.user.<id>`) แล้วส่งต่อ ถ้ามี gateway หลายตัว ตัวไหนถือ connection ของคนนั้นก็รับ subject นั้นเอง

**กฎกลางที่ทุก service ทำเหมือนกัน**

- health check `/healthz` (ยังมีชีวิต) และ `/readyz` (พร้อมรับงาน, เช็ค DB/NATS)
- graceful shutdown: หยุดรับงานใหม่ ทำงานค้างให้เสร็จ แล้วค่อยปิด
- config จาก environment variables เท่านั้น
- log เป็น JSON มี `trace_id` ทุกบรรทัด
- ทุก gRPC call มี timeout และส่ง trace context ต่อ

## Data model และ event catalog

ใช้ PostgreSQL (เปิด PostGIS) แยก database ต่อ service และใช้ Redis กับข้อมูลที่เปลี่ยนเร็วหรือมีอายุสั้น ทุก id เป็น UUIDv7 (เรียงตามเวลาได้)

**ตารางหลัก**

| Service | ตาราง / key | คอลัมน์สำคัญ |
| --- | --- | --- |
| auth | `users` | id, phone, email, password\_hash, role (passenger/driver/admin), created\_at |
| auth | `drivers`, `vehicles` | driver: user\_id, license\_no, approved · vehicle: id, driver\_id, plate, type (car/moto), model |
| location | Redis `drivers:geo:<vehicle_type>` | GEO set ของคนขับที่ออนไลน์และว่าง |
| location | Redis `driver:<id>` | status, last\_seen, active\_job\_id, TTL 30 วินาที (ไม่ส่ง ping = หลุดเอง) |
| pricing | `fare_rules` | vehicle\_type, base\_fare, per\_km, per\_min, min\_fare |
| pricing | Redis `quote:<id>` | pickup, dropoff, distance\_m, duration\_s, fare, TTL 5 นาที |
| dispatch | `jobs` | id, type, ref\_id, pickup (geography), dropoff, status, attempts, created\_at |
| dispatch | `job_offers` | job\_id, driver\_id, offered\_at, response (accepted/rejected/expired), responded\_at |
| dispatch | Redis `lock:driver:<id>` | `SET NX` กันคนขับโดนเสนอสองงานพร้อมกัน |
| trip | `trips` | id, passenger\_id, driver\_id, status, pickup, dropoff, quote\_id, fare\_estimate, fare\_final, distance\_m, timestamps ทุกสถานะ, cancel\_reason, version |
| trip | `trip_events` | trip\_id, from\_status, to\_status, actor, at (ประวัติทุกการเปลี่ยนสถานะ) |
| ทุก service ที่ส่ง event | `outbox` | id, subject, payload (jsonb), created\_at, published\_at |

`trips.version` ใช้ทำ optimistic locking: `UPDATE ... WHERE id = $1 AND version = $2` ถ้าไม่มีแถวถูกแก้แปลว่ามีคนเปลี่ยนก่อน ให้อ่านใหม่

**Outbox pattern:** เขียนข้อมูลกับ event ลง `outbox` ใน transaction เดียว แล้วมี worker อ่าน outbox ไป publish เข้า NATS กันกรณี DB บันทึกแล้วแต่ส่ง event ไม่ทัน

**Event catalog (NATS JetStream)**

รูปแบบ subject: `<domain>.<entity>.<event>` · payload เป็น JSON มี `event_id`, `occurred_at`, `version` เสมอ · consumer ต้อง idempotent (เจอ `event_id` ซ้ำให้ข้าม)

| Subject | ผู้ส่ง | ผู้รับ | ข้อมูลหลัก |
| --- | --- | --- | --- |
| `dispatch.job.requested` | trip (ภายหลัง order) | dispatch | job\_type, ref\_id, pickup, dropoff, vehicle\_type |
| `dispatch.job.offered` | dispatch | gateway (push หาคนขับ) | job\_id, driver\_id, pickup, expires\_at |
| `dispatch.job.assigned` | dispatch | trip, gateway | job\_id, ref\_id, driver\_id |
| `dispatch.job.unfulfilled` | dispatch | trip | job\_id, ref\_id, reason |
| `ride.trip.status_changed` | trip | gateway, notification | trip\_id, from, to, driver\_id, passenger\_id |
| `ride.trip.completed` | trip | payment | trip\_id, distance\_m, duration\_s, fare\_final |
| `location.driver.moved` | location | gateway | driver\_id, job\_id, lat, lng, heading (ส่งเฉพาะคนขับที่มีทริป, ไม่เกิน 1 ครั้ง/วินาที) ส่งผ่าน core NATS ไม่เก็บใน JetStream เพราะพิกัดเก่าไม่มีประโยชน์ถ้าเล่นซ้ำ |
| `payment.charge.succeeded` | payment | trip, notification | trip\_id, amount, method |

คนขับรับ/ปฏิเสธงานผ่าน gRPC `RespondOffer` (sync) ไม่ใช่ event เพราะคนขับต้องรู้ผลทันทีว่าได้งานหรือมีคนอื่นได้ไปแล้ว

JetStream stream ครอบ subject `auth.>`, `dispatch.>`, `ride.>` และ `payment.>` payload เป็น JSON ที่แปลงจาก proto ด้วยชื่อ field แบบ snake\_case (`event_id`, `occurred_at`) และ publish ด้วย `Nats-Msg-Id` เท่ากับ `event_id` เพื่อกัน event ซ้ำฝั่งส่ง ฝั่งรับข้าม `event_id` ที่เคยจัดการแล้ว

การส่งข้อมูลกลับหาผู้ใช้ทำผ่าน subject `push.user.<user_id>` บน NATS (ไม่ผ่าน JetStream เพราะถ้าผู้ใช้ไม่ได้ต่ออยู่ก็ไม่ต้องเก็บไว้) ข้อความเป็น JSON `{"type":"offer.new","data":{...}}` service ที่ต้องแจ้งผู้ใช้ publish ลง subject นี้เอง แล้ว gateway ที่ถือ WebSocket ของคนนั้นส่งต่อ

## Tech stack และ tools

ทุกตัวในตารางนี้ฟรีและ self-host ได้ เลือกให้รันบน home server เครื่องเดียวได้ตั้งแต่วันแรก ใช้เวอร์ชันเสถียรล่าสุด ณ วันที่เริ่ม

| หมวด | Tool | ใช้ทำอะไร | ทำไมเลือก / ทางเลือกอื่น |
| --- | --- | --- | --- |
| ภาษา | Go | ทุก service, simulator | ถนัดอยู่แล้ว, concurrency ดี, binary เล็ก |
| HTTP | `net/http` (routing แบบใหม่) หรือ chi | REST ใน gateway | เบา ไม่ผูก framework |
| WebSocket | coder/websocket | real-time ระหว่าง gateway กับ client | API เรียบง่าย รองรับ context |
| RPC ภายใน | gRPC + Protocol Buffers | service คุยกันแบบ sync | type-safe, มี streaming สำหรับพิกัด |
| จัดการ proto | Buf CLI | lint, ตรวจ breaking change, generate code | ดีกว่าใช้ protoc ตรง ๆ มาก |
| Message broker | NATS + JetStream | event ระหว่าง service, push หา user | เขียนด้วย Go, เบา, ตั้งง่าย · ทางเลือก: Kafka (หนักกว่า), RabbitMQ |
| Database | PostgreSQL + PostGIS | ข้อมูลถาวร, query เชิงพื้นที่ | มาตรฐาน, PostGIS เก่งเรื่องพิกัด |
| DB driver | pgx | เชื่อม Postgres | เร็วสุดใน Go |
| Query | sqlc | generate Go จาก SQL | เขียน SQL จริงแต่ type-safe ไม่ต้องใช้ ORM |
| Migration | goose | เวอร์ชัน schema | รันใน CI และตอน deploy ได้ |
| Cache / GEO | Redis (หรือ Valkey) | พิกัดคนขับ, lock, quote, rate limit | `GEOSEARCH` เร็วมาก · ทางเลือกภายหลัง: Uber H3 |
| Routing | OSRM + ข้อมูลกรุงเทพฯ ตัดจากไฟล์ประเทศไทยของ Geofabrik (OpenStreetMap) | ระยะทาง/เวลาจริงบนถนน, เส้นทางของ simulator | ฟรี เร็ว รันใน Docker ได้ |
| แผนที่หน้าเว็บ | MapLibre GL JS ผ่าน react-map-gl + PMTiles (Protomaps) | แสดงแผนที่และรถ | self-host tile ได้ ไม่ติด usage policy ของ tile server สาธารณะ |
| Frontend | Vite + React + TypeScript (React Router, TanStack Query, Zustand, Tailwind CSS, pnpm) | หน้าผู้โดยสาร คนขับ และ ops dashboard | ไม่ต้องใช้ SSR เพราะ backend อยู่ที่ gateway, build เป็นไฟล์ static เบากว่ารัน Node server |
| Auth | JWT (access สั้น + refresh ยาว), argon2id | ยืนยันตัวตน | เขียนเองเพื่อเรียนรู้ · ทางเลือก: Keycloak, Zitadel |
| Config | caarlos0/env | อ่าน env เข้า struct | เล็กและพอ |
| Logging | `log/slog` (JSON) | log ทุก service | อยู่ใน standard library |
| Telemetry | OpenTelemetry SDK + Collector | trace, metric, log | มาตรฐานกลาง เปลี่ยน backend ได้ |
| Metrics | Prometheus | เก็บ metric | มาตรฐานวงการ |
| Tracing | Grafana Tempo (หรือ Jaeger) | ดู request ข้าม service | เห็นว่าช้าที่ตรงไหน |
| Logs | Grafana Loki | รวม log | query คู่กับ trace ได้ |
| Dashboard | Grafana | dashboard + alert | รวมทุกอย่างไว้ที่เดียว |
| Test | Go `testing` + testcontainers-go | unit + integration กับ Postgres/Redis/NATS จริง | ไม่ต้อง mock database |
| Load test | k6 | ยิงโหลด REST/WebSocket | เขียน script ด้วย JS, ได้ตัวเลข p95 |
| Lint | golangci-lint, buf lint, ESLint | คุณภาพโค้ด | รันใน CI |
| Task runner | Task (Taskfile.yml) | คำสั่ง build/test/run ทุก repo เหมือนกัน | อ่านง่ายกว่า Makefile |
| Hot reload | air | รัน service ตอน dev | แก้โค้ดแล้ว restart เอง |
| Container | Docker, multi-stage build, distroless (service Go) และ Caddy เสิร์ฟเว็บ static | image ทุก service | image เล็ก ปลอดภัยกว่า |
| Registry | GitHub Container Registry (ghcr.io) | เก็บ image | ฟรี ผูกกับ org |
| CI/CD | GitHub Actions (reusable workflows) | test, build, scan, push, deploy | ตั้งที่เดียวใช้ทุก repo |
| Security scan | Trivy, govulncheck | สแกน image และ dependency | ฟรี ใส่ใน CI ง่าย |
| Dependency update | Renovate หรือ Dependabot | อัปเดต library อัตโนมัติ | ลดงานดูแล |
| Deploy ช่วงแรก | Docker Compose บน home server | รันทั้งระบบ | ง่ายที่สุด |
| Deploy ช่วงหลัง | k3s + Helm/Kustomize + Argo CD | Kubernetes + GitOps | ตรงสาย DevOps, ใช้ทรัพยากรน้อย |
| เปิดสู่ภายนอก | Cloudflare Tunnel | เปิด demo ให้คนนอกเข้า | ไม่ต้องเปิด port ที่บ้าน |
| Secrets | GitHub org secrets, SOPS + age | เก็บ secret ใน CI และใน repo deploy | เข้ารหัสก่อน commit |
| API docs | OpenAPI (REST), proto (gRPC), AsyncAPI (event) | เอกสาร API | สร้างหน้า docs อัตโนมัติได้ |
| Diagram / ADR | Mermaid, C4 model, ADR แบบ Markdown | เอกสาร architecture และเหตุผลการตัดสินใจ | อยู่ใน git ร่วมกับโค้ด |

## โครงสร้าง GitHub org และ repo

org `zova-platform` มี 18 repo แบ่งเป็น 5 กลุ่ม และเพิ่มอีก 4 repo ตอนทำ ZOVA Food ชื่อ repo ไม่ใส่ `zova-` นำหน้า เพราะชื่อ org บอกอยู่แล้ว (`github.com/zova-platform/trip-service` ไม่ใช่ `github.com/zova-platform/zova-trip`) ส่วนที่ออกไปนอก org มีชื่อ org ติดไปเอง เช่น image `ghcr.io/zova-platform/trip-service` และ Go module `github.com/zova-platform/proto` แนะนำให้ทุก repo เป็น public เพราะ branch protection/rulesets ใช้ฟรีกับ repo public และคนสัมภาษณ์เปิดดูได้

| Repo | กลุ่ม | เก็บอะไร | Phase |
| --- | --- | --- | --- |
| `.github` | org | profile README ของ org (โลโก้ ZOVA + ภาพรวม), template issue/PR | 0 |
| `docs` | org | architecture, ADR, runbook, branding, benchmarks | 0 |
| `proto` | shared | ไฟล์ `.proto` ทั้งหมด + Go code ที่ generate แล้ว ติด tag SemVer | 0 |
| `platform-go` | shared | library กลาง: slog, OpenTelemetry, NATS helper, outbox worker, gRPC interceptor, graceful shutdown | 0 |
| `ci-workflows` | shared | reusable GitHub Actions workflows | 0 |
| `deploy` | infra | `compose/` (local + home server), `k8s/`, config observability, secrets ที่เข้ารหัส | 0 |
| `gateway` | service | REST + WebSocket ทางเข้าเดียวของทุก client | 1 |
| `auth-service` | service | ผู้ใช้ คนขับ รถ | 1 |
| `location-service` | service | พิกัด สถานะคนขับ | 1 |
| `pricing-service` | service | ประเมินราคา (Car/Moto แยกสูตร) | 1 |
| `dispatch-service` | service | จับคู่งานกับคนขับ | 1 |
| `trip-service` | service | วงจรทริป | 1 |
| `passenger-web` | client | เว็บผู้โดยสาร (Vite + React) | 1 |
| `driver-web` | client | เว็บคนขับ (Vite + React, PWA ใช้ GPS มือถือได้) | 1 |
| `admin-web` | client | ops dashboard แผนที่รวม จัดการคนขับ | 2 |
| `simulator` | tooling | คนขับ/ผู้โดยสารจำลอง + script k6 ใน `loadtest/` | 2 |
| `payment-service` | service | ZOVA Wallet และการชำระเงิน | 4 |
| `notification-service` | service | แจ้งเตือนนอกแอป | 4 |
| `restaurant-service` | service | ร้านค้า เวลาเปิดปิด พื้นที่ส่ง | 5 |
| `menu-service` | service | เมนู ราคา ของหมด | 5 |
| `order-service` | service | ออเดอร์อาหาร state machine | 5 |
| `merchant-web` | client | เว็บร้านค้ารับออเดอร์ | 5 |

เว็บทั้งสามใช้ชุดสีและ component เดียวกัน ถ้าเริ่มก๊อปโค้ดกันเยอะ ให้แยก `ui-kit` (design token + component แผนที่) เป็น package กลาง

spec ของแต่ละ repo แยกไว้ในแท็บ รายละเอียดแต่ละ repo

`platform-go` ต้องเล็ก ใส่เฉพาะโค้ด infrastructure ที่ทุก service ใช้เหมือนกัน ห้ามใส่ business logic ไม่งั้นจะกลายเป็น distributed monolith

**โครงในแต่ละ service repo**

```text
trip-service/
├── cmd/trip-service/main.go   # ประกอบทุกอย่าง, อ่าน config, start server
├── internal/
│   ├── domain/                 # entity, state machine, กฎธุรกิจ (ไม่ import infra)
│   ├── app/                    # use case: RequestTrip, CancelTrip, ...
│   ├── grpc/                   # handler แปลง proto <-> domain
│   ├── events/                 # publisher / consumer ของ NATS
│   └── store/                  # Postgres (sqlc) และ Redis
├── migrations/                 # SQL migration
├── queries/                    # SQL สำหรับ sqlc
├── Dockerfile
├── Taskfile.yml
├── .golangci.yml
└── README.md                   # วิธีรัน, env ที่ต้องใช้, API, event ที่ส่ง/รับ
```

**Convention**

- Commit แบบ Conventional Commits (`feat:`, `fix:`, `chore:`) เพื่อ generate changelog อัตโนมัติ
- Trunk-based: branch สั้น ๆ → PR → CI ผ่าน → squash merge เข้า `main`
- Tag SemVer (`v0.3.1`) ต่อ repo, image tag = git SHA สั้น และ tag SemVer เมื่อ release
- Image ชื่อ `ghcr.io/zova-platform/<repo>`
- proto เปลี่ยนแบบ breaking ไม่ได้ ถ้าจำเป็นให้ขึ้น package ใหม่ (`trip.v2`) และ `buf breaking` ใน CI จะจับให้
- ตั้งค่า org: บังคับ 2FA, org secrets สำหรับ ghcr และ deploy, pin repo สำคัญที่หน้า org และ pin `docs` หรือ `gateway` ไว้ที่โปรไฟล์ส่วนตัวด้วย

## Local dev, CI/CD, deployment, observability

เป้าคือ clone repo `deploy` แล้วพิมพ์คำสั่งเดียวได้ระบบทั้งชุดบนเครื่อง และทุก merge เข้า `main` ไปถึง home server เองโดยไม่ต้อง SSH เข้าไปทำมือ

**Local dev**

1. clone ทุก repo ไว้โฟลเดอร์เดียวกัน (ทำ script `task clone-all` ใน `deploy`)
2. `task up` ใน `deploy` → Docker Compose เปิด Postgres, Redis, NATS, OSRM, OTel Collector, Prometheus, Tempo, Loki, Grafana
3. รัน service ที่กำลังแก้ด้วย `air` บนเครื่อง ส่วน service อื่นรันจาก image ใน compose
4. ถ้าต้องแก้ `proto` หรือ `platform-go` พร้อมกัน ใช้ `go.work` ชี้ไปโฟลเดอร์ local ชั่วคราว (อย่า commit `go.work`)

OSRM ต้องเตรียมข้อมูลครั้งแรก: ดาวน์โหลด `thailand-latest.osm.pbf` จาก Geofabrik ตัดเหลือกรอบกรุงเทพฯ ด้วย `osmium extract` แล้วรัน `osrm-extract`, `osrm-partition`, `osrm-customize` ทำเป็น task เดียวไว้ใน `deploy`

**CI/CD (GitHub Actions)**

`ci-workflows` เก็บ workflow กลาง 3 ตัว แต่ละ service repo มีไฟล์สั้น ๆ เรียกใช้:

- `go-ci.yml`: golangci-lint → `go test -race` (มี testcontainers) → govulncheck
- `docker-publish.yml`: build multi-stage → Trivy scan → push `ghcr.io/zova-platform/<repo>:<sha>`
- `deploy.yml`: แก้ image tag ใน repo `deploy` แล้ว commit (GitOps)

`proto` มี workflow ของตัวเอง: `buf lint` + `buf breaking` เทียบกับ `main` แล้ว generate และ tag

**Deployment**

- ช่วงแรก: home server pull repo `deploy` แล้ว `docker compose up -d` ทำได้สองแบบ คือ self-hosted GitHub runner บน server หรือ cron/webhook ที่ `git pull` เมื่อมี commit ใหม่
- ช่วงหลัง: ติดตั้ง k3s บน home server, ทำ Helm chart กลางตัวเดียวใช้กับทุก service, Argo CD sync จาก `deploy/k8s` อัตโนมัติ
- เปิด demo สู่ภายนอกด้วย Cloudflare Tunnel เฉพาะ `web` กับ `gateway` ส่วน Grafana ให้อยู่หลัง Cloudflare Access
- migration รันเป็น init container (k8s) หรือ one-shot service ใน compose ก่อน service หลักเริ่ม

**Observability**

ทุก service ส่ง trace, metric, log ผ่าน OpenTelemetry ไป Collector ตัวเดียว แล้ว Collector แจกต่อไป Tempo, Prometheus, Loki

metric ที่ควรทำ dashboard ตั้งแต่แรก:

| Metric | ดูอะไร |
| --- | --- |
| `dispatch_match_duration_seconds` (histogram) | ตั้งแต่ขอรถจนได้คนขับ ใช้เวลาเท่าไร |
| `dispatch_offers_total{result}` | สัดส่วนรับ/ปฏิเสธ/หมดเวลา |
| `dispatch_unfulfilled_total` | หาคนขับไม่ได้กี่ครั้ง |
| `location_updates_total` | รับพิกัดได้กี่ ping ต่อวินาที |
| `drivers_online` (gauge) | จำนวนคนขับออนไลน์ แยกตามสถานะ |
| `trips_by_status` (gauge) | ทริปค้างอยู่สถานะไหนบ้าง |
| `gateway_ws_connections` (gauge) | จำนวน WebSocket ที่เปิดอยู่ |
| RED ของทุก gRPC/HTTP (rate, errors, duration) | สุขภาพของแต่ละ service |
| `outbox_pending` (gauge) | event ค้างยังไม่ได้ publish |

alert เริ่มต้น: error rate เกิน 5% นาน 5 นาที, `outbox_pending` เพิ่มต่อเนื่อง, service ไม่ตอบ `/readyz`

## Simulator และการทดสอบ

simulator คือสิ่งที่ทำให้ demo มีชีวิต: รัน 1 process ปล่อยคนขับปลอมได้หลายร้อยคัน แต่ละคันเป็น goroutine ที่ทำตัวเหมือนคนขับจริงผ่าน API ปกติ ไม่มีทางลัดเข้า database

**พฤติกรรมคนขับจำลอง**

1. สุ่มจุดเริ่มในกรอบพื้นที่ (เช่น กรุงเทพฯ ชั้นใน) ล็อกอินแล้วเปิดสถานะออนไลน์
2. ไม่มีงาน: ขอเส้นทางจาก OSRM ไปจุดสุ่ม แล้วขยับตามเส้นทางนั้น ส่งพิกัดทุก 3 วินาที
3. ได้รับข้อเสนองาน: รับตามความน่าจะเป็นที่ตั้งไว้ (เช่น 80%) หน่วงเวลาตอบแบบสุ่ม หรือไม่ตอบเลยเพื่อทดสอบ timeout
4. รับงาน: วิ่งไปจุดรับ → กด "ถึงแล้ว" → วิ่งไปจุดส่ง → กดจบทริป → กลับไปข้อ 2

ผู้โดยสารจำลองขอรถตามอัตราที่ตั้ง (เช่น 2 ทริปต่อวินาที) จากจุดสุ่ม ทุกค่าตั้งผ่าน config: จำนวนคนขับ, อัตราขอรถ, อัตรารับงาน, ความเร็วการขับ (เร่งเวลาได้เพื่อ demo)

**ระดับการทดสอบ**

| ระดับ | ทดสอบอะไร | เครื่องมือ |
| --- | --- | --- |
| Unit | state machine ของทริป, สูตรราคา, ลำดับการเลือกคนขับ | Go `testing` (table-driven) |
| Integration | store กับ Postgres/Redis จริง, publish/consume กับ NATS จริง | testcontainers-go |
| Contract | proto ไม่ breaking, รูปแบบ event ตรงกับ schema | `buf breaking`, JSON Schema ของ event |
| End-to-end | ขอรถ → match → จบทริป ผ่าน gateway ทั้งเส้น | Go test ที่ยิงเข้า compose ทั้งชุด, รันใน CI ทุกคืน |
| Concurrency | คนขับคนเดียวต้องไม่ได้สองงาน, ทริปเดียวต้องไม่ได้สองคนขับ | `go test -race` + test ที่ยิงพร้อมกันหลาย goroutine |
| Load | ระบบรับได้แค่ไหน | k6 + simulator |
| Chaos | ระบบฟื้นเองได้ไหม | `docker kill` service ระหว่าง load แล้วดู dashboard |

**Load test scenario ที่ควรเก็บตัวเลข**

- พิกัด: คนขับ 1,000 คัน ส่งทุก 3 วินาที → ดู CPU ของ location-service และ latency ของ `NearbyDrivers`
- ขอรถพร้อมกันในจุดเดียว 200 คน (จำลองคอนเสิร์ตเลิก) → ต้องไม่มีคนขับถูกจับคู่ซ้ำ ดู p95 ของ match
- WebSocket ค้างพร้อมกัน 2,000 connection ที่ gateway

เก็บผลลัพธ์ (ฮาร์ดแวร์ที่ใช้, p50/p95/p99, throughput, จุดคอขวด) ลง `docs/benchmarks.md` นี่คือของที่เอาไปเล่าตอนสัมภาษณ์ได้ดีที่สุด

## Roadmap

แบ่งเป็น 6 phase ทำทีละ phase และจะข้ามไป phase ถัดไปก็ต่อเมื่อผ่านเกณฑ์ "เสร็จเมื่อ" แล้ว Phase 1 คือเป้าหมายแรกที่สำคัญที่สุด

&#91;embedded content: roadmap · 6 phase พร้อมเกณฑ์เสร็จ\]

แต่ละ phase จบด้วยสิ่งที่ demo ได้จริง ไม่ใช่แค่เขียนโค้ดเสร็จ หยุดที่ phase ไหนก็ยังมี portfolio ที่ใช้งานได้

**Phase 0: Foundation**

- [ ] สร้าง org, เปิด 2FA, สร้าง repo `.github`, `docs`, `proto`, `platform-go`, `ci-workflows`, `deploy`
- [ ] เขียน ADR แรก: เลือก polyrepo, gRPC + NATS, Postgres + Redis
- [ ] `deploy/compose`: Postgres, Redis, NATS, OSRM (ข้อมูลกรุงเทพฯ), OTel Collector, Prometheus, Tempo, Loki, Grafana
- [ ] `platform-go`: slog, OpenTelemetry, gRPC interceptor, graceful shutdown, health check
- [ ] reusable workflow `go-ci.yml` และ `docker-publish.yml`
- [ ] service ตัวอย่าง (`hello`) ผ่าน CI, push ขึ้น ghcr และเห็น trace ใน Grafana

**Phase 1: Ride MVP**

- [ ] `proto`: นิยาม API ของ auth, location, pricing, dispatch, trip
- [ ] `auth-service`: สมัคร, ล็อกอิน, JWT, บทบาท
- [ ] `location-service`: รับพิกัด, Redis GEO, `NearbyDrivers`
- [ ] `pricing-service`: เรียก OSRM, สูตรราคา, quote มี TTL
- [ ] `trip-service`: state machine + outbox + test ทุกการเปลี่ยนสถานะ
- [ ] `dispatch-service`: เสนองานทีละคน, timeout, lock กันจับคู่ซ้อน + test แบบยิงพร้อมกัน
- [ ] `gateway`: REST, WebSocket, push event หา user
- [ ] `passenger-web และ driver-web: หน้าเรียกรถและรับงานบนแผนที่ ใช้ชุดสี ZOVA`
- [ ] e2e test: ขอรถ → จบทริป ผ่าน gateway

ขั้นตอนละเอียดของทุก phase พร้อม commit ทีละตัวอยู่ในแท็บ แผนงานทีละขั้น

## การขยายไป ZOVA Food

เพิ่ม 3 service ใหม่ และใช้ของเดิมซ้ำได้ทั้งหมด เพราะ dispatch รู้จักแค่ "งาน" ไม่ผูกกับผู้โดยสาร

|  | Repo |
| --- | --- |
| ใช้ซ้ำ | `gateway`, `auth-service` (เพิ่มบทบาท `merchant`), `location-service`, `pricing-service` (เพิ่มสูตรค่าส่ง), `dispatch-service`, `payment-service`, `notification-service`, `simulator` |
| เพิ่มใหม่ | `restaurant-service` (ร้าน เวลาเปิดปิด พื้นที่ส่ง), `menu-service` (เมนู ราคา ของหมด), `order-service` (ตะกร้า ออเดอร์ state machine) |
| หน้าเว็บ | `merchant-web` ให้ร้านรับออเดอร์และกดว่าอาหารใกล้เสร็จ, เพิ่มหน้าสั่งอาหารใน `passenger-web` |

**สถานะออเดอร์:** `placed → accepted_by_merchant → preparing → ready_for_pickup → picked_up → delivered` และยกเลิกได้ก่อน `picked_up`

**จังหวะการเรียกคนขับ:** `order-service` ส่ง `dispatch.job.requested` แบบ `job_type=delivery` เมื่อร้านกดว่าจะเสร็จใน X นาที ไม่ใช่ตอนลูกค้ากดสั่ง เพื่อไม่ให้คนขับไปรอนานที่ร้าน

**สิ่งที่ต่างจาก ride:** มีสามฝ่าย (ลูกค้า ร้าน คนขับ) และจุดรับเป็นร้าน ส่วนคนขับเห็นเหมือนเดิม คือไปรับที่ A ไปส่งที่ B

ต่อยอดได้อีก: งานส่งพัสดุ (`job_type=parcel`), รวมหลายออเดอร์ในเที่ยวเดียว (batching), surge pricing ใน pricing-service

## ความเสี่ยง ข้อควรระวัง และแหล่งเรียนรู้

ความเสี่ยงใหญ่สุดคือทำ infrastructure จนไม่มีแรงเหลือทำ flow หลักให้จบ ให้ทำ flow ขอรถถึงจบทริปให้วิ่งได้ทั้งเส้นก่อน แล้วค่อยขัดเกลา

| ความเสี่ยง | ป้องกันอย่างไร |
| --- | --- |
| ขอบเขตบาน (เพิ่มฟีเจอร์ก่อน MVP จบ) | ยึด roadmap, ฟีเจอร์ใหม่ลง issue ไว้ก่อน |
| แตก service ละเอียดเกินจนจัดการไม่ไหวคนเดียว | ถ้าหนักเกิน รวม pricing เข้า trip ชั่วคราวได้ แยกออกทีหลัง |
| distributed monolith (service ต้อง deploy พร้อมกันตลอด) | สื่อสารผ่าน proto/event ที่มีเวอร์ชัน, `platform-go` ไม่มี business logic |
| event หาย หรือได้ซ้ำ | outbox + JetStream ack + consumer idempotent |
| คนขับโดนจับคู่ซ้อน | Redis `SET NX` + TTL และเช็ค version ของทริปตอนเขียน |
| home server ทรัพยากรไม่พอ (OSRM ใช้ RAM เยอะ) | ใช้ข้อมูลเฉพาะกรุงเทพฯ (ทั้งประเทศไทยสร้างกราฟไม่ผ่านที่ RAM 7 GB), ตั้ง resource limit, ปิด service ที่ไม่ใช้ตอน dev |
| ใช้ tile แผนที่สาธารณะเกินนโยบาย | self-host PMTiles หรือใช้ผู้ให้บริการที่มี free tier |
| เปิด demo สู่ภายนอกแล้วโดนสแปม | rate limit ใน gateway, ให้สมัครด้วย invite code |

**แหล่งเรียนรู้**

- หนังสือ *Designing Data-Intensive Applications* (Martin Kleppmann): ความสอดคล้องของข้อมูล, event, replication
- หนังสือ *Building Microservices* (Sam Newman): แบ่ง service, การสื่อสาร, deployment
- Uber Engineering Blog: เรื่อง dispatch, H3 และระบบ real-time ของ Uber
- เอกสารทางการของ NATS JetStream, OSRM, OpenTelemetry Go, Buf, sqlc, k6
- microservices.io (Chris Richardson): pattern เช่น outbox, saga, API gateway
