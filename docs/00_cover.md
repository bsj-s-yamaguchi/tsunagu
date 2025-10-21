# Tsunagu Project Master Pack

**Service Name**: Tsunagu (SMB向け採用自動化SaaS)

**Document Set**: Project Overview / Business Plan / Go-to-Market / Financial Model / Operational Plan / Appendices / Feature Catalog / Requirements / Non-Functional / Architecture / Risk / Competitor / Partner Strategy

**Prepared For**: Internal Stakeholders & Investors

**Prepared By**: Product Strategy Team

**Version**: 0.2 (Working Draft)

**Date**: 2025-10-20  
**Last Updated**: 2025-10-20 (Multi-channel & DB design refresh)

---

## Document Inventory
| File | Purpose | Status |
| --- | --- | --- |
| 00_cover.md | 表紙とドキュメント一覧 | Draft |
| 01_project_overview.md | プロジェクト概要と価値提案 | Outline |
| 02_business_plan.md | 事業計画・市場分析・収益モデル | Working Draft |
| 03_go_to_market.md | Go-to-Market戦略と施策ロードマップ | Outline |
| 04_financial_plan.md | 財務計画・シナリオ分析 | Outline |
| 05_operational_plan.md | 組織・開発体制・オペレーション設計 | Working Draft |
| 06_appendices.md | 参考資料・調査データ | Outline |
| 07_feature_catalog.md | 機能マップ・差別化設計 | Working Draft |
| 08_requirements.md | 要件定義書（プロダクト／ビジネス要件） | Working Draft |
| 09_competitor_analysis.md | 競合分析・収益モデル整理 | Draft |
| 10_partner_prioritization.md | パートナー優先度・交渉計画 | Draft |
| 11_nonfunctional_requirements.md | 非機能要件（性能・可用性・セキュリティ） | Outline |
| 12_architecture_overview.md | システム／データ／インフラアーキテクチャ | Outline |
| 13_risk_register.md | リスク管理台帳・レビュー運用 | Draft |
| 14_detailed_design.md | 詳細設計（アプリ／外部連携／インフラ） | Working Draft |
| 15_db_design.md | データベース設計（MVP） | Working Draft |

> このドキュメントセットは継続更新を前提としたドラフトです。各ファイルのステータスを更新しながら内容を充実させてください。

---

## Latest Updates（2025-10-20）
- **マルチチャネル対応強化**: LINE／SMS／Chatwork／Slackを統合するCommunication ServiceとWebhook仕様を `docs/14_detailed_design.md` に追加。通知要件・監視指標を再定義。
- **候補者重複検知 & タレントプール**: `docs/07_feature_catalog.md` と `docs/08_requirements.md` に重複マージ、タグ管理、エージェント発注ワークフローを追記。DBスキーマを `docs/15_db_design.md` で拡張。
- **PWA／モバイル体験**: フロントエンドのPWA化、モバイルチェックイン要件、NFR測定指標を `docs/08_requirements.md` / `docs/14_detailed_design.md` に追加。
- **セキュリティ強化**: 二要素認証＋SSO、Cloudflare Zero Trust統合を `docs/07_feature_catalog.md` / `docs/08_requirements.md` / `docs/14_detailed_design.md` に反映。
- **データ移行 & サポート運用**: CSVインポート検証、サポート伴走メニュー、重複検知Runbookを `docs/08_requirements.md` / `docs/14_detailed_design.md` / `docs/15_db_design.md` に整理。
- **オペレーション刷新**: `docs/05_operational_plan.md` をマルチチャネルQA、PWAサポート、重複検知KPI、サポートSLAに対応するよう更新。
- **ビジネスプラン初稿**: 市場規模、プラン別価格、収益モデル、三ヵ年計画、リスク・マイルストーンを `docs/02_business_plan.md` に反映。

---

## Decision Highlights
- **コミュニケーション基盤**: MVPでLINE／SMS／Chatwork通知を提供し、Slackと同じテンプレ管理・監査ログを共通化。（Ref: `docs/14_detailed_design.md §2.2, 4.2, 5`）
- **候補者データモデル**: `candidate_aliases`, `candidate_tags`, `agent_orders` など新テーブルを導入し、重複検知とエージェント成果追跡を実装。（Ref: `docs/15_db_design.md §3-4`）
- **PWA＆オフライン対応**: Next.js App Router + Workbox で現場利用を想定したPWAを構築。非機能指標（オフライン同期成功率）を設定。（Ref: `docs/14_detailed_design.md §2.1, §8`）
- **二要素＋SSO**: Better Auth × Cloudflare Access の条件付きアクセスを採用し、FIDO2/TOTPを標準化。（Ref: `docs/07_feature_catalog.md`, `docs/08_requirements.md`, `docs/14_detailed_design.md §6.2`）

---

## Open Actions（抜粋）
| Due | Owner | Task | Reference |
| --- | --- | --- | --- |
| 2025-11-05 | BE Lead | ER図（dbdiagram.io）を作成し `docs/06_appendices.md` へリンク | `docs/15_db_design.md §11` |
| 2025-11-10 | EM | マルチチャネル・選考フローのシーケンス図を追加 | `docs/14_detailed_design.md §10` |
| 2025-11-12 | Ops | Slack/LINE/SMS通知テンプレ＋KYCチェックリスト整備 | `docs/14_detailed_design.md §10` |
| 2025-11-15 | CX Lead | LINE/Twilio/Chatwork Runbook・禁止語リストをまとめる | `docs/14_detailed_design.md §10` |
| 2025-11-18 | Data Lead | 候補者重複検知・タグ規約ガイドを作成しRunbookに追記 | `docs/14_detailed_design.md §10` |

---

## Next Milestones（MVP）
1. **SP3-4（〜2026-01-25）**: 候補者重複検知、タレントプール、LINE/Twilio PoC完了（Ref: `docs/14_detailed_design.md §9.1`）。
2. **SP5-6（〜2026-02-22）**: エージェント発注ポータル、Chatwork通知、決済UI統合を完了。
3. **SP7-8（〜2026-03-22）**: DR/性能テストとBetaローンチ準備。マルチチャネル運用とサクセスRunbook完成。

---

## Key Contacts
| Role | Name | Primary Areas |
| --- | --- | --- |
| Head of Product | TBD | プロダクト戦略、優先度調整 |
| Engineering Manager | TBD | 技術アーキテクチャ、スプリント推進 |
| Lead Data Scientist | TBD | AIスコアリング、重複検知、Explainability |
| CX Manager | TBD | オンボーディング、サポートRunbook |
