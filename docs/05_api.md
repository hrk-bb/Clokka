# Clokka API設計（Phase 3 レビュー版）

| 項目 | 内容 |
| --- | --- |
| 目的 | UI・定時処理・将来の移行先が共有するREST APIの契約を定義する。 |
| 対象読者 | フロントエンド/バックエンド開発者、テスト担当者、レビュー担当者 |
| 更新タイミング | エンドポイント、認可、入出力、エラー形式を変更する時 |
| 状態 | 承認済み（Q-01 2026-08-28）+ 追補 2026-08-28（Bootstrap/招待/最後のADMIN） |
| 最終更新日 | 2026-08-28 |

## 1. 共通規約

- ベースURLは`/api/v1`。UIと同一オリジンで提供する。
- JSONはUTF-8、応答日時はISO 8601 UTC、勤務日は`YYYY-MM-DD`（JST）とする。締切更新の`dueAt`だけはJSTを明示したISO 8601の`+09:00`オフセットを必須とする。
- 認証は`HttpOnly; Secure; SameSite=Lax`のセッションCookie。状態変更リクエストにはCSRFトークンを必須とする。
- すべてのレスポンスに`X-Request-Id`を付け、監査・障害調査に使う。
- エラー形式は`{ "code": "ATTENDANCE_LOCKED", "message": "提出済みの月は編集できません。", "fieldErrors": [] }`とする。

## 2. 認証・本人情報

| Method / Path | 権限 | 概要 |
| --- | --- | --- |
| `POST /auth/login` | 未認証 | 社員ID・パスワードでログインし、セッションを発行する。失敗時は同一メッセージを返す。 |
| `POST /auth/logout` | ログイン済み | セッションを破棄する。 |
| `GET /me` | ログイン済み | 自分の表示名、権限、部署、Push集約状態を返す。 |
| `GET /csrf` | ログイン済み | 状態変更用のCSRFトークンを返す。 |
| `POST /auth/activate` | 未認証（招待トークン） | 招待トークンと新パスワードで社員を有効化する。`token`（平文）と `newPassword` を受け取る。成功時は `is_active=true` となりセッションを発行する。トークンは1回限り、有効期限24時間。 |

`POST /auth/activate` は `employee_invitations.token_hash` と突合し、`expires_at` と `used_at IS NULL` を検証する。失敗時は `404`（トークン不明）または `410`（期限切れ/使用済み）を返し、成功時は `used_at` を設定し監査ログ `EMPLOYEE_ACTIVATED` を残す。`BOOTSTRAP_ADMIN_*` による初期ADMIN作成はAPIではなく、1つのトランザクションで `SELECT pg_advisory_xact_lock(hashtext('bootstrap_admin'))` 後に件数検証する `ApplicationRunner` の冪等起動処理であり、`POST /auth/activate` や `POST /admin/employees` で代替しない。公開の `/setup` 画面/APIは設けない。`departments` は `V2` で既定の `未所属`（`00000000-0000-0000-0000-000000000001`）がSeedされ、空DBでもBootstrap可能である。`BOOTSTRAP_ADMIN_DEPARTMENT_ID` が未指定の場合は既定部署を使用する。初期管理者のパスワード変更は推奨（必須ではなく、専用のパスワード変更API/UIは未定義のため任意）とし、実施した場合は監査する。

## 3. 社員勤怠・提出

| Method / Path | 権限 | 概要・主な応答 |
| --- | --- | --- |
| `GET /attendance?month=YYYY-MM` | ログイン済み（自分。EMPLOYEE/ADMIN） | 自分の月次日別勤怠、対象日、未入力日、合計勤務時間、提出状態。 |
| `PUT /attendance/{workDate}` | ログイン済み（自分。EMPLOYEE/ADMIN） | 指定勤務日の勤怠をupsertする。`clockIn`, `clockOut`, `breakMinutes`, `note`を受け取る。提出済み月は`409 ATTENDANCE_LOCKED`。 |
| `DELETE /attendance/{workDate}` | ログイン済み（自分。EMPLOYEE/ADMIN） | 提出前の勤怠を削除する。監査ログを残す。 |
| `POST /submissions/{month}/validate` | ログイン済み（自分。EMPLOYEE/ADMIN） | 提出前チェック。未入力日・不正項目を返す。状態は変更しない。 |
| `POST /submissions/{month}` | ログイン済み（自分。EMPLOYEE/ADMIN） | チェック成功時に`DRAFT/RETURNED`から`SUBMITTED`へ遷移する。 |
| `GET /submissions/{month}` | ログイン済み（自分。EMPLOYEE/ADMIN） | 自分の対象月の提出状態・差戻し理由を返す。 |

`PUT /attendance/{workDate}`の例：

```json
{
  "clockIn": "09:00",
  "clockOut": "18:00",
  "breakMinutes": 60,
  "note": ""
}
```

サーバーは`workDate`とJST時刻から時刻を構成する。`clockOut <= clockIn`の場合のみ翌日退勤と判定する。

対象月の`monthly_submissions`が未作成なら、`PUT /attendance/{workDate}`で勤怠保存と同時に`DRAFT`を作成する。提出APIは先にチェックを行い、成功時だけ`SUBMITTED`レコードを作成する。`GET`系APIはレコードを作成せず、未作成状態を`DRAFT`として返す。これにより、未操作の社員も管理一覧から漏れない。

## 4. Push通知状態

| Method / Path | 権限 | 概要 |
| --- | --- | --- |
| `PUT /push-subscriptions/status` | 社員 | `installationId`とブラウザの通知許可状態（`DENIED`/`DEFAULT`/`UNSUPPORTED`）を報告する。購読なしの`GRANTED`は`DEFAULT`として扱う。 |
| `POST /push-subscriptions` | 社員 | `installationId`とVAPID購読情報を登録・更新する。登録成功時にだけ`GRANTED`とする。 |
| `DELETE /push-subscriptions/{id}` | 社員 | 自分の端末購読を論理解除する。物理削除は行わず、購読データを消去して端末状態を`DEFAULT`へ遷移させる。 |

ブラウザがPushを未対応、または一度も状態を報告していない社員は、管理者一覧でそれぞれ`UNSUPPORTED`、`UNKNOWN`として扱う。複数端末では有効購読が1件でもあれば`GRANTED`、なければ`DEFAULT`端末が1件でもあれば`DEFAULT`、すべての対応端末が`DENIED`なら`DENIED`、すべて非対応なら`UNSUPPORTED`、報告がなければ`UNKNOWN`とする。

`installationId`はブラウザ初回設定時に生成するランダムUUIDであり、物理端末情報やフィンガープリントではない。同一`employeeId`と`installationId`の報告はupsertし、同じブラウザでPush endpointが変わっても1設置として扱う。Push購読のendpoint・鍵はレスポンス、監査ログ、アプリケーションログへ含めない。

## 5. 管理API

| Method / Path | 権限 | 概要 |
| --- | --- | --- |
| `GET /admin/submissions?month=&departmentId=&status=&sort=&order=` | 管理者 | 提出状態、未入力日数、提出日時をページング付きで返す。 |
| `POST /admin/submissions/{employeeId}/{month}/return` | 管理者 | `SUBMITTED`を`RETURNED`へ変更。`reason`必須。 |
| `GET /admin/push-status?status=` | 管理者 | 社員単位に集約したPush状態の一覧。`DENIED`で拒否者を抽出可能。 |
| `GET /admin/exports/attendance.xlsx?month=&departmentId=&status=` | 管理者 | フィルタ済み月次Excelをストリーム返却し、出力監査ログを残す。 |
| `GET /admin/employees` | 管理者 | 社員一覧を返す。パスワードやトークンは含めない。 |
| `POST /admin/employees` | 管理者 | 社員を招待として作成する。`employee_code`, `display_name`, `department_id`, `role` を受け取る。`password` は受け取らない。作成時は `is_active=false` とし、招待トークンを発行して `employee_invitations` に保存する。平文トークンはレスポンスに1回のみ含め、管理画面でコピーして本人へ渡す。監査 `EMPLOYEE_INVITED` を残す。再招待時は既存の有効な未使用招待を無効化して新トークンを発行し、社員ごとに有効な未使用招待は1件のみとする。 |
| `PATCH /admin/employees/{id}` | 管理者 | 部署・権限・有効状態を変更する。`role` または `is_active` の変更時、有効な `ADMIN` が0人になる場合は `409 LAST_ADMIN_RESTRICTION` で拒否する（同一トランザクションで `SELECT pg_advisory_xact_lock(hashtext('last_admin_protection'))` 後に `SELECT COUNT(*) FROM employees WHERE role='ADMIN' AND is_active=true` で検証）。成功時は監査 `ROLE_CHANGED` / `EMPLOYEE_DEACTIVATED` を残す。 |
| `GET/POST/PATCH /admin/calendar` | 管理者 | 会社休日・勤務日例外を管理する。 |
| `GET /admin/deadlines/{month}` | 管理者 | 対象月の永続化済み締切を返す。`dueAt`はISO 8601 UTC、画面表示と入力基準はJST。 |
| `PUT /admin/deadlines/{month}` | 管理者 | 月次締切を作成・更新する。本文の`dueAt`はJSTオフセット付き日時を必須とし、変更は監査ログへ記録する。 |

## 6. 定時ジョブAPI

`POST /internal/jobs/monthly-reminders`はGitHub Actionsだけが呼び出す。`X-Job-Secret`が一致せず、または呼出元のレート制限に違反した場合は`401/429`とする。ジョブは、対象月の永続化済み締切とJST日付から通知段階を判定する。

社員ごとに、Pushサービスへの送信前に`notification_deliveries`へ`UNIQUE(employee_id, notification_date)`を満たす予約をトランザクションで確定する。競合した場合はその社員へ当日中に再送しない。予約のコミット後に全ての有効購読端末へ送信し、論理送信と端末別試行の結果を記録する。外部Push送信後のクラッシュでは送達漏れを許容し、重複送信を避けるat-most-once方式とする。

## 7. HTTP状態コード

| 状態 | 用途 |
| --- | --- |
| 200 / 201 / 204 | 取得成功 / 作成成功 / 本文なし成功 |
| 400 | 入力形式・業務バリデーション違反 |
| 401 / 403 | 未認証 / 権限不足・CSRF不正 |
| 404 | 非公開または存在しないリソース |
| 409 | 提出済み編集、状態遷移競合、`LAST_ADMIN_RESTRICTION`（最後の有効ADMINの降格/無効化を拒否） |
| 410 | 招待トークンの期限切れ/使用済み（`POST /auth/activate`） |
| 422 | 提出前チェック不通過 |
| 429 | ログイン・ジョブAPIのレート超過 |
| 500 | 予期しない障害（詳細・Secretsを返さない） |

