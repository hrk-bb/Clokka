# Clokka ブランチ戦略とGitルール（Phase 4）

| 項目 | 内容 |
| --- | --- |
| 目的 | AI・人間が履歴を汚染せず秘密情報を漏らさずにブランチ、コミット、PRを作成する方法を定義する。 |
| 対象読者 | 開発者、AIエージェント、レビュー担当者 |
| 状態 | レビュー待ち — Q-04承認待ち |
| 最終更新日 | 2026-08-28 |
| 依存関係 | `08_directory.md`、`09_development-rules.md`、`AGENTS.md` |

## 1. ブランチモデル

* **メインブランチ:** `main`（保護）。直接pushは禁止し、すべての変更はPR経由。`main` は常にRenderへデプロイ可能で、常にCIが成功する状態を保つ。
* **機能ブランチ:** `feat/<短いトピック>` またはAI作業では `ai/<トピック>`。例: `feat/attendance-validation`、`ai/fix-push-encryption`。
* **修正ブランチ:** `fix/<短いトピック>`（`main` のバグ修正）。
* **ドキュメント専用ブランチ:** `docs/<短いトピック>`。
* **長期ブランチを作らない。** 1ブランチは3日以内または300 LOC以内とし、超える場合は分割する。
* **MVPでは `develop` ブランチは持たない。** `main` 単独 + 短期ブランチにすることでRenderのGitHub自動デプロイを単純に保つ。

## 2. ブランチからPRまでの流れ

```
main（保護）
  └─ feat/xxx  ──PR──► main
```

1. `git fetch origin && git checkout -b feat/xxx origin/main`
2. ブランチ上で小さなコミットを積む（§4参照）
3. `git push -u origin feat/xxx` し `gh pr create`（またはGitHub UI）でPR作成
4. PRは**1名の人間または指名されたAIレビュー担当者**のレビューを経る
5. マージは **Squash and merge**（`main` を直線的に保つ）。スカッシュ後のコミットメッセージはPRタイトルとする。

## 3. コミット規約

* **1コミット1目的。** 接頭辞 `feat:`、`fix:`、`docs:`、`chore:`、`test:`、`refactor:`。例:
  * `feat: add attendance PUT validation`
  * `fix: reject attendance write when submission is SUBMITTED`
  * `docs: update 04_database DDL for break check`
* **秘密情報、社員データ、`.env` をコミットしない。** Phase 6で追加される pre-commitフックは `*.env`、`*.pem`、`*.key` および実名を含む `*_test.csv` を拒否する。
* **不要な `Co-authored-by` は付けない**（ペア作業を除く）。
* ** `main` へのforce-push禁止。** レビュー前のfeatureブランチでのforce-pushは許可する。

## 4. PR規約

* **テンプレート:** `.github/pull_request_template.md` を必ず埋める。PRごとに目的、関連するFR/ADR/Q、テスト内容、秘密情報/マイグレーションの有無を記述する。
* **サイズ:** 本番コード400行以内（生成されたマイグレーションを除く）。超える場合は分割する。
* **チェック:** `ci.yml` がグリーン（ビルド + テスト + 秘密情報スキャン）でなければレビューしない。
* **レビュー:** 1名以上の承認を必須とする。レビューでは `AGENTS.md:5`（推測しない、契約を無断変更しない）と `09_development-rules.md:3-6` を確認する。
* **CI回避のためのDraft化禁止。** Draftは早期フィードバック用のみ。

## 5. タグとリリース

* Phase 6〜9ではタグを切らない。タグ `v0.1.0-mvp` は Phase 10の本番デプロイでPOのリリース承認後にのみ作成する。

## 6. ローカルでのGit運用

* `core.autocrlf=input`（macOS/Linux）、`core.autocrlf=true`（Windows）（または `.editorconfig` の `lf` に依存）。CRLFのみの変更をコミットしない。
* `git config pull.rebase false`（merge pull）でよい。本レビュー開始後に `main` をfeatureブランチへrebaseしない。
* `git fetch --prune` を習慣化し、マージ済みブランチは `git branch -d` で削除する。

## 7. 禁止事項

* `main` への直接push、`main` へのforce-push、レビューなしでの自己マージ。
* `main` にマージ済みの `V{NN}__` マイグレーションのコミット（不変）。
* `frontend/node_modules/`、`build/`、`out/`、IDEファイルのコミット（`.gitignore` 対象）。

## 8. Phase 6前の検証

使い捨てのブランチ `docs/test-branch-strategy` を `main` から作成し、push、PR作成、`ci.yml`（Phase 6で追加）の実行を確認した後、マージせずにクローズする。これにより `main` を汚さずにブランチ保護を検証できる。
