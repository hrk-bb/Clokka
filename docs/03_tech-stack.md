# Clokka 技術選定（Phase 2 承認済み版）

| 項目 | 内容 |
| --- | --- |
| 目的 | 優先順位（保守性、単純性、無料、セキュリティ、拡張性、性能）に基づく採用技術と移行判断を記録する。 |
| 対象読者 | プロダクトオーナー、開発者、運用管理者、レビュー担当者 |
| 更新タイミング | 技術・ライブラリ・ホスティングサービスを追加、変更、廃止する時 |
| 状態 | 承認済み・完了 |
| 最終更新日 | 2026-08-24 |

## 採用一覧

| 領域 | 採用案 |
| --- | --- |
| 言語・実行基盤 | Java 21 LTS、Spring Boot 3、Gradle Wrapper |
| UI | HTML、CSS、ES ModulesによるVanilla JavaScript（ビルドツール・UIフレームワークなし） |
| DB | Neon Free PostgreSQL（PostgreSQL 16以上、MVP） |
| DB変更管理 | Flyway OSS、バージョン管理したSQLマイグレーション |
| 認証 | 社員ID・パスワード、Spring Securityのサーバーセッション、Argon2idハッシュ |
| 通知 | Web Push（VAPID）＋アプリ内未提出警告 |
| Excel | Apache POI（XLSX） |
| 実行・公開 | Render Free Web Service、Dockerfile、GitHub連携による`main`自動デプロイ |
| CI | GitHub Actions |
| テスト | JUnit 5、Spring Boot Test、Testcontainers PostgreSQL、Playwright |

## 技術比較と決定

| 領域 / 比較対象 | 採用・不採用 | 採用理由・メリット | 不採用理由・デメリット | 将来の乗り換えコスト |
| --- | --- | --- | --- | --- |
| バックエンド: Spring Boot / Node.js Express / Cloudflare Workers | **Spring Bootを採用** | 指定構成に適合し、認証・検証・定時処理・監視を1つの成熟した枠組みで提供する。組込みサーバー、ヘルスチェック、外部設定を備える。[Spring Boot資料](https://docs.spring.io/spring-boot/docs/3.0.x/reference/htmlsingle/) | Expressは設計規約を別途揃える必要がある。Workersは運用負荷が小さい反面、Java/Spring資産を使えずベンダー固有APIが増える。 | REST API・SQL・UIを分離するため中程度。Workers移行はAPIと認証・通知の実装し直しが必要。 |
| UI: Vanilla JavaScript（HTML/CSS + ES Modules） / React / Next.js | **Vanilla JavaScriptを採用** | HTMLとCSSで画面を構成し、ES Modulesで分割したフレームワーク非依存のJavaScriptを使う。画面数・状態が限定的なため、依存関係・ビルド・更新作業を最小化できる。 | React/Next.jsは複雑なUIには有利だが、初期規模には学習・依存更新・ビルドの負担が上回る。 | 中程度。APIを維持して画面を段階的に置換できる。 |
| DB: Neon PostgreSQL / Render Free Postgres / MariaDB / SQLite / Cloudflare D1 | **Neon Free PostgreSQLを採用** | PostgreSQLの整合性・時刻型・制約を使え、Freeプランはクレジットカード不要・期限なしで0.5GB、100 CU時間/月を提供する。RenderサービスとはTLSで接続する。[Neon Pricing](https://neon.com/pricing) | Render Free Postgresは1GBだが30日で失効し、バックアップ非対応。MariaDBは標準化で劣後し、SQLite/D1は将来の配置先に強く依存する。 | 低〜中。標準SQL中心にし、`pg_dump`/`pg_restore`でAWS RDS PostgreSQLへ移行できる。 |
| 認証: ローカルセッション / OAuth/OIDC / Keycloak / JWT | **ローカルセッションを採用** | 50名の社内利用では最少の構成で、同一オリジンのHttpOnly CookieによりトークンをJSから隔離できる。パスワードはArgon2idで保存する。OWASPはArgon2idを推奨している。[OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) | OAuth/OIDCは既存IdPがあれば利便性が高いが、前提情報がなく、外部依存・設定・費用を増やす。Keycloakは自前運用負荷が大きい。JWTは失効・ブラウザ保管の設計を増やす。 | 中程度。`UserIdentity`境界を設ければ、後にOIDCログインを追加可能。 |
| パスワードハッシュ: Argon2id / bcrypt | **Argon2idを採用** | 新規システム向けのメモリハードな推奨方式。個別saltを使い、パラメータは本番VMで負荷確認する。 | bcryptは広く使われるが、OWASPはArgon2id等が利用できないレガシー用途に限定する。 | 低。ログイン成功時に旧ハッシュを再ハッシュする移行が可能。 |
| DB変更: Flyway OSS / 手動SQL / Liquibase | **Flyway OSSを採用** | SQLをコミットして変更履歴と本番適用順を管理できる。OSS版はOSSソース由来である。[Flyway資料](https://documentation.red-gate.com/fd/the-flyway-engine-oss-open-source-software-and-redgate-205226036.html) | 手動SQLは環境差分を生みやすい。Liquibaseは高機能だが、小規模では記法・設定が増える。 | 低。既存SQLを別ツールへ取り込める。 |
| 通知: Web Push / Email / SMS | **Web Pushを採用** | ブラウザのみで実現し、許可済み端末へ能動通知でき、ランタイム費用を増やさない。 | Emailには送信サービスと到達性の管理、SMSには従量課金が必要。Push拒否者へ届かないことが残る。 | 中。通知を`NotificationPort`で抽象化し、後に社内メール等を追加できる。 |
| Excel: Apache POI / CSVのみ / クライアント生成 | **Apache POIを採用** | 指定されたExcel形式（`.xlsx`）をサーバーで一貫して生成でき、権限制御と監査を集中できる。 | CSVのみは書式・複数列の互換性要求に弱い。クライアント生成は改ざん・再現性に不利。 | 低。出力DTOを保てばライブラリを差替え可能。 |
| MVPホスティング: Render / Fly.io / Railway | **Render Freeを採用** | GitHubリポジトリのDockerfileを使って自動デプロイでき、Java/JVMアプリをDockerで実行できる。[Render Docker](https://render.com/docs/docker) 支払方法を登録しなければ、無料枠超過時には課金でなくサービス停止となるため、MVPの費用上限を0円にできる。[Render Free](https://render.com/docs/free) | Fly.ioは新規組織にクレジットカードを要求するため不採用。[Fly Pricing](https://fly.io/docs/about/pricing/) Railwayは無カードの試用が可能だが、継続Free枠は月$1クレジットであり、MVPの継続稼働を保証しにくい。[Railway Pricing](https://docs.railway.com/pricing/plans) Renderも15分のアイドル停止・一時ファイルシステムという制約がある。 | 低〜中。Dockerfile、環境変数、PostgreSQL、GitHub Actionsを標準的に保つため、AWS ECS/EC2等へ移せる。 |
| 本番配備: AWS / Render有料 / Fly.io / Railway | **会社導入決定後に選定（現時点では未採用）** | 会社名義のアカウント、予算、個人情報取扱い、監査要件に沿って選べる。AWSはDockerイメージをECS/EC2へ置き、DBをRDS PostgreSQLへ段階移行できる。 | AWSを含む商用クラウドは継続費用が発生し得る。Render Freeは本番利用非推奨で、Fly.ioはカード必須、Railwayは使用量制である。 | 低〜中。クラウド固有機能を導入しないため、主な移行はIaC、Secrets、DB移送、通知スケジュールの置換。 |
| TLS: Render管理TLS / Caddy / Nginx + Certbot | **Render管理TLSを採用（MVP）** | RenderのHTTPS URLを利用し、MVPで証明書運用を増やさない。AWS移行後にEC2を採用する場合のみCaddy、ALB、Nginx等を比較して選定する。 | Caddy/NginxはVM上では有用だが、Render MVPでは不要なコンポーネントになる。アプリ直公開はTLS設定の責任が増える。 | 低。標準HTTPSのため、移行時にリバースプロキシのみ追加すればよい。 |
| CI: GitHub Actions / 手動確認 / 外部CI | **GitHub Actionsを採用** | リポジトリと近く、PRごとにテストを必須化できる。 | 手動確認は再現性が低い。外部CIはアカウント・無料枠・連携を増やす。privateリポジトリの利用枠は運用時に確認する。 | 低。CI設定を他サービスへ移植できる。 |

## バージョン・依存関係の方針

1. JavaはLTSのJava 21を使用する。Spring BootはJava 21に対応する現行安定版を、実装開始時点で固定する。
2. Dockerイメージは可変タグでなく、メジャー・マイナーを固定し、更新はDependabot等のPRで検証する。
3. 依存関係は最小限にし、追加時はこの文書に選定理由を追記する。
4. ランタイムAI/APIは使用しない。AIは設計・実装支援に限定し、社員データやSecretsを外部プロンプトへ送らない。

## 決定・リスク記録

| ID | 内容 | 状態 |
| --- | --- | --- |
| D-05 | Render Freeの15分アイドル停止・再起動は、会社提出用MVPの制約として受容する。正式運用前にはホスティングを再選定・移行する。 | 承認済み |
| R-02 | Render Freeでは初回応答遅延と常時実行の定時処理不可が起こり得る。GitHub Actions起動の通知と管理者一覧でMVPは補完する。 | MVPでは受容、本番移行時に解消 |

## 承認記録

次の構成はPhase 2で承認済みである。(1) Render Free + Neon Freeによる会社提出用MVP、(2) DockerfileとGitHub連携による`main`自動デプロイ、(3) Spring Boot + PostgreSQL +同一オリジンVanilla JS、(4) ローカルセッションとArgon2id、(5) Web Pushと管理者のPush通知状態一覧による提出漏れ対策。本番クラウドは導入決定後に会社名義で選定する。現在はPhase 3で実装詳細をレビュー中である。
