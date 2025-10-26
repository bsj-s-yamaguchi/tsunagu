# Tsunagu Operational Plan

## 0. Document Purpose & References
- 本書は `docs/07_feature_catalog.md` セクション2（機能マップ）、`docs/08_requirements.md` セクション2（要件定義）、および `docs/11_nonfunctional_requirements.md` の品質基準を運用面に落とし込むための指針。
- 関連ドキュメント: `docs/01_project_overview.md`, `docs/02_business_plan.md`, `docs/03_go_to_market.md`, `docs/04_financial_plan.md`, `docs/06_appendices.md`, `docs/09_competitor_analysis.md`, `docs/10_partner_prioritization.md`, `docs/12_architecture_overview.md`, `docs/13_risk_register.md`, `docs/14_detailed_design.md`, `docs/15_db_design.md`.
- 更新ルール: セクションごとに「Owner / Update Frequency / 次アクション」を明記し、変更点は `docs/00_cover.md` のステータス表にも反映する。

## 1. Organization & Roles
### 1.1 Core Team Structure (2025-10 Draft)
| Role | Headcount (2025Q4) | Must-have Skills | Owner | Next Review |
| --- | --- | --- | --- | --- |
| Product Management | 1 Lead / 1 PM | SaaS要件定義、金融SaaS、KYC知識 | Head of Product | 2025-11-05 |
| Compliance & Legal | 0.5 FTE | 労働法、職業安定法、特商法 | Legal & Security Lead | 2025-11-01 |
| Customer Success & Ops | 1 CS / 1 Ops | Onboarding、Stripe Connect運用 | CX Manager | 2025-10-28 |

> 採用状況・アサイン変更は `docs/10_partner_prioritization.md` のパートナー候補リストと連動してアップデート。

### 1.2 Hiring Plan & Ramp-up
- 2025-11までにCS 1名を採用。面接評価テンプレートは `docs/07_feature_catalog.md` の「評価テンプレート／カルマスコア」PoCを兼ねて実施。
- エージェント契約要件（報酬、KPI）は `docs/02_business_plan.md` のGTMセクションに準拠。

### 1.3 Partner & Vendor Management
- KYCベンダー（Prove / Liquid）評価: 契約締結までにセキュリティレビューシートとSOC2資料を取得。
- Stripe Connectチームとの月次チェックインを設定し、`docs/07_feature_catalog.md` のパートナー収益オーケストレーション構想をPoCに落とし込む。
- データ匿名化BPOパートナー候補は `docs/10_partner_prioritization.md` の優先度A企業から選定。

## 2. 製品開発体制

### 2.1 チーム構成と責任
| チーム | 責任範囲 | 主要KPI |
| --- | --- | --- |
| Core Platform | ユーザー認証、サブスクリプション、サポート、セキュリティ基盤 | システム可用性99.9%、セキュリティインシデント0件/月 |
| Recruitment Services | 求人管理、候補者スクリーニング、選考管理 | AI精度80%以上、重複検知Recall 0.92以上 |
| Communication Services | マルチチャネル通知、面接スケジューリング | 通知配信成功率99%以上 |
| Payment Services | 成果報酬決済、KYC/AML | 決済成功率95%以上、AML違反0件 |
| AI Services | AI Performance Review、Culture Matching、Learning Plans | 予測精度75%以上、ユーザー満足度80%以上 |
| Data & Analytics | データ分析、レポート、監査 | レポート生成時間<5秒、データ更新遅延<1時間 |

### 2.2 開発プロセス
- **スプリント計画**: 2週間サイクルのアジャイル開発
- **コードレビュー**: PR必須、カバレッジ80%以上
- **セキュリティレビュー**: すべての機能変更にセキュリティチェック
- **コンプライアンスレビュー**: 法令遵守チェックと記録保持

## 3. サポート体制

### 3.1 サポートチャネルとSLA
| プラン | チャネル | 応答時間 | 解決時間 | 対応言語 |
| --- | --- | --- | --- | --- |
| Free | メール | 24時間 | 72時間 | 日本語 |
| Pro | メール＋チャット | 8時間 | 24時間 | 日本語 |
| Growth | メール＋チャット＋Slack | 4時間 | 12時間 | 日本語＋英語 |
| Enterprise | 専任CSM＋24/7チャネル | 2時間 | 4時間 | 日本語＋英語 |

### 3.2 サポートプロセス
- **チケット分類**: 問題、質問、機能要望、不具合の4カテゴリ
- **エスカレーション**: L1（ボット）→L2（一般）→L3（専門）の3層構造
- **ナレッジベース更新**: 解決済みチケットからFAQ自動生成
- **顧客満足度調査**: チケット解決後72時間以内にCSAT調査

### 3.3 セルフサービス機能
- **コミュニティフォーラム**: ユーザー同士の情報交換
- **ハウツー動画**: 主要機能の操作方法ガイド
- **テンプレートライブラリ**: 求人テンプレート、通知テンプレート
- **APIドキュメント**: Swagger UIによるインタラクティブドキュメント

## 4. セキュリティ・コンプライアンス運用

### 4.1 セキュリティ監視
- **脆弱性スキャン**: 週次自動スキャン＋月次手動検証
- **侵入検知**: SIEMツールによるリアルタイム監視
- **セキュリティインシデント対応**: 15分以内の検知、2時間以内の初期対応

### 4.2 コンプライアンス管理
- **個人情報保護**: 年次監査＋四半期レビュー
- **GDPR対応**: データ主体の権利実装と記録保持
- **電子帳簿保存法対応**: 請求書・領収書の適格電子帳簿保存
- **労働関連法対応**: 求人広告の法令準拠チェック

### 4.3 監査ログ運用
- **ログ保持期間**: 30日間の詳細ログ、3年間の監査ログ
- **ログ分析**: 異常アクセスパターンの自動検知
- **コンプライアンスレポート**: 月次自動生成と経営陣レポート

## 5. 品質保証とテスト

### 5.1 QAマトリクス（サービス基盤機能）

| 機能領域 | テスト観点 | P0（必須） | P1（重要） | P2（望ましい） |
| --- | --- | --- | --- | --- |
| ユーザー認証 | SSOログイン、2FA、パスキー | ログイン成功率99.5%、2FA設定率80% | パスワードリセット成功率99% | パスキー登録成功率95% |
| サブスクリプション | プラン変更、請求処理 | 請求エラー率<0.1%、プラン変更成功率99% | 自動更新成功率99.5% | クーポン適用成功率98% |
| サポート機能 | チケット作成、ナレッジ検索 | チケット作成成功率99.9% | ナレッジ検索精度85% | AIボット解決率60% |
| セキュリティ | 認証試行制限、セッション管理 | 不正アクセス0件/月 | セッションタイムアウト100% | 脆弱性修正時間<72時間 |
| モバイル対応 | PWAインストール、オフライン機能 | インストール成功率95% | オフライン同期成功率90% | プッシュ通知配信率95% |

## 6. ロードマップとアクションアイテム

### 6.1 直近アクションアイテム（2025年10月〜12月）
| 期限 | 責任者 | アクション | 参照 |
| --- | --- | --- | --- |
| 2025-11-05 | Founder | サービス基盤API（認証・請求・サポート）の初期実装 | `docs/14_detailed_design.md §4` |
| 2025-11-10 | Founder | セキュリティ監視ダッシュボード設定 | `docs/05_operational_plan.md §4.1` |
| 2025-11-15 | CX Ops Lead | サポートナレッジベース初期コンテンツ作成 | `docs/05_operational_plan.md §3.3` |
| 2025-11-20 | Founder | コンプライアンス監査ログスキーマ設計 | `docs/05_operational_plan.md §4.3` |
| 2025-11-25 | Founder | モバイルPWA初期実装とオフライン機能検証 | `docs/14_detailed_design.md §6` |
| 2025-12-05 | Founder | サービス基盤機能のQAテスト計画策定・実施 | `docs/05_operational_plan.md §5.1` |
| 2025-12-15 | Product | サブスクリプションプランの価格設定レビュー | `docs/02_business_plan.md §3` |

### 6.2 MVPリリース準備（2026年1月〜3月）
| 期限 | 責任者 | アクション | 参照 |
| --- | --- | --- | --- |
| 2026-01-15 | Founder | セキュリティペネトレーションテスト実施・対策実施 | `docs/05_operational_plan.md §4.1` |
| 2026-02-01 | Compliance Lead | 個人情報保護法対応チェックリスト完成 | `docs/05_operational_plan.md §4.2` |
| 2026-02-15 | CX Manager | ベータ顧客向けオンボーディングマニュアル完成 | `docs/05_operational_plan.md §3` |
| 2026-03-01 | Founder | 全機能QAテスト完了とバグ修正 | `docs/05_operational_plan.md §5` |
| 2026-03-15 | Founder | パフォーマンステストとスケーリング検証 | `docs/05_operational_plan.md §7` |
| 2026-03-31 | Product | MVPリリース（ベータ10社） | `docs/01_project_overview.md §2` |

## 7. Change Log
| Date | Summary | Sections |
| --- | --- | --- |
| 2025-10-20 | マルチチャネル通知・候補者重複検知・PWAサポートに合わせてQA/サポート/データ移行計画を更新 | §2.2, §2.3, §3, §4, §5, §6 |
| 2025-10-21 | 新AIサービス（パフォーマンスレビュー、カルチャーマッチング、学習プラン、面接官トレーニング）の運用計画を追加 | §2.2, §2.3, §4.1, §4.3, §4.4, §5.1, §5.3, §5.4, §6 |
| 2025-10-26 | 単独開発体制を反映してチーム構造更新・アクションアイテムをFounder担当に変更 | §1.1, §1.2, §6.1, §6.2 |
