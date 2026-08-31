# Clokka セキュリティ設計（Phase 3 レビュー版）

| 項目 | 内容 |
| --- | --- |
| 目的 | 勤怠・認証情報・Push購読情報を守るための脅威、対策、実装・運用上の確認点を定義する。 |
| 対象読者 | 開発者、運用管理者、レビュー担当者、プロダクトオーナー |
| 更新タイミング | 認証、公開範囲、依存関係、データ取扱い、脅威の変化時 |
| 状態 | 承認済み（Q-01 2026-08-28）+ 追補 2026-08-28（Bootstrap/招待/最後のADMIN） |
| 最終更新日 | 2026-08-28 |

## 1. 保護対象と脅威

| 保護対象 | 主な脅威 | 対策 |
| --- | --- | --- |
| パスワード | 漏えい、総当たり、平文保存 | Argon2idハッシュ、ログインレート制限、パスワードをログ・API応答に含めない。 |
| セッション | 窃取、固定化、CSRF | HTTPS、`HttpOnly`/`Secure`/`SameSite=Lax` Cookie、ログイン時のセッションID再発行、CSRFトークン、ログアウト時失効。 |
| 勤怠データ | 他社員閲覧、改ざん、削除 | サーバー側認可、IDORテスト、提出済みロック、監査ログ、DB最小権限。 |
| 管理操作 | 権限昇格、誤操作 | `ADMIN`のサーバー側認可、差戻し理由必須、Excel出力・権限変更の監査。 |
| Push購読情報 | endpoint・購読鍵の漏えい、なりすまし | 購読JSON全体をアプリケーション層AES-256-GCMで暗号化してDBへ保存し、管理画面・API応答・ログへ出さない。VAPID秘密鍵と暗号化鍵は環境変数で管理する。 |
| 端末識別 | 過度な追跡・フィンガープリント | 物理端末情報を収集せず、ブラウザごとのランダムUUIDだけをPush購読に紐づける。 |
| Render/Neon Secrets | Git漏えい、CIログ漏えい | `.env`をGit除外、Render環境変数・GitHub Secretsのみで注入、マスク・ローテーション手順。 |
| 外部公開API | XSS、SQL注入、DoS | 入力検証、パラメータ化SQL/JPA、出力エスケープ、CSP、レート制限、リクエスト上限。 |

## 2. 認証・認可

1. パスワードはArgon2id（最低19MiB、反復2、並列度1を出発点とし、本番負荷で調整）でハッシュ化する。
2. アカウント作成・初期パスワード配付・再設定は管理者が本人確認の上で行う。メール送信によるリセットは初期対象外。
3. ログイン失敗は社員IDの存在を示さず、IP・社員ID単位でレート制限する。
4. `EMPLOYEE`は常に認証済み社員IDを条件に検索し、リクエストの`employeeId`を信用しない。
5. `ADMIN` APIはロールだけでなく、無効化済みアカウントを拒否する。
6. 初期管理者のBootstrapは `BOOTSTRAP_ADMIN_*` 環境変数からのみ実行し、有効な `ADMIN` が0人の場合のみ冪等に1件を作成する。公開の `/setup` 画面/APIは設けない。`departments` は `V2` で既定の `未所属` をSeedするため空DBでも実行可能である。初期管理者のパスワード変更は推奨（必須ではなく、専用API/UIは未定義のため任意）とする。招待トークンは128bit以上のランダム値、SHA-256でハッシュ化して保存し、有効期限24時間、1回限りで無効化する。再招待時は既存の有効な未使用招待を無効化して新トークンを発行する。
7. 最後の有効 `ADMIN`（`role='ADMIN' AND is_active=true` が1件）の `role` 降格または `is_active=false` は、アプリケーションで `SELECT pg_advisory_xact_lock(hashtext('last_admin_protection'))` 後に `SELECT COUNT(*) FROM employees WHERE role='ADMIN' AND is_active=true` で件数検証した後に `409 LAST_ADMIN_RESTRICTION` で拒否する。DBの `CHECK` では人数を強制しない。

## 3. Web防御

| 項目 | 方針 |
| --- | --- |
| HTTPS | RenderのHTTPS URLをMVPの必須経路とする。HTTPアクセスを許可しない。 |
| CSP | `default-src 'self'`を基本とし、Pushに必要な接続先だけを明示追加する。インラインscriptは避ける。 |
| XSS | テンプレート・DOM挿入時にエスケープし、ユーザー備考をHTMLとして解釈しない。 |
| CSRF | Cookie認証の状態変更APIにCSRFトークンを必須化する。 |
| CORS | 同一オリジンのため原則無効。将来別オリジンにしてもワイルドカードを使わない。 |
| Security Headers | HSTS、`X-Content-Type-Options: nosniff`、`Referrer-Policy: strict-origin-when-cross-origin`、クリックジャッキング対策を設定する。 |

## 4. データ・運用セキュリティ

- 実社員データをRender Free + Neon FreeのMVP基盤で扱うことは承認済み（`D-06`）。HTTPS、認可、監査ログ、Secrets管理を必須とする。
- Neon接続はTLS必須とし、接続文字列をGit・画面・ログに出さない。
- DBユーザーはアプリ専用とし、スキーマ変更用ユーザーを分離する。`audit_logs`はアプリ用DBロールへ`SELECT`/`INSERT`だけを付与し、`UPDATE`/`DELETE`は付与しない。さらにDBトリガーで更新・削除を拒否する。
- 依存関係の脆弱性チェックをCIに組み込み、重大脆弱性はリリースを停止する。
- 監査ログはアプリ利用者が編集・削除できない。
- Secrets漏えいが疑われた場合、Render/Neon/GitHubの該当Secretsを直ちに失効・再発行し、監査ログを調査する。
- `BOOTSTRAP_ADMIN_PASSWORD` と `PUSH_SUBSCRIPTION_ENCRYPTION_KEY` / `VAPID_PRIVATE_KEY` はいずれもRender環境変数でのみ管理し、Git、DB、監査ログ、通常ログ、APIレスポンスに出さない。Bootstrapで作成された初期管理者のパスワードは初回ログイン後に変更可能とし、変更は `audit_logs` に `PASSWORD_CHANGED` として記録する（平文は記録しない）。招待トークンの平文はDBに保存せず、発行時に1回のみ管理者へ返却する。

## 5. 定時通知APIの防御

`/internal/jobs/monthly-reminders`は、ランダム生成した`X-Job-Secret`、レート制限、リクエストID、JST日付ごとのDB一意制約で守る。SecretはGitHub Actions SecretsとRender環境変数にだけ設定する。失敗詳細をHTTP応答へ出さず、ログにもPush endpointを記録しない。送信前に永続予約をコミットするため、障害時は重複送信より送達漏れを優先して防ぐ。

### Push購読データの保護

- `endpoint`、`p256dh`、`auth`を含む購読JSONは、アプリケーションでAES-256-GCM暗号化してから`BYTEA`として保存する。PostgreSQLには暗号文、96-bit IV、鍵バージョンだけを置く。
- 32-byteの`PUSH_SUBSCRIPTION_ENCRYPTION_KEY`はRender環境変数だけで管理し、Git、DB、監査ログ、通常ログ、APIレスポンスへ置かない。VAPID秘密鍵とは別の鍵にする。
- 鍵ローテーションは鍵バージョンを増やし、読取時または保守ジョブで再暗号化する。旧鍵は未ローテーション暗号文がなくなるまで環境変数の限定された復旧経路だけに保持する。
- Push購読を解除または404/410で失効した時は、暗号文・IV・鍵バージョンを消去し、端末設置行と失効時刻・理由は保持する。これにより不要な秘密データを残さず、状態集約と監査性を維持する。
- DBバックアップから復元するには対応する暗号化鍵が必要である。鍵のバックアップ・復旧手順はPhase 5運用設計で検証する。

### Bootstrapと最後の管理者

- Bootstrapは `ApplicationRunner` で `BOOTSTRAP_ADMIN_*` を読み、1つのトランザクションで `SELECT pg_advisory_xact_lock(hashtext('bootstrap_admin'))` 後に件数検証し、有効ADMINが0件の時のみ実行する。既に存在する場合は何もせず、再実行してもADMINを増やさない。公開APIではないためレート制限の対象外だが、環境変数が未設定の場合は起動時に警告ログを残しつつ何も作成しない。
- 最後のADMIN保護は、UIでは該当行の降格/無効化ボタンを無効化しつつ、APIでは `409` で二重に拒否する。DBでは人数を `CHECK` しない。

## 6. MVPの受容リスク

| ID | リスク | 条件・対応 |
| --- | --- | --- |
| D-06 | Render Free + Neon FreeのMVP基盤で実社員データを扱うことをプロダクトオーナーが承認した。 | 承認済み。実装・運用ではHTTPS、認可、監査、Secrets管理を必須とする。 |
| R-05 | Render FreeとGitHub Actionsのスケジュールは高可用性を保証しない。 | 月次通知はベストエフォートとし、管理者の未提出一覧を確認手段とする。受容済み。正式運用ホスティング移行前に再設計する。 |

