# データベース命名規則・設計基準（完全版） {#db-style-guide}

- **対象**：PostgreSQL を前提とした論理名（テーブル・ビュー・カラム・制約・インデックス・トリガ・関数・シーケンス等）
- **目的**：読解性の向上、衝突回避、差分レビュー容易化、AI/CI による自動検査の標準化
- **互換**：現行のプレフィックス体系（`mst_/trn_/wrk_/tmp_/log_/hist_/cfg_/view_`）を継続採用
  ※ 将来的にスキーマ分離（例：`mst.product`）へ移行可能。移行手順は付録参照。
- **前提**：原則、業務・顧客は日本国内限定、通貨は日本円（JPY）とする（外貨対応は例外として定義）。

---

## 1. 基本原則 {#principles}

### 1.0 基本原則

- **原則**：テーブルおよびカラムへの日本語でのコメントは必須（そのテーブルの用途を必ず記載すること）

### 1.1 使用可能文字と形式 {#charset}

- **許可**：英小文字 `a–z`、数字 `0–9`、アンダースコア `_`
- **禁止**：日本語、全角文字、大文字、ハイフン `-`、空白、その他記号、引用識別子（`"Name"`）
- **先頭文字**：英小文字（`a–z`）
- **形式**：**snake_case**（小文字+アンダースコア）
- **長さ**：63 文字以内（PostgreSQL 制限）
- **予約語**：PostgreSQL 公式予約語リストに準拠（単体名として使用禁止）
  - 参考：https://www.postgresql.org/docs/current/sql-keywords-appendix.html
  - CI 実装：`pg_get_keywords()`の結果を使用
  - 追加禁止語：プロジェクト固有の禁止語は `docs/glossary_exceptions.md` で管理

### 1.2 命名スタイル {#style}

- **必須**：snake_case
- **禁止**：camelCase、PascalCase、kebab-case
- **言語**：意味が明確な英単語を用いる
- **略語**：原則禁止。**例外のみ許可**（`id`, `no`, `cd` / 追加例外は運用台帳で管理）

---

## 2. テーブル命名規則 {#table-naming}

### 2.1 プレフィックス体系（現行標準） {#prefixes}

| プレフィックス | 用途                                       | 例                       |
| -------------- | ------------------------------------------ | ------------------------ |
| `mst_`         | マスタデータ                               | `mst_product`            |
|                | ※ 認証情報も `mst_` (例: `mst_staff_auth`) |                          |
| `trn_`         | トランザクション                           | `trn_sales_order`        |
| `wrk_`         | ワークテーブル（同期/中間）                | `wrk_phone_sync_state`   |
| `tmp_`         | 一時テーブル（永続利用の一時領域）         | `tmp_price_calc`         |
| `log_`         | ログテーブル（イベント履歴）               | `log_phone_call`         |
| `hist_`        | 履歴テーブル（SCD）                        | `hist_product_price`     |
| `cfg_`         | 設定・定義                                 | `cfg_tax_rule`           |
| `view_`        | ビュー                                     | `view_inventory_summary` |
| `mvw_`         | マテリアライズドビュー                     | `mvw_latest_orders`      |

> ※ 将来スキーマ分離を行う場合は、`mst.product` のように**スキーマ**で分類し、**プレフィックスを外す**。どちらか一方に統一すること。

### 2.1.1 業務ドメインによるグループ化 (Domain Prefix) {#domain-prefix}

- **ルール**：特定の業務領域（ドメイン）に強く紐づくテーブルは、プレフィックスの直後に**ドメイン名**を付与してグループ化することを推奨する。
- **形式**：`{prefix}_{domain}_{entity}`
- **メリット**：アルファベット順でソートした際に、関連テーブルがまとまり、可読性が向上する。

**採用基準（判断フロー）：**

1. **横断ドメイン**（例：`shipping_`, `system_`, `auth_`）
   → 必ず付与（他の業務領域でも参照される）
2. **テーブルプレフィックスと意味が重複**
   → 省略可（例：`trn_sales_order`の`sales`は省略）
3. **5 テーブル以上の関連テーブルがある**
   → 付与推奨（可読性・保守性向上）
4. **外部キー名の衝突が予想される**
   → 付与必須（例：`shipping_carrier_id` vs `carrier_id`）

| 変更前 (旧)          | 変更後 (Domain 付与)          | ドメイン          | 用途                                              |
| :------------------- | :---------------------------- | :---------------- | :------------------------------------------------ |
| `mst_carrier`        | `mst_shipping_carrier`        | `shipping` (配送) | 配送業者                                          |
| `mst_fuel_surcharge` | `mst_shipping_fuel_surcharge` | `shipping` (配送) | 燃料サーチャージ                                  |
| `mst_shipping_fee`   | `mst_shipping_fee`            | `shipping` (配送) | 送料（※元から shipping を含むため変更なし）       |
| `trn_sales_order`    | `trn_sales_order`             | `sales` (販売)    | 受注（※標準プレフィックスと重複する場合は省略可） |
| `mst_staff_auth`     | `mst_system_auth`             | `system` (基盤)   | システム認証情報                                  |

### 2.2 単数形で命名 {#singular}

- **必須**：テーブル名は**単数形**（`mst_products` ではなく `mst_product`）

### 2.3 日付サフィックスの廃止 {#no-date-suffix}

- **禁止例**：`tbl_mラック_価格情報202401`、`tbl_mチャーター金額_20221011`
- **是正**：履歴化（`hist_`）で表現し、**有効区間カラム**で管理（後述 SCD2）

### 2.4 中間テーブルの命名 {#junction}

- **明細（子）**：`{親テーブル}_detail`（例：`trn_purchase_order_detail`）
- **多対多**：`{左テーブル}_{右テーブル}_link`（例：`mst_product_category_link`）
  ※ `_map` や `_rel` に分散させない

### 2.5 変換例（既存 ⇒ 新） {#rename-examples}

| 現在のテーブル名              | 新テーブル名                 | 説明                      |
| ----------------------------- | ---------------------------- | ------------------------- |
| `tbl_m会社`                   | `mst_company`                | 会社マスタ                |
| `tbl_m商品_基本情報`          | `mst_product`                | 商品マスタ                |
| `tbl_m商品_価格情報`          | `mst_product_price`          | 商品価格マスタ            |
| `tbl_mラック_基本情報`        | `mst_rack`                   | ラックマスタ              |
| `tbl_t出荷指示`               | `trn_shipment_order`         | 出荷指示                  |
| `tbl_t在庫`                   | `trn_inventory`              | 在庫トランザクション      |
| `tbl_t発注`                   | `trn_purchase_order`         | 発注                      |
| `tbl_mアマゾン運送会社対応表` | `mst_amazon_carrier_mapping` | Amazon 運送会社マッピング |
| `tbl_mカレンダー_休業日`      | `mst_calendar_holiday`       | 休業日カレンダー          |

---

## 3. カラム命名規則 {#column-naming}

### 3.1 共通カラム（標準セット） {#common-columns}

**監査（推奨）**

- `created_at TIMESTAMPTZ`
- `updated_at TIMESTAMPTZ`
- `deleted_at TIMESTAMPTZ`（論理削除）。`is_deleted` フラグは持たず、`IS NOT NULL` で判定する（二重管理防止）。
  - **使い分け基準**:
    - **論理削除 (`deleted_at`)**: 誤登録データや、システム上「最初から存在しなかった」かのように扱いたい場合。
    - **無効 (`is_active = false`)**: 運用上の停止（商品廃盤、担当外れ、休会など）。データおよび参照関係は維持される。
- `created_by_id BIGINT` / `updated_by_id BIGINT` / `deleted_by_id BIGINT`（`mst_staff.id` 参照）

**監査カラム運用ルール：**

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

-- トリガで自動更新（アプリ側では触らない）
CREATE TRIGGER trg_{table}_bu_set_updated_at
BEFORE UPDATE ON {table}
FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

**重要**：アプリケーションコードで`updated_at`を明示的に設定しても、トリガで上書きされる。

**主キー・外部キー**

- 主キー：`id BIGINT GENERATED ALWAYS AS IDENTITY`（**標準**）
  - **標準**：DB 採番 (`GENERATED ALWAYS AS IDENTITY`)
  - **例外 1 (交差テーブル)**：多対多の中間テーブルで、属性を持たない場合は複合キー (`user_id`, `role_id`) を許可。ただし履歴管理が必要な場合は `id` を推奨。
  - **例外 2 (列挙型テーブル)**：変更頻度が極めて低く、コード値自体が不変な場合（例: `code TEXT PRIMARY KEY`）。ただし原則は `id` 推奨。
  - **例外 3 (分散 ID)**：UUID / ULID（要申請）
  - **禁止**：アプリ側での連番管理、業務キー（会員番号など）の主キー利用（変更時に破綻するため）。
- 外部キー：`{参照先テーブル}_id BIGINT`（例：`customer_id`, `store_id`, `staff_id`）
  - **命名衝突回避**：ドメインを含める（例：`shipping_carrier_id`, `payment_gateway_id`）
  - **判断基準**：`carrier_id` では曖昧な場合に `shipping_carrier_id` とするなど。

### 3.2 日付・時刻の語尾ルール {#datetime-suffix}

- `…_at`：`TIMESTAMPTZ`（イベントの瞬間・区間端点）
- `…_date`：`DATE`（日単位）。`_on` は使用しない。
- `…_time`：`TIME`（時刻のみ）
- **タイムゾーン運用方針**：
  - **DB 設定**：`timezone = 'Asia/Tokyo'`
  - **保存型**：必ず `TIMESTAMPTZ`（`TIMESTAMP` 使用禁止）
  - **運用**：`now()` または明示的な `TIMESTAMPTZ` リテラルを使用。文字列保存禁止。
- 期間表現：`valid_from_at` / `valid_to_at`（SCD2）

### 3.3 真偽値 {#boolean}

- **肯定形のみ**：`is_` / `has_`（例：`is_active`, `has_recording`）
- **NULL 禁止**：原則 `NOT NULL DEFAULT false` (または `true`) とする。3 値論理（True/False/NULL）によるバグを防ぐため。
- 否定形（`is_not_xxx`）は禁止

### 3.4 数値・単位 {#numeric}

**基本方針（用途別ルール）：**

- **日本円・支払額 (Amount)**：`NUMERIC(18,0)`（小数部なし、円単位）
  - 理由：実際の決済・請求は 1 円単位のため。
  - 例：`total_amount NUMERIC(18,0)`, `tax_amount NUMERIC(18,0)`
- **単価・レート (Unit Price / Rate)**：**`NUMERIC(18,4)` 等**
  - 理由：重量課金、税計算、割引按分などで小数点以下の精度が必要な場合があるため。
  - 例：`unit_price NUMERIC(18,4)`, `discount_rate NUMERIC(5,4)`
- **外貨 (FX)**：`NUMERIC(18,2)` + `currency_code CHAR(3)`
  - 注：この場合でも JPY 列は `.00` で統一する（計算時の型不一致防止）

**禁止事項：**

- JPY 固定システムで `NUMERIC(18,2)` を使用すること
- 同一システム内で JPY 列と FX 列を混在させること（カラム名で明確に区別）

**その他の数値カラム：**

- 単価：`…_unit_price`
- 数量：`…_quantity`（`_qty` は不採用。原則 `INTEGER`。重量等で小数が必要な場合は `NUMERIC`）
- 率（百分率）：`…_rate_pct`、比率（小数）：`…_ratio`
- 経過時間：`…_sec`（秒）、`…_ms`（ミリ秒）

### 3.5 名称・文字列 {#string}

- 基本名：`name`、表示名：`display_name`、説明：`description`、備考：`remarks`、メモ：`note`
  **型選択基準：**
- **原則**：`TEXT`（PostgreSQL では性能差なし）
- **長さ制約が必要な場合**：`TEXT` + `CHECK` 制約、または `VARCHAR(n)`（既存互換時のみ）
- 電話番号・コード類は**数値型にしない**（先頭ゼロ保持のため）。
  - 推奨：`TEXT`（PostgreSQL では VARCHAR(n) より TEXT が推奨される）。
  - 形式：ハイフンなしの E.164 形式（例: `+819012345678`）または国内形式（例: `09012345678`）に統一し、**ハイフンは除去して保存**することを推奨。表示時に整形する。

### 3.6 JSONB {#jsonb}

- 任意構造：`…_jsonb`
- 外部 API の生データ：`…_payload`
- **禁止事項**: 検索条件(`WHERE`)に使用する重要項目を JSONB 内に埋没させること。
- **必須**: 検索キーや集計対象となる項目は、必ず**正規化列（通常のカラム）**として定義すること。

### 3.8 NULL 許容方針 {#null-policy}

**デフォルト**：`NOT NULL`（3 値論理を避ける）

**NULL 許容が認められるケース：**

1. 外部システム連携で欠損データがあり得る場合
2. オプショナルな属性（例：`middle_name`, `fax_number`）
3. 段階的なデータ登録が業務要件の場合

**必須対応：**

- NULL 許容列には必ずコメントで理由を記載
- デフォルト値で対応できないか検討する

### 3.7 カラム名変換例 {#column-rename-examples}

| 現在       | 新                    | 説明                            |
| ---------- | --------------------- | ------------------------------- |
| 削除フラグ | -                     | `deleted_at` で管理するため廃止 |
| 登録日     | `created_date`        | 日付のみ                        |
| 登録日時   | `created_at`          | タイムスタンプ                  |
| 更新日時   | `updated_at`          | タイムスタンプ                  |
| 仕入単価   | `purchase_price`      | 仕入価格                        |
| 販売単価   | `selling_price`       | 販売価格                        |
| 商品名     | `product_name`        | 名称                            |
| 品番       | `product_code`        | コード                          |
| 担当者     | `person_in_charge_id` | 担当者 ID                       |

---

## 4. 業務用語対訳辞典（抄） {#glossary}

### 4.1 基本

| 日本語 | 英語             | 例                     |
| ------ | ---------------- | ---------------------- |
| 会社   | company          | `mst_company`          |
| 顧客   | customer         | `mst_customer`         |
| 取引先 | business_partner | `mst_business_partner` |
| 店舗   | store            | `mst_store`            |
| 部署   | department       | `mst_department`       |
| 職員   | staff            | `mst_staff`            |
| 担当者 | person_in_charge | `person_in_charge_id`  |
| 役職   | position         | `mst_position`         |

### 4.2 商品・在庫

| 日本語       | 英語              | 例                    |
| ------------ | ----------------- | --------------------- |
| 商品         | product           | `mst_product`         |
| 品番         | product_code      | `product_code`        |
| 商品名       | product_name      | `product_name`        |
| 分類         | category          | `product_category_id` |
| 在庫         | inventory / stock | `trn_inventory`       |
| 倉庫         | warehouse         | `mst_warehouse`       |
| ロケーション | location          | `mst_location`        |
| 棚卸         | stock_taking      | `trn_stock_taking`    |

### 4.3 受発注

| 日本語   | 英語                  | 例                          |
| -------- | --------------------- | --------------------------- |
| 受注     | sales_order           | `trn_sales_order`           |
| 発注     | purchase_order        | `trn_purchase_order`        |
| 発注先   | supplier              | `mst_supplier`              |
| 発注明細 | purchase_order_detail | `trn_purchase_order_detail` |
| 案件     | project               | `project_id`                |
| 見積     | quotation             | `trn_quotation`             |
| 請求     | invoice               | `trn_invoice`               |
| 支払     | payment               | `trn_payment`               |
| 返品     | return                | `trn_sales_return`          |
| 値引     | discount              | `discount_amount`           |
| 与信     | credit                | `credit_limit`              |

### 4.4 物流・配送

| 日本語           | 英語           | 例                            |
| ---------------- | -------------- | ----------------------------- |
| 出荷             | shipment       | `trn_shipment`                |
| 出荷指示         | shipment_order | `trn_shipment_order`          |
| 納品             | delivery       | `delivery_date`               |
| 配送             | delivery       | `delivery_method_id`          |
| 運送会社         | carrier        | `mst_shipping_carrier`        |
| チャーター       | charter        | `charter_request`             |
| 送料             | shipping_fee   | `mst_shipping_fee`            |
| 燃料サーチャージ | fuel_surcharge | `mst_shipping_fuel_surcharge` |
| 梱包             | packing        | `packing_unit`                |

### 4.5 ラック・棚（業界固有）

| 日本語 | 英語          | 例                  |
| ------ | ------------- | ------------------- |
| ラック | rack          | `mst_rack`          |
| 棚板   | shelf_board   | `shelf_board_count` |
| 支柱   | column / post | `column_height`     |
| 耐荷重 | load_capacity | `max_load_capacity` |
| 段数   | tier_count    | `rack_tier_count`   |
| 奥行   | depth         | `rack_depth`        |
| 横幅   | width         | `rack_width`        |
| 高さ   | height        | `rack_height`       |

### 4.6 金額・会計

| 日本語   | 英語          | 例                    |
| -------- | ------------- | --------------------- |
| 金額     | amount        | `total_amount`        |
| 単価     | unit_price    | `unit_price`          |
| 原価     | cost          | `product_cost`        |
| 差額     | difference    | `price_difference`    |
| 税別     | excluding_tax | `price_excluding_tax` |
| 課税区分 | tax_category  | `tax_category_id`     |

### 4.7 日付・時刻

| 日本語   | 英語          | 例                           |
| -------- | ------------- | ---------------------------- |
| 登録日   | created_date  | `created_date`               |
| 登録日時 | created_at    | `created_at`                 |
| 更新日時 | updated_at    | `updated_at`                 |
| 開始日   | start_date    | `valid_from_at` との整合注意 |
| 終了日   | end_date      | `valid_to_at` との整合注意   |
| 納品日   | delivery_date | `delivery_date`              |
| 出荷日   | shipment_date | `shipment_date`              |

### 4.8 状態・区分

| 日本語 | 英語            | 例                    |
| ------ | --------------- | --------------------- |
| 状態   | status          | `order_status`        |
| 区分   | type / category | `payment_type`        |
| 種別   | kind            | `document_kind`       |
| フラグ | flag            | `is_active`（肯定形） |
| 削除   | delete          | `deleted_at`          |
| 有効   | active          | `is_active`           |

### 4.9 その他

| 日本語       | 英語                  | 例                      |
| ------------ | --------------------- | ----------------------- |
| 摘要         | summary / description | `transaction_summary`   |
| 備考         | remarks               | `remarks`               |
| 添付ファイル | attachment            | `attachment_file_name`  |
| メール       | email                 | `email_template`        |
| テンプレート | template              | `mst_email_template`    |
| 休業日       | holiday               | `calendar_holiday`      |
| 個体識別     | individual_id         | `individual_identifier` |

### 4.10 システム関連

| 日本語   | 英語  | 例                    |
| -------- | ----- | --------------------- |
| バッチ   | batch | `log_batch_execution` |
| ジョブ   | job   | `job_status`          |
| エラー   | error | `error_message`       |
| リトライ | retry | `retry_count`         |

---

## 5. 制約・インデックス・オブジェクト命名 {#constraints-indexes}

### 5.1 制約名 {#constraints}

| 種別     | プレフィックス | 形式                           | 例                     |
| -------- | -------------- | ------------------------------ | ---------------------- |
| 主キー   | `pk_`          | `pk_{テーブル}`                | `pk_product`           |
| 外部キー | `fk_`          | `fk_{子テーブル}_{親テーブル}` | `fk_order_customer`    |
| ユニーク | `uq_`          | `uq_{テーブル}_{カラム}`       | `uq_product_code`      |
| チェック | `chk_`         | `chk_{テーブル}_{条件略称}`    | `chk_order_amount_pos` |

- **外部キー削除動作**：原則 `ON DELETE RESTRICT`（親削除時にエラー）。`CASCADE` は明確な意図がある場合のみ許可。

**論理削除採用時の UNIQUE 制約ルール：**

- **デフォルト**：部分ユニーク制約（再登録を許可）。`WHERE deleted_at IS NULL` を付与。
- **例外**：再利用禁止が業務要件の場合、通常の UNIQUE 制約（設計書に明記）。

### 5.2 インデックス名 {#indexes}

| 種類 | プレフィックス | 形式                                 | 例                        |
| ---- | -------------- | ------------------------------------ | ------------------------- |
| 通常 | `idx_`         | `idx_{テーブル}_{カラム1}_{カラム2}` | `idx_product_code`        |
| 複合 | `idx_`         | 同上                                 | `idx_order_customer_date` |
| 部分 | `idx_`         | `idx_{テーブル}_{カラム}_{条件略称}` | `idx_order_active`        |

**インデックス作成基準：**

- **必須条件**：目的が明確であること（WHERE, ORDER BY, JOIN 等）
- **複合インデックス順序**：1.等価条件(`=`) → 2.範囲条件(`>`,`BETWEEN`) → 3.カーディナリティ高
- **JSONB 専用**：`CREATE INDEX idx_{table}_{column}_gin ON {table} USING gin({jsonb_column});`
- **禁止**：「念のため」作成する、使用頻度不明なもの
- **自動生成名**：マイグレーションツール等が生成する末尾ハッシュ付きの名称（例: `..._176768436`）は**許容 (WARN/INFO)** とする。可能な限りリネーム推奨だが、実運用上は必須としない。

### 5.3 トリガ・関数・シーケンス {#triggers}

- **Trigger**：`trg_{テーブル}_{timing略}_{event略}`（例：`trg_sales_order_bu_set_updated_at`）
  - `bi`=Before Insert, `bu`=Before Update, `bd`=Before Delete
  - `ai`=After Insert, `au`=After Update, `ad`=After Delete
- **Function**：`fn_{目的}`（例：`fn_set_updated_at()`）
- **Sequence**：`seq_{テーブル}_{カラム}`（IDENTITY 採用時は原則不要）

### 5.4 パーティション・ビュー {#partitions-views}

- マテビュー：`mvw_{名前}`
- パーティション親：`{テーブル}_prt`、子：`{テーブル}_prt_{単位}`
  - 月次：`_YYYYMM` (例: `log_access_prt_202501`)
  - 日次：`_YYYYMMDD` (例: `log_access_prt_20250101`)
  - 年次：`_YYYY` (例: `log_access_prt_2025`)

---

## 6. データ型選択指針（抜粋） {#types}

- **ID**：`BIGINT GENERATED ALWAYS AS IDENTITY` 推奨。UUID/ULID は例外申請。
- **タイムスタンプ**：`TIMESTAMPTZ` を標準。日本国内限定のため JST 運用を許容
- **金額・数値**：用途により精度を使い分ける。
  - **支払額 (Amount)**：`NUMERIC(18,0)` (円単位・端数なし)
  - **単価・レート (Unit Price / Rate)**：`NUMERIC(18,4)` 等、必要な精度を確保する (計算途中での丸め誤差防止)
  - **外貨 (FX)**：`NUMERIC(18,2)` + `currency_code`
- **数量**：原則 `INTEGER`。重量・長さ等で小数を扱う場合のみ `NUMERIC`
- **文字列**：`TEXT` 推奨。`TEXT` + `CHECK` 制約で長さ制限を行う。
- **ENUM**：原則使用しない。`TEXT` + `CHECK` 制約、または参照マスタテーブルを使用する。
- **電話番号**：`TEXT`（ハイフン除去・国内形式または E.164）

---

## 7. 履歴（SCD）設計指針 {#scd}

- `hist_` テーブルは **SCD2** を標準
  - カラム：`valid_from_at TIMESTAMPTZ`, `valid_to_at TIMESTAMPTZ`, `is_current BOOLEAN` (Optional)
  - 一意性：`(business_key, valid_from_at)` 等で範囲重複を禁止
  - 取得：`valid_to_at IS NULL` (または `infinity`) で現行レコードを判定
- SCD1（上書き）は許容するが、採否を設計書に明記

**区間表現の標準：**

- 期間：`[valid_from_at, valid_to_at)`
- 現行レコード：`valid_to_at IS NULL` (推奨) または `'infinity'`
  - ※ PostgreSQL の `tstzrange(from, NULL)` は「終了なし（未来永劫）」として扱われるため、NULL 運用で問題ない。
  - ※ アプリケーション互換性のため NULL を採用するケースが多い。
- 重複検証：**Exclusion Constraint (必須)**
  - これがないと期間重複が発生し、データ不整合の原因となる。
  - 実装には `btree_gist` 拡張が必要。

**インデックス構成（標準セット）:**

1. **Exclusion Constraint** (`gist`): 期間重複の防止（整合性担保）
2. **Unique Index** (`btree`): `valid_to_at IS NULL` に対するユニーク制約（現行データの重複防止、高速化）

### 7.1 SCD2 実装例

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist SCHEMA core; -- 事前に必須

CREATE TABLE hist_product_price (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  product_id      BIGINT NOT NULL REFERENCES mst_product(id),
  unit_price      NUMERIC(18,0) NOT NULL,
  valid_from_at   TIMESTAMPTZ NOT NULL,
  valid_to_at     TIMESTAMPTZ, -- NULL = 現在有効

  -- 重複期間の禁止 (Exclusion Constraint - GiST)
  -- business_key + 期間がかぶるものをブロック
  CONSTRAINT exclude_hist_product_price_overlap
  EXCLUDE USING gist (
    product_id WITH =,
    tstzrange(valid_from_at, valid_to_at) WITH &&
  )
);

-- 現行データの検索用・重複防止 (B-Tree + Partial Index)
-- ※ PostgreSQLの通常UNIQUE制約はNULLを複数許容してしまうため、現行レコード(valid_to_at IS NULL)の一意性を担保するには
--    以下のように「部分インデックス」を用いるのが標準 (Best Practice)。
CREATE UNIQUE INDEX uq_hist_product_price_current
ON hist_product_price(product_id)
WHERE valid_to_at IS NULL;
```

---

## 8. DDL テンプレート（抜粋） {#ddl-templates}

### 8.1 監査トリガ（任意） {#audit-trigger}

```sql
CREATE OR REPLACE FUNCTION fn_set_updated_at()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END $$;

CREATE TRIGGER trg_product_bu_set_updated_at
BEFORE UPDATE ON mst_product
FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- 必須拡張機能（coreスキーマに集約）
CREATE EXTENSION IF NOT EXISTS btree_gist SCHEMA core; -- SCD2 Exclusion Constraint用

-- 推奨：必要に応じて core スキーマに導入
CREATE EXTENSION IF NOT EXISTS pg_trgm SCHEMA core;   -- あいまい検索
CREATE EXTENSION IF NOT EXISTS pg_bigm SCHEMA core;   -- 日本語2文字検索精度向上
CREATE EXTENSION IF NOT EXISTS "uuid-ossp" SCHEMA core; -- UUID
```

### 8.2 ログ：発着信（実例） {#phone-log-example}

```sql
CREATE TABLE IF NOT EXISTS log_phone_call (
  id                       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  call_log_id              TEXT,
  direction                TEXT,
  result                   TEXT,
  other_party_number_e164  TEXT,
  other_party_number_domestic TEXT,
  our_number_e164          TEXT,
  our_number_domestic      TEXT,
  occurred_at              TIMESTAMPTZ NOT NULL,
  duration_sec             INTEGER,
  staff_id                 BIGINT REFERENCES mst_staff(id) ON DELETE RESTRICT,
  customer_id              BIGINT REFERENCES mst_customer(id) ON DELETE RESTRICT,
  store_id                 BIGINT REFERENCES mst_store(id) ON DELETE RESTRICT,
  is_business_hours        BOOLEAN NOT NULL DEFAULT false,
  raw_payload              JSONB,
  created_at               TIMESTAMPTZ DEFAULT now(),
  updated_at               TIMESTAMPTZ DEFAULT now()
);

COMMENT ON TABLE log_phone_call IS '通話履歴ログ：ZoomPhoneからの発着信情報を記録';
COMMENT ON COLUMN log_phone_call.direction IS '通話方向（inbound/outbound/internal）';

CREATE UNIQUE INDEX IF NOT EXISTS uq_log_phone_call_call_log_id
  ON log_phone_call(call_log_id) WHERE call_log_id IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_log_phone_call_occurred_at ON log_phone_call(occurred_at DESC);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_other_party_domestic ON log_phone_call(other_party_number_domestic);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_our_domestic ON log_phone_call(our_number_domestic);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_direction_result ON log_phone_call(direction, result);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_customer_id ON log_phone_call(customer_id);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_store_id ON log_phone_call(store_id);

CREATE TABLE IF NOT EXISTS log_phone_call_leg (
  id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  phone_call_id  BIGINT NOT NULL REFERENCES log_phone_call(id) ON DELETE CASCADE,
  leg_kind       TEXT,
  leg_name       TEXT,
  occurred_at    TIMESTAMPTZ,
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_log_phone_call_leg_call ON log_phone_call_leg(phone_call_id);
CREATE INDEX IF NOT EXISTS idx_log_phone_call_leg_kind ON log_phone_call_leg(leg_kind);

CREATE TABLE IF NOT EXISTS wrk_phone_sync_state (
  id               TEXT PRIMARY KEY,
  last_synced_to   DATE,
  checkpoint_token TEXT,
  updated_at       TIMESTAMPTZ DEFAULT now()
);
```

---

## 9. AI / CI 自動チェック基準（人間向け要約） {#lint-summary}

**重大（ERROR） - 機械的に 100%判定可能な違反**

1. 識別子が snake_case/許可文字以外
2. 未許可プレフィックス or 混在
3. テーブル名が複数形 (チーム文化として統一)
4. 日付サフィックスで履歴表現 (`_202401` 等)
5. 真偽値が肯定形でない (`is_active` ok, `is_deleted` ng-logic-delete)
6. 日付・時刻語尾と型の不一致 (`_at`!=TIMESTAMPTZ, `_date`!=DATE)
7. 予約語の単体使用
8. **SCD2 履歴テーブルの期間重複防止 (Exclude Constraint) 欠落**
   - `gist` 排他制約は必須 (不整合即事故のため)

**警告（WARN） - 設計判断や例外が絡むもの**

1. **主キー名が `id` 以外**
   - 原則 `id` 推奨だが、複合主キー・交差テーブル・外部キー PK 等の正当な理由は許容。
2. **外部キー列が `{参照先}_id` 以外**
   - 略称(org*id)やポリモーフィック(*\_type + \_\_id)などは例外許容。
3. **`created_at`, `updated_at` の欠落/設定不足**
   - 更新されない静的マスタ等は例外許容。
4. **真偽値カラム (`is_*`) の NULL 許容**
   - 原則 `NOT NULL DEFAULT` だが、3 値論理が必要な正当な理由は許容。
5. **外部キー列に対するインデックス欠落**
   - マスタ等で更新頻度が低く、ロックリスクが低い場合は許容されることもあるため。

---

## 10. 自動検証ツール (Automated Verification)

本ガイドラインの遵守状況を機械的に検証・監査するためのツール群が `scripts/` に用意されています。

### 10.1 Police: スキーマロジック検査 (`scripts/lint_schema_logic.py`)

稼働中の DB（または CI 用の一時 DB）に接続し、メタデータを直接検査します。

- **役割**: "Mechanically Verifiable" なルール（上記 ERROR 項目 + 一部 WARN 項目）の違反を検出。
- **出力**: `artifacts/schema/consistency_report.json`
- **実行**:
  ```bash
  python scripts/lint_schema_logic.py
  ```

### 10.2 Judge: AI/レポート生成 (`scripts/generate_ai_report.py`)

Police の検査結果を解析し、人間や AI が判断しやすい形式に変換します。

- **役割**: JSON レポートの可読化、および AI への修正指示プロンプトの生成。
- **出力**:
  - `artifacts/schema/consistency_report.md` (Markdown レポート)
  - `artifacts/schema/ai_prompt.txt` (AI への指示プロンプト)
- **実行**:
  ```bash
  python scripts/generate_ai_report.py
  ```

6. **業務キー (`code`, `name` 等) に対する UNIQUE 制約欠落**
7. **数値の整合性 CHECK 制約欠落** (金額 < 0 許容かどうか等は仕様による)
8. **金額列の精度 (`NUMERIC(18,0)` vs `18,2` vs `18,4`)**
   - 用途（支払額 vs 単価）に応じた適切な精度か確認を促す。

**提案（INFO） - 将来の改善余地**

1. `BIGSERIAL` → `IDENTITY` 移行提案
2. `Check(Text)` → DB ENUM / 参照テーブル化の提案
   - ENUM 採用時は変更手順の確立が必要
3. `TEXT` 列への文字数制限 (`CHECK (length(c) <= N)`) の提案

---

## 10. 機械判定用ルール（YAML 仕様） {#yaml-rules}

````yaml
# SQLFluff や 独自Lintツールでの利用を想定
db_style_guide:
  version: 1.1
  engine: postgresql

  identifier:
    max_length: 63
    reserved_words_source: "postgresql_reserved_words" # pg_get_keywords()準拠
    additional_prohibited: # プロジェクト固有禁止語
       - data
       - info
       - from
       - where
       - table
       - column
    allowed_regex: "^[a-z][a-z0-9_]*$"

  table:
    singular_required: true
    disallow_date_suffix_regex: '(_)?\\d{6,8}$'
    allowed_prefixes:
      - mst_
      - trn_
      - wrk_
      - tmp_
      - log_
      - hist_
      - cfg_
      - view_
      - mvw_
    junction:
      detail_suffix: _detail
      m2m_suffix: _link

  column:
    pk:
      name_recommendation: id
      type_required: bigint # 原則
      is_identity_required: true # 原則
    fk:
      name_regex: "^[a-z][a-z0-9_]*_id$"
      type_required: bigint
    datetime_suffix:
      at:
        type_required:
          - timestamptz
      date:
        type_required:
          - date
      time:
        type_required:
          - time
    boolean_prefix:
      allowed:
        - is_
        - has_
      disallow_negative: true
      nullable: false

    money_jpy:
      suffixes: [_amount, _unit_price, _tax_amount, _discount_amount]
      type_required: "numeric(18,0)"

    money_fx:
      suffixes: [_amount_fx, _unit_price_fx]
      type_required: "numeric(18,2)"
      required_sibling: currency_code

    quantity:
      suffix: _quantity
    ratio_rate:
      rate_pct_suffix: _rate_pct
      ratio_suffix: _ratio
    duration:
      sec_suffix: _sec
      ms_suffix: _ms
    jsonb:
      allowed_suffix:
        - _jsonb
        - _payload

  constraints:
    pk_format: "pk_{table}"
    fk_format: "fk_{child}_{parent}"
    uq_format: "uq_{table}_{column_list}"
    chk_format: "chk_{table}_{cond_hint}"

  indexes:
    idx_format: "idx_{table}_{column_list}"
    partial_idx_hint_allowed: true

  triggers_functions_sequences:
    trigger_format: "trg_{table}_{timing}_{event}"
    function_format: "fn_{purpose}"
    sequence_format: "seq_{table}_{column}"

  glossary:
    company: company
    customer: customer
    business_partner: business_partner
    store: store
    department: department
    staff: staff
    person_in_charge: person_in_charge
    position: position
    product: product
    product_code: product_code
    product_name: product_name
    category: category
    inventory: inventory
    warehouse: warehouse
    location: location
    stock_taking: stock_taking
    sales_order: sales_order
    purchase_order: purchase_order
    supplier: supplier
    purchase_order_detail: purchase_order_detail
    project: project
    quotation: quotation
    invoice: invoice
    payment: payment
    shipment: shipment
    shipment_order: shipment_order
    delivery: delivery
    carrier: carrier
    charter: charter
    shipping_fee: shipping_fee
    packing: packing
    rack: rack
    shelf_board: shelf_board
    column: column
    load_capacity: load_capacity
    tier_count: tier_count
    depth: depth
    width: width
    height: height
    amount: amount
    unit_price: unit_price
    cost: cost
    difference: difference
    excluding_tax: excluding_tax
    tax_category: tax_category
    created_at: created_at
    updated_at: updated_at
    delivery_date: delivery_date
    shipment_date: shipment_date
    hire_date: hire_date
    termination_date: termination_date
    status: status
    type: type
    category_word: category
    kind: kind
    is_active: is_active
    deleted_at: deleted_at
    remarks: remarks
    attachment: attachment
    email: email
    template: template
    holiday: holiday
    individual_identifier: individual_identifier

  comments:
    table:
      required: true
    column:
      required: true
      exemptions: [id, created_at, updated_at, deleted_at, created_by_id, updated_by_id, deleted_by_id]

---

## 11. 例外運用と承認フロー（推奨） {#exceptions}

1. **Github Issue 作成**：例外申請テンプレートを使用し、理由・期間（恒久か一時的か）・影響範囲を記載する。
2. **レビュー**：アーキテクチャ責任者および領域オーナーの 2 名以上が `Approve` すること。
3. **台帳反映**：承認後、`docs/glossary_exceptions.md`（例外管理台帳）に追記し、Linter の無視リスト(`ignore_words`)に反映する。
4. **期限管理**：一時的な例外の場合、カレンダーまたは Issue の DueDate でリファクタリング期限を管理する。

---

## 12. 付録：スキーマ階層移行の考え方（任意） {#schema-migration}

- フェーズ 1：現行の `mst_` 等プレフィックスを維持
- フェーズ 2：新規はスキーマを採用（`mst.product`）、旧テーブルはビューで互換
- フェーズ 3：アプリ参照先を順次スキーマへ切替、最後に旧テーブルのエイリアスを廃止

---

# 参考：チェック用 OK/NG スニペット {#ok-ng}

**OK（命名・型一致）**

```sql
CREATE TABLE trn_purchase_order_detail (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  purchase_order_id BIGINT NOT NULL REFERENCES trn_purchase_order(id) ON DELETE RESTRICT,
  product_id BIGINT NOT NULL REFERENCES mst_product(id) ON DELETE RESTRICT,
  quantity INTEGER NOT NULL,

  -- ✅ OK例1: JPY固定システム（標準）
  unit_price NUMERIC(18,0) NOT NULL,
  amount NUMERIC(18,0) NOT NULL,

  -- ✅ OK例2: 外貨対応システム（例外申請後）
  -- unit_price_fx NUMERIC(18,2) NOT NULL,
  -- currency_code CHAR(3) NOT NULL DEFAULT 'JPY',

  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE trn_purchase_order_detail IS '発注明細：発注書に含まれる商品ごとの明細行';
````

**NG（複数形・UTC サフィックス・否定形・型不一致）**

```sql
CREATE TABLE trn_orders (
  id SERIAL PRIMARY KEY,
  customerId INTEGER,
  order_timestamp_utc TIMESTAMP,
  is_not_canceled BOOLEAN,
  -- ❌ NG: 方針が不明確（JPY固定なのに小数部がある）
  unit_price NUMERIC(18,2)
);
```
