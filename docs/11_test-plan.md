# Clokka テスト計画（Phase 4 — レベルとトレーサビリティ）

| 項目 | 内容 |
| --- | --- |
| 目的 | 何をどのレベルでテストし、要件からテスト成功までの追跡を実装の詳細を捏造せずに定義する。 |
| 対象読者 | 開発者、AIエージェント、QA、プロダクトオーナー |
| 状態 | レビュー待ち — Q-04承認待ち |
| 最終更新日 | 2026-08-28 |
| 依存関係 | `01_requirements.md`、`04_database.md`、`05_api.md`、`06_screen-design.md`、`07_security.md`、`08_directory.md`、`09_development-rules.md` |

## 1. テストレベル

| レベル | 対象範囲 | 技術 | 実行タイミング | 責任者 |
| --- | --- | --- | --- | --- |
| 単体 | 単一クラス/関数（例: JSTのwork_date計算、休憩検証） | JUnit 5、Mockito | `./gradlew test`（CIおよびローカル） | 開発者 |
| 結合（スライス） | Springスライス + Testcontainersによる実PostgreSQL | Spring Boot Test、Testcontainers `postgres:16`、Flyway、RestAssured/MockMvc | `./gradlew test`（CI） | 開発者 |
| API（契約） | `05_api.md` の契約に対する `backend` 全体（認証、勤怠、提出、管理、Push、締切、リマインド、監査） | Testcontainers、MockMvc、AssertJ | `./gradlew test` および毎晩のCI | 開発者 |
| E2E（ブラウザ） | `frontend` + `backend` + DB（Playwright） | Playwright（Phase 9） | `npx playwright test`（Phase 9、Phase 6ではない） | QA / AI |
| セキュリティ/負荷 | 認証バイパス、CSRF、レート制限、50名×31日のExcel | OWASP ZAP（任意）、JMeter/k6（Phase 9） | Phase 9 | QA |

* 手動の `curl` を自動テストの代替としない。すべてのFRは適切なレベルで少なくとも1つの自動テストを持つこと。
* `frontend` の単体テスト（Vitest/Jest）はMVPでは**使わない** — Vanilla JSは結合/APIとE2Eでテストする。

## 2. トレーサビリティ（FR → テスト）

| 要件 | 単体 | 結合 | API | E2E（Phase 9） |
| --- | --- | --- | --- | --- |
| FR-01 認証/権限 | Argon2idベクタ、セッションCookie属性 | `POST /auth/login` 成功/失敗、`GET /me` の権限 | `401`/`403` マトリクス、CSRF、IDOR | ログインS-01、401リダイレクト、403画面 |
| FR-02 日次入力 | JSTの `work_date` 計算、`clock_out - clock_in < 24h`、`break < elapsed` | `PUT /attendance/{workDate}` のupsert、`UNIQUE(employee_id,work_date)` | `status=SUBMITTED` 時の `409` | S-03の保存/削除、検証メッセージ |
| FR-03 日跨ぎ | `23:00→02:00` が開始日に紐づく | 同上（DBの `CHECK ck_attendance_work_date_jst` 経由） | APIの `clockOut <= clockIn` は翌日として扱う | S-03の翌日ヒント、S-02の月次表示 |
| FR-04 月次表示 | `target_month = date_trunc` | `GET /attendance?month=` が対象日と `DRAFT` 表示を返す | `GET /submissions/{month}` | S-02の月移動、合計表示 |
| FR-05 提出前チェック | 休日由来の必須日 | `POST /submissions/{month}/validate` が `422` と `fieldErrors` を返す | `422` のボディ | S-04の検証リスト、S-03へのリンク |
| FR-06 提出/差戻し | 状態FSM `DRAFT→SUBMITTED→RETURNED→SUBMITTED` | `POST /submissions`、`POST /admin/.../return` と監査 | 同時遷移時の `409` | S-04提出、S-06差戻し、S-02ロック |
| FR-07 通知漏れ防止 | `due_at` からの `reminder_stage` 算出 | `notification_deliveries` の `UNIQUE(employee_id,notification_date)` | `POST /internal/jobs/monthly-reminders` の冪等性 | S-05のPush状態、S-07一覧、アプリ内バナー |
| FR-08 管理一覧 | ソート比較 | `GET /admin/submissions` のフィルタ、ソート、ページング | `X-Request-Id` の存在 | S-06の検索/ソート |
| FR-12 Push状態 | 集約 `GRANTED>DEFAULT>DENIED>UNSUPPORTED>UNKNOWN` | `push_subscriptions` の暗号化/復号、`PUT /push-subscriptions/status` | `GET /admin/push-status?status=DENIED` | S-07の絞り込み表示 |
| FR-09 Excel | `workMinutes = elapsed - break` | `GET /admin/exports/attendance.xlsx` がXLSXをストリームし監査が残る | `Content-Disposition` + 監査 | S-06のExcelダウンロード |
| FR-10 管理設定 | `is_active` 切替 | `POST/PATCH /admin/employees`、`.../calendar`、`.../deadlines/{month}` | `PUT dueAt +09:00` の扱い | S-08、S-09、S-10 |
| FR-11 監査 | `actor_type` のCHECK | `audit_logs` のトリガーがUPDATE/DELETEを拒否、`GRANT` | 変更系エンドポイントは全て監査を挿入 | 監査ログ照会（内部） |
| NFR-02 性能 | — | — | 50名×31日のExcelが30秒以内（タイマー付きAPIテスト） | Playwrightの360px + Chrome/Edge/Safari |

## 3. データベース契約テスト（機能より先に必ず成功させること）

* 新品の `postgres:16`（Testcontainers）にFlyway `V1`〜`V3` を適用する。
* `04_database.md:4` の `CHECK` 制約を検証する：JST境界（`23:00 JST = 14:00 UTC`）での `ck_attendance_work_date_jst`、`ck_submission_status_fields`、`ck_notification_*`、`ck_push_payload_state`、および `audit_logs` トリガー。
* ロール：`clokka_app` は `audit_logs` に対して `SELECT, INSERT` はできるが `UPDATE`/`DELETE` はトリガーで拒否される。`PUBLIC` に権限は与えない。

## 4. Push暗号化テスト

* 購読JSONは `AES-GCM`（12バイトIV）で暗号化し、同じ `key_version` で復号できること。異なる鍵での復号は `AEADBadTagException` で失敗すること。
* `DELETE /push-subscriptions/{id}` は `ciphertext/iv/version` をクリアし `revoked_at` を設定すること。`GET /admin/push-status` は平文のendpoint/鍵を決して返さないこと。

## 5. APIテスト規約

* ベースURL `http://localhost:8080/api/v1`（Testcontainers）で、`HttpOnly` セッションCookie + `X-CSRF-Token` ヘッダを使用する。
* すべてのテストで2xxおよび4xxに `X-Request-Id` が存在することを検証する。
* `401` vs `403` vs `404` は `07_security.md:2` のマトリクス（IDOR、無効化された管理者、CSRF）でテストする。
* `notification_deliveries` のat-most-onceは、同じ `notification_date` で `POST /internal/jobs/monthly-reminders` を2回実行し、2回目は `RESERVED` 行が0件挿入されることで検証する。

## 6. フロントエンド / E2E（Phase 9）

* Playwrightで S-01 → S-02/S-06、S-03保存、S-04提出、S-06差戻し、および `06_screen-design.md:6` と `NFR-01` の360pxレスポンシブをカバーする。
* MVPでは `frontend` の単体テストは行わない。50名規模ではE2EとAPIカバレッジで十分である。

## 7. CI品質ゲート（Phase 6で追加）

* `ci.yml` は `./gradlew check`（Spotless + テスト + Testcontainers）と秘密情報スキャン（例 `gitleaks` や `trufflehog`）を実行する。いずれかが失敗したPRはマージ不可とする。
* テストレポートとカバレッジ（JaCoCo、任意）は成果物としてアップロードする。

## 8. Phase 4でテストしないこと

* コードは書かない。本計画はPhase 6〜9の契約である。Phase 4で `V{NN}__` ファイルや `*Test.java` は作成しない。
* 負荷・セキュリティスキャンはPhase 9のみであり、Phase 6ではない。
