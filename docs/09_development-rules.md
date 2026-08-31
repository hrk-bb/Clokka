# Clokka 開発ルール（Phase 4）

| 項目 | 内容 |
| --- | --- |
| 目的 | 承認済みのPhase 3契約に沿って、AI・人間の開発者が一貫した方法でコードを記述・レビュー・テストするためのルールを定める。 |
| 対象読者 | 開発者、AIエージェント、レビュー担当者 |
| 状態 | レビュー待ち — Q-04承認待ち |
| 最終更新日 | 2026-08-28 |
| 依存関係 | `08_directory.md`、`01_requirements.md`〜`07_security.md`、`03_tech-stack.md` |

## 1. 基本原則

1. **Single Source of TruthはGit。** 設計や要件をチャットやローカルファイルだけに残さない。
2. **承認ゲートは必須:** `設計 → レビュー → PO承認 → 実装 → レビュー → 修正 → テスト → コミット → 次工程`。Q-04承認前にPhase 6のコードを開始しない。
3. **契約の無断変更禁止。** 要件・ADR・API・DDLの不変条件を変更する必要がある場合は `Q-xx` を起票し、`decisions.md` にADRを追記する。
4. **小さく単一目的のコミット。** 1コミット1目的。ドキュメントとコードの混在は `chore: normalize structure` のような場合を除き行わない。
5. **Gitに秘密情報を置かない。** `07_security.md:4` と `.gitignore` のルールを参照。

## 2. 言語とツール

| 領域 | ルール |
| --- | --- |
| Java | **Java 21 LTS**（`eclipse-temurin:21`）。プレビュー機能は使わない。 |
| ビルド | Gradle Wrapper（`./gradlew`）。Mavenは使わない。`gradle-wrapper.jar` はコミットする。 |
| バックエンド | Spring Boot 3.x（Phase 6開始時点で最新の3.2.xを `backend/gradle/libs.versions.toml` または `build.gradle` に固定）。 |
| フロントエンド | Vanilla JS（ES Modules）、HTML、CSS。React/Vue、バンドラ、`node_modules` はMVPでは使わない。 |
| DBマイグレーション | Flyway OSS、SQLファイルのみ: `backend/src/main/resources/db/migration/V{NN}__{説明}.sql`。 |
| フォーマット | Java: Spotless経由のGoogle Java Format（または `gradle spotlessCheck`）。JS/CSS: MVPではフォーマッタなし、`editorconfig` は必要に応じて最小限に追加。 |
| Lint / 静的解析 | バックエンド: SpotBugsまたはErrorProneのいずれか一方 + Checkstyle（import用）。フロントエンド: MVPでは手動レビューのみ。 |
| エディタ | `.editorconfig` で `charset=utf-8`、`end_of_line=lf`、`insert_final_newline=true`、`trim_trailing_whitespace=true`。 |

## 3. バックエンド規約

* **パッケージ構成**は `08_directory.md:2` に従う（`identity`、`attendance`、`submission`、`admin`、`notification`、`audit`、`config`）。ADRなしでトップレベルパッケージを新設しない。
* **コントローラは薄く保つ:** 検証、サービス呼び出し、DTO返却のみ。業務ロジックはコントローラではなく `Service` に置く。
* **検証**
  * リクエストDTOは `jakarta.validation`（`@NotBlank`、`@Pattern` 等）と明示的なJSTの `work_date` チェックを併用する。
  * `04_database.md:4` のDB不変条件（`CHECK` 制約）はアプリケーション側でSQLとして二重化しない。DBで強制し、結合テストでカバーする。
* **トランザクション:** サービス層で `@Transactional` を使用する。勤怠書き込み+監査+`DRAFT`生成（`PUT /attendance`）および提出+監査は単一トランザクション。通知の予約はPush呼び出しの前にコミットされる別トランザクション（`04:5` のat-most-once）。
* **時刻:** 永続化する瞬間は `Instant`/`TIMESTAMPTZ`、業務日付は `Asia/Tokyo` として解釈される `LocalDate` を使用する。保存に `LocalDateTime` は使わない。変換は `clock_in_at.atZone(ZoneId.of("Asia/Tokyo")).toLocalDate()` で行う。
* **エラーハンドリング:** `ResponseStatusException` または小さな `@ControllerAdvice` で `05_api.md:1` の `{code, message, fieldErrors}` を返し、秘密情報やスタックトレースを漏らさない。
* **ログ:** すべてのリクエストで `X-Request-Id` を記録する。`password`、`subscription_ciphertext`、`subscription_iv`、`PUSH_SUBSCRIPTION_ENCRYPTION_KEY`、VAPID秘密鍵をログに出さない。Pushのendpointは `INFO` で記録しない。

## 4. フロントエンド規約

* **フレームワークなし。** `fetch` + `async/await`、ES Modules、素のDOMを使用する。`js/api.js` を唯一の `fetch` ラッパーとし、`X-Requested-With`/`X-CSRF-Token` の付与と `401` → `S-01` へのリダイレクトを集約する。
* **CSRF:** 起動時および状態変更前に `GET /api/v1/csrf` でトークンを取得し `X-CSRF-Token` で送信する。トークンはメモリに保持し、ログアウトを跨いで `localStorage` に残さない。
* **JST表示:** APIはUTCで返すが、UIでは `Intl.DateTimeFormat("ja-JP", {timeZone:"Asia/Tokyo"})` で整形する。`work_date` や `due_at` にブラウザのローカルタイムゾーンを使わない。
* **アクセシビリティ:** `06_screen-design.md:6`（360px、ラベル、フォーカス、44pxターゲット、色+テキスト）に準拠する。
* **状態管理:** グローバルストアは持たない。各画面は `GET /me`、`GET /attendance?month=` 等で自身のデータを個別に取得し、変更後に再取得する。

## 5. データベースとマイグレーション

* **マイグレーションはSQLのみ**で、`main` にマージされた後は不変とする。マージ済みの `V1__` ファイルを編集せず、必ず新規の `V{NN}__` を追加する。
* **最初のマイグレーション（Phase 6骨組み）:**
  * `V1__extensions_and_roles.sql` — `pgcrypto`（`gen_random_uuid()`）、ロール `clokka_owner` / `clokka_app`。
  * `V2__core_tables.sql` — `departments`、`employees`、`attendance_records`、`monthly_submissions`、`company_calendar`、`submission_deadlines`、`push_subscriptions`、`notification_deliveries`、`notification_delivery_attempts`、`audit_logs` および全ての `CHECK`、`UNIQUE`、`FK ON DELETE RESTRICT`。
  * `V3__indexes_and_triggers.sql` — `04:7` のインデックスと `reject_audit_log_mutation` トリガー。
  * `V4__employee_invitations.sql` — `employee_invitations`（招待トークンのハッシュ、期限、単回利用）とそのインデックス（追補 2026-08-28）。
* **コミットされる `application.yml` に秘密情報を置かない。** `${DATABASE_URL}`、`${PUSH_SUBSCRIPTION_ENCRYPTION_KEY}`、`${VAPID_PRIVATE_KEY}`、`${BOOTSTRAP_ADMIN_PASSWORD}` 等はRenderの環境変数またはローカルの `.env` から注入する。`.env` はgitignoreする。

## 6. セキュリティ規約（`07_security.md` より）

* パスワード: Argon2id（`m=19MiB`、`t=2`、`p=1` を最小値としPhase 6でチューニング）で、承認済みライブラリを使用し1つのテストベクタで検証する。
* セッション: `HttpOnly; Secure; SameSite=Lax`、`SESSION_COOKIE` 名、`server.servlet.session.cookie.secure=true`、セッション固定化対策（`migrateSession`）、ログアウト時の無効化。
* レート制限: ログインと `POST /internal/jobs/monthly-reminders` はレート制限する（MVPではDBではなくインメモリ）。
* `audit_logs` のトリガーと `GRANT`（`04:7` / `07:4`）は、監査に関わる機能をマージする前に必須とする。
* Push購読の `subscription_ciphertext`/`iv`/`key_version` の扱い（`07:5`）は、Push機能をマージする前に必須とする。
* Bootstrapの `BOOTSTRAP_ADMIN_*` は `ApplicationRunner` で1つのトランザクションで `SELECT pg_advisory_xact_lock(hashtext('bootstrap_admin'))` 後に件数検証し、有効ADMINが0件の時のみ冪等に1件を作成し、既に存在する場合は何もしない。`departments` は `V2` で既定の `未所属` をSeedするため空DBでも作成可能である。Git・DB・監査ログ・通常ログに平文を出さない。初期管理者のパスワード変更は推奨（必須ではなく、専用APIは未定義のため任意）とし、実施した場合は `PASSWORD_CHANGED` として監査する。再招待時は既存の有効な未使用招待を無効化して新トークンを発行する。
* 最後の有効ADMINの降格/無効化は `409 LAST_ADMIN_RESTRICTION` で拒否する。1つのトランザクションで `SELECT pg_advisory_xact_lock(hashtext('last_admin_protection'))` 後に `SELECT COUNT(*) FROM employees WHERE role='ADMIN' AND is_active=true` で件数を検証し、0件になる操作はロールバックする。DBの `CHECK` では人数を強制しない。

## 7. Gitの運用

* ブランチ、コミット、PRは `10_branch-strategy.md` を参照。
* コミットメッセージの接頭辞: `feat:`、`fix:`、`docs:`、`chore:`、`test:`、`refactor:`（conventional、英小文字）。
* プッシュ前にローカルで `./gradlew check`（Spotless + テスト）を必ず通過させること。

## 8. Phase 6タスクの完了条件

タスクは以下を全て満たした時のみ完了とする：本ファイルと `08_directory.md` に準拠していること、Flywayマイグレーションが空のDBで適用できること、単体・結合テストが成功すること、秘密情報がコミットされていないこと、変更したエンドポイントに `X-Request-Id` が存在すること、関連する `known-issues.md` の項目が更新されていること。

## 9. Phase 4で許可されないこと

* Java/JSコード、`Dockerfile`、`docker-compose.yml` の本体、`実値の入った.env` の作成。
* 新たな `Q-xx` とADRなしでの `01`〜`07` の契約変更。
