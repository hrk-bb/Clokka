# Clokka ディレクトリ設計（Phase 4）

| 項目 | 内容 |
| --- | --- |
| 目的 | AI・人間の開発者が、コード・テスト・マイグレーション・ドキュメント・CIの配置場所を推測なく把握できるリポジトリ構成を定義する。 |
| 対象読者 | 開発者、AIエージェント、レビュー担当者、プロダクトオーナー |
| 状態 | レビュー待ち — Q-04承認待ち |
| 最終更新日 | 2026-08-28 |
| 依存関係 | `01_requirements.md`、`02_architecture.md`、`03_tech-stack.md`、`04_database.md`、`05_api.md`、`06_screen-design.md`、`07_security.md` |

## 1. トップレベル構成

```
Clokka/
├── .github/
│   ├── workflows/          # GitHub Actions（CI、定時リマインド）
│   └── pull_request_template.md
├── backend/                # Spring Bootアプリケーション（Java 21）
├── frontend/               # Vanilla JS UI（HTML/CSS/ES Modules）
├── docs/                   # 設計・プロジェクト資料（本ディレクトリ）
│   ├── 00_project/         # current-state、handoff、decisions、roadmap、known-issues
│   ├── 00_プロジェクトロードマップ.md # 日本語版オリジナルロードマップ（履歴）
│   ├── 01_requirements.md .. 14_backlog.md
│   └── 06_screen-images/   # 06_screen-design.drawio からのPNGエクスポート
├── .gitignore
├── .env.example            # 秘密情報を含まない、キーとプレースホルダのみ
├── docker-compose.yml      # ローカル開発用（任意）
├── Dockerfile              # Render用（backend + frontend静的配信を同梱）
└── README.md
```

* リポジトリ直下の `src/` と `tests/` は**使用しない**。Phase 4前の正規化で `README.md:11-12` およびPhase 2の「frontendを分離して配信する」決定に合わせて `frontend/` と `backend/` に置換済み。
* `frontend/` と `backend/` は Phase 6で骨組みができるまで `.gitkeep` のみを置く。追跡を維持するため削除しないこと。

## 2. バックエンド（`backend/`）

```
backend/
├── build.gradle            # Gradle Wrapper、Java 21、Spring Boot 3
├── settings.gradle
├── src/
│   ├── main/
│   │   ├── java/com/clokka/
│   │   │   ├── ClokkaApplication.java
│   │   │   ├── identity/       # 認証、セッション、UserDetails、Argon2id
│   │   │   ├── attendance/     # 勤怠、work_dateのJSTロジック
│   │   │   ├── submission/     # 月次提出と検証
│   │   │   ├── admin/          # 管理者検索、差戻し、Excel、休日、締切
│   │   │   ├── notification/   # Push、deliveries、attempts、job API
│   │   │   ├── audit/          # 監査ログ（追記専用）
│   │   │   └── config/         # Security、Web、Flyway、Jackson設定
│   │   └── resources/
│   │       ├── application.yml          # 秘密情報なし、環境変数で上書き
│   │       ├── application-local.yml    # ローカル用プロファイル（必要ならgitignore）
│   │       └── db/migration/            # Flyway V1__..., V2__...（SQLのみ）
│   └── test/
│       ├── java/com/clokka/   # mainパッケージと同一構成
│       └── resources/
│           └── application-test.yml
├── .gitkeep                # Phase 6まで
└── README.md               # バックエンド単体の起動手順（Phase 6で追加）
```

* パッケージ境界は `02_architecture.md:3` のモジュラモノリスに従う：`identity`、`attendance`、`submission`、`admin`、`export`（`admin`内）、`notification`、`audit`。
* `db/migration` はFlywayのバージョン管理されたSQLのみを置く。MVPではJavaマイグレーションは使わない。
* `src/main/resources/static/` は**ビルド時に生成される**。`frontend/` をコピーしたものであり、手編集しないこと。

## 3. フロントエンド（`frontend/`）

```
frontend/
├── index.html              # 単一エントリ、Spring Bootから同一オリジン配信
├── css/
│   └── style.css           # 単一スタイルシート（フレームワークなし）
├── js/
│   ├── app.js              # ルーター / 起動処理
│   ├── api.js              # fetchラッパー + CSRF + X-Request-Id
│   ├── auth.js             # ログイン / me / ログアウト
│   ├── attendance.js       # S-02、S-03、S-04
│   ├── admin.js            # S-06、S-07、S-08、S-09、S-10
│   └── notification.js     # S-05、installation_id、Push
├── .gitkeep
└── README.md               # Phase 6で追加
```

* MVPではビルドツール・`node_modules`・バンドラは使わない（`03_tech-stack.md`）。ES Modulesを直接利用する。
* `frontend/` は `./gradlew build` 時に `backend/src/main/resources/static/` へコピーされる（Gradleの `copy` タスク）。正本は常に `frontend/` である。

## 4. ドキュメントと画像

```
docs/
├── 01_requirements.md      # Phase 1 承認済み
├── 02_architecture.md      # Phase 2 承認済み
├── 03_tech-stack.md        # Phase 2 承認済み
├── 04_database.md          # Phase 3 承認済み（DDL契約を含む）
├── 05_api.md               # Phase 3 承認済み
├── 06_screen-design.md     # Phase 3 承認済み
├── 06_screen-design.drawio # 視覚的正本（12ダイアグラム）
├── 06_screen-images/       # ダイアグラムごとのPNGエクスポート（記述名付き）
├── 07_security.md          # Phase 3 承認済み
├── 08_directory.md         # 本ファイル（Phase 4）
├── 09_development-rules.md
├── 10_branch-strategy.md
├── 11_test-plan.md
├── 12_deployment.md        # Phase 5（未着手）
├── 13_operation.md         # Phase 5
└── 14_backlog.md           # Phase 11
```

* `00_プロジェクトロードマップ.md`（日本語）は当初の計画、`00_project/roadmap.md`（英語）は進捗管理表。両方を保持し、英語版が `roadmap.md:8` の `Audit basis` である。
* `06_screen-images/` は draw.ioの各ダイアグラムに対応するPNGを1枚ずつ、記述名（`S-01 共通ログイン (PC).png` 等）で置く。`S-01.png` のような汎用連番は使わない。

## 5. CIとGitHub

```
.github/
├── workflows/
│   ├── ci.yml              # PRとmain：ビルド、テスト、Lint、秘密情報スキャン（Phase 6）
│   └── reminders.yml       # 定時 POST /internal/jobs/monthly-reminders（Phase 10、MVPでは任意）
└── pull_request_template.md
```

* `.github/workflows/` は現在 `.gitkeep` のみ。ワークフローはリポジトリ骨組みができた後のPhase 6で追加する。

## 6. ルート直下のファイル

| ファイル | 用途 | 秘密情報の扱い |
| --- | --- | --- |
| `.gitignore` | `.env`、`build/`、`.gradle/`、`node_modules/`（将来）、IDEファイルを無視 | Phase 6前に `.env` を必ず記載 |
| `.env.example` | 必要な環境変数のキーをプレースホルダ値で文書化（例 `DATABASE_URL=`） | 実際の秘密情報は書かない |
| `docker-compose.yml` | 開発用PostgreSQL（Phase 6） | `env_file` で `.env` を参照、ハードコードしない |
| `Dockerfile` | `eclipse-temurin:21-jre` ベース、ビルド済みbackend jarをコピー | 秘密情報を焼き込まない |
| `README.md` | `01_requirements.md` と `00_project/current-state.md` への入口 | 秘密情報を書かない |

## 7. 作成してはならないもの

* リポジトリ直下の `src/` や `tests/`（`frontend/`/`backend/` に置換済み）。
* `backend/src/main/resources/static/` の手編集ファイル（生成物）。
* `node_modules/`、`dist/`、`build/` のコミット（ignore対象）。
* 実値の入った `.env`、`*.pem`、`*.key`、社員CSV。

## 8. Phase 6前の検証

新規の開発者/AIが以下で起動できること：

```bash
git clone <url> && cd Clokka
cp .env.example .env   # ローカル値を記入、.envはコミットしない
docker compose up -d   # ローカルPostgreSQL起動
./gradlew :backend:build
```

手動操作なしで `frontend/` が `backend/build/resources/main/static/` にコピーされること。この検証はPhase 6の受け入れ条件であり、Phase 4では行わない。
