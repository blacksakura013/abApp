# Payment & GP Flow — Aidly Buddy

**สถานะการตรวจ:** 8 ตุลาคม 2026  
**ขอบเขต:** การรับเงินลูกค้า, GP (Platform Fee), ยอดค้างจ่ายผู้รับงาน, คืนเงิน และการจ่ายเงินจริง

## ข้อสรุป

ระบบรับ PromptPay และบันทึกบัญชีคู่สำหรับยอดชำระได้ โดย GP ตั้งไว้ **20%** และเก็บเป็น `platform_fee_minor` (หน่วยสตางค์) ระบบ payout ถูกเพิ่มแล้ว: มีบัญชีรับเงินแบบ tokenized, การอนุมัติบัญชี, คิว payout แบบ idempotent, clearing ledger, webhook ผลการโอน และรายงาน reconciliation. ยัง **ห้ามเปิดเงินจริง** จนกว่าจะตั้งค่า payout provider/webhook secret, ทดสอบ sandbox และทำ live pilot ที่กระทบยอดได้ครบ

> คำว่า **GP คืนระบบ** ในเอกสารนี้หมายถึงการกลับรายการรายได้แพลตฟอร์มเมื่อคืนเงินให้ลูกค้า ไม่ใช่ค่าธรรมเนียมที่ payment gateway เรียกเก็บ ซึ่งต้องบันทึกแยกเป็นต้นทุนผู้ให้บริการชำระเงิน

## Flow ปัจจุบันที่ตรวจพบ

```text
ลูกค้าสร้าง booking ที่ accepted
  -> POST /payment พร้อม Idempotency-Key และยอดต้องเท่าราคา booking
  -> PromptPay: สร้าง charge กับ provider แล้วได้ QR/checkout URL
  -> provider webhook หรือ status check ยืนยันยอดและสกุลเงิน
  -> payment = paid และ booking = confirmed
  -> ledger: Dr customer_funds = ยอดรวม (G)
             Cr worker_payable = ยอดผู้รับงานสุทธิ (N)
             Cr platform_revenue = GP (F)
```

สูตรที่ใช้ในโค้ด: `F = round(G x 20%)`, `N = G - F` โดยทุกยอดเป็นสตางค์ (`*_minor`)  
ตัวอย่าง: ลูกค้าจ่าย 1,000.00 บาท → GP 200.00 บาท → ยอดค้างจ่ายผู้รับงาน 800.00 บาท

การคืนเงิน PromptPay ปัจจุบันเรียก provider ก่อน แล้วจึงบันทึก local ledger เมื่อ provider ตอบผลสำเร็จขั้นสุดท้าย:

```text
Dr worker_payable       = ยอดสุทธิส่วนที่คืน
Dr platform_revenue     = GP ส่วนที่คืน
Cr customer_funds       = ยอดคืนลูกค้า
```

## ช่องว่างและความเสี่ยงที่ต้องปิด

| ระดับ | เรื่อง | หลักฐาน/ผลกระทบ | สิ่งที่ต้องทำ |
| --- | --- | --- | --- |
| P0 | ยังไม่มี provider ที่ตั้งค่าและยืนยันเงินจริง | ระบบส่ง payout แบบ tokenized และรับ webhook ได้ แต่ไม่มีหลักฐาน sandbox/live transfer | ตั้งค่า provider, secret และ webhook URL; ทดสอบ/กระทบยอดก่อนเปิดเงินจริง |
| P0 | Refund หลัง payout ถูกกันไว้เพื่อความปลอดภัย | ระบบปฏิเสธ refund เมื่อ payout อยู่ queued/submitted/paid เพื่อไม่ให้เงินติดลบ | เพิ่ม manual clawback/worker receivable พร้อม approval ก่อนเปิด policy คืนหลัง payout |
| P0 | ไม่พบหลักฐาน UAT กับ payment provider จริง | unit test ใช้ mock; integration test ใช้ cash แล้ว admin settle | ทำ sandbox webhook + refund + duplicate delivery + live pilot ที่ควบคุมได้ |
| P1 | GP จาก partial refund มีโอกาสเหลือเศษจากการปัด | โค้ดคำนวณ GP ใหม่จากแต่ละยอดคืน; ผลรวมหลายครั้งอาจไม่เท่ากับ GP ของรายการเดิม | เก็บ `fee_reversed_minor` และให้ refund ครั้งสุดท้ายคืน `platform_fee_minor - fee_reversed_minor` |
| P1 | Cash ถูก settle ผ่าน admin ได้ แต่ไม่มีการยืนยันรับเงิน/หลักฐาน | อาจ confirm booking โดยเงินสดยังไม่รับจริง | ใช้ `cash_pending` และบังคับหลักฐาน/ผู้อนุมัติ/เวลา; ไม่สร้าง worker payable จนยืนยัน |
| P1 | Payout hold ไม่ถูกตรวจตอนสั่งจ่าย | hold/release มีอยู่ แต่ไม่มี payout executor ให้บังคับใช้ | payout eligibility ต้องปฏิเสธทุกรายการที่มี hold active |
| P1 | รายงานนับเฉพาะ `paid` | ยอด `partially_refunded` และ GP ที่กลับรายการอาจไม่สะท้อนผลสุทธิ | รายงานจาก ledger/event ที่ posted แยก gross, refund, GP, provider fee, payout |
| P1 | ค่าธรรมเนียม gateway ยังไม่แยกบัญชี | เสี่ยงสับสนระหว่างรายได้ GP กับต้นทุนรับชำระ | เพิ่ม `provider_fee_expense` และ clearing account จาก settlement report ของ provider |

## Flow เป้าหมายที่ควรใช้

```text
1. Create payment (idempotency key)
2. Provider confirms terminal payment by signed/verified webhook
3. Post immutable ledger event: gross / worker payable / GP
4. Confirm booking
5. Complete job -> รอ dispute window และตรวจ KYC/บัญชีรับเงิน/hold
6. Create payout batch -> submit transfer -> provider confirms transfer
7. Post payout ledger event and reconcile bank/provider settlement
8. Refund path
   - ก่อน payout: reverse payable + GP + customer funds
   - หลัง payout: provider refund ลูกค้า + สร้าง worker receivable/clawback
```

### สถานะที่เสนอ

```text
Payment: created -> pending -> paid -> partially_refunded | refunded | failed
Payout:  eligible -> held | queued -> submitted -> paid | failed | reversed
Refund:  requested -> provider_pending -> completed | failed | manual_review
```

### กติกาก่อนเข้าคิว payout

- booking ต้อง `completed` และพ้นระยะ dispute ที่กำหนด
- ผู้รับงานผ่าน KYC, มีบัญชีรับเงินที่ยืนยันแล้ว และไม่ถูกระงับ
- payment ต้อง `paid` หรือ `partially_refunded` และยอดสุทธิที่เหลือต้องมากกว่า 0
- ไม่มี `PayoutHold` ที่ active, ไม่มี dispute/SOS ที่ยังเปิด, ไม่มี payout ก่อนหน้ากำลังดำเนินการ
- สร้าง idempotency key ระดับ payout และ lock ยอดที่เลือกใน transaction เดียว

### Ledger ที่เสนอสำหรับ payout สำเร็จ

```text
Dr worker_payable       = N
Cr payout_clearing      = N

เมื่อผู้ให้บริการโอนยืนยันสำเร็จ
Dr payout_clearing      = N
Cr bank_cash            = N
```

Provider fee ต้องเป็น event แยก:

```text
Dr provider_fee_expense = P
Cr bank_cash            = P
```

## แบบข้อมูลและ API ที่ต้องเพิ่ม

### ตาราง/ข้อมูล

- `payout_accounts`: worker_id, bank/provider token, verified_at, status (เข้ารหัสข้อมูลอ่อนไหว)
- `payouts`: id, worker_id, currency, amount_minor, status, idempotency_key, provider_reference, batch_id, failure_reason, submitted_at, paid_at
- `payout_items`: payout_id, payment_id, amount_minor, locked_at (unique ต่อ payment/ยอดที่จ่าย)
- `refunds`: เพิ่ม provider_reference, idempotency_key, provider_status, `fee_reversed_minor`
- `ledger_entries`: เพิ่ม immutable `event_type`, `external_reference`, `posted_at`; ทำ unique key เพื่อกัน event ซ้ำ
- `reconciliation_runs`: รอบกระทบยอด, ยอด provider/bank, ผลต่าง, ผู้อนุมัติ, หลักฐานไฟล์

### API ที่ควรมี

- `POST /worker/payout-accounts` และ flow ยืนยันบัญชี
- `GET /worker/earnings` แสดง available / held / paid / refunded
- `POST /admin/payouts/prepare` สร้าง batch แบบ preview
- `POST /admin/payouts/:id/submit` ส่งให้ provider ด้วย idempotency key
- `POST /payment/webhooks/payout-provider` รับผล transfer และเก็บ raw event/audit hash
- `GET /admin/reconciliation` รายงานยอดต่างจาก provider และธนาคาร

สิทธิ์: ผู้รับงานอ่านเฉพาะรายได้ของตน; admin แยก role สำหรับ approve/submit/reconcile และต้องมี audit log ทุกครั้ง

## ลำดับการทำงานที่แนะนำ

1. เสร็จแล้ว: แก้การปัดเศษ GP ของ partial refund และเพิ่ม unit test
2. เสร็จแล้ว: payout data model, tokenized payout account, ledger events, eligibility check และ hold enforcement
3. เสร็จแล้วในระดับ integration contract: outbound provider request, HMAC webhook และ reconciliation endpoint
4. เสร็จแล้ว: หน้ารายได้/ตั้งค่าบัญชีรับเงินของผู้รับงานใน mobile
5. ต้องทำกับ provider จริง: sandbox duplicate webhook, timeout, payout fail, dispute hold และ concurrent request
6. ต้องทำก่อน production: live pilot เงินจำนวนน้อยและกระทบยอดรายวัน

## เกณฑ์ผ่านก่อนเปิดเงินจริง

- ทุก event เงินมี idempotency และ ledger balance = 0 เสมอ
- webhook ตรวจลายเซ็น/ยอด/สกุลเงิน/reference และกัน replay ได้
- ไม่มี payout ออกจากยอดที่ hold, dispute, ไม่ผ่าน KYC หรือยังไม่พ้น dispute window
- refund ทุกกรณีมี provider reference และไม่ทำให้ GP/worker payable ค้างผิดยอด
- รายงานรายวัน reconcile กับ provider และบัญชีธนาคารได้; ความต่างต้องถูกติดตามจนปิด
- UAT กับ sandbox และ live pilot มีหลักฐานผลลัพธ์จริงก่อน release

## ขอบเขตหลักฐานการตรวจครั้งนี้

ตรวจจาก source, API runtime, Swagger, mobile typecheck/lint/Jest และ Android debug build. ยังไม่มีการโอนเงินจริงหรือการทดสอบ provider sandbox/live ในผลตรวจนี้ จึงไม่ใช่การยืนยันว่าระบบพร้อมจ่ายเงินจริงใน production.
