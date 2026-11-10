# 0002. gRPC สำหรับ sync และ NATS JetStream สำหรับ event

- สถานะ: Accepted
- วันที่: 2026-11-10

## Context

service ของ ZOVA ต้องคุยกันสองแบบ

- **ต้องได้คำตอบทันที:** ขอราคา, คนขับกดรับงาน, dispatch ถามหาคนขับใกล้สุด
- **บอกว่ามีอะไรเกิดขึ้นแล้ว และผู้รับอาจมีหลายตัว:** ทริปเปลี่ยนสถานะ, งานถูก assign, ทริปจบ

นอกจากนี้ต้อง push ข้อมูลกลับหาผู้ใช้แบบ real-time ผ่าน WebSocket ของ gateway
และต้องรันทั้งหมดบน home server เครื่องเดียวได้

## Decision

- **การเรียกแบบ sync ใช้ gRPC + Protocol Buffers** สัญญาอยู่ใน repo `proto` จัดการด้วย Buf
- **Event ระหว่าง service ใช้ NATS JetStream** subject รูปแบบ `<domain>.<entity>.<event>` เช่น `ride.trip.status_changed`
- **Push หาผู้ใช้ใช้ NATS core** subject `push.user.<user_id>` ไม่เก็บถาวร เพราะถ้าผู้ใช้ไม่ได้ต่ออยู่ก็ไม่ต้องส่ง
- Event ทุกตัวมี `event_id`, `occurred_at`, `version` และ consumer ต้อง idempotent
- Service ที่ส่ง event ใช้ outbox pattern

## Consequences

ข้อดี

- gRPC ได้ type-safety, streaming (ใช้กับพิกัดคนขับ) และตรวจ breaking change ได้ใน CI
- NATS เขียนด้วย Go, ตัวเดียวเล็ก, ตั้งง่าย, ใช้ RAM น้อย เหมาะกับ home server
- ใช้ NATS ตัวเดียวได้ทั้ง event ถาวร (JetStream) และ push ชั่วคราว (core)

ข้อเสียและวิธีรับมือ

- debug ข้าม service ยาก → OpenTelemetry trace ส่ง context ผ่านทั้ง gRPC metadata และ NATS header
- event อาจซ้ำหรือมาไม่เรียง → dedupe ด้วย `event_id` และใช้ `version` ของ entity ตัดสิน
- browser เรียก gRPC ตรงไม่ได้ → gateway แปลง REST/WebSocket เป็น gRPC

## Alternatives

- **REST ทุกที่:** ง่ายกว่าแต่ไม่มี contract ที่ตรวจได้ และไม่มี streaming ที่ดี
- **Kafka:** มาตรฐานวงการ แต่หนักเกินไปสำหรับเครื่องเดียว
- **RabbitMQ:** ใช้ได้ดี แต่ NATS เบากว่าและใช้ทำ push ได้ในตัวเดียวกัน
