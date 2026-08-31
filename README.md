# Clokka

社員約50名向けの、月末の勤務実績提出忘れをなくす出退勤管理Webアプリです。現在はPhase 3（詳細設計のレビュー段階）であり、アプリケーションコード・テスト・CI/CD・デプロイ設定はまだ作成されていません。
Codex autonomous Git workflow test
opencode autonomous Git workflow verification (2026-08-25)

設計の正本は [docs/01_requirements.md](docs/01_requirements.md) です。開発の現在地と次工程は [docs/00_project/current-state.md](docs/00_project/current-state.md) を参照してください。`docs/01_要件定義.md` は履歴文書です。

## 構成

- `frontend/`: HTML、CSS、JavaScript（未実装）
- `backend/`: Spring Boot API（未実装）
- `docs/`: 要件・設計・プロジェクト引き継ぎ資料
- `.github/workflows/`: CI（未作成）

> **公開リポジトリ確認 (2026-08-28):** 本リポジトリはプライベートからパブリック (`PUBLIC`) へ変更されました。 `gh repo view` で `visibility: PUBLIC` を確認済み。 — https://github.com/hrk-bb/Clokka
