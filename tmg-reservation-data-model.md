# TRP 예약 데이터 모델 — `bookings` / `accoms` / `flights` / `options`

> 2026-05-14 작성 · 김서현 PM
> TMG 숙소 + 항공권 통합 예약 화면 (`tmg-reservation-admin.html` v5) 의 백엔드 데이터 구조 제안
> ⚠️ 실제 TRP DB 스키마는 따로 있을 것 — 본 문서는 **목업이 가정하고 있는 데이터 형태**. 실제 스키마와 대조하여 갭 파악용.

---

## 0. ERD 개요

```
                       ┌────────────────────┐
                       │     bookings       │  예약 1건 (= 패키지 1개)
                       │  (예약 메인)        │
                       └──────┬─────────────┘
                              │ 1
                              │
              ┌───────────────┼──────────────────┐
              │ N             │ N                │ N
       ┌──────▼──────┐ ┌─────▼──────┐ ┌─────────▼────────┐
       │   accoms    │ │  flights   │ │     options      │
       │ (TMG 숙박권) │ │ (항공권)    │ │ (투어/티켓)        │
       └─────────────┘ └────────────┘ └──────────────────┘
```

- **예약 단위 = 상품(패키지) 단위**. 한 booking에 N개의 accoms/flights/options 결합.
- accoms는 **TMG 연동이 기본** (source=api). 일부 수동 입력 가능성 열어둠.
- flights는 항공사 API 연동 or 수동 PNR 입력. options는 대부분 수동.

---

## 1. `bookings` — 예약 메인 테이블

어드민 표준 19컬럼이 매핑되는 메인 엔티티. **결제 + 예약 상태 + 패키지 메타** 전부 보관.

| 컬럼 | 타입 | 설명 | 예시 |
|---|---|---|---|
| `id` | BIGINT PK | 내부 ID | `1` |
| `reservation_no` | VARCHAR(20) UNIQUE | 트립스토어 예약번호 (16자리) | `"8368771260859228"` |
| `channel` | VARCHAR(40) | 채널명 | `"트립스토어"` |
| `payment_type` | ENUM | 결제유형 | `"전액결제"` / `"분할결제"` |
| `ext_order_no` | VARCHAR(40) NULL | 채널사 주문번호 (외부 OTA 연동 시) | `null` |
| `customer_name` | VARCHAR(40) | 예약자명 | `"김민준"` |
| `customer_phone` | VARCHAR(20) | 연락처 | `"010-1234-5678"` |
| `pax_count` | SMALLINT | 인원 | `2` |
| `product_code` | VARCHAR(40) | 상품코드 | `"TPSOLPQC52001"` |
| `event_code` | VARCHAR(40) | 행사코드 | `"TPSOLPQC52001-260520A"` |
| `product_name` | VARCHAR(255) | 패키지명 | `"Sol by Melia Phu Quoc 4박 5일 · Air+Hotel 패키지"` |
| `departure_date` | DATE | 출발일 | `"2026-05-20"` |
| `return_date` | DATE NULL | 귀국일 (편도 시 null) | `"2026-05-24"` |
| `list_price` | INT | 총 정가 (KRW) | `4500000` |
| `discount_amount` | INT | 총 할인금 | `820000` |
| `selling_price` | INT | 총 판매가 | `3680000` |
| `payment_amount` | INT | 결제금액 | `3680000` |
| `unpaid_amount` | INT | 미수금액 | `0` |
| `payment_status` | ENUM | 결제상태 | `"결제완료"` / `"결제대기"` / `"결제실패"` / `"환불완료"` |
| `reservation_status` | ENUM | **예약상태 (5종)** | `"예약접수"` / `"예약완료"` / `"출발완료"` / `"취소요청"` / `"취소완료"` |
| `has_flight` | BOOLEAN | 항공 연동 여부 (Phase 4 분기 기준) | `true` |
| `aggregate_op_status` | VARCHAR(50) | 운영 집계 상태 (계산 컬럼 — 하위 accoms/flights 상태 집계) | `"TMG_CONFIRM_WAITING"` / `"OVERDUE"` / `"ALL_CONFIRMED"` / `"UNAVAILABLE_DETECTED"` / `"HOTEL_CANCELLED"` |
| `reserved_at` | TIMESTAMP | 예약일시 | `"2026-05-13 09:08:00"` |
| `completed_at` | TIMESTAMP NULL | 예약완료 일시 (운영자) | `"2026-05-13 09:12:00"` |
| `completed_by` | VARCHAR(40) NULL | 완료 처리 운영자 | `"트립스토어"` |
| `payment_url_sent_at` | TIMESTAMP NULL | 결제URL 발송일시 | `null` |
| `payment_url_valid` | BOOLEAN NULL | 결제URL 유효기간 여부 | `null` |
| `itinerary_url` | TEXT NULL | 여행 일정표 URL | `null` |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |

### 1.1 `reservation_status` 5종 정의

| 값 | 의미 | 전이 트리거 |
|---|---|---|
| `예약접수` | 결제완료, 운영 검토 전 (기본) | 결제완료 시 자동 |
| `예약완료` | 영업실 확정 (운영자 수동 변경 OR 항공+숙박 모두 CONFIRMED 시 자동) | 운영자 수동 변경 OR aggregate_op_status=ALL_CONFIRMED |
| `출발완료` | 출발일 경과 | 배치 (출발일+1일에 자동 변경) |
| `취소요청` | 고객 또는 운영자가 취소 요청 | 운영자 수동 변경 |
| `취소완료` | 취소 처리 완료 (위약금 정산 포함) | cancelBooking 응답 + 환불 처리 후 자동 |

### 1.2 예시 데이터 — 7건 (mockup 매칭)

```json
[
  {
    "id": 1, "reservation_no": "8368771260859228", "customer_name": "김민준",
    "product_name": "Sol by Melia Phu Quoc 4박 5일", "departure_date": "2026-05-20",
    "list_price": 4500000, "discount_amount": 820000, "selling_price": 3680000,
    "payment_status": "결제완료", "reservation_status": "예약접수",
    "has_flight": true, "aggregate_op_status": "TMG_CONFIRM_WAITING"
  },
  {
    "id": 2, "reservation_no": "8368771260859302", "customer_name": "이서연",
    "product_name": "JW Marriott Hanoi 3박 4일", "departure_date": "2026-06-01",
    "list_price": 7200000, "discount_amount": 1380000, "selling_price": 5820000,
    "reservation_status": "예약접수", "has_flight": true,
    "aggregate_op_status": "FLIGHT_PNR_WAITING_TMG_OVERDUE_SOON"
  },
  {
    "id": 3, "reservation_no": "8368771260858917", "customer_name": "최아인",
    "product_name": "Sheraton Saigon 2박 3일", "departure_date": "2026-05-25",
    "list_price": 3890000, "selling_price": 3160000,
    "aggregate_op_status": "TMG_HOLD_EXPIRED"
  },
  {
    "id": 4, "reservation_no": "8368771260858824", "customer_name": "박지호",
    "product_name": "InterContinental Danang 4박 (숙박 단독)",
    "departure_date": "2026-07-15", "has_flight": false,
    "aggregate_op_status": "MANUAL_CONFIRM_PENDING"
  },
  {
    "id": 5, "reservation_no": "8368771260857411", "customer_name": "정유나",
    "product_name": "Vinpearl Resort Nha Trang 3박 4일", "departure_date": "2026-05-18",
    "reservation_status": "예약완료", "has_flight": true,
    "aggregate_op_status": "ALL_CONFIRMED"
  },
  {
    "id": 6, "reservation_no": "8368771260859441", "customer_name": "한도윤",
    "product_name": "Pullman Saigon Centre 2박 3일", "departure_date": "2026-06-08",
    "aggregate_op_status": "UNAVAILABLE_DETECTED"
  },
  {
    "id": 7, "reservation_no": "8368771260854882", "customer_name": "강민서",
    "product_name": "Lotte Hotel Hanoi 2박 3일", "departure_date": "2026-05-30",
    "reservation_status": "예약완료", "has_flight": true,
    "aggregate_op_status": "HOTEL_CANCELLED_POST_CONFIRM"
  }
]
```

---

## 2. `accoms` — 숙박 (TMG 호텔 연동)

| 컬럼 | 타입 | 설명 | 예시 |
|---|---|---|---|
| `id` | BIGINT PK | | `1` |
| `booking_id` | BIGINT FK | `bookings.id` 참조 | `1` |
| `seq` | SMALLINT | 한 예약 내 순번 (복수 객실 시) | `1` |
| `source` | ENUM | **API 연동 / 수동 등록** | `"api"` / `"manual"` |
| `supplier_id` | VARCHAR(20) | 공급사 식별 | `"tmg"` |
| `supplier_name` | VARCHAR(200) | 호텔명 | `"Sol by Melia Phu Quoc"` |
| `property_id` | VARCHAR(40) NULL | TMG `propertyId` (API 연동 시) | `"PROP-TMG-PQC-0428"` |
| `booking_code` | VARCHAR(60) NULL | **TMG `bookingCode`** | `"TMG-HL-8N3K7Y-2026"` |
| `rate_key` | TEXT NULL | TMG `rateKey` (체크용) | `"rk_x9a8b7c6d3e2f1..."` |
| `room_info_encrypted` | TEXT NULL | TMG `roomInfoEncrypted` (v1.0.9: rate별 컬럼) | `"enc_4f5g6h7..."` |
| `rate_type` | ENUM NULL | **TMG `rateType`** (v1.0.9 신규) | `"STATIC"` / `"ALM-DYNAMIC"` / `"DYNAMIC"` |
| `room_name` | VARCHAR(120) | 객실명 | `"Deluxe Ocean View"` |
| `room_count` | SMALLINT | 객실 수 | `1` |
| `nights` | SMALLINT | 박 수 | `4` |
| `checkin_date` | DATE | 체크인 | `"2026-05-20"` |
| `checkout_date` | DATE | 체크아웃 | `"2026-05-24"` |
| `meal_plan` | VARCHAR(20) NULL | 식사 코드 (TMG `boardCode`) | `"BB"` |
| `adults_count` | SMALLINT | 성인 | `2` |
| `children_count` | SMALLINT | 아동 | `0` |
| `total_rate_vnd` | BIGINT NULL | 총 요금 (VND, TMG 원본) | `98640000` |
| `total_rate_krw` | INT NULL | 총 요금 (KRW, 환산) | `5400000` |
| `currency` | VARCHAR(3) | `"VND"` |
| `cancellation_policies` | JSON NULL | TMG 위약금 정책 (v1.0.9: `hoursBeforeCheckin`/`feePercentage`) | `[{"hoursBeforeCheckin":72,"feePercentage":0},{"hoursBeforeCheckin":24,"feePercentage":50},{"hoursBeforeCheckin":0,"feePercentage":100}]` |
| `confirm_before` | TIMESTAMP NULL | 확정 시한 (`reserveBooking` 응답) | `"2026-05-14 02:00:00"` |
| `confirm_before_type` | ENUM NULL | 시한 타입 (계약 12h / 다이나믹 ~30min) | `"contracted"` / `"dynamic"` |
| `tmg_status` | ENUM | **TMG 응답 상태값** | `"RESERVED"` / `"CONFIRMED"` / `"CANCELLED"` / `"UNAVAILABLE"` / `"EXPIRED"` |
| `internal_status` | VARCHAR(40) | **트립 내부 운영 상태** | `"TMG_RESERVED"` / `"TMG_CONFIRMED"` / `"HOTEL_CANCELLED_POST_CONFIRM"` / `"AWAITING_FLIGHT_RESERVED"` / `"HOLD_EXPIRED"` / `"ALT_HOTEL_PROPOSED"` |
| `inventory_deduct` | BOOLEAN | **재고 차감 적용 여부** (`rate_type=STATIC`일 때만 true) | `true` (STATIC) / `false` (ALM-DYN, DYNAMIC) |
| `cancel_fees_vnd` | BIGINT NULL | 취소 위약금 (cancelBooking 응답) | `null` |
| `cancel_fees_krw` | INT NULL | 위약금 (KRW 환산) | `null` |
| `cancelled_at` | TIMESTAMP NULL | 취소 처리 일시 | `null` |
| `cancel_reason` | VARCHAR(255) NULL | 취소 사유 | `null` |
| `voucher_url` | TEXT NULL | TMG 바우처 PDF URL | `null` |
| `webhook_received_at` | TIMESTAMP NULL | 최근 Webhook 수신 시각 (Phase 1: 요금 갱신 / Phase 5: 상태 변경) | `null` |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |

### 2.1 `tmg_status` ↔ `internal_status` 매핑

| TMG 응답 | 트립 내부 상태 | UI 표시 | 활성 액션 |
|---|---|---|---|
| RESERVED + confirmBefore OK | `TMG_RESERVED` | "TMG_RESERVED" 파랑 | 상태값 확인 / 숙소 예약 취소 |
| RESERVED + 항공 미RESERVED (대기) | `AWAITING_FLIGHT_RESERVED` | "대기 · 항공권 PNR 대기" 주황 | 상태값 확인 / 숙소 예약 취소 |
| RESERVED + 시한 임박 (15min/계약 1h) | `TMG_RESERVED` (countdown danger) | "TMG_RESERVED + 카운트다운 빨강" | 상태값 확인 / 숙소 예약 취소 |
| EXPIRED (시한 초과) | `HOLD_EXPIRED` | "예약 만료 (홀드 해제)" 빨강 | **재예약 시도** / 숙소 예약 취소 |
| CONFIRMED | `TMG_CONFIRMED` | "TMG_CONFIRMED ✓" 초록 | 상태값 확인 / 숙소 예약 취소 |
| UNAVAILABLE | `UNAVAILABLE_DETECTED` | "UNAVAILABLE" 빨강 | **🔄 다른 숙소 대체** / 숙소 예약 취소 |
| CANCELLED (호텔측, CONFIRMED 이후) | `HOTEL_CANCELLED_POST_CONFIRM` | "호텔측 CANCELLED" 빨강 | **🔄 다른 숙소 대체** / 숙소 예약 취소 |

### 2.2 예시 데이터 (7건)

```json
[
  { "booking_id": 1, "source": "api", "supplier_name": "Sol by Melia Phu Quoc",
    "booking_code": "TMG-HL-8N3K7Y-2026", "rate_type": "ALM-DYNAMIC",
    "room_name": "Deluxe Ocean View", "nights": 4,
    "checkin_date": "2026-05-20", "checkout_date": "2026-05-24",
    "tmg_status": "RESERVED", "internal_status": "TMG_RESERVED",
    "confirm_before": "2026-05-14 02:00:00", "confirm_before_type": "contracted",
    "inventory_deduct": false },
  { "booking_id": 2, "supplier_name": "JW Marriott Hanoi",
    "booking_code": "TMG-HL-7K9M2X-2026", "rate_type": "DYNAMIC",
    "room_name": "Executive Suite", "nights": 3,
    "tmg_status": "RESERVED", "internal_status": "AWAITING_FLIGHT_RESERVED",
    "confirm_before": "2026-05-13 13:48:00", "confirm_before_type": "dynamic",
    "inventory_deduct": false },
  { "booking_id": 3, "supplier_name": "Sheraton Saigon Hotel",
    "booking_code": "TMG-HL-3P5L8Q-2026", "rate_type": "DYNAMIC",
    "tmg_status": "EXPIRED", "internal_status": "HOLD_EXPIRED",
    "inventory_deduct": false },
  { "booking_id": 4, "supplier_name": "InterContinental Danang",
    "booking_code": "TMG-HL-9R2N4S-2026", "rate_type": "STATIC",
    "room_name": "Sun Peninsula Suite", "nights": 4,
    "tmg_status": "RESERVED", "internal_status": "TMG_RESERVED",
    "inventory_deduct": true },
  { "booking_id": 5, "supplier_name": "Vinpearl Resort Nha Trang",
    "booking_code": "TMG-HL-5J8Z1W-2026", "rate_type": "STATIC",
    "room_name": "Garden View Twin", "room_count": 2, "nights": 3,
    "tmg_status": "CONFIRMED", "internal_status": "TMG_CONFIRMED",
    "inventory_deduct": true },
  { "booking_id": 6, "supplier_name": "Pullman Saigon Centre",
    "booking_code": null, "rate_type": "ALM-DYNAMIC",
    "tmg_status": "UNAVAILABLE", "internal_status": "UNAVAILABLE_DETECTED",
    "inventory_deduct": false },
  { "booking_id": 7, "supplier_name": "Lotte Hotel Hanoi",
    "booking_code": "TMG-HL-2H7Q4R-2026", "rate_type": "STATIC",
    "tmg_status": "CANCELLED", "internal_status": "HOTEL_CANCELLED_POST_CONFIRM",
    "webhook_received_at": "2026-05-13 11:48:02",
    "inventory_deduct": true }
]
```

---

## 3. `flights` — 항공권

| 컬럼 | 타입 | 설명 | 예시 |
|---|---|---|---|
| `id` | BIGINT PK | | `1` |
| `booking_id` | BIGINT FK | bookings 참조 | `1` |
| `seq` | SMALLINT | 한 예약 내 순번 (다구간 시) | `1` |
| `source` | ENUM | API 연동 / 수동 PNR | `"api"` / `"manual"` |
| `airline_code` | VARCHAR(3) | IATA 항공사 코드 | `"VJ"` |
| `airline_name` | VARCHAR(80) | 항공사명 | `"VietJet Air"` |
| `flight_no_outbound` | VARCHAR(10) | 출국편 항공편명 | `"VJ871"` |
| `flight_no_inbound` | VARCHAR(10) NULL | 귀국편 (편도 시 null) | `"VJ872"` |
| `origin` | VARCHAR(3) | 출발지 IATA | `"ICN"` |
| `destination` | VARCHAR(3) | 도착지 IATA | `"PQC"` |
| `depart_at` | TIMESTAMP | 출발 시각 | `"2026-05-20 06:30:00"` |
| `arrive_at` | TIMESTAMP NULL | 도착 시각 | `"2026-05-20 10:20:00"` |
| `return_depart_at` | TIMESTAMP NULL | 귀국편 출발 | `"2026-05-24 14:50:00"` |
| `return_arrive_at` | TIMESTAMP NULL | 귀국편 도착 | `"2026-05-24 21:35:00"` |
| `seat_count` | SMALLINT | 좌석 수 | `2` |
| `cabin_class` | ENUM | 좌석 등급 | `"Economy"` / `"Business"` / `"First"` |
| `pnr_code` | VARCHAR(20) NULL | PNR 코드 | `"ABC123"` |
| `gds_record_locator` | VARCHAR(20) NULL | GDS 레코드 | `null` |
| `status` | ENUM | 항공권 상태 | `"BOOKING"` / `"RESERVED"` / `"CONFIRMED"` / `"CANCELLED"` |
| `eticket_url` | TEXT NULL | E-Ticket PDF | `"vj_pnr_0520.pdf"` |
| `cancellation_policy` | JSON NULL | 항공사별 취소 위약금 (수동 입력 or API) | `[{...}]` |
| `total_fare_krw` | INT NULL | 총 요금 (KRW) | `1850000` |
| `confirmed_at` | TIMESTAMP NULL | 확정 일시 | `null` |
| `cancelled_at` | TIMESTAMP NULL | | |
| `created_at, updated_at` | TIMESTAMP | | |

### 3.1 예시 데이터 (6건)

```json
[
  { "booking_id": 1, "source": "api", "airline_name": "VietJet Air",
    "flight_no_outbound": "VJ871", "flight_no_inbound": "VJ872",
    "origin": "ICN", "destination": "PQC",
    "depart_at": "2026-05-20 06:30:00", "seat_count": 2,
    "status": "RESERVED", "eticket_url": "vj_pnr_0520.pdf" },
  { "booking_id": 2, "source": "api", "airline_name": "Korean Air",
    "flight_no_outbound": "KE679", "status": "BOOKING" },
  { "booking_id": 3, "source": "api", "airline_name": "Asiana Airlines",
    "flight_no_outbound": "OZ731", "status": "RESERVED" },
  { "booking_id": 5, "source": "api", "airline_name": "Vietnam Airlines",
    "flight_no_outbound": "VN417", "seat_count": 4, "status": "CONFIRMED" },
  { "booking_id": 6, "source": "api", "airline_name": "Jeju Air",
    "flight_no_outbound": "7C2387", "status": "RESERVED" },
  { "booking_id": 7, "source": "api", "airline_name": "Asiana Airlines",
    "flight_no_outbound": "OZ711", "status": "CONFIRMED" }
  // booking_id=4 (숙박 단독)는 flights row 없음
]
```

---

## 4. `options` — 옵션 (Tour / Ticket / Transfer 등)

| 컬럼 | 타입 | 설명 | 예시 |
|---|---|---|---|
| `id` | BIGINT PK | | `1` |
| `booking_id` | BIGINT FK | | `2` |
| `type` | ENUM | 옵션 종류 | `"tour"` / `"ticket"` / `"transfer"` / `"meal"` |
| `source` | ENUM | API / 수동 | `"manual"` (대부분 수동) |
| `supplier_name` | VARCHAR(200) | 공급사명 | `"Halong Bay Day Tour"` |
| `item_code` | VARCHAR(40) NULL | 내부 옵션 코드 | `"VINPEARL-TR-2810"` |
| `item_name` | VARCHAR(255) | 옵션명 | `"하롱베이 1일 투어"` |
| `usage_date` | DATE | 사용일 | `"2026-06-02"` |
| `quantity` | SMALLINT | 수량 (티켓·인원) | `2` |
| `detail` | TEXT NULL | 상세 (가이드/픽업 등) | `"07:30 픽업 · 한국어 가이드"` |
| `status` | ENUM | | `"CONFIRMED"` / `"PENDING"` / `"CANCELLED"` |
| `voucher_url` | TEXT NULL | 바우처 | `"halong_voucher.pdf"` |
| `cancellation_policy` | JSON NULL | | |
| `total_price_krw` | INT NULL | | `380000` |

### 4.1 예시 데이터 (2건)

```json
[
  { "booking_id": 2, "type": "tour", "source": "manual",
    "supplier_name": "Halong Bay Day Tour", "item_code": "VINPEARL-TR-2810",
    "usage_date": "2026-06-02", "quantity": 2, "status": "CONFIRMED" },
  { "booking_id": 5, "type": "ticket", "source": "manual",
    "supplier_name": "Vinpearl Land 입장권", "item_code": "VPL-TKT-4421",
    "usage_date": "2026-05-19", "quantity": 4, "status": "CONFIRMED" }
]
```

---

## 5. 보조 테이블 후보 (필요 시)

### 5.1 `accom_alt_proposals` — 대체 숙소 제안 이력
UNAVAILABLE / HOTEL_CANCELLED_POST_CONFIRM 케이스에서 어떤 대체 숙소를 제안했고 고객 반응이 어땠는지 추적.

| 컬럼 | 설명 |
|---|---|
| `id`, `original_accom_id` FK | |
| `proposed_property_id` | 제안한 호텔 |
| `proposed_room_name` | |
| `proposed_rate_krw` | |
| `price_diff_krw` | 차액 |
| `proposed_at` | 제안 시각 |
| `customer_response` | `accepted` / `rejected` / `no_response` |
| `responded_at` | |
| `swapped_accom_id` FK NULL | swap 시 새 accoms.id |

### 5.2 `tmg_api_logs` — TMG API 호출 로그
모든 reserveBooking/confirmBooking/cancelBooking 호출 + 응답 보관 (감사/디버깅).

| 컬럼 | 설명 |
|---|---|
| `id`, `accom_id` FK | |
| `endpoint` | `"/checkAvailability"` / `"/reserveBooking"` / ... |
| `request_payload` JSON | |
| `response_payload` JSON | |
| `http_status` | |
| `error_code` | TMG errorCode |
| `called_at` | |

### 5.3 `reservation_state_log` — 예약상태 변경 이력
운영자가 `reservation_status` 변경한 이력 (감사용).

---

## 6. 핵심 비즈니스 룰 (구현 시 검증 필요)

1. **bookings.aggregate_op_status는 계산 컬럼** — accoms/flights의 상태 변화에 따라 자동 업데이트되어야 함 (DB 트리거 or 애플리케이션 레벨 hook)
2. **inventory_deduct는 rate_type에 종속** — STATIC만 true. 변경 시 stock 시스템에 시그널 전송 필요
3. **confirm_before 시한 폴링** — TMG_RESERVED + AWAITING_FLIGHT_RESERVED 상태인 accoms에 대해 주기 polling (시한 임박 시 빈도 ↑)
4. **호텔측 취소 Webhook** — TMG가 CONFIRMED 이후 CANCELLED 통보 시 별도 핸들러 (운영팀 알림 + ALT_HOTEL 흐름 트리거)
5. **항공 RESERVED → TMG 자동 confirm** — flights.status가 RESERVED로 바뀌면 같은 booking의 TMG_RESERVED accoms에 대해 자동 confirmBooking 트리거

---

## 7. 목업과의 매핑 요약

| 목업 컬럼 | bookings 컬럼 |
|---|---|
| 예약번호 | `reservation_no` |
| 채널명 | `channel` |
| 결제유형 | `payment_type` |
| 예약자명 | `customer_name` |
| 연락처 | `customer_phone` |
| 상품코드 / 행사코드 | `product_code` / `event_code` |
| 총 정가 / 할인금 / 판매가 / 결제금액 / 미수금액 | `list_price`/`discount_amount`/`selling_price`/`payment_amount`/`unpaid_amount` |
| 결제상태 | `payment_status` |
| 예약상태 변경 | `reservation_status` (dropdown) |
| 예약일시 · 완료일시 | `reserved_at` / `completed_at` |

| 목업 펼침 패널 | accoms / flights / options |
|---|---|
| ✈️ Flight 행 | `flights` row |
| 🏨 Hotel TMG 행 | `accoms` (where `source='api'` and `supplier_id='tmg'`) |
| 🎫 Tour / Ticket 행 | `options` |
| 🔌 API / ✏️ 수동 뱃지 | `source` 컬럼 |
| `STATIC` / `ALM-DYN` / `DYNAMIC` 뱃지 | `accoms.rate_type` |
| `TMG_RESERVED` / `TMG_CONFIRMED` 등 상태 | `accoms.internal_status` |
| 잔여 시한 카운트다운 | `accoms.confirm_before` (실시간 차이) |
| 📎 voucher / e-ticket | `accoms.voucher_url` / `flights.eticket_url` |

---

## 8. 다음 단계

- [ ] 실제 TRP DB 스키마(`extriber/hackers`?)와 대조
- [ ] `bookings.aggregate_op_status` 의 enum 값 운영팀과 합의 (7~10종)
- [ ] `accoms.internal_status` 의 enum 값 운영팀과 합의
- [ ] `inventory_deduct` 시그널 → stock 시스템 인터페이스 정의 (정근님)
- [ ] `accom_alt_proposals` 도입 여부 결정 (대체 숙소 이력 트래킹 가치)
- [ ] flights 측 데이터 모델은 누아 항공 API 연동 스펙과 align (별도 프로젝트)
