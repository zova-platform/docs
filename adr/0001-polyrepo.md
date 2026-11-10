# 0001. Polyrepo ใน GitHub org

- สถานะ: Accepted
- วันที่: 2026-11-10

## Context

ZOVA เป็นระบบ microservices ที่มี service Go หลายตัว เว็บ 3–4 ตัว และ repo ฝั่ง infra
ต้องเลือกว่าจะเก็บโค้ดทั้งหมดใน repo เดียว (monorepo) หรือแยก repo ต่อส่วน (polyrepo)

ข้อเท็จจริงที่มีผล:

- ทำคนเดียว monorepo จะง่ายกว่าเรื่องแก้หลาย service พร้อมกัน
- เป้าหมายของโปรเจกต์คือฝึกและโชว์งานแบบที่ทีมจริงทำ: แต่ละ service มี lifecycle, version และ CI ของตัวเอง
- repo public ใน org แผน Free ใช้ Actions ได้ไม่จำกัด และตั้ง ruleset ได้ต่อ repo

## Decision

ใช้ polyrepo ใน org `zova-platform`

- 1 service หรือ 1 เว็บ = 1 repo, ชื่อ repo ไม่ใส่ `zova-` นำหน้า
- ของที่ใช้ร่วมกันแยกเป็น repo ที่มี version: `proto`, `platform-go`, `ci-workflows`
- สร้าง service ใหม่จาก `service-template` เสมอ
- ทุก repo เป็น public

## Consequences

ข้อดี

- แต่ละ service build, test, release และ deploy แยกกันได้จริง
- ถูกบังคับให้คิดเรื่อง contract (proto) และ versioning ตั้งแต่ต้น
- CI ของแต่ละ repo เล็กและเร็ว

ข้อเสียและวิธีรับมือ

- แก้ proto หรือ platform-go แล้วต้องไล่อัปเดตทุก repo → ใช้ tag SemVer และ Renovate ช่วยเปิด PR
- แก้หลาย repo พร้อมกันตอน dev ยุ่ง → ใช้ `go.work` ในเครื่อง (ไม่ commit)
- node_modules ซ้ำหลาย repo → ใช้ pnpm
- เสี่ยงกลายเป็น distributed monolith → ห้ามใส่ business logic ใน `platform-go` และสื่อสารผ่าน proto/event ที่มี version เท่านั้น

## Alternatives

- **Monorepo + go.work:** ง่ายกว่าสำหรับคนเดียว แต่ไม่ได้ฝึกเรื่อง versioning และ independent deploy ซึ่งเป็นเป้าหมายหลัก
