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
| Backend Engineering | 3 BE | Stripe/Payments、KYC API、Ruby/Node | Engineering Manager | 2025-10-30 |
| Frontend Engineering | 2 FE | React/TypeScript、デザインシステム | Engineering Manager | 2025-10-30 |
| Data & AI | 1 DS / 1 MLE | LLM活用、Explainability、MLOps | Lead Data Scientist | 2025-11-12 |
| Compliance & Legal | 0.5 FTE | 労働法、職業安定法、特商法 | Legal & Security Lead | 2025-11-01 |
| Customer Success & Ops | 1 CS / 1 Ops | Onboarding、Stripe Connect運用 | CX Manager | 2025-10-28 |

> 採用状況・アサイン変更は `docs/10_partner_prioritization.md` のパートナー候補リストと連動してアップデート。

### 1.2 Hiring Plan & Ramp-up
- 2025-11までにBE 1名、CS 1名を採用。面接評価テンプレートは `docs/07_feature_catalog.md` の「評価テンプレート／カルマスコア」PoCを兼ねて実施。
- エージェント契約要件（報酬、KPI）は `docs/02_business_plan.md` のGTMセクションに準拠。
- 技術オンボーディング: 初週で `docs/08_requirements.md` セクション2のAPI連携要件レビューと、KYC/AML PoC環境セットアップ（`docs/07_feature_catalog.md` の成果報酬エスクロー機能）を完了させる。

### 1.3 Partner & Vendor Management
- KYCベンダー（Prove / Liquid）評価: 契約締結までにセキュリティレビューシートとSOC2資料を取得。
- Stripe Connectチームとの月次チェックインを設定し、`docs/07_feature_catalog.md` のパートナー収益オーケストレーション構想をPoCに落とし込む。
- データ匿名化BPOパートナー候補は `docs/10_partner_prioritization.md` の優先度A企業から選定。

## 2. Development Process
### 2.1 Product Development Lifecycle
1. Discover: 問題仮説→ユーザー調査（アウトリーチは `docs/07_feature_catalog.md` の「マルチチャネルAIアウトリーチ」構想で支援）
2. Define: 要件定義 → `docs/08_requirements.md` へ反映 → `docs/01_project_overview.md` 更新
3. Deliver: Sprint実装 → QAチェックリスト → リリース承認
4. Measure: KPI確認→学習事項を`docs/06_appendices.md`の調査ノートへ追記

### 2.2 Release Governance & Quality
- リリース頻度: MVP期間は隔週、本番反映前にSandbox（Stripe / KYC）でE2Eテスト。
- 変更管理: PRテンプレにAI/決済の影響分析を義務化。承認者2名（EM + Legal）でGo/No-Go判断。
- QAマトリクス: 機能別に`docs/07_feature_catalog.md` モジュールと紐付け、テスト観点を洗い出し。
| モジュール | 主要テスト観点 | バグトリアージ閾値 |
| --- | --- | --- |
| 成果報酬自動決済 | KYC未完了時の支払ブロック、エスクロー解放条件 | Sev1: 支払い誤作動、Sev2: 通知遅延 |
| AI Explainabilityビュー | 根拠表示の整合性、バイアスアラート発火条件 | Sev1: 根拠欠落、Sev2: グラフズレ |
| マルチチャネルAIアウトリーチ | チャネル別送信制御、返信率予測のログ、LINE/Chatwork/Twilio連携 | Sev1: 誤送信、Sev2: レコメンド遅延 |
| 候補者重複検知＋マージ | 別媒体・エージェントの重複検知、マージ承認フロー | Sev1: 誤マージによるデータ損失、Sev2: アラート未発火 |
| エージェント発注ポータル | 発注→応募→成果報酬の進捗追跡、レポーティング | Sev1: SLA逸脱通知欠落、Sev2: レポート値不整合 |
| PWAモバイル体験 | オフライン状態での評価入力・同期、Push通知 | Sev1: オフラインデータ消失、Sev2: 同期遅延 |

- リリースチェックリスト（抜粋）
| 項目 | Owner | タイミング | Exit条件 |
| --- | --- | --- | --- |
| Regression & Accessibility | QA Lead | Release-3d | Cypress/Playwright合格、AXE自動検証0件 |
| 多チャネル通知ドライラン | CX Ops | Release-2d | LINE/SMS/Chatwork/メールのテスト送信成功、Webhooks遅延<1分 |
| データ移行バリデーション | Data Eng | Release-2d | ドライランCSV 1000件以内でエラー率<0.5%、重複検知誤判定0件 |
| モデル監査ログ確認 | Lead DS | Release-1d | 最新モデルバージョンが監査ダッシュボードに反映、Explainabilityサンプルレビュー |
| SRE Go/No-Go | Engineering Manager | Release-0d | CloudWatch/AWS Config異常0件、バックアップ完了通知取得 |

- サービスごとのリリース後モニタリング期間: 48時間はPagerDuty待機を強化し、オーナーがKPI・ログをレビュー（Slack #ops-release にテンプレート報告）。Deviationが発生した場合は即座に`docs/13_risk_register.md`へ記録。

### 2.3 Data & AI Operations (MLOps)
- モデル管理: Feature Storeとモデルレジストリを構築し、AIスコアリング/バイアス監視/アウトリーチ予測/重複検知を共通管理。候補者重複モデルは月次で再学習し、Precision/Recall≥0.92を維持。
- モニタリング: 精度ドリフト>5%で自動再学習タスクを起動し、`docs/04_financial_plan.md` のROIシナリオに影響を反映。チャネル送達率の低下（-10%）は自動アラート。
- データ品質: `docs/15_db_design.md` の `communication_message`/`candidate_aliases` テーブルを対象に、毎日0:00に重複・NULL・暗号化漏れチェックを実行。結果はDataHub（予定）またはSlackに投稿。
- 監査ログ: `docs/07_feature_catalog.md` のリーガルアップデート自動運用と連携し、モデル変更履歴と重複判定の監査証跡を保存。Explainabilityサンプルは四半期ごとに法務レビュー。
- アーキテクチャ連携: マイクロサービス構成・データフローの更新は `docs/12_architecture_overview.md` に反映し、リリース前レビューで差分を確認。データ移行ツールのスキーマ変更はCSチームへ48時間前に通知。
- データアクセス要求: 社内利用者がタレントプール生データをダウンロードする場合、Jiraでチケット化し、Legal/Security Leadが承認。処理はDatabricks経由で匿名化しダウンロードURLを提供。

## 3. Compliance & Security
### 3.1 Data Governance
- データ区分: PII/Financial/KYC/操作ログ/コミュニケーションログ/学習データを分類し、アクセスはRBAC (`docs/07_feature_catalog.md` 管理・プラットフォームセクション) に従う。
- 暗号化: 保存時AES256、通信時TLS1.2以上。キー管理はAWS KMS、日本リージョン。`communication_messages.body` は`pgp_sym_encrypt`、`candidate_aliases.alias_value` はハッシュ＋ソルト化。
- データ保持: `docs/15_db_design.md` のライフサイクル表に従い、候補者データ（24ヶ月）、重複検知ログ（12ヶ月）、コミュニケーションログ（INBOUND 12ヶ月 / OUTBOUND 24ヶ月）を管理。自動削除ジョブは毎週日曜2:00に実行。
- 非機能要件リンク: 性能・可用性・セキュリティの基準値は `docs/11_nonfunctional_requirements.md` に従い、Deviationは是正計画を策定。
- エクスポート統制: CSV/PDFエクスポートはすべて`audit_logs`に記録し、月次でLegalがサンプリングレビュー。大量エクスポート（>500件）は二段階承認。

### 3.2 KYC/AML Workflow
- フロー: 応募時→KYCチェック→成果報酬エスクロー投入→承認→支払実行。ワークフロー詳細は `docs/07_feature_catalog.md` の成果報酬エスクロー機能に準拠。
- 監査: 月次でコンプライアンスレビュー、年度末に第三者監査。`docs/06_appendices.md`に監査証跡を保管。
- チャネル制限: LINE/Chatworkで送信するKYC関連情報はマスキングし、重要通知はメール/Secureリンクに誘導。送信テンプレはLegalレビュー済みのみ使用可。

### 3.3 Security & Incident Response
- インシデント定義: PII漏洩、支払遅延、AI差別判定。各ケースに対し24時間以内に初動対応報告。
- プレイブック: 3段階（Contain → Remediate → Learn）。再発防止アクションは`docs/05_operational_plan.md`内に追記。
- MFA/SSO運用: TOTP/FIDO2登録率90%以上を維持。SSO/SAMLメタデータ更新は年2回レビューし、Cloudflare Accessポリシーと同期。
- セキュリティ演習: 半期に1回、マルチチャネル不正投稿／重複マージ誤判定のシナリオでTable Top演習を実施。結果は`docs/13_risk_register.md`へ反映。

## 4. Customer Support & Success
### 4.1 Onboarding Flow
1. Kickoff (Day0): 要件ヒアリング、成果報酬条件設定。
2. Day3: 媒体連携とStripe Connect設定。`docs/08_requirements.md` 管理・設定セクションのRBACテンプレを利用。
3. Day7: KYC/AMLフロー導入、AIスコアの使い方トレーニング、候補者重複アラートとタレントプール運用レクチャー。
4. Day14: LINE/SMS/Chatworkテンプレートの公開、PWAモバイルチェックインのセットアップ。

### 4.2 Support Model & SLA
| サポートチャネル | SLA (応答/解決) | Owner | 備考 |
| --- | --- | --- | --- |
| 電話 | 5分 / 4時間 | CX Manager | Enterprise/Growthプラン |
| Slack Connect / Chatwork | 15分 / 4時間 | CX Ops Lead | Pro以上プラン対象。LINE通知とも連動。 |
| メール / チケット | 4時間 / 1営業日 | Support Ops | Freeプランは応答のみ24時間 |
| AIサポートBot | 即時 / 継続改善 | Knowledge Lead | ナレッジベースは`docs/06_appendices.md`で更新 |

### 4.3 Knowledge Management
- サクセスノート: 導入結果・ROI・課題を `docs/01_project_overview.md` セクション6へ反映。
- FAQ更新: 解決率80%未満の問い合わせは月次レビューでプロダクト改善 or ドキュメント更新。マルチチャネルRunbookと連動してテンプレ見直し。
- データ移行ケース: CSVインポートで検出された重複/欠損をトリアージし、改善フィードバックをプロダクトに共有。移行完了レポートは週次でCS/PMに共有し、顧客へサマリーを送付。

### 4.4 Data Migration & Enablement
- プレ移行チェック: 顧客提供のCSVをAIクレンジングツールで分析し、重複推定率・欠損率・NGワードをレポート。承認後に本番変換。
- 本番移行: `docs/14_detailed_design.md` の `/admin/import/csv` エンドポイントを使用し、Dry-run → 本番の二段階で実行。実行ログはS3に保存し、`docs/15_db_design.md` の保持ポリシーに従う。
- 移行後検証: 候補者件数・タレントタグ数・エージェント発注データをダッシュボードで突合し、ズレが±1%以内であることを確認。差異が出た場合は24時間以内に補正。
- 再教育: 大口顧客には移行完了後72時間以内にタレントプール・重複アラート運用のトレーニングを実施し、NPS/CSATを記録。

## 5. KPIs & Continuous Improvement
### 5.1 Operational KPIs
- Onboardingリードタイム (Day0→Day7完了率)
- 成果報酬支払自動化率 (`docs/02_business_plan.md`の財務計画指標と同期)
- AI提案採用率 / 面接バイアスアラート解決時間 (`docs/07_feature_catalog.md` の分析・最適化セクション)
- 候補者重複検知MTTR（アラート発生→マージ完了まで） / 誤マージ率
- チャネル送達率（Slack/LINE/SMS/Chatwork）と応答率、PWA同期成功率

### 5.2 Review Rhythm
- Weekly Ops Sync: KPI速報、インシデント、開発進捗。重複検知アラート・チャネル送達率を定点観測。
- Monthly Exec Review: `docs/04_financial_plan.md`のシナリオと整合。サポートSLA/NPS、モバイル利用率を報告。
- Quarterly Retro: 市場動向（`docs/09_competitor_analysis.md`）と照合し、優先度再評価。モデル精度レビューとリスク対策の有効性を検証。
- 半期毎: セキュリティ/コンプラ演習レビュー、Chatwork/LINE/Twilioリレーション進捗報告。

### 5.3 Risk Management & Incident Response
- リスクレジスターを `docs/13_risk_register.md` に常時更新し、Severity/Impact/Owner/Statusを明記。マルチチャネルAPI制限やデータ移行失敗を追加管理。
- 重大インシデント発生時は24時間以内にポストモーテムを作成し、本章にリンクを追加。オフライン同期不整合や重複マージミスの場合も同様。

### 5.4 Instrumentation & Reporting
- ダッシュボード: Looker Studioで運用KPI（5.1）をリアルタイム可視化。データソースはBigQuery `analytics.application_fact` / `communication_message`。
- ログ連携: Cloudflare Logpush（WAF/Bot）、Chatwork/LINE Webhookログ、Stripe ConnectイベントをOpenSearchに集約し、Opsアラートと連動。
- レポートテンプレ: 週次Opsレポート、月次Execレポート、四半期Retroレポートのテンプレートを `docs/06_appendices.md` に格納し、主要数値（KPI、重大インシデント、Runbook改善）を固定フォーマットで報告。
- データ品質監査: Airflowで毎朝8:00に通信ステータス・重複検知成功率を検証し、閾値超過時は#ops-data-healthに通知。

## 6. Workstream Checklist
| Task | Owner | Due | Notes |
| --- | --- | --- | --- |
| 役割・採用計画の具体化とオーナー割当（§1.1, §1.2） | Head of Product | 2025-10-31 | 採用進捗は週次共有 |
| リリースガバナンスの承認プロセス策定（§2.2） | Engineering Manager | 2025-10-27 | PRテンプレ＋Exitチェックリスト整備 |
| KYC/AMLオペレーションにおける監査ログ運用開始（§3.2） | Legal & Security Lead | 2025-11-05 | 監査ダッシュボード初版構築 |
| オンボーディングSOPをヘルプセンター化（§4.1〜4.3） | CX Manager | 2025-11-08 | 多言語テンプレと動画作成 |
| KPIダッシュボード要件を機能カタログと整合（§5.1） | Data Lead | 2025-10-30 | Lookerダッシュボード要件書を提出 |
| 候補者重複検知・タレントプールRunbook整備＆CSトレーニング（§2.3, §4.1） | Data Lead / CX Ops | 2025-11-12 | トレーニング記録＋CSAT取得 |
| LINE/SMS/Chatwork通知テンプレと禁止語リストを公開、PWA動作検証完了（§4.2, §4.3） | CX Ops Lead | 2025-11-15 | Runbook添付、E2Eテストログ保存 |
| 非機能要件の測定計画を `docs/11_nonfunctional_requirements.md` に沿って策定 | SRE Lead | 2025-11-10 | 合意済みSLO/SLAを記録 |
| アーキテクチャ変更レビューを `docs/12_architecture_overview.md` へ反映し、承認プロセスを整備 | Engineering Manager | 2025-11-05 | Change calendar更新 |
| リスクレビューの結果を `docs/13_risk_register.md` と本書に反映 | Program Manager | 月次 | Execレビュー後24時間以内に更新 |

> RACIチャート、詳細フローチャート、外部監査報告などは完成次第、該当セクションまたは `docs/06_appendices.md` にリンクを追加してください。

## 7. Change Log
| Date | Summary | Sections |
| --- | --- | --- |
| 2025-10-20 | マルチチャネル通知・候補者重複検知・PWAサポートに合わせてQA/サポート/データ移行計画を更新 | §2.2, §2.3, §3, §4, §5, §6 |
