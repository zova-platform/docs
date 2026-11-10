# 0003. Database แยกต่อ service ด้วย Postgres และ Redis

- สถานะ: Accepted
- วันที่: 2026-11-10

## Context

ข้อมูลของ ZOVA มีสองลักษณะ

- **ข้อมูลถาวรที่ต้องถูกต้อง:** ผู้ใช้, ทริป, งาน, การจ่ายเงิน
- **ข้อมูลที่เปลี่ยนเร็วและมีอายุสั้น:** พิกัดคนขับทุก 3 วินาที, lock ตอนจับคู่, quote ราคา 5 นาที

ถ้าหลาย service อ่านเขียนตารางเดียวกันได้ จะเปลี่ยน schema ไม่ได้โดยไม่กระทบกัน และ service จะผูกกันแน่น

## Decision

- **แต่ละ service เป็นเจ้าของข้อมูลของตัวเอง** service อื่นเข้าถึงได้ผ่าน gRPC หรือ event เท่านั้น ห้ามต่อ database ของคนอื่น
- **ข้อมูลถาวรใช้ PostgreSQL + PostGIS** แยก database ต่อ service (`auth`, `trip`, `dispatch`, ...) บน Postgres instance เดียวกันในช่วงแรก แต่ละ database มี user ของตัวเอง
- **ข้อมูลร้อนใช้ Redis** แยก logical DB ต่อ service (`/0` gateway, `/1` location, `/2` pricing, `/3` dispatch)
- เข้าถึง Postgres ด้วย pgx + sqlc, จัดการ schema ด้วย migration ใน repo ของ service นั้น
- ID ทุกตัวเป็น UUIDv7

## Consequences

ข้อดี

- เปลี่ยน schema ของ service หนึ่งได้โดยไม่กระทบ service อื่น
- พิกัดคนขับไม่ไปถ่วง Postgres, Redis GEO ค้นหาคนขับใกล้สุดได้เร็ว
- ใช้ instance เดียวกันช่วงแรกประหยัด RAM แต่แยก instance ได้ทีหลังโดยไม่ต้องแก้โค้ด

ข้อเสียและวิธีรับมือ

- join ข้าม service ไม่ได้ → เก็บข้อมูลที่ต้องแสดงซ้ำไว้ในฝั่งที่ใช้ (เช่น trip เก็บ driver_id และขอชื่อจาก auth เมื่อจำเป็น)
- ไม่มี transaction ข้าม service → ใช้ outbox + event และออกแบบให้แต่ละขั้นทำซ้ำได้
- ข้อมูลใน Redis หายได้เมื่อ restart → เก็บเฉพาะข้อมูลที่สร้างใหม่ได้ (พิกัดส่งมาใหม่ทุก 3 วินาที, quote ขอใหม่ได้)

## Alternatives

- **Database เดียวใช้ร่วมกัน:** ง่ายกว่าในช่วงแรก แต่ทำให้ microservices ไม่เป็นอิสระจริง
- **MongoDB:** ยืดหยุ่นเรื่อง schema แต่ทริปและเงินต้องการ transaction และความถูกต้องที่ Postgres ทำได้ดีกว่า และ PostGIS เก่งเรื่องพิกัด
