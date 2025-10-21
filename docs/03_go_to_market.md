# Tsunagu Go-to-Market計画（ドラフト｜2025-10-20）

## 1. ターゲットセグメントとペルソナ
| セグメント | 主要課題・トリガー | キー連絡先 | 成功指標 | 参考ドキュメント |
| --- | --- | --- | --- | --- |
| **SMB採用兼務者（従業員1〜50名）** | 媒体投稿の属人化、候補者連絡の遅延、成果報酬計算がExcel頼り | 代表／バックオフィス／総務 | Onboarding完了率、求人公開数、Slack/LINE通知利用率 | `docs/01_project_overview.md`、`docs/07_feature_catalog.md` |
| **スタートアップCTO/HR（50〜200名）** | 面接スピード低下、評価バイアス、データ統合不足 | CTO、人事責任者 | AI提案採用率、重複検知精度、Explainability活用度、定着率改善 | `docs/02_business_plan.md`, `docs/14_detailed_design.md` |
| **紹介エージェント／SES** | 成果報酬未回収、共有スプレッドシートの混乱、透明性不足 | 代表／アカウントMgr | GMV、報酬回収リードタイム、紹介者信頼スコア、Culture Fit精度 | `docs/05_operational_plan.md`, `docs/15_db_design.md` |

**バイイングステージ**
1. *Awareness*: 課題認識（媒体費高騰、決済手続負荷、AI活用ニーズ）。 → ブログ・ウェビナー・SNSで課題記事を提供。
2. *Consideration*: プロダクトツアーでAIスコア・成果報酬デモを確認。 → ベンチマーク提示、ROIシミュレータ提供。
3. *Decision*: 14日トライアル＋CS伴走。 → Stripe/KYC設定、LINEテンプレをセットアップ。
4. *Retention/Expansion*: 重複検知・Explainability・AI Performance Reviewへアップセル。 → 定例レビュー、共催イベント。

## 2. ポジショニングとメッセージング
- **ブランドスローガン**: *「信頼できるAIで成果報酬採用を最短に。」*
- **キーメッセージ**
  1. **Trust-first AI** – Explainability & 監査証跡でAI採用の透明性を守る。
  2. **Outcome-based Monetization** – Stripeエスクロー＋KYCで成果報酬決済を自動化。
  3. **Integrated Japan Stack** – LINE/Chatwork/PWA対応で現場のスピードを維持。
  4. **Future-proof Hiring** – AI Performance ReviewとCulture Matchingで長期定着を最大化。
- **競合比較（抜粋）**
  | 競合 | 強み | 弱み | Tsunaguの優位性 |
  | --- | --- | --- | --- |
  | ジョブカン採用管理 | 国内導入実績、LINE連携 | 成果報酬決済なし、AI説明責任弱 | KYC決済＋Explainability＋重複検知を標準化 |
  | 採用一括かんりくん | Chatwork/LINE連携、エージェント機能 | 決済/KYCが手動、AI機能限定 | Stripe自動化、AIスコア、PWAで差別化 |
  | Ashby/Greenhouse | 豊富なAI/ワークフロー | 日本の法規制・メッセージング対応が薄い | 法令準拠テンプレ、LINE/Slack/SMS対応、Culture Matching |
  | **Tsunaguの新優位性** | - | - | AI Performance Review、Culture Matching、Interviewer Trainingで長期定着とチーム適性を最適化 |

**Elevator Pitch**
> Tsunaguは、AIで候補者の適合性を説明付きで評価し、LINE/Chatworkから成果報酬決済まで一気通貫で自動化する、SMB特化型採用SaaSです。AI Performance ReviewとCulture Matchingで長期定着率も向上させます。

## 3. 獲得チャネル
- **インバウンド**
  - **コンテンツマーケティング**: 毎週月曜日に「AI採用ガイド」を公開し、採用担当者の課題解決に寄与。コンテンツカレンダーで半年分を事前準備。SEOキーワード「成果報酬 採用 自動化」「LINE 採用 通知」「AI Culture Matching」をターゲット。
  - **ウェビナー**: 月2回開催（第2火曜・第4木曜）、テーマをローテーション（AIスコアの透明性、成果報酬の自動化事例、Culture Matchingの実践）。登録者には事前資料を配布し、アテンド者には個別デモを提供。
  - **SNS戦略**: LinkedInでスタートアップCTO向けにテクニカルコンテンツを配信、NoteでSMB採用担当者向けに実践事例を共有。ハッシュタグ戦略で「#採用DX」「#AI採用」などを活用。
- **パートナー & リファラル**
  - **会計事務所／労務士**: 成果報酬の決済効率化を提案し、導入企業から10%のリファラルフィーを支払う。導入事例を共同で作成し、セミナーで紹介。パートナー専用LPと導入支援ツールキットを提供。
  - **HR SaaS連携**: freee、MF給与、SmartHRとのデータ連携PoCを実施し、共催セミナーで成功事例を紹介。API連携による業務効率化をデータで示し、導入企業に特典を提供。
  - **決済・チャネルパートナー**: Stripe、KYCベンダー、LINE公式とのケーススタディを共同で発表し、信頼性を高める。API連携記事をメディアで配信し、導入企業の声を紹介。
- **アウトバウンド / 有料施策**
  - **LinkedIn広告**: スタートアップCTO/HR向けに「信頼できるAI採用」を訴求し、LPに誘導。A/Bテストで広告クリエイティブを最適化し、コンバージョン率を向上。
  - **Wantedlyスカウト広告**: 採用担当者向けに「採用業務の自動化」を訴求し、資料DLに誘導。求人票にTsunaguの特徴を記載し、導入企業の声を紹介。
  - **産業イベント**: HR EXPO、小規模ビジネスフェアへ年4回出展し、デモ環境を用意。Stripe連携体験を提供し、導入企業の声を紹介。参加者には特典を提供し、フォローアップを実施。

## 4. セールスモーションとオンボーディング
- **フリートライアル〜有料化フロー**
  1. LP→資料DL/デモ予約→プロダクトツアー（Self-serve Demo）。
  2. 14日トライアル開始（AIスコア・重複検知のサンプルデータ付き）。
  3. Day3：Stripe/KYC接続、媒体連携、LINE通知テンプレ設定。
  4. Day7：タレントプール／重複アラートトレーニング、AI Performance Review/Culture Matchingデモ、Successプラン合意。
  5. Day14：ROIレビュー（応募数・重複検知成果・Time-to-hire短縮・定着率予測）→ Pro/Growth契約。
- **セールス組織とKPI**
  | 役割 | 主要KPI | 備考 |
  | --- | --- | --- |
  | Demand Gen (1) | MQL 120/月、CTR2% | コンテンツ/広告運用 |
  | Inside Sales (2) | SQL 45/月、トライアル着地率40% | ABM、デモ実施 |
  | Sales Engineer (1) | トライアル→有料化50%、Stripe設定リードタイム7日 | テクニカル支援 |
  | Customer Success (2) | Onboarding完了率85%、NPS +30 | Day0→Day14伴走 |
- **サクセスKPI & ヘルス**
  - Trial→Paid Conversion 45%以上。
  - 成果報酬設定完了率70%以上（Day14）。
  - LINE/Chatwork通知利用率 60%以上（初期ミッション）。
  - 重複検知アラート解決MTTR < 24h。
  - AI Performance Review/Culture Matching利用率 40%以上（Growthプラン導入後）。
- **オペレーション連携**
  - セールス→CS引き継ぎテンプレをNotionベースで共有（顧客課題、対象チャネル、導入目標）。
  - `docs/05_operational_plan.md` のオンボーディングSOP（Day0/3/7/14）を標準実施。

## 5. マーケティングカレンダーと予算（2026上期）
| 月 | 主要施策 | 詳細 | 予算 (¥K) | KPI |
| --- | --- | --- | --- | --- |
| Jan | AI採用ウェビナー #1 | Stripe / KYC共催、参加100名 | 600 | 登録120 / SQL15 |
| Feb | HR EXPO出展 | デモブース・顧客事例配布 | 1,500 | リード250 / SQL30 |
| Mar | "成果報酬自動化"キャンペーン | ホワイトペーパー＋メール5通 | 400 | DL300 / Trial40 |
| Apr | LINEテンプレ配布 | LP＋広告（LINE Ads Platform） | 700 | DL200 / Trial25 |
| May | パートナー共催セミナー | freee会計 × Tsunagu | 300 | 参加80 / SQL10 |
| Jun | Customer Day | 既存顧客ワークショップ＋ケース撮影（AI Performance Review/Culture Matching） | 500 | CSAT+5pt / Upsell5 |
- **総予算（上期）**: ¥4.0M（マーケ費の約38%）。残りは下期Growth施策（Marketplace/EOR連携）へ。
- **トラッキング方法**
  - HubSpot（MQL→SQL→Trial→Paid）× Tableauダッシュボード。
  - AdPlatform / SNS / Webinarツールを週次でデータ連携。
  - スプリントごとの事後レビューでCAC・トライアル率を見直し。

## 6. イネーブルメントとアセット
- **セールスデック / ワンペイジャー**: "Trust-first AI Hiring"メッセージ、AIスコア可視化、Stripeエスクロー図解、Culture Matchingデモを含む（11月初稿）。各セグメント向けの価値提案を明確化し、競合との差別化ポイントを強調。
- **ROIシミュレータ**: 売上貢献、成果報酬削減、通知チャネル別レスポンス改善、定着率向上効果を試算（In-app＆Excel版）。顧客の実データを入力し、導入前後の比較を可視化。
- **デモ環境**: ベータ顧客の匿名化データ＋ストーリーボード（AIスコア→重複検知→決済→Performance Review）を再現。PWAオフラインデモも用意し、現場での利用シーンを体験。
- **ケーススタディ計画**: 2026Q1にベータ10社から2社をケースとして取材（採用スピード＋成果報酬＋定着率の変化）。成功事例を動画や記事でまとめ、導入検討企業に提供。
- **Webサイト／LP要件**
  - Heroエリア：Explainable AI＋成果報酬自動化＋Culture Matchingを強調、デモリクエストCTA。
  - セグメント別LP：SMB向け／スタートアップ向け／エージェント向けの3種を用意。
  - リソースセンター：ホワイトペーパー、テンプレ、ウェビナーアーカイブをTag管理。
- **教育・内部イネーブルメント**
  - Playbook（Discovery質問、デモスクリプト、Pricingガイド）をSalesforce/Notionで配布。
  - 月次イネーブルメントトレーニング：AI説明責任、成果報酬FAQ、法務アップデート、Culture Matching活用をCS/Salesと共有。

> このGTMプランは `docs/02_business_plan.md`, `docs/05_operational_plan.md`, `docs/07_feature_catalog.md` と整合しています。施策・予算・KPIの変更は `docs/00_cover.md` のリビジョンログにも反映してください。