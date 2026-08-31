# Swagger Petstore API Test Cases for Postman

## 1. Document Control

| Item | Detail |
| --- | --- |
| System under test | Swagger Petstore - OpenAPI 3.0 |
| Swagger UI | `https://petstore3.swagger.io/` |
| Base URL | `https://petstore3.swagger.io/api/v3` |
| OpenAPI source | `https://petstore3.swagger.io/api/v3/openapi.json` |
| Contract version inspected | `1.0.27` |
| Test design date | 2026-08-19 |
| Intended execution tool | Postman |
| Full catalog status | Designed; not executed as a complete set |
| Executed subset | `PET-001` through `PET-007`; local run passed 24/24 assertions |

เอกสารนี้เป็น full Test Case catalog สำหรับออกแบบ coverage และยังไม่ได้ execute ครบทุกเคส อย่างไรก็ตาม subset `PET-001` ถึง `PET-007` ถูกสร้างเป็น Postman Collection และรันจริงใน local แล้ว โดย assertions ผ่าน 24/24 รายการ หลักฐานอยู่ในโฟลเดอร์ `evidence/`

## 2. Objective and Scope

วัตถุประสงค์คือทดสอบ REST API ของ Petstore ให้ครอบคลุม:

- Functional และ CRUD behavior ของ `pet`, `store`, `user`
- Request validation: path, query, header, body, data type, enum และ required field
- Response validation: status code, header, content type, schema และค่าข้อมูล
- Authentication/authorization ตาม OpenAPI security scheme
- Data consistency หลัง Create, Update และ Delete
- Boundary value, special character, duplicate, idempotency และ error handling
- End-to-end business flow ที่สามารถจัดลำดับใน Postman Collection Runner ได้

อยู่นอก scope ในรอบนี้: load/stress test, penetration test เชิงรุก, OAuth provider internals, database validation และการแก้ไข API

## 3. Contract Facts and Test Oracle

### 3.1 Schema rules confirmed by OpenAPI

- `Pet` required: `name`, `photoUrls`
- `Pet.status`: `available`, `pending`, `sold`
- `Order.status`: `placed`, `approved`, `delivered`
- `Order.id`, `Order.petId`, `Pet.id`, `Category.id`, `Tag.id`, `User.id`: `int64`
- `Order.quantity`, `User.userStatus`: `int32`
- `Order.shipDate`: `date-time`
- `Order` และ `User` ไม่มี required property ใน schema
- `email` เป็นเพียง `string`; contract ไม่ได้กำหนด `format: email`
- string และ array ส่วนใหญ่ไม่มี `minLength`, `maxLength`, `minItems`, `maxItems`

### 3.2 Case classification

| Code | Meaning | Pass/Fail rule |
| --- | --- | --- |
| C | Contract | ต้องตรง status/schema/rule ที่ OpenAPI ระบุ มิฉะนั้นเป็น Contract defect |
| F | Functional flow | ต้องคงข้อมูลและ state ตาม flow ที่สร้างไว้ มิฉะนั้นเป็น Functional defect |
| E | Exploratory / contract gap | Swagger ไม่กำหนดผลลัพธ์ชัดเจน ให้เก็บ Actual Result; ต้องไม่เกิดข้อมูลเสียหรือ `5xx` จาก input ปกติ |
| S | Security | ตรวจ security declaration และการป้องกันข้อมูล; หาก public sample ไม่ enforce ให้บันทึกเป็น implementation/security gap |

### 3.3 General assertions for every response

ทุกเคสต้องตรวจเพิ่มเมื่อเกี่ยวข้อง:

1. Response time ถูกบันทึกไว้เพื่อทำ baseline; รอบแรกยังไม่กำหนด SLA
2. Status code ตรงกับ Expected Result
3. `Content-Type` สอดคล้องกับ `Accept` และ body จริง
4. JSON/XML parse ได้และตรง schema ที่ประกาศ
5. Response ไม่มี stack trace, SQL error, secret, token หรือ internal path
6. ID และค่าหลักใน response ตรงกับ request และข้อมูลที่เตรียมไว้
7. Error response อ่านเข้าใจได้และไม่คืน `2xx` ให้ invalid request ที่ contract ระบุว่าต้อง reject
8. Create/Update/Delete ต้องตรวจซ้ำด้วย GET เมื่อมี endpoint รองรับ

## 4. Proposed Postman Variables and Test Data

| Variable | Example | Purpose |
| --- | --- | --- |
| `baseUrl` | `https://petstore3.swagger.io/api/v3` | Base URL |
| `runId` | timestamp เช่น `20260819153000` | ทำข้อมูลแต่ละรอบให้ไม่ชนกัน |
| `petId` | `{{runId}}01` | Pet ที่ใช้ใน positive flow |
| `petIdDelete` | `{{runId}}02` | Pet สำหรับ delete flow |
| `unknownPetId` | `899999999999999999` | ID ที่คาดว่าไม่มี |
| `orderId` | สุ่ม `11..999` และสร้างก่อนใช้ทันที | Order positive/get/delete flow; สอดคล้องกับเงื่อนไข `<1000` ของ DELETE |
| `username` | `qa_petstore_{{runId}}` | User positive flow |
| `usernameDelete` | `qa_delete_{{runId}}` | User delete flow |
| `unknownUsername` | `qa_missing_{{runId}}` | User ที่คาดว่าไม่มี |
| `apiKey` | ค่า test key ที่ได้รับอนุญาต | Header `api_key` |
| `oauthToken` | OAuth test token | Bearer token/scopes |

Baseline Pet body:

```json
{
  "id": 202608190001,
  "name": "QA-Dog-20260819",
  "category": { "id": 1, "name": "Dogs" },
  "photoUrls": ["https://example.test/images/qa-dog.png"],
  "tags": [{ "id": 101, "name": "automation" }],
  "status": "available"
}
```

Baseline Order body:

```json
{
  "id": 519,
  "petId": 202608190001,
  "quantity": 1,
  "shipDate": "2026-08-20T09:00:00Z",
  "status": "placed",
  "complete": false
}
```

Baseline User body:

```json
{
  "id": 202608190201,
  "username": "qa_petstore_20260819",
  "firstName": "QA",
  "lastName": "Tester",
  "email": "qa.petstore@example.test",
  "password": "Postman-Test-Only-123!",
  "phone": "+66812345678",
  "userStatus": 1
}
```

## 5. Pet API Test Cases

### 5.1 `POST /pet` - Add a new pet

Common step: ส่ง `POST {{baseUrl}}/pet` ด้วย `Content-Type: application/json` และ OAuth scope ที่ถูกต้อง จากนั้นตรวจ response และ `GET /pet/{id}`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-ADD-001 | P0 | C,F | ส่ง Baseline Pet body ด้วย ID ใหม่ | `200`; response เป็น `Pet`; `id/name/status/photoUrls` ตรง request; GET ด้วย ID เดียวกันคืนข้อมูลที่สร้าง |
| PET-ADD-002 | P1 | C | ส่งเฉพาะ required fields: `{"name":"Minimum Pet","photoUrls":[]}` | `200`; response parse ได้เป็น `Pet`; ระบบสร้างข้อมูลได้ตาม contract |
| PET-ADD-003 | P1 | C | ไม่ส่ง `name` | `400` Invalid input หรือ `422` Validation exception; ต้องไม่สร้างข้อมูล |
| PET-ADD-004 | P1 | C | ไม่ส่ง `photoUrls` | `400` หรือ `422`; ต้องไม่สร้างข้อมูล |
| PET-ADD-005 | P1 | C | `name:null` | `400` หรือ `422`; required field ต้องไม่ยอมรับ null |
| PET-ADD-006 | P1 | C | `photoUrls:null` | `400` หรือ `422`; required array ต้องไม่ยอมรับ null |
| PET-ADD-007 | P1 | C | `status:"pending"` และ `status:"sold"` แยกสอง iteration | แต่ละค่าคืน `200`; response/GET คง status ตามที่ส่ง |
| PET-ADD-008 | P1 | C | `status:"AVAILABLE"`, `"invalid"`, `""` แยก iteration | `400` หรือ `422`; enum เป็น case-sensitive; ไม่สร้างข้อมูล invalid |
| PET-ADD-009 | P1 | C | `id:"abc"` | `400` หรือ `422`; ไม่ coerce string ที่ไม่ใช่เลขเป็น int64 |
| PET-ADD-010 | P2 | E | `id` ที่ขอบเขต int64: `9223372036854775807`; ส่งเป็น raw JSON และตรวจ precision | ไม่เกิด `5xx`; หากรับต้องคืน ID เดิมโดยไม่ปัดเศษ; บันทึกข้อจำกัด JSON/Postman precision |
| PET-ADD-011 | P2 | E | `id:0` และ `id:-1` แยก iteration | เก็บ Actual Result เพราะ schema ไม่มี minimum; ถ้ารับต้อง GET ได้ตรง ID และข้อมูลไม่ชน record อื่น |
| PET-ADD-012 | P2 | E | `name:""`, whitespace-only และ Unicode/Thai/emoji แยก iteration | Unicode ต้องไม่ทำให้ `5xx`; blank acceptance เป็น contract gap; ค่าที่รับต้อง round-trip ไม่เพี้ยน |
| PET-ADD-013 | P2 | C | `photoUrls` มี 0, 1 และหลาย URL | array ทุกขนาดที่ไม่ขัด schema ควรรับ; ลำดับและค่าต้องคงเดิม |
| PET-ADD-014 | P2 | E | `photoUrls:[123,null]` | ควรถูก reject ด้วย `400/422`; หากรับถือเป็น schema validation gap |
| PET-ADD-015 | P2 | C | `tags` หลายรายการและ `category` ครบ | `200`; nested objects และ array ตรง schema/ค่าที่ส่ง |
| PET-ADD-016 | P2 | E | ส่ง unknown property เช่น `"isAdmin":true` | ไม่เกิด `5xx`; บันทึกว่า ignore, echo หรือ persist; contract ไม่กำหนด `additionalProperties:false` |
| PET-ADD-017 | P1 | C | body ว่าง หรือไม่มี body | `400` หรือ `422`; requestBody เป็น required |
| PET-ADD-018 | P1 | C | malformed JSON | `400` หรือ `422`; ไม่มี record ถูกสร้าง; response ไม่เผย parser stack trace |
| PET-ADD-019 | P1 | C | `Content-Type: application/xml` ด้วย Pet XML ที่ถูกต้อง | `200`; response ตาม `Accept`; ข้อมูลเทียบเท่า JSON และ GET ได้ |
| PET-ADD-020 | P2 | E | `Content-Type: text/plain` แต่ body เป็น JSON | ควร `415 Unsupported Media Type`; หากรับให้บันทึก contract/content negotiation gap |
| PET-ADD-021 | P1 | S | ไม่ส่ง OAuth token | ควร `401/403` ตาม security declaration; ต้องไม่สร้างข้อมูล |
| PET-ADD-022 | P1 | S | token ไม่มี `write:pets` หรือหมดอายุ | `401/403`; ต้องไม่สร้างข้อมูล |
| PET-ADD-023 | P1 | F | ส่ง ID เดิมซ้ำโดยเปลี่ยนชื่อ | เก็บ Actual: API ต้องมี behavior ชัดว่า update/reject; ห้ามสร้าง duplicate ที่ GET ให้ผลไม่แน่นอน |

### 5.2 `PUT /pet` - Update an existing pet

Precondition: สร้าง Pet เป้าหมายไว้และเก็บ `petId`; หลัง PUT ให้ GET ตรวจ persisted state

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-PUT-001 | P0 | C,F | เปลี่ยน `name`, `category`, `photoUrls`, `tags`, `status` ของ Pet ที่มีอยู่ | `200`; response และ GET สะท้อนค่าทุก field ใหม่; `id` เดิม |
| PET-PUT-002 | P1 | C,F | เปลี่ยน status `available -> pending -> sold` ตามลำดับ | ทุกครั้ง `200`; GET หลังแต่ละรอบตรง state ล่าสุด |
| PET-PUT-003 | P1 | C | body ไม่ส่ง `id` | `400` Invalid ID, `404` หรือ `422`; ต้องไม่อัปเดต record อื่น |
| PET-PUT-004 | P1 | C | `id:"abc"` | `400` หรือ `422`; ไม่มีข้อมูลเปลี่ยน |
| PET-PUT-005 | P1 | C | ใช้ ID ที่ไม่มีอยู่ | `404 Pet not found`; ไม่สร้าง Pet ใหม่ผ่าน PUT |
| PET-PUT-006 | P1 | C | ไม่ส่ง `name` | `422` Validation exception หรือ `400`; record เดิมต้องไม่เสีย |
| PET-PUT-007 | P1 | C | ไม่ส่ง `photoUrls` | `422` หรือ `400`; record เดิมต้องไม่เสีย |
| PET-PUT-008 | P1 | C | status นอก enum | `422` หรือ `400`; GET ยังคืน status เดิม |
| PET-PUT-009 | P2 | E | ส่งเฉพาะ `id`, `name`, `photoUrls` | `200` ตาม minimal valid schema; ตรวจว่า optional fields ถูกล้างหรือคงเดิมและบันทึก semantics |
| PET-PUT-010 | P2 | F | ส่ง payload เดิมซ้ำสองครั้ง | ทั้งสองครั้งต้องให้ final state เหมือนกัน; ไม่เกิด duplicate/side effect เพิ่ม |
| PET-PUT-011 | P1 | C | body ว่าง, malformed JSON | `400/422`; record เดิมต้องไม่เปลี่ยน |
| PET-PUT-012 | P1 | S | ไม่มี token / token ไม่มี scope | `401/403`; record เดิมต้องไม่เปลี่ยน |
| PET-PUT-013 | P2 | C | JSON, XML และ form-urlencoded ด้วยข้อมูลสมมูล | แต่ละ supported content type ทำงานสอดคล้องกันและ response ตรง schema |
| PET-PUT-014 | P2 | E | ID เกิน int64 เช่น `9223372036854775808` | `400/422`; ไม่ overflow, wrap หรืออัปเดต record อื่น |

### 5.3 `GET /pet/findByStatus`

Precondition: มี Pet ที่รู้จักอย่างน้อยหนึ่งตัวต่อ status หรือสร้างด้วย `POST /pet` ก่อน

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-STATUS-001 | P0 | C,F | `?status=available` | `200`; JSON array; ทุกรายการมี `status=available`; Pet ที่เตรียมไว้ถูกพบ |
| PET-STATUS-002 | P1 | C,F | `pending` และ `sold` แยก iteration | `200`; ทุก element ตรง status ที่ค้นหา |
| PET-STATUS-003 | P1 | C | ไม่ส่ง query `status` | `400 Invalid status value` เพราะ parameter required |
| PET-STATUS-004 | P1 | C | `status=invalid`, `AVAILABLE`, empty | `400`; ไม่คืนรายการแบบไม่ filter |
| PET-STATUS-005 | P2 | C | status ที่ถูกต้องแต่ไม่มีข้อมูล | `200` และ `[]`; ไม่ควร `404` |
| PET-STATUS-006 | P2 | E | หลายค่าแบบ `status=available,pending` | ตรวจ behavior ตามคำอธิบาย comma-separated; ทุกผลต้องอยู่ในชุดที่ขอ; หาก reject ให้บันทึกความขัดแย้งระหว่าง description กับ schema string enum |
| PET-STATUS-007 | P2 | E | query ซ้ำ `status=available&status=sold` | เก็บ actual serialization behavior; ห้ามมี status อื่นและห้าม `5xx` |
| PET-STATUS-008 | P1 | S | ไม่มี auth / api key อย่างเดียว / OAuth read scope | OAuth ที่ถูกต้องควร `200`; no auth ควร `401/403`; บันทึกหาก sample ไม่ enforce |
| PET-STATUS-009 | P2 | C | `Accept: application/xml` | `200`; XML array parse ได้และทุกรายการตรง filter |

### 5.4 `GET /pet/findByTags`

Precondition: สร้าง Pet ที่มี tag เฉพาะ run เช่น `qa-{{runId}}`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-TAG-001 | P0 | C,F | `?tags=qa-{{runId}}` | `200`; array; Pet ที่เตรียมไว้ถูกพบ; แต่ละผลมี tag ที่ค้นหา |
| PET-TAG-002 | P1 | C | หลาย tags ตาม serialization ที่ Swagger UI สร้าง | `200`; ผลต้อง match อย่างน้อยหนึ่ง/ตาม semantics ที่ APIใช้และบันทึก OR/AND ให้ชัด |
| PET-TAG-003 | P1 | C | ไม่ส่ง `tags` | `400 Invalid tag value` |
| PET-TAG-004 | P1 | E | `tags=` ว่าง | ควร `400`; หาก `200` ต้องไม่กลายเป็น unfiltered data leak |
| PET-TAG-005 | P2 | C | tag ที่ไม่มีข้อมูล | `200` และ empty array |
| PET-TAG-006 | P2 | E | tag มี space, Thai, emoji และ URL-encoded special chars | ไม่เกิด `5xx`; encoding ถูก decode ครั้งเดียว; ผลที่รับต้องตรง tag จริง |
| PET-TAG-007 | P2 | S | SQL/script-like string เช่น `' OR 1=1 --` และ `<script>` | ไม่เกิด `5xx`, ไม่คืนข้อมูลทั้งหมด, ไม่ execute/reflect แบบอันตราย |
| PET-TAG-008 | P1 | S | ไม่มี auth / OAuth scope ไม่ถูกต้อง | ควร `401/403`; valid OAuth ควร `200` |
| PET-TAG-009 | P2 | C | `Accept: application/xml` | XML parse ได้และผลตรง tag filter |

### 5.5 `GET /pet/{petId}`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-GET-001 | P0 | C,F | ID ของ Pet ที่สร้างไว้ | `200`; response เป็น Pet; ID และข้อมูลตรง persisted state |
| PET-GET-002 | P1 | C | ID ที่ไม่มีอยู่ | `404 Pet not found` |
| PET-GET-003 | P1 | C | `petId=abc`, decimal, whitespace | `400 Invalid ID supplied`; ไม่เกิด `5xx` |
| PET-GET-004 | P2 | E | `petId=0`, `-1` | หากไม่มีให้ `404`; หากมีก็ `200`; ห้าม overflow หรือ `5xx` |
| PET-GET-005 | P2 | E | int64 min/max | ไม่เกิด precision loss/overflow; `200` เมื่อมีจริง มิฉะนั้น `404` |
| PET-GET-006 | P1 | C | ID เกิน int64 | `400`; ไม่ wrap เป็น ID อื่น |
| PET-GET-007 | P1 | S | ส่ง valid `api_key` | `200` ตาม OR security alternative |
| PET-GET-008 | P1 | S | ส่ง valid OAuth read token | `200` |
| PET-GET-009 | P1 | S | ไม่มี credential และ credential ผิด/หมดอายุ | `401/403`; response ไม่เผย Pet; บันทึกหาก sample ไม่ enforce |
| PET-GET-010 | P2 | C | `Accept: application/json` และ `application/xml` | `200`; content type/schema ตรง Accept และข้อมูลสมมูลกัน |
| PET-GET-011 | P2 | E | URL path injection/encoded slash | `400/404`; ไม่ route ไป resource อื่นและไม่เกิด `5xx` |

### 5.6 `POST /pet/{petId}` - Update with query/form-style data

Precondition: มี Pet เป้าหมาย; endpoint นี้ประกาศ `name` และ `status` เป็น query parameters

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-FORM-001 | P0 | C,F | `?name=QA-Renamed&status=pending` | `200`; response/GET มีชื่อและ status ใหม่; field อื่นไม่เสีย |
| PET-FORM-002 | P1 | C,F | ส่งเฉพาะ `name` | `200`; name เปลี่ยน; status เดิมคงอยู่ |
| PET-FORM-003 | P1 | C,F | ส่งเฉพาะ `status` | `200`; status เปลี่ยน; name เดิมคงอยู่ |
| PET-FORM-004 | P2 | E | ไม่ส่งทั้ง name และ status | บันทึก actual; ไม่ควรทำข้อมูลเสียหรือ `5xx`; GET ต้องยังอ่านได้ |
| PET-FORM-005 | P1 | C | `petId=abc` | `400 Invalid input`; ไม่มี Pet ถูกเปลี่ยน |
| PET-FORM-006 | P1 | E | ID ที่ไม่มีอยู่ | ควร `404`; Swagger operation ระบุเพียง `400/default`, จึงเป็น contract gap ถ้าผลไม่ชัด |
| PET-FORM-007 | P1 | E | `status=invalid` | ควร `400`; แม้ parameter schema ไม่กำหนด enum แต่ business schema Pet กำหนด status |
| PET-FORM-008 | P2 | E | name ว่าง/space/Thai/emoji/URL encoded | ไม่เกิด `5xx`; ค่าที่รับต้อง round-trip ถูกต้อง; blank acceptance เป็น gap |
| PET-FORM-009 | P1 | S | ไม่มี OAuth/ไม่มี write scope | `401/403`; state เดิมไม่เปลี่ยน |
| PET-FORM-010 | P2 | F | ส่ง update เดิมซ้ำ | final state เหมือนเดิม; ไม่เกิด side effect เพิ่ม |

### 5.7 `DELETE /pet/{petId}`

Precondition: สร้าง Pet เฉพาะสำหรับ delete เพื่อไม่ทำลาย test data ของผู้อื่น

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-DEL-001 | P0 | C,F | ลบ `petIdDelete` ที่มีอยู่ | `200 Pet deleted`; GET เดิมต้อง `404` |
| PET-DEL-002 | P1 | F | ลบ ID เดิมซ้ำ | ครั้งที่สองควร `404` หรือ non-2xx ที่สื่อว่าไม่มี; ห้าม `500` |
| PET-DEL-003 | P1 | E | ลบ ID ที่ไม่มีตั้งแต่แรก | ควร `404`; contract ไม่มี explicit 404 จึงเก็บ actual/contract gap |
| PET-DEL-004 | P1 | C | `petId=abc`, decimal | `400 Invalid pet value` |
| PET-DEL-005 | P2 | E | `0`, negative, int64 max | ถ้าไม่มีควร `404`; ไม่ overflow/ลบ record อื่น |
| PET-DEL-006 | P1 | S | valid OAuth write scope | `200` เมื่อ record มีอยู่ |
| PET-DEL-007 | P1 | S | ไม่มี token/ไม่มี write scope/expired token | `401/403`; GET ยืนยัน Pet ยังอยู่ |
| PET-DEL-008 | P2 | S | ส่ง optional header `api_key` ที่ถูก/ผิด โดย OAuth เหมือนเดิม | authorization ต้องอิง security declaration อย่างสม่ำเสมอ; header นี้ไม่ควร bypass OAuth |
| PET-DEL-009 | P1 | F | Delete แล้วสร้าง ID เดิมใหม่ | Create ควรสำเร็จและ GET คืน record ใหม่ ไม่ใช่ข้อมูล stale |

### 5.8 `POST /pet/{petId}/uploadImage`

Precondition: มี Pet เป้าหมายและไฟล์ทดสอบที่ไม่มีข้อมูลส่วนบุคคล

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| PET-IMG-001 | P0 | C | upload PNG ขนาดเล็กด้วย `application/octet-stream` | `200`; response ตรง `ApiResponse` (`code/type/message` type ถูกต้อง) |
| PET-IMG-002 | P1 | C | upload JPEG ขนาดเล็ก | `200`; API ไม่ corrupt/timeout |
| PET-IMG-003 | P1 | C | ส่ง `additionalMetadata=qa-{{runId}}` | `200`; message/behavior สอดคล้องและไม่ทำ metadata เพี้ยน |
| PET-IMG-004 | P1 | C | ไม่มี file body | `400 No file uploaded` |
| PET-IMG-005 | P1 | C | ID ที่ไม่มีอยู่พร้อมไฟล์ | `404 Pet not found` |
| PET-IMG-006 | P1 | C | `petId=abc` | `400` หรือ non-2xx; ไม่เกิด `5xx` |
| PET-IMG-007 | P2 | E | ไฟล์ 0 byte | ควร `400`; หากรับต้องบันทึก validation gap |
| PET-IMG-008 | P2 | E | text/PDF renamed เป็น image | ควร reject หาก business ต้องการ image; contract รับ binary ทุกชนิด จึงเก็บ actual |
| PET-IMG-009 | P2 | E | ไฟล์ชื่อ/metadata มี Thai, space, emoji และ special chars | ไม่เกิด injection/encoding error; response ไม่สะท้อนข้อความแบบอันตราย |
| PET-IMG-010 | P2 | E | ไฟล์ขนาดใกล้ limit และเกิน limit | หา upload limit จาก actual; over-limit ต้อง reject แบบควบคุม ไม่ timeout/`5xx` |
| PET-IMG-011 | P1 | S | ไม่มี token/ไม่มี write scope | `401/403`; ไม่ผูกไฟล์กับ Pet |
| PET-IMG-012 | P2 | S | binary ที่มี signature/script test แบบปลอดภัย | ระบบไม่ execute content; response ไม่เผย storage path; เป็น security exploratory |

## 6. Store API Test Cases

### 6.1 `GET /store/inventory`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| STORE-INV-001 | P0 | C | ส่ง valid `api_key` | `200`; JSON object/map; value ทุกตัวเป็น int32 integer |
| STORE-INV-002 | P1 | C | ตรวจ key ที่เป็น status เช่น available/pending/sold เมื่อมี | count เป็นจำนวนเต็มและไม่ติดลบ |
| STORE-INV-003 | P1 | F | สร้าง Pet status ใหม่แล้วเรียก inventory ก่อน/หลัง | ตรวจแนวโน้ม count ตาม implementation; หาก API ถูกออกแบบให้ real-time count ต้องเพิ่มอย่างสอดคล้อง |
| STORE-INV-004 | P1 | S | ไม่ส่ง api_key | `401/403`; ไม่คืน inventory |
| STORE-INV-005 | P1 | S | api_key ผิด/ว่าง/ซ้ำหลาย header | `401/403`; ห้าม bypass |
| STORE-INV-006 | P2 | C | `Accept: application/xml` | contract มีเฉพาะ JSON; ควรคืน JSON ตาม capability หรือ `406`; ไม่คืน body/content-type ขัดกัน |
| STORE-INV-007 | P2 | E | เรียกซ้ำหลายครั้งโดยไม่เปลี่ยนข้อมูล | รูปแบบ schema คงที่; count ไม่แกว่งผิดเหตุผล; บันทึก shared-environment limitation |

### 6.2 `POST /store/order` - Place order

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| STORE-ADD-001 | P0 | C,F | Baseline Order อ้าง `petId` ที่สร้างไว้ | `200`; response เป็น Order; ทุก field ตรง request; GET order ด้วย ID เดียวกันสำเร็จ |
| STORE-ADD-002 | P1 | C | แต่ละ status: placed, approved, delivered | `200`; response คง enum ที่ส่ง |
| STORE-ADD-003 | P1 | C | status นอก enum/ตัวพิมพ์ใหญ่/ว่าง | `400` หรือ `422`; ไม่สร้าง order invalid |
| STORE-ADD-004 | P1 | C | `shipDate` ISO 8601 UTC ที่ถูกต้องและมี timezone offset | `200`; response parse เป็น date-time และสื่อ instant ถูกต้อง |
| STORE-ADD-005 | P1 | C | `shipDate` invalid เช่น `2026-02-30`, `not-a-date`, ไม่มี timezone | `400/422`; ไม่สร้าง order |
| STORE-ADD-006 | P2 | C,E | body `{}` | Contract อนุญาตเพราะไม่มี required property; หาก `200` response ต้องยังตรง Order schema; บันทึก business validation gap |
| STORE-ADD-007 | P2 | E | ไม่มี body | เก็บ actual เพราะ requestBody ไม่ marked required; ต้องไม่ `5xx` |
| STORE-ADD-008 | P1 | C | `quantity:"one"`, decimal, null | `400/422` สำหรับ type ผิด; ไม่สร้างข้อมูลเสีย |
| STORE-ADD-009 | P2 | E | quantity `0`, `-1`, int32 min/max | schema ไม่มี minimum; เก็บ actual; หากรับต้อง round-trip ถูกต้องและไม่ overflow |
| STORE-ADD-010 | P1 | C | quantity เกิน int32 | `400/422`; ไม่ wrap เป็นค่าติดลบ |
| STORE-ADD-011 | P1 | C | `complete:true` และ `false` | `200`; response boolean จริง ไม่ใช่ string |
| STORE-ADD-012 | P1 | C | `complete:"true"` | `400/422`; ไม่ coerce แบบไม่ชัดเจน |
| STORE-ADD-013 | P2 | E | `petId` ที่ไม่มีอยู่ | ควร reject หาก validate relation; contract ไม่ระบุ foreign-key behavior จึงเก็บ actual |
| STORE-ADD-014 | P2 | E | `id=0`, negative, int64 max | ไม่ overflow/`5xx`; หากรับ GET ต้องได้ ID เดิม |
| STORE-ADD-015 | P1 | C | malformed JSON / unsupported media type | `400/422` หรือ `415`; ไม่สร้าง order |
| STORE-ADD-016 | P2 | C | JSON, XML, form-urlencoded ด้วยข้อมูลสมมูล | supported request formats ทำงานสอดคล้องกัน; response JSON ตาม contract |
| STORE-ADD-017 | P1 | F | ส่ง Order ID เดิมซ้ำด้วยข้อมูลต่างกัน | บันทึก create/update/reject semantics; ห้ามเกิดหลายผลลัพธ์ที่ GET แยกไม่ได้ |
| STORE-ADD-018 | P2 | F | ส่ง payload เดิมซ้ำ | final state ต้อง deterministic; บันทึกว่า endpoint idempotent หรือไม่ |

### 6.3 `GET /store/order/{orderId}`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| STORE-GET-001 | P0 | C,F | orderId ที่สร้างไว้ | `200`; Order schema; ID และข้อมูลตรง create response |
| STORE-GET-002 | P1 | C | ID ที่ไม่มีอยู่ | `404 Order not found` |
| STORE-GET-003 | P1 | C | non-integer, decimal, whitespace | `400 Invalid ID supplied` |
| STORE-GET-004 | P2 | C | ทดสอบ ID `1..5` ที่มี/ไม่มี | ตาม description: valid range format; record ที่มี `200`, ไม่มีก็ `404` |
| STORE-GET-005 | P2 | C | ID `6..10` แยก iteration | description ระบุว่าจะ generate exceptions; ต้องเป็น controlled `400/404`, ไม่ใช่ `5xx`/stack trace |
| STORE-GET-006 | P2 | C | ID `11` และมากกว่า | format valid; ถ้ามี `200`, ไม่มี `404` ตาม description |
| STORE-GET-007 | P2 | E | `0` และ negative | ควร `404` หรือ `400`; ไม่เกิด `5xx` |
| STORE-GET-008 | P2 | E | int64 max และเกิน int64 | max: `200/404`; เกิน max: `400`; ไม่ overflow |
| STORE-GET-009 | P2 | C | `Accept: application/xml` | `200` เมื่อ order มี; XML schema/ค่าตรง JSON |
| STORE-GET-010 | P2 | S | path injection/encoded special chars | `400/404`; ไม่เปิดเผยข้อมูลอื่นหรือ internal error |

### 6.4 `DELETE /store/order/{orderId}`

Precondition: สร้าง Order สำหรับลบโดยเฉพาะ

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| STORE-DEL-001 | P0 | C,F | ลบ Order ที่มีอยู่และ ID `<1000` ตาม description | `200 order deleted`; GET หลังลบเป็น `404` |
| STORE-DEL-002 | P1 | F | ลบ ID เดิมซ้ำ | ครั้งที่สอง `404`; ไม่ `500` |
| STORE-DEL-003 | P1 | C | ID ที่ไม่มี | `404 Order not found` |
| STORE-DEL-004 | P1 | C | non-integer/decimal | `400 Invalid ID supplied` |
| STORE-DEL-005 | P1 | C | ID `1000` และ `1001` | ตรวจ boundary ตามคำอธิบาย `<1000`; `1001` ต้อง error แบบควบคุม; บันทึก behavior ของ `1000` ซึ่งข้อความไม่ชัด |
| STORE-DEL-006 | P2 | E | ID `0`, negative | `400/404`; ไม่ลบ order อื่น |
| STORE-DEL-007 | P2 | E | int64 max และเกิน int64 | max `404/controlled error`; เกิน max `400`; ไม่ overflow |
| STORE-DEL-008 | P1 | F | ลบ Order ต้องไม่ลบ Pet ที่ถูกอ้างอิง | GET Pet ยังคง `200` |

## 7. User API Test Cases

### 7.1 `POST /user` - Create user

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-ADD-001 | P0 | C,F | Baseline User ด้วย username ใหม่ | `200`; response เป็น User; GET username คืนข้อมูลที่สร้าง |
| USER-ADD-002 | P2 | C,E | body `{}` | Contract อนุญาตเพราะไม่มี required property; ไม่ควร `5xx`; หากรับให้บันทึก business/identity gap |
| USER-ADD-003 | P2 | E | ไม่มี body | requestBody ไม่ marked required; เก็บ actual; ห้าม `5xx` |
| USER-ADD-004 | P1 | C | type ผิด: `id:"abc"`, `userStatus:"active"` | ควร reject non-2xx; หากรับเป็น schema validation gap |
| USER-ADD-005 | P2 | E | email invalid/ว่าง/ไม่มี @ | contract เป็น string ธรรมดา; เก็บ actual; หากระบบต้องใช้อีเมลจริงถือเป็น validation gap |
| USER-ADD-006 | P2 | E | password ว่าง, 1 char, ยาวมาก | contract ไม่มี policy; ไม่เกิด `5xx`; ห้ามสะท้อน password ใน log/error ที่เห็นจาก response |
| USER-ADD-007 | P2 | E | username ว่าง/space/Thai/emoji/special char | ไม่เกิด encoding error; acceptance ต้อง deterministic; GET ด้วย encoded username ต้องตรง |
| USER-ADD-008 | P1 | F | username ซ้ำแต่ ID/ข้อมูลต่าง | ควร reject conflict หรือกำหนด update semantics ชัด; ห้ามมี identity กำกวม |
| USER-ADD-009 | P1 | F | ID ซ้ำแต่ username ต่าง | บันทึก uniqueness rule; GET ของแต่ละ username ต้องไม่คืนข้อมูลสลับกัน |
| USER-ADD-010 | P2 | E | userStatus `0`, negative, int32 max | ไม่มี enum/minimum; หากรับต้องคืนค่าเดิมไม่ overflow |
| USER-ADD-011 | P1 | C | userStatus เกิน int32 | ควร reject non-2xx; ไม่ wrap |
| USER-ADD-012 | P1 | C | malformed JSON/unsupported media type | `400` หรือ `415`/controlled error; ไม่สร้าง user |
| USER-ADD-013 | P2 | C | JSON, XML, form-urlencoded ด้วยข้อมูลสมมูล | supported formats ทำงานสอดคล้อง; GET คืนข้อมูลเท่ากัน |
| USER-ADD-014 | P2 | E,S | ส่ง unknown role/admin property | ต้องไม่ยกระดับสิทธิ์; ignore/reject ตาม implementation; ไม่ persist สิทธิ์ที่ contract ไม่มี |
| USER-ADD-015 | P1 | S | สร้าง user โดยไม่ได้ login ทั้งที่ description ระบุ logged-in only | ควร `401/403`; หาก `200` ให้บันทึก security implementation gap |

### 7.2 `POST /user/createWithList`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-LIST-001 | P0 | C,F | array ผู้ใช้ใหม่ 2 คน | `200`; response ตรง declared User schema; GET แต่ละ username ต้อง `200` และข้อมูลถูกคน |
| USER-LIST-002 | P1 | C | array 1 คน | `200`; user ถูกสร้าง |
| USER-LIST-003 | P2 | C,E | empty array `[]` | Contract อนุญาต; ควร `200` no-op หรือ controlled validation; ไม่ `5xx` |
| USER-LIST-004 | P1 | E | ไม่มี body | requestBody ไม่ required; เก็บ actual; ไม่ `5xx` |
| USER-LIST-005 | P1 | C | body เป็น object แทน array | ควร reject non-2xx; ไม่สร้างข้อมูลบางส่วน |
| USER-LIST-006 | P1 | C | array มี element `null`, string หรือ number | ควร reject; ตรวจว่าไม่มี partial create หรือบันทึก atomicity |
| USER-LIST-007 | P1 | F | array มี valid user และ invalid user | ควร atomic reject หรือระบุ partial behavior ชัด; GET ตรวจทุกรายการ |
| USER-LIST-008 | P1 | F | username ซ้ำกันภายใน array | reject/resolve อย่าง deterministic; ไม่สร้าง identity กำกวม |
| USER-LIST-009 | P1 | F | หนึ่ง username ชน user เดิม | ตรวจ atomicity และ conflict behavior; user เดิมต้องไม่เสีย |
| USER-LIST-010 | P2 | E | array ใหญ่ขึ้นตาม incremental sizes เช่น 10, 100 | ไม่ timeout/`5xx`; หา practical limit โดยไม่ทำ load test |
| USER-LIST-011 | P2 | C | ตรวจความแปลกของ contract: request เป็น array แต่ response เป็น User เดียว | Actual response ต้อง validate ตาม declared schema; บันทึก contract design gap หากไม่สื่อผลทุก user |

### 7.3 `GET /user/login`

Precondition: สร้าง User และทราบ username/password; ห้ามใช้ credential จริง

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-LOGIN-001 | P0 | C,F | username/password ถูกต้อง | `200`; body เป็น string; มี `X-Rate-Limit` เป็น int32 และ `X-Expires-After` เป็น date-time ตาม contract |
| USER-LOGIN-002 | P1 | C | username ถูก password ผิด | `400 Invalid username/password supplied`; ไม่คืน session/token สำเร็จ |
| USER-LOGIN-003 | P1 | C | username ไม่มีอยู่ | `400`; ไม่บอกข้อมูลละเอียดจนใช้ enumerate user ได้ |
| USER-LOGIN-004 | P1 | E | ไม่ส่ง username | parameter ไม่ required ตาม contract; เก็บ actual; ไม่ควร login สำเร็จ |
| USER-LOGIN-005 | P1 | E | ไม่ส่ง password | เก็บ actual; ไม่ควร login สำเร็จ |
| USER-LOGIN-006 | P1 | E | ไม่ส่งทั้งคู่ | ไม่ควร login สำเร็จ; controlled `400`; ไม่ `5xx` |
| USER-LOGIN-007 | P2 | E | username/password ว่างและ whitespace | ไม่ควร authenticate; response ไม่เผย credential |
| USER-LOGIN-008 | P2 | S | SQL/script injection strings | ไม่ authenticate, ไม่ `5xx`, ไม่สะท้อน/execute input อันตราย |
| USER-LOGIN-009 | P2 | S | password มี reserved URL chars (`+`, `&`, `%`, `#`) โดย encode ถูกต้อง | valid credential ต้อง authenticateได้; server decode ครั้งเดียว |
| USER-LOGIN-010 | P2 | C | ตรวจ `X-Expires-After` | parse เป็น RFC3339/date-time, อยู่ในอนาคตเมื่อ login สำเร็จ |
| USER-LOGIN-011 | P2 | C | ตรวจ `X-Rate-Limit` | เป็น integer ไม่ติดลบ; header มีตาม contract |
| USER-LOGIN-012 | P2 | S | login ผิดซ้ำจำนวนเล็กน้อย | บันทึก rate-limit behavior โดยไม่ brute force; ไม่มี token/session หลุด |
| USER-LOGIN-013 | P2 | S | ตรวจ URL/query exposure | password ถูกส่งผ่าน query ตาม contract ซึ่งเสี่ยง log/history; บันทึกเป็น security/design observation ไม่ใส่ credential จริง |

### 7.4 `GET /user/logout`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-LOGOUT-001 | P0 | C,F | login แล้ว logout | `200`; session เดิมควรใช้กับ operation ที่ต้อง login ไม่ได้อีก ถ้ามี session enforcement |
| USER-LOGOUT-002 | P1 | E | logout โดยไม่เคย login | controlled response; ไม่ `5xx`; contract ระบุ `200/default` |
| USER-LOGOUT-003 | P2 | F | logout ซ้ำ | idempotent/safe; ไม่ `5xx` |
| USER-LOGOUT-004 | P2 | S | ใช้ session/token เดิมหลัง logout | ควร `401/403` ใน protected operation; ถ้ายังใช้ได้ให้บันทึก security gap |

### 7.5 `GET /user/{username}`

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-GET-001 | P0 | C,F | username ที่สร้างไว้ | `200`; User schema; username/fields ตรง persisted state |
| USER-GET-002 | P1 | C | username ไม่มีอยู่ | `404 User not found` |
| USER-GET-003 | P1 | C | username invalid ตาม server เช่น empty path/encoded invalid | `400 Invalid username supplied` หรือ `404`; ไม่ `5xx` |
| USER-GET-004 | P2 | E | username case ต่างจากที่สร้าง | บันทึกว่า case-sensitive หรือ insensitive; ต้อง deterministic |
| USER-GET-005 | P2 | E | Thai, emoji, space, slash/percent encoded | ค่าที่สร้างได้ต้องดึงได้ด้วย encoding ถูกต้อง; ไม่ route ผิด/`5xx` |
| USER-GET-006 | P1 | S | ตรวจ response ไม่เปิดเผย password | แม้ User schema มี password ระบบที่ปลอดภัยไม่ควรคืน plaintext; หากคืนให้บันทึก High security/design issue |
| USER-GET-007 | P1 | S | user A ขอข้อมูล user B โดยไม่มี auth | ควร `401/403` หากข้อมูลเป็น private; description ไม่มี security declaration จึงเป็น security gap assessment |
| USER-GET-008 | P2 | C | `Accept: application/xml` | `200`; XML parse ได้และข้อมูลสมมูล JSON |

### 7.6 `PUT /user/{username}`

Precondition: สร้าง User เป้าหมายและ login ตามคำอธิบาย endpoint

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-PUT-001 | P0 | C,F | เปลี่ยน firstName, lastName, email, phone, status | `200`; GET username เดิมสะท้อนทุกค่าใหม่ |
| USER-PUT-002 | P1 | F | เปลี่ยน username ใน body แต่ path เป็น username เดิม | บันทึก rename semantics; ต้องไม่เหลือ identity ซ้ำ/ข้อมูลเข้าถึงไม่ได้ |
| USER-PUT-003 | P1 | C | username path ไม่มีอยู่ | `404 user not found`; ห้าม upsert โดยไม่ระบุ |
| USER-PUT-004 | P1 | C | body type ผิด/malformed JSON | `400 bad request`; user เดิมไม่เปลี่ยน |
| USER-PUT-005 | P2 | C,E | body `{}` | contract อนุญาต fields optional; ไม่ควร `5xx`; ตรวจว่าไม่ล้างข้อมูลโดยไม่ตั้งใจ |
| USER-PUT-006 | P2 | E | email/password invalid/ว่าง | เก็บ actual ตาม contract gap; GET/login ตรวจผลและ data integrity |
| USER-PUT-007 | P1 | C | userStatus เกิน int32 | `400`; ค่าเดิมไม่เปลี่ยน |
| USER-PUT-008 | P1 | S | ไม่ login หรือ login เป็น user อื่น | `401/403` ตาม description logged-in only; state ไม่เปลี่ยน |
| USER-PUT-009 | P2 | F | payload เดิมซ้ำ | final state เหมือนเดิม; ไม่สร้าง duplicate |
| USER-PUT-010 | P2 | C | JSON, XML, form-urlencoded สมมูล | supported formats ให้ final state เดียวกัน |
| USER-PUT-011 | P2 | S | พยายามเพิ่ม unknown role/admin field | ไม่ยกระดับสิทธิ์และไม่ persist field นอก contract |

### 7.7 `DELETE /user/{username}`

Precondition: สร้างและใช้ username สำหรับลบโดยเฉพาะ

| ID | P | Class | Scenario / Test data | Expected result |
| --- | --- | --- | --- | --- |
| USER-DEL-001 | P0 | C,F | ลบ username ที่มีอยู่ขณะ login ถูก user | `200 User deleted`; GET หลังลบ `404`; login เดิมไม่สำเร็จ |
| USER-DEL-002 | P1 | F | ลบ username เดิมซ้ำ | ครั้งที่สอง `404`; ไม่ `500` |
| USER-DEL-003 | P1 | C | username ไม่มี | `404 User not found` |
| USER-DEL-004 | P1 | C | invalid/empty encoded username | `400 Invalid username supplied` หรือ controlled `404`; ไม่ `5xx` |
| USER-DEL-005 | P1 | S | ไม่ login | `401/403`; user ยังอยู่ |
| USER-DEL-006 | P1 | S | login เป็น user A แล้วลบ user B | `403`; user B ยังอยู่ |
| USER-DEL-007 | P2 | E | case variation ของ username | ทำตาม case sensitivity ที่ค้นพบ; ห้ามลบคนผิด |
| USER-DEL-008 | P1 | F | ลบ user ต้องไม่ลบ Pet/Order ที่ ID บังเอิญเท่ากัน | resources คนละ domain ยังอยู่ |

## 8. Cross-cutting and End-to-End Test Cases

| ID | P | Class | Scenario / Steps | Expected result |
| --- | --- | --- | --- | --- |
| E2E-001 | P0 | F | Create User -> Login -> Create Pet -> GET Pet -> Create Order -> GET Order | ทุก step สำเร็จ; IDs เชื่อมกัน; response data ตรง request |
| E2E-002 | P0 | F | Create Pet available -> update pending -> update sold -> query each status | Pet ปรากฏเฉพาะ state ล่าสุดและไม่ค้างใน status เก่า |
| E2E-003 | P0 | F | Create Pet -> update full body -> update form -> GET | final state รวมการเปลี่ยนล่าสุดโดย field ที่ไม่ได้แก้ไม่สูญหาย |
| E2E-004 | P0 | F | Create Pet -> upload image -> delete Pet -> GET | upload สำเร็จ; delete สำเร็จ; GET เป็น `404` |
| E2E-005 | P0 | F | Create Order -> GET -> DELETE -> GET | ก่อนลบข้อมูลตรง create; หลังลบ `404` |
| E2E-006 | P0 | F | Create User -> GET -> PUT -> GET -> DELETE -> GET | CRUD state ถูกต้องทุกช่วง; หลังลบ `404` |
| E2E-007 | P1 | F | ลบ Pet ที่มี Order อ้างอิง | บันทึก referential behavior; Order ต้องไม่กลายเป็น response เสีย/`5xx` |
| E2E-008 | P1 | F | สร้างข้อมูลด้วย runId สองชุด | การแก้/ลบชุด A ต้องไม่กระทบชุด B |
| E2E-009 | P1 | F | Collection rerun ด้วย runId ใหม่ | ไม่ชนข้อมูลเดิม; test independent และ repeatable |
| E2E-010 | P1 | F | Cleanup ลำดับ Order -> Pet -> User | ทุก resource ที่สร้างโดย test ถูกลบหรือรายงาน cleanup failure ชัดเจน |
| API-COMMON-001 | P1 | C | ใช้ HTTP method ผิดกับ endpoint เช่น PATCH/OPTIONS ที่ไม่ประกาศ | `405` หรือ controlled `404`; ไม่ mutate data |
| API-COMMON-002 | P1 | C | URL/resource path สะกดผิด | `404`; ไม่คืน Swagger HTML พร้อม `200` แทน API error |
| API-COMMON-003 | P1 | E | `Accept: application/json` และ unsupported Accept | supported type ถูกต้อง; unsupported ควร `406` หรือ documented fallback |
| API-COMMON-004 | P1 | E | duplicate/conflicting `Content-Type`, `Accept`, `api_key` headers | reject หรือเลือก deterministic; ห้าม security bypass/request smuggling behavior |
| API-COMMON-005 | P1 | S | ตรวจ error ทุกประเภท | ไม่มี stack trace, framework version, filesystem path, credential หรือ DB detail |
| API-COMMON-006 | P2 | S | input มี CRLF เช่น `%0d%0aX-Test: injected` | ไม่สร้าง response header ใหม่และไม่เกิด `5xx` |
| API-COMMON-007 | P2 | S | JSON string มี `<script>`, quotes, backslash | จัดเก็บ/คืนเป็น data เท่านั้น; JSON escape ถูกต้อง; ไม่ execute |
| API-COMMON-008 | P2 | S | very long path/query/string แบบจำกัดขนาด | controlled `4xx`; ไม่ timeout/`5xx`; บันทึก limit โดยไม่ทำ DoS |
| API-COMMON-009 | P2 | E | request มี unknown query parameter | ignore/reject อย่าง deterministic; parameter ต้องไม่เปลี่ยนผลลัพธ์โดยไม่ตั้งใจ |
| API-COMMON-010 | P2 | C | ตรวจ JSON number types | integer ต้องไม่มีทศนิยม/quote; boolean ต้องเป็น true/false; null ตาม schema |
| API-COMMON-011 | P2 | C | ตรวจ schema nested/array ในทุก 200 response | object/array/property type ตรง `Pet`, `Order`, `User`, `ApiResponse` |
| API-COMMON-012 | P2 | E | เรียก GET เดิมซ้ำโดยไม่ mutate | response data เท่ากัน ยกเว้น shared data เปลี่ยนจากผู้ใช้รายอื่น; GET ไม่มี side effect |
| API-COMMON-013 | P2 | E | ตรวจ response time ของแต่ละ endpoint 3 รอบ | บันทึก min/avg/max เป็น baseline; ยังไม่ Fail เพราะ contract ไม่มี SLA |
| API-COMMON-014 | P1 | S | ตรวจ HTTPS และไม่เรียก HTTP ใน collection | ทุก request ใช้ HTTPS; ไม่ส่ง token/password ผ่าน plaintext transport |
| API-COMMON-015 | P1 | S | ตรวจ secret handling ใน Postman | token/api key/password อยู่ใน local/current value ไม่ commit ลง collection/evidence |

## 9. Recommended Execution Order

1. Contract smoke: `POST /pet`, `GET /pet/{id}`, `POST /store/order`, `GET /store/order/{id}`, `POST /user`, `GET /user/{username}`
2. Positive CRUD/functional flow ของแต่ละ resource
3. Required field, enum, type และ malformed-request negatives
4. Authentication/authorization cases
5. Boundary, encoding และ exploratory contract gaps
6. Cross-resource end-to-end cases
7. Cleanup cases และยืนยันว่า test data ถูกลบ

เพื่อหลีกเลี่ยงผลทดสอบไม่นิ่งบน public shared environment ทุก record ควรมี `runId`, ห้ามใช้ ID ตัวอย่าง `1` หรือ `10` สำหรับ Create/Update/Delete และต้องเก็บ Actual Response ไว้เป็น evidence เมื่อเริ่ม execution จริง สำหรับ Order delete flow ให้สุ่ม ID ช่วง `11..999`, สร้างแล้วตรวจ GET ทันที ก่อนลบในรอบเดียวกัน เพราะ Swagger ระบุว่า ID เหนือ `1000` จะเกิด API error และ environment นี้เป็นพื้นที่สาธารณะที่ข้อมูลอาจชนหรือถูกเปลี่ยนโดยผู้อื่นได้

## 10. Entry and Exit Criteria

Entry criteria:

- Swagger/OpenAPI เปิดได้
- Postman environment มี `baseUrl` และ test credentials/key ที่ได้รับอนุญาต
- กำหนด unique `runId`
- Public service ไม่อยู่ในช่วง outage/rate-limit

Exit criteria:

- P0/P1 ที่อยู่ใน scope ถูกรันครบ
- ทุก case มี Actual Result, status และ evidence
- ทุก defect มี request, response, timestamp, environment และ steps to reproduce
- Cleanup test data เสร็จ หรือมีรายการ residual data ชัดเจน
- Contract gap แยกจาก implementation defect และ test-data/environment issue
