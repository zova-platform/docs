# ZOVA Color Palette

ชุดสีเดียวกันทั้งโลโก้และ UI ของทุกเว็บ (`passenger-web`, `driver-web`, `admin-web`, `merchant-web`)

## สีหลัก

| ชื่อ | Hex | บทบาท | ใช้กับ |
| --- | --- | --- | --- |
| Navy | `#172B3D` | Primary | header, navigation, ตัวอักษรหลัก |
| Orange | `#F4A340` | Accent | ปุ่ม CTA, ไฮไลต์, องค์ประกอบแบรนด์ |
| Mint | `#54B8A1` | Secondary | สถานะสำเร็จ, ไฮไลต์รอง |
| White | `#F5F6F7` | Background | พื้นหลัง, card, surface |

## สีสถานะ

| ชื่อ | Hex | ใช้กับ |
| --- | --- | --- |
| Red | `#C53030` | ยกเลิก, หาคนขับไม่ได้ (`cancelled`, `no_driver_found`), error |
| Yellow | `#F2C94C` | คำเตือน, รอคนขับตอบรับ |
| Mint | `#54B8A1` | สำเร็จ (`completed`) |

## คู่สีตัวอักษรกับพื้นหลัง (contrast)

มาตรฐาน WCAG AA ต้องการอย่างน้อย 4.5:1 สำหรับตัวอักษรปกติ และ 3:1 สำหรับตัวใหญ่หรือตัวหนา

| ตัวอักษร | พื้นหลัง | Contrast โดยประมาณ | ใช้ได้ไหม |
| --- | --- | --- | --- |
| Navy | White | 13:1 | ✅ ทุกขนาด |
| Navy | Orange | 7:1 | ✅ ใช้กับปุ่ม CTA |
| White | Orange | 2:1 | ❌ ห้ามใช้ |
| Navy | Mint | 6:1 | ✅ |
| White | Mint | 2.4:1 | ❌ ห้ามใช้ |
| White | Red | 5.5:1 | ✅ |
| Navy | Yellow | 9:1 | ✅ |
| White | Navy | 13:1 | ✅ |

กฎจำง่าย: **บนพื้นสีสว่าง (ส้ม, มิ้นต์, เหลือง) ใช้ตัวอักษร Navy เสมอ**

## สีบนแผนที่

| สิ่งที่แสดง | สี |
| --- | --- |
| เส้นทางที่กำลังเดินทาง | Orange |
| หมุดจุดรับ | Navy |
| หมุดจุดส่ง | Orange |
| รถว่าง (admin-web) | Mint |
| รถมีงาน (admin-web) | Orange |
| รถออฟไลน์ (admin-web) | เทา `#9AA5B1` |

## Design tokens

ใช้ใน CSS ของทุกเว็บ ไฟล์เดียวกัน (ภายหลังย้ายไป `ui-kit` ได้)

```css
@theme {
  --color-navy: #172b3d;
  --color-orange: #f4a340;
  --color-mint: #54b8a1;
  --color-surface: #f5f6f7;
  --color-danger: #c53030;
  --color-warning: #f2c94c;
  --color-muted: #9aa5b1;
}
```

## โลโก้

- ไฟล์: `logo.png` (และ `logo.svg` เมื่อมี)
- โลโก้ต้องใช้ Navy `#172B3D` และ Orange `#F4A340` ตามชุดนี้
- ทดสอบที่ 16–32 px (favicon, หมุดบนแผนที่) ถ้าช่องระหว่างสองท่อนหาย ให้ทำเวอร์ชันย่อที่ช่องกว้างขึ้น
- พื้นหลังมืด: ใช้ท่อนสีกรมท่าเป็นสีขาวแทน
