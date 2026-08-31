# Clokka

社員約50名向けの、月末の勤務実績提出忘れをなくす出退勤管理Webアプリです。現在はPhase 4（開発・品質・Gitルール設計のレビュー段階。Phase 3までは承認済み）であり、アプリケーションコード・テスト・CI/CD・デプロイ設定はまだ作成されていません。

設計の正本は [docs/01_requirements.md](docs/01_requirements.md) です。開発の現在地と次工程は [docs/00_project/current-state.md](docs/00_project/current-state.md) を参照してください。移行前に存在した日本語ファイル名の設計文書は、内容を上記の正本へ移行した後、2026-08-29に削除済みです（[docs/01_requirements.md](docs/01_requirements.md) 8章「文書移行方針」参照）。

## 構成

- `frontend/`: HTML、CSS、JavaScript（未実装）
- `backend/`: Spring Boot API（未実装）
- `docs/`: 要件・設計・プロジェクト引き継ぎ資料
- `.github/workflows/`: CI（未作成）
