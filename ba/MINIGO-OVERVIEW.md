# MiniGo — Hệ thống quản lý kho phân tán nhiều chi nhánh

Tài liệu đặc tả đầy đủ để xây dựng hệ thống. Đọc hết trước khi sinh code.

---

## 1. Bối cảnh và mục tiêu

Đồ án môn Phát triển hệ thống phân tán. Chuỗi cửa hàng tiện lợi MiniGo: 3 chi nhánh (Cầu Giấy, Hà Đông, Đà Nẵng), trụ sở tại Hà Nội, 5.000 mã hàng mỗi chi nhánh, tổng 15.000 dòng tồn kho.

Bài toán trung tâm: 15.000 dòng tồn kho đó nằm ở đâu, và ghép lại thế nào khi cần nhìn toàn cảnh.

Ba yêu cầu bắt buộc của đề:

| | Yêu cầu | Bằng chứng phải trình được |
|---|---|---|
| A | Tồn kho phân đoạn theo chi nhánh, đặt tại node riêng | 3 container PostgreSQL độc lập, mỗi con chỉ chứa dòng của chi nhánh mình |
| B | Dịch vụ tổng hợp trung tâm truy vấn từ nhiều node | `EXPLAIN (VERBOSE)` cho thấy `SUM` được đẩy xuống từng node |
| C | Đồng bộ định kỳ hoặc theo sự kiện | Ngắt mạng một node, bán offline, nối lại thì số liệu trung tâm tự khớp |

Làm thêm giao dịch hai pha (2PC) cho nghiệp vụ điều chuyển hàng — ngoài đề nhưng là phần ăn điểm cộng lớn nhất.

**Ngoài phạm vi, không làm:** kế toán giá vốn FIFO/LIFO, quản lý khách hàng, khuyến mãi, nhân sự, vận đơn giao hàng.

**Đánh đổi CAP đã chọn:** ưu tiên tính sẵn sàng cho bán hàng cục bộ. Mất mạng thì chi nhánh vẫn bán, báo cáo trung tâm chấp nhận trễ. Trong phạm vi một node vẫn nhất quán mạnh.

---

## 2. Hai loại dữ liệu ứng xử ngược nhau

Đây là cơ sở của toàn bộ thiết kế:

| | Tồn kho (`qty_on_hand`) | Danh mục sản phẩm |
|---|---|---|
| Ghi ở đâu | Tại chi nhánh | Tại trung tâm |
| Cách xử lý | Phân mảnh, mỗi mảnh một bản chính duy nhất | Nhân bản xuống mọi site |
| Chiều dữ liệu | Chi nhánh → trung tâm (CDC qua Kafka) | Trung tâm → chi nhánh (logical replication) |
| Vì sao | Hai nơi cùng trừ tồn thì sai số | Đọc nhiều ghi ít, bản sao vô hại |

---

## 3. Kiến trúc và tech stack

11 container, Docker Compose, chạy trên một máy.

| Nhóm | Container | Vai trò |
|---|---|---|
| Chi nhánh ×3 | `pg_cn1/2/3`, `api_cn1/2/3` | Mỗi chi nhánh một PostgreSQL giữ mảnh riêng và một FastAPI |
| Bắt thay đổi | `dbz` | Một Debezium Connect chạy 3 connector, đọc WAL của 3 node |
| Hàng đợi | `kafka`, `kafka-ui` | Giữ sự kiện tồn kho; giao diện xem sự kiện khi demo |
| Trung tâm | `pg_hq`, `api_hq`, `metabase` | Danh mục, snapshot, kho ảo TRANSIT; dịch vụ tổng hợp; báo cáo |

Ba `api_cn` dùng chung một image, khác nhau bằng biến môi trường `BRANCH_ID`.

| Vai trò | Công nghệ | Lý do chọn |
|---|---|---|
| CSDL mọi node | PostgreSQL 16 | Có sẵn `postgres_fdw` và 2PC thật, không mô phỏng bằng code |
| Truy vấn phân tán | `postgres_fdw` + view `UNION ALL` | Chứng minh pushdown bằng query plan |
| Giao dịch xuyên site | `PREPARE TRANSACTION` | Giao thức gốc của PostgreSQL |
| CDC | Debezium qua plugin `pgoutput` | Không phải sửa ứng dụng để phát sự kiện |
| Hàng đợi | Apache Kafka | Trung tâm chết thì sự kiện vẫn nằm chờ |
| API | FastAPI + psycopg3 | Nhẹ, chạy được tại node kể cả khi mất WAN |
| Báo cáo | Metabase | Cắm vào PostgreSQL là chạy |
| Triển khai | Docker Compose | Ngắt mạng từng node bằng `docker network disconnect` |

**Đã cân nhắc rồi loại:** StarRocks (quá nặng cho 15.000 dòng), Citus và YugabyteDB (công cụ lo hết phần phân mảnh, mất nội dung thiết kế), Superset (nhiều thành phần phụ trợ hơn mức cần), nginx trước mỗi chi nhánh (không có giá trị khi chạy một máy).

---

## 4. Bốn instance PostgreSQL

| Container | Database | Nội dung |
|---|---|---|
| `pg_cn1` | `kho` | Mảnh I₁: 5.000 dòng tồn Cầu Giấy, outbox, stock_movement, chứng từ |
| `pg_cn2` | `kho` | Mảnh I₂: Hà Đông, cùng cấu trúc |
| `pg_cn3` | `kho` | Mảnh I₃: Đà Nẵng, cùng cấu trúc |
| `pg_hq` | `kho` + `transit` | `kho`: danh mục, `inventory_snapshot`, `transfer_log`. `transit`: mảnh I_TRANSIT |

**Instance khác database.** Bốn container là bốn tiến trình PostgreSQL riêng, bốn cổng, bốn volume — đây mới là thứ Yêu cầu A đòi hỏi. Bốn database trong một instance thì vẫn là một điểm chết đơn lẻ.

TRANSIT là database riêng trong `pg_hq`, không cần container riêng: kết nối tới nó vẫn độc lập nên `PREPARE TRANSACTION` vẫn là 2PC thật.

---

## 5. Lược đồ dữ liệu

### Node chi nhánh (`pg_cn1/2/3`, database `kho`)

```
-- master data: ban sao chi doc, nhan tu pg_hq qua logical replication
-- KHONG dat khoa ngoai giua cac bang master o node, de replication khong bi chan
category(category_id PK, name)
uom(uom_id PK, code)
supplier(supplier_id PK, name, phone)
branch(branch_id PK, name, region, branch_type)        -- STORE | DC | TRANSIT
product(product_id PK, sku, name, category_id, uom_id, is_active)

-- ton kho: manh A(i), ban chinh duy nhat tai node nay
inventory(inventory_id PK, branch_id, product_id,
          qty_on_hand, qty_reserved, version,
          UNIQUE(branch_id, product_id),
          CHECK qty_on_hand >= 0)

-- chinh sach ton: manh doc B(i)
inventory_policy(inventory_id PK FK, min_level, max_level, reorder_point, bin_location)

-- chung tu: phan manh phai sinh theo branch_id
stock_movement(movement_id PK, inventory_id FK, branch_id, product_id,
               move_type, qty_delta, qty_after, ref_id, created_at)
               -- move_type: SALE | RECEIPT | ADJUST | TRANSFER_OUT | TRANSFER_IN
goods_receipt(receipt_id PK, branch_id, supplier_id, status, created_at)
goods_issue(issue_id PK, branch_id, issue_type, created_at)

-- ha tang dong bo
outbox(event_id PK, aggregate, aggregate_key, payload JSONB, created_at)
       -- event_id = movement_id, dung lam idempotency key
       -- aggregate_key = "branch_id:product_id" -> khoa phan hoach Kafka
sync_watermark(branch_id PK, last_movement_id, last_sync_at)
```

### Trung tâm (`pg_hq`, database `kho`)

```
-- master data ban chinh + PUBLICATION master_data
category, uom, supplier, branch, product        -- cung cau truc, co khoa ngoai day du

-- duong nguoi
inventory_snapshot(branch_id, product_id, qty_on_hand, last_movement, updated_at,
                   PRIMARY KEY(branch_id, product_id))
reconcile_log(id PK, branch_id, checked_rows, diff_rows, variance_pct, ran_at)

-- dieu chuyen xuyen site
transfer(transfer_id PK, from_branch_id, to_branch_id, product_id, qty, status,
         created_at, updated_at)
         -- status: DRAFT | IN_TRANSIT | RECEIVED | CANCELLED | DISPUTED
transfer_log(id PK, transfer_id, leg, phase, decision, gids JSONB, decided_at)
         -- leg: OUT | IN ; phase: PREPARE | DECIDE | DONE ; decision: COMMIT | ROLLBACK
```

### `pg_hq`, database `transit`

```
inventory(inventory_id PK, branch_id DEFAULT 99, product_id, qty_on_hand, version, ...)
stock_movement(...)   -- cung cau truc node chi nhanh
```

### Ba cột cần chú ý

- `stock_movement` là sổ cái chỉ ghi thêm, không sửa. Vừa là nguồn đồng bộ, vừa là căn cứ đối chiếu.
- `inventory.version` phục vụ khoá lạc quan, chống hai giao dịch cùng trừ tồn.
- `transfer.status` là máy trạng thái, sinh ra nhu cầu 2PC.

### Máy trạng thái phiếu điều chuyển

```
DRAFT ──(2PC lần 1)──> IN_TRANSIT ──(2PC lần 2)──> RECEIVED
  │                        │
  └─> CANCELLED            └─> DISPUTED
```

**Phân biệt hai tên gần giống nhau:** `IN_TRANSIT` là giá trị cột `transfer.status` (trạng thái tờ phiếu). `TRANSIT` là chi nhánh ảo `branch_id = 99` có tồn kho riêng, nằm ở database `transit`.

---

## 6. Yêu cầu A — Phân mảnh

**Bước 1, ngang theo `branch_id`:** 15.000 dòng cắt thành I₁, I₂, I₃ mỗi mảnh 5.000 dòng, cộng I_TRANSIT.

**Bước 2, dọc trên từng mảnh, tách theo tần suất ghi:**

| Mảnh dọc | Cột | Xử lý |
|---|---|---|
| Aᵢ — số lượng tồn | `inventory_id, branch_id, product_id, qty_on_hand, qty_reserved, version` | Ghi liên tục, một bản chính tại chi nhánh, không nhân bản |
| Bᵢ — chính sách tồn | `inventory_id, min_level, reorder_point, bin_location` | Đọc nhiều ghi ít, nhân bản lên trung tâm |

**Tái thiết:** Iᵢ = Aᵢ ⋈ Bᵢ trên `inventory_id`, INVENTORY = I₁ ∪ I₂ ∪ I₃ ∪ I_TRANSIT. Phép hợp này chính là view `global_inventory`.

Phân mảnh phái sinh: `stock_movement`, `goods_receipt`, `goods_issue` cắt theo cùng `branch_id` và đặt cùng node, để mọi phép nối là nối cục bộ.

**Bảng cấp phát mảnh × site:**

| Mảnh | CN1 | CN2 | CN3 | TRANSIT | HQ |
|---|---|---|---|---|---|
| A₁ / A₂ / A₃ | chính | chính | chính | — | — |
| A_TRANSIT | — | — | — | chính | — |
| B₁ B₂ B₃ | chính | chính | chính | — | bản sao |
| product, branch, supplier | bản sao | bản sao | bản sao | bản sao | chính |
| inventory_snapshot | — | — | — | — | chính |

---

## 7. Yêu cầu B — Truy vấn phân tán

Hai con đường, chọn theo mức ảnh hưởng của kết quả:

| | Đường nóng (`postgres_fdw`) | Đường nguội (snapshot) |
|---|---|---|
| Cơ chế | Hỏi thẳng cả 4 node, cộng tại chỗ | Đọc bảng đã cộng sẵn ở `pg_hq` |
| Độ chính xác | Tuyệt đối tại thời điểm hỏi | Trễ 1–5 giây |
| Node chết | Không trả lời được, `503` | Vẫn trả lời, kèm cờ `stale` |
| Dùng cho | Chi nhánh nào còn hàng để điều sang | Dashboard, cảnh báo dưới ngưỡng |

Cấu hình trên `pg_hq`:

```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER cn1 FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'pg_cn1', port '5432', dbname 'kho', use_remote_estimate 'true');
CREATE USER MAPPING FOR app SERVER cn1 OPTIONS (user 'app', password 'secret');
CREATE SCHEMA cn1_remote;
IMPORT FOREIGN SCHEMA public LIMIT TO (inventory, stock_movement, inventory_policy)
  FROM SERVER cn1 INTO cn1_remote;
-- lap lai cho cn2, cn3, transit (transit tro toi host pg_hq, dbname transit)

CREATE VIEW global_inventory AS
          SELECT * FROM cn1_remote.inventory
UNION ALL SELECT * FROM cn2_remote.inventory
UNION ALL SELECT * FROM cn3_remote.inventory
UNION ALL SELECT * FROM transit_remote.inventory;
```

`use_remote_estimate = true` là điều kiện để bộ tối ưu đẩy được phép tổng hợp xuống node.

Ba truy vấn phải chạy được:

```sql
-- Q1 aggregation pushdown
SELECT product_id, SUM(qty_on_hand) FROM global_inventory GROUP BY product_id;
-- Q2 predicate pushdown
SELECT branch_id, qty_on_hand FROM global_inventory WHERE product_id = 1234;
-- Q3 join phan tan
SELECT g.branch_id, g.product_id, g.qty_on_hand, p.reorder_point
  FROM global_inventory g JOIN global_policy p USING (inventory_id)
 WHERE g.qty_on_hand < p.reorder_point;
```

Bằng chứng: `EXPLAIN (VERBOSE, COSTS)` phải có `Foreign Scan` kèm `Remote SQL: SELECT product_id, sum(qty_on_hand) ... GROUP BY`. Để lấy số so sánh, chạy lại với view bọc `OFFSET 0` để chặn pushdown rồi đo `EXPLAIN ANALYZE`.

---

## 8. Yêu cầu C — Ba cơ chế đồng bộ

| Cơ chế | Chiều | Kích hoạt | Độ trễ |
|---|---|---|---|
| C1 — CDC theo sự kiện | CN → HQ | Mỗi lần tồn kho đổi | 1–5 giây |
| C2 — đối chiếu định kỳ | CN → HQ | 15 phút một lần | 15 phút |
| C3 — nhân bản danh mục | HQ → CN | Khi danh mục đổi | Vài giây |

### C1 — Outbox pattern

Bài toán ghi kép: trừ tồn xong rồi mới phát sự kiện mà chết ở giữa thì sự kiện mất vĩnh viễn. Giải bằng cách ghi sự kiện vào `outbox` trong **cùng một transaction** với nghiệp vụ:

```sql
BEGIN;
  UPDATE inventory SET qty_on_hand = qty_on_hand - 2, version = version + 1
   WHERE branch_id = 1 AND product_id = 1234 AND version = 7;
  INSERT INTO stock_movement(...) VALUES (...) RETURNING movement_id INTO v_mid;
  INSERT INTO outbox(event_id, aggregate_key, payload)
  VALUES (v_mid, '1:1234', jsonb_build_object(
          'branch_id',1,'product_id',1234,'qty_delta',-2,
          'qty_after',118,'movement_id',v_mid));
COMMIT;
```

Debezium đọc `outbox` qua WAL rồi phát lên Kafka topic `inventory.movement`, dùng `EventRouter` transform. Vì đọc từ transaction log chứ không quét bảng, **outbox không cần cột `is_sent` và không cần xoá dòng sau khi phát**.

Khoá message Kafka là `(branch_id, product_id)` để mọi sự kiện của cùng một dòng tồn vào cùng partition, giữ đúng thứ tự.

### Nạp sự kiện idempotent tại HQ

```sql
INSERT INTO inventory_snapshot(branch_id, product_id, qty_on_hand, last_movement, updated_at)
VALUES (:branch_id, :product_id, :qty_after, :movement_id, now())
ON CONFLICT (branch_id, product_id) DO UPDATE
   SET qty_on_hand = EXCLUDED.qty_on_hand,
       last_movement = EXCLUDED.last_movement, updated_at = now()
 WHERE inventory_snapshot.last_movement < EXCLUDED.last_movement;
```

Dòng `WHERE` cuối là thành phần quan trọng nhất: sự kiện tới muộn hoặc bị phát lại sau khi consumer restart sẽ bị bỏ qua.

### C2 — Đối chiếu định kỳ

Chạy 15 phút một lần qua chính FDW, so từng dòng giữa node và snapshot:

```sql
SELECT n.product_id, n.qty_on_hand AS node_qty, s.qty_on_hand AS snap_qty
  FROM cn1_remote.inventory n
  LEFT JOIN inventory_snapshot s ON s.branch_id = n.branch_id AND s.product_id = n.product_id
 WHERE n.qty_on_hand IS DISTINCT FROM s.qty_on_hand;
```

Lệch thì nạp đè bằng số của node — node luôn là chân lý cho mảnh của nó. Ghi `reconcile_log` với `variance_pct` để hiện trên dashboard.

### C3 — Nhân bản danh mục

```sql
-- tren pg_hq
CREATE PUBLICATION master_data FOR TABLE product, category, uom, supplier, branch;
-- tren moi node chi nhanh
CREATE SUBSCRIPTION sub_master
  CONNECTION 'host=pg_hq dbname=kho user=repl password=replsecret'
  PUBLICATION master_data;
```

Mỗi node đúng một subscription, không phải mỗi bảng hay mỗi mặt hàng một cái. Chi nhánh offline lúc danh mục đổi sẽ tự nhận phần thiếu khi nối lại.

---

## 9. Giao dịch 2PC cho điều chuyển hàng

Chuyển 50 đơn vị CN1 → CN3 ghi lên hai node. Một nửa thành công là mất hàng hoặc đẻ ra hàng không có thật.

Nghiệp vụ gồm **hai giao dịch 2PC độc lập**, không phải một:

- Chặng OUT: CN gửi −50, TRANSIT +50 → `transfer.status = IN_TRANSIT`
- Chặng IN: TRANSIT −50, CN nhận +50 → `transfer.status = RECEIVED`

Nhờ kho ảo TRANSIT, tổng tồn toàn chuỗi không đổi tại mọi thời điểm:

| Thời điểm | CN1 | TRANSIT | CN3 | Tổng |
|---|---|---|---|---|
| Trước khi chuyển | 118 | 0 | 60 | 178 |
| Đang vận chuyển | 68 | 50 | 60 | 178 |
| Sau khi nhận | 68 | 0 | 110 | 178 |

Mỗi chặng chạy đúng trình tự:

```
PHA 1: coordinator -> mỗi bên: UPDATE ... ; PREPARE TRANSACTION 'gid'
       mỗi bên trả READY (đã ghi đĩa, đang giữ khoá) hoặc ABORT
PHA 2: ghi VÀ COMMIT transfer_log(phase=DECIDE) TRƯỚC
       rồi mới gửi COMMIT PREPARED / ROLLBACK PREPARED cho tất cả
```

Sau khi trả READY, node tham gia không được đổi ý. Bất kỳ bên nào ABORT ở pha 1 thì rollback tất cả.

Coordinator là một module trong `api_hq`, tự cài đặt bằng psycopg, **không dùng transaction manager có sẵn**.

**Lưu ý code:** `PREPARE TRANSACTION` cần connection riêng ở `autocommit=False`, không dùng được connection pool cho đường này.

**Phục hồi:** endpoint `/transfer/recover` đọc `transfer_log` tìm chặng đã `DECIDE` nhưng chưa `DONE`, gửi lại lệnh pha 2. `COMMIT PREPARED` idempotent nên gửi lại bao nhiêu lần cũng được.

| Sự cố | Xử lý |
|---|---|
| Coordinator chết sau pha 1 | Đọc `transfer_log` khi khởi động lại; job tự `ROLLBACK PREPARED` giao dịch treo quá 5 phút |
| Node tham gia chết sau `PREPARE` | PostgreSQL giữ nguyên qua restart, gửi lại `COMMIT PREPARED` |
| Hai lệnh điều chuyển cùng lô | Tuần tự hoá theo `transfer_id`, đặt `lock_timeout` |

Chống tồn âm trong một node: khoá lạc quan bằng cột `version`, cộng ràng buộc `CHECK (qty_on_hand >= 0)` làm lớp phòng thủ cuối.

---

## 10. Hợp đồng API

### `api_cn` — 3 bản, cổng 8001/8002/8003

| Method | Endpoint | Body | Trả về |
|---|---|---|---|
| POST | `/sale` | `{product_id, qty, version?}` | `{movement_id, qty_on_hand, version}` |
| POST | `/receipt` | `{product_id, qty, supplier_id?}` | như trên |
| POST | `/stocktake` | `{product_id, counted_qty}` | như trên |
| GET | `/inventory` | `?product_id=&limit=&offset=` | `{branch_id, items[]}` |
| POST | `/transfer/prepare` | `{transfer_id, product_id, qty_delta, gid}` | `{status: READY\|ABORT}` |
| POST | `/transfer/commit` | `{gid, decision}` | `{status}` |
| GET | `/transfer/pending` | — | nội dung `pg_prepared_xacts` |
| GET | `/health` | — | `{status, inventory_rows, subscriptions}` |

### `api_hq` — cổng 8000

| Method | Endpoint | Ghi chú |
|---|---|---|
| GET | `/inventory/global?product_id=` | Đường nóng qua FDW, `503` nếu một node chết |
| GET | `/inventory/snapshot?product_id=` | Đường nguội, có cờ `stale` và `updated_at` |
| GET | `/inventory/below-reorder` | Cảnh báo dưới ngưỡng |
| POST | `/transfer` | `{from_branch_id, to_branch_id, product_id, qty}` → 2PC chặng OUT |
| POST | `/transfer/{id}/receive` | 2PC chặng IN |
| GET | `/transfer/{id}` | Trạng thái phiếu kèm `transfer_log` |
| POST | `/transfer/recover` | Hoàn tất chặng đã quyết định mà chưa xong |
| POST | `/reconcile` | Chạy đối chiếu ngay |
| GET | `/nodes/status` | Node sống chết, mốc snapshot, variance %, trạng thái subscription |

**Mã lỗi thống nhất:** `409` xung đột version (kèm `current_version`), `422` vi phạm nghiệp vụ (`insufficient_stock`), `503` không với tới node, `404` mã hàng không có ở chi nhánh.

Mọi response làm đổi tồn kho đều trả `version` mới để client retry. Không service nào gọi thẳng CSDL của chi nhánh khác — mọi thao tác xuyên site đi qua coordinator.

---

## 11. Năm luồng xử lý dữ liệu

| # | Luồng | Đường đi | Độ trễ |
|---|---|---|---|
| 1 | Bán hàng cục bộ | POS → `api_cn` → `pg_cn` (1 transaction: trừ tồn + movement + outbox) | < 10 ms |
| 2 | Đồng bộ lên trung tâm | WAL → Debezium → Kafka → sink consumer → `inventory_snapshot` | 1–5 giây |
| 3 | Báo cáo toàn chuỗi | Đường nóng qua FDW, hoặc đường nguội đọc snapshot | tức thời / 1–5 giây |
| 4 | Điều chuyển hàng | 2PC chặng OUT → IN_TRANSIT → 2PC chặng IN | theo thao tác |
| 5 | Nhân bản danh mục | `pg_hq` → logical replication → 3 `pg_cn` | vài giây |

---

## 12. Cấu hình bắt buộc

| Đặt ở đâu | Tham số | Thiếu thì sao |
|---|---|---|
| `pg_cn1/2/3` | `wal_level = logical` | Debezium không đọc được WAL |
| Cả 4 instance | `max_prepared_transactions = 20` | `PREPARE TRANSACTION` báo lỗi ngay (mặc định là 0) |
| `pg_cn1/2/3` | `max_replication_slots`, `max_wal_senders` ≥ 10 | Không đủ slot cho Debezium và subscription |
| `pg_hq` | `CREATE EXTENSION postgres_fdw` | Không truy vấn sang node được |

---

## 13. Thứ tự khởi tạo — bắt buộc đúng trình tự

1. Tạo `PUBLICATION master_data` ở `pg_hq`
2. Tạo `SUBSCRIPTION sub_master` ở cả ba node
3. Seed danh mục sản phẩm vào `pg_hq`, **chờ replication đẩy xuống đủ ba node**
4. Seed tồn kho ở từng node
5. Cấu hình FDW và view toàn cục trên `pg_hq`
6. Đăng ký ba Debezium connector

Đảo bước 3 và 4 sẽ lỗi khoá ngoại, vì tồn kho tham chiếu mã hàng chưa kịp về node.

---

## 14. Quy tắc bất biến

1. **Không bao giờ `DELETE` trên bảng `product`.** Chỉ `UPDATE SET is_active = false`. Xoá một mã hàng mà chi nhánh đang tham chiếu tới sẽ làm subscription của chi nhánh đó kẹt cứng, và từ đó nó không nhận được bất kỳ cập nhật danh mục nào nữa. Chặn từ gốc bằng cách không cấp quyền `DELETE` cho tài khoản ứng dụng.
2. Node là chân lý duy nhất cho mảnh tồn kho của mình. Lệch thì luôn nạp đè snapshot bằng số của node, không bao giờ ngược lại.
3. Không đặt khoá ngoại giữa các bảng master ở node chi nhánh, để logical replication không bị chặn bởi thứ tự áp dụng thay đổi.
4. Thứ tự sự kiện xác định bằng cặp `(branch_id, movement_id)`, không dựa vào đồng hồ vật lý.
5. Ghi và commit `transfer_log` trước khi gửi lệnh pha 2 của 2PC.

---

## 15. Thứ tự triển khai

| Bước | Nội dung | Xong khi |
|---|---|---|
| 1 | Hạ tầng Compose, 4 PostgreSQL lên được | `docker compose ps` không container nào restart liên tục |
| 2 | Lược đồ và seed dữ liệu | Mỗi `pg_cn` có đúng 5.000 dòng inventory và 5.000 dòng product |
| 3 | Yêu cầu A | `SELECT branch_id, count(*)` trên từng node ra đúng một chi nhánh |
| 4 | Yêu cầu B | Có 2 query plan kèm số đo thời gian có và không pushdown |
| 5 | Yêu cầu C | Ngắt mạng một node, bán 20 đơn, nối lại thì snapshot khớp đúng 20 biến động |
| 6 | Giao dịch 2PC | Kill coordinator giữa pha 1, khởi động lại, tổng tồn trước sau bằng nhau |
| 7 | Ba màn hình giao diện | Ngắt mạng một chi nhánh thì màn hình trụ sở tô đỏ node đó, POS vẫn bán được |
| 8 | Đo đạc và báo cáo | Chạy trôi cả 5 kịch bản, số liệu thật đã vào báo cáo |

Ba màn hình cần làm: POS bán hàng, quản lý kho chi nhánh, quản trị toàn chuỗi. Màn hình thứ ba gọi `/nodes/status` để tô đỏ node chết và hiện mốc "số liệu tính đến 14:20".

---

## 16. Năm kịch bản nghiệm thu

| # | Thao tác | Kết quả phải thấy |
|---|---|---|
| 1 | Bán 2 đơn vị tại CN1 | Chỉ `pg_cn1` có movement mới, dưới 10 ms, hai node kia không đổi |
| 2 | Truy vấn tồn toàn chuỗi | `EXPLAIN VERBOSE` có `Remote SQL` chứa `sum(...)` và `GROUP BY` |
| 3 | Kill coordinator ngay sau pha 1 của 2PC | Giao dịch treo trong `pg_prepared_xacts`, khởi động lại thì tự hoàn tất, tổng tồn không đổi |
| 4 | Ngắt mạng CN2, bán 20 đơn, nối lại | Mất mạng vẫn bán được; nối lại thì snapshot khớp đúng 20 biến động |
| 5 | Bán tới khi tồn dưới `reorder_point` | Sự kiện chạy qua Kafka, dashboard hiện cảnh báo trong vài giây |

Kịch bản 3 và 4 ăn điểm nhất: chúng chứng minh tính đúng đắn dưới sự cố.

---

## 17. Những chỗ dễ vấp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Debezium không bắt được thay đổi | `wal_level` vẫn là `replica` | Đặt `logical`, kiểm `curl localhost:8083/connectors/outbox-cn1/status` |
| `PREPARE TRANSACTION` lỗi ngay | `max_prepared_transactions = 0` | Đặt thành 20, restart PostgreSQL |
| Ổ đĩa node phình to | Replication slot không ai đọc, WAL bị giữ | `SELECT * FROM pg_replication_slots`, xoá slot khi ngừng dùng Debezium |
| Chi nhánh không nhận danh mục mới | Subscription kẹt do lỗi khoá ngoại | Xem `pg_stat_subscription`; gỡ bằng `ALTER SUBSCRIPTION ... SKIP (lsn = ...)` |
| Snapshot lệch số | CDC mất sự kiện | Gọi `/reconcile`; kiểm lại điều kiện `last_movement <` khi UPSERT |
| Seed lỗi khoá ngoại | Sai thứ tự khởi tạo | Xem mục 13 |
| Container tự restart | Máy không đủ RAM cho 11 container | Tắt Metabase và kafka-ui khi phát triển |
