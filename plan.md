# แผนงานทีละขั้น

ทำตามลำดับจากบนลงล่าง แต่ละขั้นคือ 1 branch + 1 PR และมี commit ที่ต้องทำเป็น checklist ติ๊กได้เมื่อ push แล้ว จบ phase แล้วต้องผ่านเกณฑ์ท้าย phase ก่อนไปต่อ

## กติกา commit และ branch

1 commit = 1 เรื่องที่ build และ test ผ่านด้วยตัวเอง ถ้ากลับไป checkout commit ไหนก็ต้องรันได้

- **รูปแบบ:** Conventional Commits `type(scope): สิ่งที่ทำ` เขียนเป็นภาษาอังกฤษ ขึ้นต้นด้วยกริยา ไม่เกิน 72 ตัวอักษร
- **type ที่ใช้:** `feat` ฟีเจอร์ใหม่ · `fix` แก้บั๊ก · `test` เพิ่ม test · `refactor` ปรับโค้ดไม่เปลี่ยนพฤติกรรม · `chore` งานตั้งค่า · `ci` workflow · `build` Dockerfile/build · `docs` เอกสาร
- **scope:** ส่วนของ repo เช่น `domain`, `db`, `grpc`, `ws`, `compose`
- **branch:** `<type>/<ขั้น>-<ชื่อสั้น>` เช่น `feat/1.6-trip-state-machine`
- **PR:** 1 ขั้น = 1 PR ชื่อ PR ใช้ commit หลักของขั้นนั้น merge แบบ squash หรือ rebase ก็ได้ แต่เลือกแบบเดียวทั้งโปรเจกต์ (ถ้าอยากให้ประวัติ commit ละเอียดตามแผนนี้ ใช้ rebase merge)
- **tag:** repo ที่คนอื่น import (`proto`, `platform-go`, `ci-workflows`) ติด tag SemVer เมื่อจบขั้น

เวลาใช้ Claude Code ให้บอกเลขขั้น เช่น "ทำขั้น 1.6 ตาม docs/plan.md ทีละ commit" แล้วตรวจ diff ทุก commit ก่อน push

## Phase 0: Foundation

ปูพื้นทุกอย่างที่ service ต้องใช้ จบ phase เมื่อ service ตัวอย่างจาก template ผ่าน CI, push image ขึ้น ghcr และเห็น trace ใน Grafana

### 0.1 `docs`: บันทึกเอกสารและการตัดสินใจ

- [ ] `docs: add project overview, repo specs and plan`
- [ ] `docs(adr): record 0001 polyrepo in GitHub org`
- [ ] `docs(adr): record 0002 gRPC for sync and NATS JetStream for events`
- [ ] `docs(adr): record 0003 database per service with Postgres and Redis`
- [ ] `docs(branding): add ZOVA logo and color palette`

### 0.2 `.github`: หน้า org

- [ ] `docs: add org profile README with logo and overview`
- [ ] `chore: add bug and feature issue templates`
- [ ] `chore: add pull request template`

### 0.3 `ci-workflows`: workflow กลาง

- [ ] `ci: add reusable go-ci workflow (lint, race test, govulncheck)`
- [ ] `ci: add reusable docker-publish workflow (buildx, trivy, ghcr)`
- [ ] `ci: add reusable web-ci workflow (pnpm lint, typecheck, build)`
- [ ] `ci: lint workflows with actionlint`
- [ ] `docs: add usage examples to README`
- [ ] ติด tag `v1.0.0` และ tag `v1` ที่เลื่อนตามได้

### 0.4 `proto`: ตั้งค่า Buf

- [ ] `chore: init buf module with lint and breaking rules`
- [ ] `feat(common): add LatLng, Money and Pagination types`
- [ ] `ci: run buf lint, buf breaking and generate check`
- [ ] ติด tag `v0.1.0`

### 0.5 `platform-go`: library กลางชุดแรก

- [ ] `chore: init module, golangci config and CI`
- [ ] `feat(config): load environment variables into struct`
- [ ] `feat(logx): add JSON slog logger with trace_id`
- [ ] `feat(otelx): set up tracer and meter providers over OTLP`
- [ ] `feat(health): serve healthz, readyz and metrics endpoints`
- [ ] `feat(grpcx): add server and client with log, trace, recover and timeout interceptors`
- [ ] `feat(run): add graceful shutdown lifecycle`
- [ ] ติด tag `v0.1.0`

### 0.6 `deploy`: infra บนเครื่อง

- [ ] `chore: add Taskfile with up, down and clone-all`
- [ ] `feat(compose): add postgres with postgis, redis and nats jetstream`
- [ ] `feat(observability): add otel collector, prometheus, tempo, loki and grafana`
- [ ] `feat(observability): provision grafana datasources`
- [ ] `feat(osrm): add thailand data prep task and osrm service`
- [ ] `docs: explain local setup in README`

### 0.7 `service-template`: แม่แบบ service

- [ ] `chore: add Go project layout and Taskfile`
- [ ] `feat: wire config, logging, tracing, health and gRPC server`
- [ ] `build: add multi-stage distroless Dockerfile`
- [ ] `ci: call go-ci and docker-publish from ci-workflows`
- [ ] `docs: add README and CLAUDE.md templates`

### 0.8 ตรวจรับ Phase 0

- [ ] `feat(compose): run service-template as smoke service` (ใน `deploy`)
- [ ] เปิด Grafana เห็น trace และ metric ของ smoke service
- [ ] ตั้ง ruleset ของ `main` ให้ทุก repo ที่มี CI แล้ว
- [ ] `chore(compose): remove smoke service`

## Phase 1: Ride MVP

ทำ backend จากล่างขึ้นบน (สัญญา → service ที่ไม่พึ่งใคร → service ที่พึ่งคนอื่น → gateway → เว็บ) จบ phase เมื่อขอรถจนจบทริปได้ครบเส้นบนแผนที่ กับคนขับจำลอง 1 คน

### 1.1 `proto`: นิยาม API ทุก service

- [ ] `feat(auth): define AuthService v1`
- [ ] `feat(location): define LocationService v1`
- [ ] `feat(pricing): define PricingService v1`
- [ ] `feat(dispatch): define DispatchService v1 and job event messages`
- [ ] `feat(trip): define TripService v1 and trip event messages`
- [ ] ติด tag `v0.2.0`

### 1.2 `platform-go`: event และ outbox

- [ ] `feat(natsx): connect JetStream and publish with trace context`
- [ ] `feat(natsx): add durable consumer with event_id dedupe`
- [ ] `feat(natsx): add push helper for push.user subjects`
- [ ] `feat(outbox): add outbox table migration and publisher worker`
- [ ] ติด tag `v0.2.0`

### 1.3 `auth-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add users, drivers, vehicles and refresh_tokens migrations`
- [ ] `feat: register and login with argon2id`
- [ ] `feat: issue EdDSA JWT and rotate refresh tokens`
- [ ] `feat(grpc): add GetUser, GetDriverProfile and UpsertVehicle`
- [ ] `feat: add seed command for test accounts`
- [ ] `test: cover auth flows with testcontainers`

### 1.4 `location-service`

- [ ] `chore: create from service-template`
- [ ] `feat: track driver status in Redis with TTL`
- [ ] `feat(grpc): stream driver locations into GEO sets`
- [ ] `feat(grpc): find nearby available drivers with GEOSEARCH`
- [ ] `feat: remove stale drivers from GEO sets`
- [ ] `feat(events): publish throttled moves for drivers on a job`
- [ ] `test: cover TTL expiry and busy filtering`

### 1.5 `pricing-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add fare_rules with car and moto seed`
- [ ] `feat: add OSRM route client with timeout`
- [ ] `feat(grpc): create and get quotes stored in Redis`
- [ ] `feat(grpc): calculate final fare`
- [ ] `test: table-driven fare formula tests`

### 1.6 `trip-service`

- [ ] `chore: create from service-template`
- [ ] `feat(domain): add trip state machine and transition rules`
- [ ] `test(domain): cover every allowed and rejected transition`
- [ ] `feat(db): add trips, trip_events, idempotency_keys and outbox`
- [ ] `feat(grpc): request trip after quote check and emit job request`
- [ ] `feat(events): apply job assigned and unfulfilled events`
- [ ] `feat(grpc): add arrived, start and complete with final fare`
- [ ] `feat(grpc): cancel trip and re-dispatch on driver cancel`
- [ ] `feat(events): push trip updates to passenger and driver`
- [ ] `test: concurrent transitions with optimistic locking`

### 1.7 `dispatch-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add jobs, job_offers and outbox`
- [ ] `feat(events): consume job requests and fetch candidates`
- [ ] `feat: lock drivers with Redis SET NX`
- [ ] `feat: offer jobs one by one with timeout`
- [ ] `feat(grpc): accept and reject offers`
- [ ] `feat: mark jobs unfulfilled after timeout and support cancel`
- [ ] `feat: recover pending offers after restart`
- [ ] `feat: export match and offer metrics`
- [ ] `test: 200 concurrent jobs never double-assign a driver`

### 1.8 `gateway`

- [ ] `chore: create from service-template`
- [ ] `feat(http): add router, JWT middleware and error mapping`
- [ ] `feat(http): add auth routes`
- [ ] `feat(http): add quote and trip routes`
- [ ] `feat(http): add driver status and offer routes`
- [ ] `feat(ws): add authenticated WebSocket hub`
- [ ] `feat(ws): forward driver locations to location-service`
- [ ] `feat(ws): deliver push.user events to connected clients`
- [ ] `feat(http): add per-user rate limit`

### 1.9 `deploy`: รวมระบบ

- [ ] `feat(compose): create a database per service on startup`
- [ ] `feat(compose): add phase 1 services with env files`

### 1.10 `passenger-web`

- [ ] `chore: scaffold Vite React TypeScript with Tailwind ZOVA tokens`
- [ ] `feat(api): add REST and reconnecting WebSocket clients`
- [ ] `feat: add login and register pages`
- [ ] `feat(map): add MapLibre map with OSM attribution`
- [ ] `feat: pick pickup and dropoff and show quote`
- [ ] `feat: request trip and follow it live`
- [ ] `feat: add trip history`
- [ ] `build: serve static build with Caddy` + `ci: call web-ci and docker-publish`

### 1.11 `driver-web`

- [ ] `chore: scaffold Vite React TypeScript with shared ZOVA tokens`
- [ ] `feat: go online and send GPS every 3 seconds`
- [ ] `feat: show offer popup with countdown`
- [ ] `feat: add job screen with arrived, start and complete`
- [ ] `feat: add PWA manifest and service worker`
- [ ] `build: serve static build with Caddy` + `ci: call web-ci and docker-publish`

### 1.12 ตรวจรับ Phase 1

- [ ] `test(e2e): request to complete trip through gateway` (ใน `deploy/e2e`)
- [ ] `ci: run e2e nightly against compose`
- [ ] ลองเองบนมือถือ: เรียกรถจาก passenger-web, รับงานจาก driver-web, จบทริป

## Phase 2: Real-time + Simulator

ทำให้ระบบมีชีวิตและวัดผลได้ จบ phase เมื่อคนขับจำลอง 500 คันวิ่งพร้อมกัน และมีตัวเลข p95 บันทึกใน `docs/benchmarks.md`

### 2.1 `simulator`

- [ ] `chore: init Go CLI with config file`
- [ ] `feat: seed riders and drivers through gateway`
- [ ] `feat(driver): add driver agent state machine`
- [ ] `feat(driver): follow OSRM routes at configurable speed`
- [ ] `feat(driver): respond to offers with accept rate and delay`
- [ ] `feat(rider): request trips at configurable rate`
- [ ] `feat(scenario): add concert-exit scenario`
- [ ] `feat: export simulator metrics and run summary`

### 2.2 แผนที่รวมสำหรับ ops

- [ ] `feat(events): publish periodic snapshot of all drivers` (ใน `location-service`)
- [ ] `feat(ws): add admin channel for driver snapshots` (ใน `gateway`)

### 2.3 `admin-web`

- [ ] `chore: scaffold Vite React TypeScript with shared ZOVA tokens`
- [ ] `feat(map): show every driver colored by status`
- [ ] `feat: search trips and show status history`
- [ ] `feat: list, approve and suspend drivers`
- [ ] `feat: embed Grafana dashboards`
- [ ] `build: serve with Caddy` + `ci: call web-ci and docker-publish`

### 2.4 ความลื่นของแผนที่

- [ ] `feat(map): interpolate vehicle markers between updates` (ทั้ง passenger-web และ driver-web)

### 2.5 Dashboard และ load test

- [ ] `feat(observability): add ZOVA overview dashboard` (ใน `deploy`)
- [ ] `feat(observability): add dispatch and location dashboards`
- [ ] `test(load): add k6 scripts for REST and WebSocket` (ใน `simulator/loadtest`)

### 2.6 ตรวจรับ Phase 2

- [ ] รัน 500 คัน + concert-exit แล้วจดผล
- [ ] `docs: record phase 2 benchmark results` (ใน `docs`)

## Phase 3: Reliability และ deploy อัตโนมัติ

ทำให้ระบบทนต่อความผิดพลาดและขึ้น home server เองทุกครั้งที่ merge จบ phase เมื่อ kill service ระหว่างโหลดแล้วระบบกลับมาเอง ไม่มีทริปค้าง

### 3.1 `platform-go`: ความทนทาน

- [ ] `feat(grpcx): add retry with backoff for idempotent calls`
- [ ] `feat(grpcx): add default deadlines per call`
- [ ] `feat(natsx): add retry and dead-letter subject for failed events`
- [ ] ติด tag `v0.3.0` แล้ว `chore(deps): bump platform-go to v0.3.0` ในทุก service

### 3.2 `gateway` และ `trip-service`: กันกดซ้ำ

- [ ] `feat(http): require Idempotency-Key on state-changing routes` (gateway)
- [ ] `feat(grpc): honor idempotency keys on all transitions` (trip)

### 3.3 ตรวจ graceful shutdown ทุก service

- [ ] `test: drain in-flight requests on SIGTERM` (ทำซ้ำในแต่ละ service)
- [ ] `fix: ...` ตามที่เจอ

### 3.4 `deploy`: chaos test

- [ ] `feat(chaos): add task to kill a random service under load`
- [ ] `docs(runbooks): record recovery behaviour per service` (ใน `docs`)

### 3.5 Deploy อัตโนมัติขึ้น home server

- [ ] `ci: add deploy-bump workflow that updates image tags` (ใน `ci-workflows`, tag `v1.1.0`)
- [ ] `ci: bump deploy repo after image publish` (ทุก service)
- [ ] `feat(server): add systemd timer that pulls deploy and runs compose` (ใน `deploy`)
- [ ] `feat(compose): add server env with resource limits and retention`

### 3.6 Alert

- [ ] `feat(observability): alert on error rate, outbox backlog and readiness` (ใน `deploy`)

### 3.7 ตรวจรับ Phase 3

- [ ] merge PR แล้วระบบบน server อัปเดตเองภายในไม่กี่นาที
- [ ] รัน chaos ระหว่าง simulator ไม่มีทริปค้างสถานะกลางทาง
- [ ] `docs: record phase 3 reliability results`

## Phase 4: Payment + Kubernetes

เพิ่มเรื่องเงินและย้ายไป Kubernetes จบ phase เมื่อคนนอกเข้า demo ได้ทางอินเทอร์เน็ต และ deploy ด้วยการ merge อย่างเดียว

### 4.1 `proto`

- [ ] `feat(payment): define PaymentService v1 and payment events`
- [ ] ติด tag `v0.3.0`

### 4.2 `payment-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add wallets, ledger_entries, payments and outbox`
- [ ] `feat(domain): post balanced double-entry ledger records`
- [ ] `feat(events): charge rider and pay driver on trip completed`
- [ ] `feat(grpc): get wallet, top up and get payment`
- [ ] `test: ledger always sums to zero and duplicates are ignored`

### 4.3 `notification-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add notification_logs and user_preferences`
- [ ] `feat: send email through SMTP (Mailpit locally)`
- [ ] `feat(events): notify on trip status, receipt and welcome`
- [ ] `feat: retry with backoff without blocking the consumer`

### 4.4 ต่อเข้าระบบเดิม

- [ ] `feat(http): add wallet routes` (gateway)
- [ ] `feat: show wallet and receipts` (passenger-web)
- [ ] `feat: show earnings` (driver-web)
- [ ] `feat(compose): add payment, notification and Mailpit` (deploy)

### 4.5 Kubernetes (`deploy/k8s`)

- [ ] `docs(adr): record 0004 k3s with Argo CD` (ใน `docs`)
- [ ] `feat(k8s): add shared Helm chart for Go services`
- [ ] `feat(k8s): add values per service`
- [ ] `feat(k8s): add infra (postgres, redis, nats, osrm, observability)`
- [ ] `feat(k8s): add Argo CD applications`
- [ ] `ci: bump image tags in k8s values` (ปรับ deploy-bump ใน `ci-workflows`)

### 4.6 เปิดสู่ภายนอก

- [ ] `feat(k8s): expose web apps and gateway through Cloudflare Tunnel`
- [ ] `feat(auth): require invite code to register` (กันสแปม)

### 4.7 ตรวจรับ Phase 4

- [ ] เพื่อนเข้าลิงก์ demo จากมือถือ เรียกรถกับคนขับจำลองได้
- [ ] ทริปจบแล้วเงินเข้า wallet คนขับ และได้อีเมลใบเสร็จ
- [ ] merge PR แล้ว Argo CD sync เอง

## Phase 5: ZOVA Food

เพิ่มส่งอาหารโดยไม่แก้ dispatch จบ phase เมื่อสั่งอาหาร → ร้านรับ → คนขับจำลองไปรับและส่งถึงลูกค้าได้ครบเส้น

### 5.1 `proto`

- [ ] `feat(restaurant): define RestaurantService v1`
- [ ] `feat(menu): define MenuService v1`
- [ ] `feat(order): define OrderService v1 and order events`
- [ ] ติด tag `v0.4.0`

### 5.2 `restaurant-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add restaurants with location and opening hours`
- [ ] `feat(grpc): get, list nearby and open or close restaurant`

### 5.3 `menu-service`

- [ ] `chore: create from service-template`
- [ ] `feat(db): add categories, items and options`
- [ ] `feat(grpc): get menu, upsert items and set availability`

### 5.4 `order-service`

- [ ] `chore: create from service-template`
- [ ] `feat(domain): add order state machine`
- [ ] `test(domain): cover every order transition`
- [ ] `feat(db): add orders, order_items, order_events and outbox`
- [ ] `feat(grpc): place, accept, mark ready and cancel orders`
- [ ] `feat(events): request delivery job when merchant marks ready soon`
- [ ] `feat(events): track courier pickup and delivery`

### 5.5 ต่อเข้าระบบเดิม

- [ ] `feat: add merchant role` (auth-service)
- [ ] `feat: add delivery fare rules` (pricing-service)
- [ ] `feat(http): add restaurant, menu and order routes` (gateway)
- [ ] `feat: add food ordering flow` (passenger-web)
- [ ] `feat: show delivery jobs with restaurant pickup` (driver-web)
- [ ] `feat(compose,k8s): add food services` (deploy)

### 5.6 `merchant-web`

- [ ] `chore: scaffold Vite React TypeScript with shared ZOVA tokens`
- [ ] `feat: receive orders with sound alert`
- [ ] `feat: accept, set prep time and mark ready`
- [ ] `feat: manage menu and availability`

### 5.7 ตรวจรับ Phase 5

- [ ] `feat(scenario): add lunch-rush food scenario` (simulator)
- [ ] ยืนยันว่า `dispatch-service` ไม่มี commit แก้ใน phase นี้เลย
- [ ] `docs: record phase 5 results and lessons learned`
