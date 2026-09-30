# LogiNet — Cấu trúc dự án và hướng mở rộng

Hệ thống quản lý mạng lưới kho phân tán. Tài liệu này mô tả cách tổ chức mã nguồn và lộ trình mở rộng từ bản demo lên bản đầy đủ.

---

## 1. Đổi domain: từ cửa hàng bán lẻ sang trung tâm phân phối

| | Bản demo cũ | Bản mở rộng |
|---|---|---|
| Đơn vị | 3 cửa hàng tiện lợi | 3 trung tâm phân phối theo vùng |
| Site | Cầu Giấy, Hà Đông, Đà Nẵng | DC Bắc (Hà Nội), DC Trung (Đà Nẵng), DC Nam (TP.HCM) |
| Trung tâm | Trụ sở | Trung tâm điều phối (Control Tower) |
| Nghiệp vụ chính | Bán hàng tại quầy | Nhập từ nhà cung cấp, soạn và xuất theo đơn, điều chuyển liên kho |
| Chiều dữ liệu tồn | `(branch, product)` | `(dc, product, lot, bin)` |
| Quy mô | 5.000 SKU × 3 | 20.000 SKU × 3, có lô và vị trí |

Ba yêu cầu A/B/C của đề giữ nguyên. Domain logistics chỉ làm dữ liệu giàu hơn và cho nhiều chỗ mở rộng hơn.

**Nghiệp vụ mới có thêm:**

| Nghiệp vụ | Tính chất phân tán |
|---|---|
| Nhận hàng từ nhà cung cấp (receiving) | Cục bộ tại DC |
| Cất hàng vào vị trí (putaway) | Cục bộ, cần biết `zone/bin` |
| Soạn hàng theo đơn (picking) | Cục bộ, giữ chỗ tồn bằng `qty_reserved` |
| Xuất hàng (shipping) | Cục bộ |
| Điều chuyển liên kho | Xuyên site, cần 2PC |
| Kiểm kê chu kỳ (cycle count) | Cục bộ |
| Phân bổ đơn về DC nào | Trung tâm quyết, cần truy vấn phân tán |
| Báo cáo tồn toàn mạng lưới | Truy vấn phân tán |

**Thực thể mới so với bản demo:** `WAREHOUSE_ZONE`, `BIN`, `LOT` (số lô, hạn dùng), `SALES_ORDER` + dòng, `PICK_WAVE`, `SHIPMENT`, `CARRIER`.

Cột tồn kho đổi từ `(branch_id, product_id)` thành `(dc_id, product_id, lot_id, bin_id)`. Khoá phân mảnh vẫn là `dc_id`, nên toàn bộ lập luận phân mảnh ở tài liệu thiết kế không phải viết lại.

---

## 2. Cây thư mục

```
loginet_ds/
├── ba/                           # phân tích nghiệp vụ và tài liệu nộp
│   ├── 01-requirements/          # đề bài, phạm vi, use case
│   ├── 02-design/                # tài liệu thiết kế, đặc tả API
│   ├── 03-diagrams/              # .drawio, .mermaid, ảnh xuất
│   └── 04-reports/               # báo cáo, slide, số đo benchmark
│
├── dev/
│   ├── db/
│   │   ├── migrations/           # versioned migration, chạy theo thứ tự
│   │   │   ├── node/             # lược đồ node DC (dùng chung cho cả 3)
│   │   │   ├── hq/               # lược đồ trung tâm
│   │   │   └── transit/          # kho ảo TRANSIT
│   │   ├── scripts/              # FDW, publication/subscription, bảo trì
│   │   └── seed/                 # sinh dữ liệu mẫu
│   │
│   ├── services/
│   │   ├── common/               # thư viện dùng chung giữa các service
│   │   ├── node_api/             # chạy tại mỗi DC
│   │   │   └── routers/          # receiving, picking, shipping, transfer, inventory
│   │   ├── hq_api/               # trung tâm điều phối
│   │   │   ├── routers/          # aggregator, allocation, transfer, admin
│   │   │   └── workers/          # sink consumer, reconciler, recovery
│   │   └── cdc_sink/             # tách riêng khi cần scale consumer
│   │
│   ├── web/
│   │   ├── pos/                  # màn hình thao tác tại kho
│   │   ├── wms/                  # quản lý kho một DC
│   │   └── control_tower/        # giám sát toàn mạng lưới
│   │
│   ├── infra/
│   │   ├── docker/               # Dockerfile từng service
│   │   ├── compose/              # compose base + override theo môi trường
│   │   ├── debezium/             # connector config
│   │   └── observability/        # prometheus, grafana (tuỳ chọn)
│   │
│   ├── .env.example
│   └── docker-compose.yml        # trỏ tới infra/compose
│
├── test/
│   ├── unit/                     # logic thuần, không cần DB
│   ├── integration/              # cần DB thật, chạy trên compose
│   ├── chaos/                    # kill node, ngắt mạng, kill coordinator
│   └── load/                     # đo throughput và latency
│
├── ops/
│   ├── runbook/                  # xử lý sự cố: slot đầy, subscription kẹt
│   ├── demo/                     # script 5 kịch bản nghiệm thu
│   └── benchmark/                # script đo, kết quả thô
│
└── venv/
```

---

## 3. Vì sao tách như vậy

**`ba/` tách khỏi `dev/`** — tài liệu nộp và mã nguồn có vòng đời khác nhau. Thầy đọc `ba/`, người code đọc `dev/`.

**`migrations/` thay cho một file schema lớn** — hệ có bốn instance với ba lược đồ khác nhau, và lược đồ sẽ đổi nhiều lần trong quá trình làm. Đặt tên `V001__init.sql`, `V002__add_lot.sql`; mỗi lần đổi là thêm file mới, không sửa file cũ.

**`common/` là thư viện dùng chung** — kết nối DB, cấu hình, model Pydantic, logger, client gọi 2PC. Không có nó thì `node_api` và `hq_api` sẽ copy code của nhau.

**`node_api` chỉ nói chuyện với một DB, `hq_api` nói chuyện với nhiều DB.** Đây là ranh giới quan trọng nhất trong toàn hệ: mọi logic xử lý chuyện phân tán — node chết, timeout, giao dịch treo — chỉ nằm trong `hq_api`. Nếu thấy code xử lý phân tán lọt vào `node_api`, đó là dấu hiệu thiết kế sai.

**`workers/` tách khỏi `routers/`** — sink consumer và reconciler là tiến trình nền, không phải endpoint. Tách ra để sau này muốn chạy riêng thành container thì chỉ việc đổi entrypoint.

**`test/chaos/` là thư mục đặc thù của hệ phân tán.** Test thường kiểm tra hệ chạy đúng khi mọi thứ bình thường; chaos test kiểm tra hệ chạy đúng khi có thứ hỏng. Hai kịch bản ăn điểm nhất của đồ án đều nằm ở đây.

**`ops/` tách khỏi `test/`** — script chạy demo và runbook xử lý sự cố không phải là test, chúng được chạy bằng tay khi bảo vệ hoặc khi hệ có vấn đề.

---

## 4. Quy ước

| Hạng mục | Quy ước |
|---|---|
| Migration | `V<số thứ tự>__<mô tả>.sql`, không sửa file đã chạy |
| Service | Một thư mục một service, có `main.py` làm entrypoint |
| Biến môi trường | Khai trong `.env.example`, không commit `.env` |
| Tên container | `pg_dc1`, `api_dc1`, `pg_hq`, `api_hq`, `dbz`, `kafka` |
| Tên site trong code | `dc1`, `dc2`, `dc3`, `transit`, `hq` |
| Compose | `compose/base.yml` + `compose/dev.yml` + `compose/demo.yml`, ghép bằng `-f` |
| Test | `pytest`, đặt cạnh loại test tương ứng, không trộn unit với integration |

---

## 5. Lộ trình mở rộng theo giai đoạn

Không làm hết một lượt. Mỗi vòng đều phải chạy được trước khi sang vòng sau.

**Vòng 1 — Bản lõi.** Đúng phạm vi bản demo: 3 node, tồn kho theo `(dc_id, product_id)`, FDW, CDC, 2PC. Đây là bản đủ điểm ba yêu cầu.

**Vòng 2 — Chiều dữ liệu logistics.** Thêm `lot` và `bin` vào khoá tồn kho, thêm nghiệp vụ receiving và putaway. Phần phân mảnh không đổi vì khoá vẫn là `dc_id`.

**Vòng 3 — Đơn hàng và phân bổ.** Thêm `sales_order`, `pick_wave`, và logic trung tâm quyết định đơn được phân về DC nào. Đây là chỗ truy vấn phân tán trở nên có ý nghĩa thật: phải hỏi cả ba DC để biết kho nào có đủ hàng và gần khách nhất.

**Vòng 4 — Vận hành.** Observability, giám sát replication slot, cảnh báo subscription kẹt, benchmark có số liệu.

**Vòng 5 — Mở rộng quy mô.** Thêm DC thứ tư để chứng minh việc mở rộng chỉ cần thêm một mảnh và một node. Đo lại độ trễ truy vấn toàn mạng lưới khi số node tăng.

Vòng 3 là vòng đáng làm nhất nếu muốn đồ án nổi bật, vì nó tạo ra một truy vấn phân tán có giá trị nghiệp vụ thật chứ không chỉ để báo cáo.

---

## 6. Chuyển từ codebase cũ sang

| File cũ | Vị trí mới |
|---|---|
| `db/branch-schema.sql` | `dev/db/migrations/node/V001__init.sql` |
| `db/hq-schema.sql` | `dev/db/migrations/hq/V001__init.sql` |
| `db/transit-schema.sql` | `dev/db/migrations/transit/V001__init.sql` |
| `db/hq-fdw.sql` | `dev/db/scripts/setup_fdw.sql` |
| `services/api_cn.py` | tách thành `dev/services/node_api/main.py` + `routers/` |
| `services/api_hq.py` | tách thành `dev/services/hq_api/main.py` + `routers/` + `workers/` |
| `services/seed.py` | `dev/db/seed/seed.py` |
| `scripts/setup.sh` | `ops/runbook/setup.sh` |
| `scripts/demo.sh` | `ops/demo/` tách thành 5 file riêng |
| `docker-compose.yml` | `dev/infra/compose/base.yml` |
| `debezium/connector-template.json` | `dev/infra/debezium/` |

Việc tách `api_cn.py` và `api_hq.py` thành router nên làm ngay ở vòng 1, khi file còn ngắn. Để tới lúc mỗi file 800 dòng thì tách rất mệt.

---

## 7. Đổi tên trong mã nguồn

| Cũ | Mới |
|---|---|
| `branch_id` | `dc_id` |
| `BRANCH` | `DISTRIBUTION_CENTER` |
| `api_cn*`, `pg_cn*` | `api_dc*`, `pg_dc*` |
| `minigo-net` | `loginet-net` |
| Chi nhánh | Trung tâm phân phối |

Giữ nguyên: `TRANSIT`, `inventory`, `stock_movement`, `outbox`, `inventory_snapshot`, `transfer`, `transfer_log`. Những tên này đã đúng nghĩa trong domain logistics.
