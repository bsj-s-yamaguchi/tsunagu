# Tsunagu 詳細設計書（ワーキングドラフト）

**バージョン**: 0.5

**日付**: 2025-10-26
**最終更新**: 2025-10-26 (AIサービス群詳細設計・サービス基盤機能統合)

## 0. ドキュメント概要
- **目的**: `docs/08_requirements.md` と `docs/07_feature_catalog.md` で定義した機能要件・差別化要件を、実装可能な技術設計と運用フローに落とし込み、MVP（2026-03-31まで）に向けた開発の共通認識を形成する。
- **スコープ**: フロントエンド／バックエンド／データ／AI／インフラ／外部連携の詳細設計。MVPに含まれないP1+機能は将来拡張の観点を添えて記載。ハードウェア調達・人員計画は `docs/05_operational_plan.md` を参照し本書では扱わない。
- **参照資料**: `docs/01_project_overview.md`, `docs/05_operational_plan.md`, `docs/07_feature_catalog.md`, `docs/08_requirements.md`, `docs/11_nonfunctional_requirements.md`, `docs/12_architecture_overview.md`, `docs/13_risk_register.md`.
- **更新ルール**: フェーズ移行（MVP→Growth）時点、および主要アーキテクチャ変更（API契約、新規外部SaaS採用、AIモデル刷新）が決定したタイミングでアップデートする。最新差分は `docs/00_cover.md` のリビジョンログへ記載。

## 1. システム全体像
### 1.1 コンテキストと境界
- 利用主体: 採用担当者・面接官・経理担当・紹介エージェント・候補者。
- 外部サービス: Stripe Connect（成果報酬決済）、KYCベンダー（eKYC/反社チェック）、求人媒体API（Indeed/Wantedly/LinkedIn等）、Slack/Email（通知）、OpenAI GPT-5 API（生成・要約）、BigQuery（分析DWH）。
- データ境界: PII/決済データはAWS 東京リージョンのPrimary環境に保存。BigQueryには匿名化した分析データのみを転送。OpenAIには最小限の必要情報（匿名化済みスキル・職務要約）を送信する。

```mermaid
graph TD
  Recruiter((Recruiter)) -->|Web/Slack| Frontend
  Agent((Agency)) -->|Referral Portal| Frontend
  Candidate((Candidate)) -->|Portal| Frontend
  Frontend[Next.js BFF] --> API[API Gateway (NestJS)]
  API --> Auth[Auth Service]
  API --> Job[Job Service]
  API --> CandidateSvc[Candidate Service]
  API --> Pipeline[Pipeline Service]
  API --> Payment[Payment Service]
  API --> Notify[Notification Service]
  API --> AIMatch[AI Match Service]
  AIMatch --> OpenAI[(GPT-5 API)]
  Payment --> Stripe[(Stripe Connect)]
  Payment --> Accounting[(B2B Payments / Bank)]
  CandidateSvc --> KYC[(eKYC Vendor)]
  Job --> Media[(Job Board APIs)]
  Pipeline --> Slack[(Slack API)]
  Report[Analytics Exporter] --> BigQuery[(BigQuery)]
  {S3/KMS}:::storage -.-> CandidateSvc
  classDef storage fill:#f3f7ff,stroke:#546ede,stroke-width:1px;
```

### 1.2 コアユーザーフロー（MVP）
1. **求人生成**: 採用担当が求人フォームに入力 → AIドラフト生成 → 承認後に媒体・自社サイトへ配信（自動 or 予約）。
2. **候補者スクリーニング**: 応募受付 → レジュメ解析 → AIスコアと根拠提示 → 面接設定・議事録 → 最終承認。
3. **成果報酬決済**: 採用承諾 → KYC再確認 → Stripe Connectエスクロー投入 → 期日到来で自動決済・紹介者分配。

### 1.3 サービス構成サマリー
| レイヤ | サービス/モジュール | 主責務 | データストア | 備考 |
| --- | --- | --- | --- | --- |
| フロントエンド | Recruiter Console, Candidate Portal, Agent Portal, Admin Console | Next.js + React/TypeScript、GraphQLクライアント（Apollo）、状態管理（TanStack Query＋Zustand）、デザインシステム | LocalStorage（セッショントークン）, Edge Cache | i18n（日英）、テーマ切替、アクセスビリティ準拠 |
| API Gateway | GraphQL（BFF） + REST passthrough | スキーマオーケストレーション、RBAC、Rate Limit、Schema Stitching | Redis（キャッシュ）, PostgreSQL(セッション) | NestJS + Apollo Federation 2 |
| ドメインサービス | Auth, Organization, Job, Candidate, Pipeline, Payment, Notification, Reporting, Admin | 業務ロジック、永続化、外部連携 | PostgreSQL (Aurora), Redis, S3互換, EventBus (Kafka) | 各サービスは独立デプロイ（コンテナ） |
| バッチ／ワーカー | Sync Worker, Invoice Worker, Model Monitor | 媒体同期、決済バッチ、AI再学習トリガ、レポートエクスポート | Redis（キュー）, BigQuery | Temporal (Self-hosted) でワークフロー管理 |
| AI基盤 | Feature Store, Prompt Orchestrator, Explainability Engine | 特徴量抽出、LLMプロンプト管理、根拠生成、モニタリング | BigQuery, Vertex AI Model Registry | OpenAI API呼び出し前後でGuardrail実装 |
| 分析 | ELT (Airflow), Dash Loader | 取引・選考ログのETL、KPI可視化 | BigQuery, Looker Studio | PIIはトークナイズ済み |

## 2. アーキテクチャ詳細
### 2.1 フロントエンド（Next.js / React）
- **アプリ構成**: App Routerを採用し、`/console`（採用担当向け）、`/candidate`、`/agent`、`/admin` のサブアプリに分割。各セクションは独立したLayout単位でコード分割し、Cloudflare Pages Functions／WorkersでエッジキャッシュとSSRを実行する。
- **状態管理**: サーバーデータは TanStack Query でキャッシュ／再検証し、応募進捗や決済操作で楽観的更新を実施。UIの短期的な状態（ドロワー、ウィザード進行、ドラフト）やクロスタブ共有は Zustand（必要箇所のみpersist）で扱う。フォームは Conform + Zod に統一し、サーババリデーションとも整合。Draft保存用にはIndexedDB（Dexie）を利用。ボット対策としてCloudflare Turnstileを応募フォームに組み込む。
- **マルチチャネルUI**: 候補者・紹介者とのメール／SMS／電話メモ／チャット／LINEの会話を単一タイムラインに表示し、テンプレート挿入・送信予約・既読管理を提供。LINE友だち登録導線やFAQボットの設定画面をConsole内に組み込み、競合（ジョブカン）の連携水準を上回る。 citeturn0open0
- **PWA対応**: Next.js App Router＋WorkboxでPWAを構成し、オフラインキャッシュ・プッシュ通知・ホーム画面追加を提供。候補者チェックインや面接官評価をモバイルからシームレスに利用できるUXを保証。 citeturn0view0
- **デザインシステム**: Figmaトークンを元にTailwind CSS＋Radix UIで実装。アクセシビリティはWCAG 2.1 AA（キーボード操作、ARIAラベル、コントラストチェック）。テーマはOrgごとのPrimaryカラー・ロゴ設定を `ThemeProvider` で配信。
- **通信層**: GraphQL（HTTPS）を基本とし、リアルタイム通知（面接調整、決済ステータス）は WebSocket（Cloudflare WebSockets）で購読。媒体投稿ジョブなどの進捗は Server Sent Events を採用し、Cloudflare Workers KVキャッシュでポーリング負荷を削減。
- **CI/CD**: Storybook + ChromaticでUIレビュー、PlaywrightによるE2EスモークをDev環境で自動実行。Cloudflare Pages Previewでデザイナーと摺り合わせ。mainマージ時にPages/Staging環境へ自動デプロイし、Wrangler経由で設定を反映。

### 2.2 バックエンド（NestJS マイクロサービス）
| サービス | 主機能 | 主要エンドポイント | データストア | 外部依存 |
| --- | --- | --- | --- | --- |
| Auth Service | OAuth2.1 / SAML, Magic Link, MFA, RBAC token発行 | `/auth/login`, `/auth/token`, `/auth/mfa/setup` | PostgreSQL(`users`, `org_memberships`), Redis(session) | Better Auth（Next.js SDK）＋Cloudflare Access（SAML IdP連携） |
| Organization Service | 組織設定、プラン、RBACマトリクス、監査設定 | `/org`, `/org/{id}/roles` | PostgreSQL(`organizations`, `roles`) | Stripe Billing API（契約プラン） |
| Job Service | 求人管理、AIドラフト、媒体投稿キューイング | GraphQL `job`, `createJob`, REST `/job-sync` | PostgreSQL(`job_postings`), S3(`job_assets/`) | OpenAI, Media API |
| Candidate Service | 応募受付、レジュメ解析、重複検知、タレントプール管理 | GraphQL `candidate`, `updateStatus`, `mergeDuplicate`, `talentPool`, REST `/webhook/kyc` | PostgreSQL(`candidates`, `applications`, `candidate_aliases`, `candidate_tags`), S3(`resume/`) | KYC API, OCR | citeturn0view0 |
| Pipeline Service | 選考ステップ、面接日程、議事録保管 | GraphQL `pipeline`, `scheduleInterview` | PostgreSQL(`pipeline_stages`, `interviews`), Redis(cache) | Google/Outlook Calendar API |
| Payment Service | 成果報酬計算、Stripe Connectエスクロー、請求書 | GraphQL `paymentIntent`, REST `/webhook/stripe` | PostgreSQL(`payment_orders`, `payouts`) | Stripe Connect, 国税庁API（法人番号検証） |
| Notification Service | Slack/Email送信、テンプレ管理 | `/notify/slack`, `/notify/email` | PostgreSQL(`notification_templates`), Redis(queue) | Slack API, SendGrid |
| Communication Service | 候補者・紹介者とのメール/SMS/LINE/Chatwork/電話メモを統合し、テンプレート・配信スケジュール・禁止リストを管理 | GraphQL `conversation`, `sendMessage`, REST `/webhook/line`, `/webhook/sms`, `/webhook/chatwork` | PostgreSQL(`communication_threads`, `communication_messages`), Redis(queue) | Twilio(音声/SMS), LINE Messaging API, Chatwork API, SendGrid | citeturn0open0turn0view0 |
| Agency Service | エージェント契約、求人配信、案件発注、成果レポートを管理 | GraphQL `agentOrders`, `createAgentBrief`, REST `/agent-order/import` | PostgreSQL(`agent_orders`, `agent_metrics`), Redis(queue) | Chatwork API, Slack API | citeturn0view0 |
| Reporting Service | KPI算出、BigQuery転送 | GraphQL `reportingSummary` | BigQuery, PostgreSQL(`report_jobs`) | Airflow |
| Admin Service | 監査ログ、設定フラグ操作 | GraphQL `auditLogs`, REST `/admin/toggles` | PostgreSQL(`audit_logs`) |  |
| Performance Review Service | 入社後実績と採用時AIスコアの照合、AIモデル精度検証 | GraphQL `performanceReview`, `modelAccuracy` | PostgreSQL(`performance_reviews`, `model_metrics`), BigQuery | AI基盤 |
| Culture Matching Service | 組織内チームカルチャーと候補者の適合度分析 | GraphQL `teamCultureMatch`, `cultureFitScore` | PostgreSQL(`culture_matches`, `team_cultures`), BigQuery | AI基盤 |
| Learning Path Service | 不足スキルの特定とカスタマイズ学習プラン生成 | GraphQL `skillGapAnalysis`, `learningPlan` | PostgreSQL(`skill_gaps`, `learning_paths`) | AI基盤 |
| Interviewer Training Service | 面接官の評価傾向・バイアス分析と個別トレーニングプログラム提供 | GraphQL `interviewerBias`, `trainingProgram` | PostgreSQL(`interviewer_metrics`, `training_programs`) | AI基盤 |

- **通信方針**: API GatewayとはGraphQL Federation。サービス間は gRPC（Protobuf）で同期呼び出し＋Kafkaトピックで非同期イベント（`candidate.created`, `payment.paid` 等）を配信。
- **コンフィグ管理**: 各サービスは12factor準拠。SecretsはAWS Secrets Manager。Feature FlagはLaunchDarkly連携。
- **スキーマバージョニング**: GraphQL Schema Registryにバージョン管理し、Breaking Changeがある場合は2 sprint前にDeprecation告知。
- **データアクセス**: 各サービスはDrizzle ORM（SQLベースのタイプセーフクエリ）を採用し、`drizzle-kit`でマイグレーションを生成・適用。共通スキーマは`packages/db`モノレポに集約し、zodスキーマと同期させる。

### 2.3 バッチ／ワーカー／スケジューラ
- Temporalを採用し、`QueueWorker`（BullMQ）からWorkflowを起動。主なWorkflow:
  - `JobSyncWorkflow`: 30分毎に媒体APIから応募データをPullし、差分をCandidate ServiceへUpsert。
  - `PaymentSettlementWorkflow`: 採用決定後＋期日条件でStripe Transferを起動、紹介者への分配と請求書PDF生成を並列実行。
  - `KycRetryWorkflow`: KYCベンダーのpendingステータスを60分間隔で再照会、3回失敗でOpsチームへSlack通知。
  - `ModelMonitoringWorkflow`: AIスコアと採用実績の乖離をBigQueryで分析し、閾値超過で再学習ジョブをAirflowに投げる。
  - `PerformanceReviewWorkflow`: 月次で入社者実績と採用時AIスコアを照合し、モデル精度レポートを生成。
  - `CultureMatchingWorkflow`: 組織カルチャーデータを定期更新し、候補者との適合度を再計算。
  - `LearningPathUpdateWorkflow`: スキルギャップ分析に基づき、学習プランを定期的に更新。
  - `InterviewerTrainingWorkflow`: 面接官評価データを分析し、個別トレーニングプログラムを更新。
- 監視: Temporal UI + Prometheusメトリクス。ジョブ失敗はPagerDutyでSLAに応じてアラート。ジョブ設定はTerraformでコード管理。

### 2.4 AI / データパイプライン
- **推論フロー**: Candidate Service → Feature Store（職務経歴/スキルタグ/エンゲージメント指標） → Prompt Orchestrator → OpenAI GPT-5（スコア算出/根拠テキスト） → Explainability EngineでSHAPベースの寄与度計算 → GraphQLレスポンス。
- **Feature Store**: Feast on GKEを採用。原データはPostgreSQLとS3からAirflowで取り込み、匿名化後BigQueryへ書き込み。Feature定義はYAMLでGit管理。
- **モデルバージョン管理**: Vertex AI Model Registryでバージョン管理し、`Model Version` と `Prompt Template` をセットでタグ付け。`docs/07_feature_catalog.md` のExplainability要件を満たすため、各スコアの根拠テーブルをPostgreSQL(`ai_explanations`)へ保存。
- **ガードレール**: OpenAI API呼び出し前にPIIフィルタリング、禁止ワード置換。出力はPydanticで厳格バリデーション。異常スコアはHuman-in-the-loop審査キューへ回送。
- **コスト最適化**: プロンプトテンプレートをキャッシュし、類似求人はEmbeddingで再利用。ピーク時はBatch推論（最大500件）に切替え、閾値を超えると社内Distilledモデルへのフォールバックを検討。
- **新機能AIモデル**:
  - **Performance Review Model**: 入社後実績（業績、評価、離職率等）と採用時AIスコアを照合し、モデル精度を継続的に検証・改善。
  - **Culture Matching Model**: 組織内各チームのカルチャー特性と候補者の価値観を分析し、最適なチーム配置を提案。
  - **Learning Path Model**: 候補者のスキルギャップを特定し、カスタマイズされた学習プランを自動生成。
  - **Interviewer Training Model**: 面接官の評価傾向とバイアスを分析し、個別に最適化されたトレーニングプログラムを提供。

## 3. 新AIサービス詳細設計（追加）

### 3.1 AIパフォーマンスレビューサービス
- **サービス名**: PerformanceReviewService
- **責任範囲**: 入社後実績と採用時AIスコアの照合、AIモデル精度検証

**主要機能**:
- 入社後実績の収集と管理
- 採用時AIスコアとの照合
- モデル精度の計算とレポート生成
- モデルの再学習と更新

**APIエンドポイント**:
```
type Query {
  performanceReview(id: ID!): PerformanceReview!
  modelAccuracy: ModelAccuracy!
}

type Mutation {
  generatePerformanceReview(applicationId: ID!): PerformanceReview!
}

```

**データスキーマ**:
```
-- パフォーマンスレビュー結果テーブル
CREATE TABLE performance_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  application_id UUID REFERENCES applications(id),
  candidate_id UUID REFERENCES candidates(id),
  ai_scores_at_hire JSONB,
  actual_performance JSONB,
  accuracy_delta JSONB,
  review_period INTERVAL,
  reviewed_at TIMESTAMP DEFAULT NOW()
);

-- モデル精度テーブル
CREATE TABLE model_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  model_version TEXT NOT NULL,
  accuracy_score NUMERIC(5,4),
  bias_indicators JSONB,
  last_reviewed_at TIMESTAMP DEFAULT NOW(),
  next_review_due_at TIMESTAMP
);

```


## 4. サービス基盤アーキテクチャ

### 4.1 認証・認可サービス
**サービス名**: AuthService
**責任範囲**: ユーザー認証、セッション管理、RBAC、SSO統合

**主要機能**:
- ユーザー登録・ログイン・ログアウト
- パスワードリセットとメール検証
- TOTPベースの二要素認証（2FA）
- FIDO2/WebAuthnパスキー認証
- SAML 2.0およびOIDCエンタープライズSSO
- ロールベースアクセス制御（RBAC）
- IP制限と条件付きアクセス制御

**APIエンドポイント**:
```
type Mutation {
  register(input: RegisterInput!): AuthResult!
  login(input: LoginInput!): AuthResult!
  logout: Boolean!
  refreshToken: AuthResult!
  enable2FA: Enable2FAResult!
  verify2FA(token: String!): Boolean!
  registerPasskey: PasskeyRegistrationResult!
  verifyPasskey(assertion: String!): AuthResult!
  initiateSSO(provider: SSOProvider!): SSORedirect!
}

type Query {
  me: User!
  sessions: [Session!]!
  permissions(resource: String!): [Permission!]!
}
```

**データスキーマ**:
```
-- ユーザーテーブル
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  is_email_verified BOOLEAN DEFAULT false,
  is_2fa_enabled BOOLEAN DEFAULT false,
  totp_secret VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- ロールテーブル
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- ユーザーロール関連テーブル
CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id),
  role_id UUID REFERENCES roles(id),
  PRIMARY KEY (user_id, role_id)
);

-- セッションテーブル
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  token_hash VARCHAR(255) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  ip_address INET,
  user_agent TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.2 サブスクリプション・請求サービス
**サービス名**: BillingService
**責任範囲**: プラン管理、サブスクリプション、請求、支払

**主要機能**:
- プラン定義と価格設定
- サブスクリプション作成・変更・キャンセル
- 請求明細生成と送付
- 支払方法管理
- 利用状況追跡と制限

**APIエンドポイント**:
```
type Mutation {
  subscribeToPlan(planId: ID!, paymentMethodId: ID): SubscriptionResult!
  changePlan(newPlanId: ID!): SubscriptionResult!
  cancelSubscription: Boolean!
  addPaymentMethod(input: PaymentMethodInput!): PaymentMethod!
  removePaymentMethod(id: ID!): Boolean!
  setDefaultPaymentMethod(id: ID!): Boolean!
}

type Query {
  plans: [Plan!]!
  currentSubscription: Subscription
  invoices(limit: Int = 10): [Invoice!]!
  paymentMethods: [PaymentMethod!]!
  usageMetrics: UsageMetrics!
}
```

### 4.3 サポート・ヘルプサービス
**サービス名**: SupportService
**責任範囲**: チケット管理、ナレッジベース、コミュニティ

**主要機能**:
- サポートチケット作成・追跡
- ナレッジベース記事管理
- コミュニティフォーラム
- リリースノート管理

**APIエンドポイント**:
```
type Mutation {
  createTicket(input: CreateTicketInput!): Ticket!
  addTicketComment(ticketId: ID!, content: String!): TicketComment!
  voteOnArticle(articleId: ID!, vote: VoteType!): ArticleVote!
  createForumPost(input: CreateForumPostInput!): ForumPost!
  markReleaseNoteAsRead(id: ID!): Boolean!
}

type Query {
  tickets(status: TicketStatus): [Ticket!]!
  knowledgeBaseArticles(category: String): [Article!]!
  forumPosts(limit: Int = 20): [ForumPost!]!
  unreadReleaseNotes: [ReleaseNote!]!
}
```

## 3. データ設計
### 3.1 エンティティと関係
| エンティティ | 主キー | 主なリレーション | 区分 | 保管先 |
| --- | --- | --- | --- | --- |
| organizations | org_id | users, job_postings, payment_orders | 機密 | PostgreSQL |
| users | user_id | org_memberships, audit_logs | 機微 | PostgreSQL |
| org_memberships | (org_id, user_id) | roles, invitations | 機密 | PostgreSQL |
| job_postings | job_id | job_channels, applications | 公開/機密混在 | PostgreSQL |
| job_channels | channel_id | job_postings | 公開 | PostgreSQL |
| candidates | candidate_id | applications, kyc_records | 特定PII | PostgreSQL + S3 |
| candidate_aliases | alias_id | candidates | 機微 | PostgreSQL |
| candidate_tags | tag_id | candidates | 機密 | PostgreSQL |
| applications | application_id | candidates, pipeline_stages, payments | 機密 | PostgreSQL |
| pipeline_stages | stage_id | applications, interviews | 機密 | PostgreSQL |
| interviews | interview_id | pipeline_stages, interview_notes | 機微 | PostgreSQL + S3 |
| interview_notes | note_id | interviews, ai_explanations | 機密 | PostgreSQL |
| payment_orders | payment_id | payouts, invoices | 極秘(金融) | PostgreSQL |
| payouts | payout_id | payment_orders, referrals | 金融 | PostgreSQL |
| referral_partners | partner_id | payouts, agents | 機密 | PostgreSQL |
| audit_logs | log_id | users | 監査 | PostgreSQL |
| ai_explanations | explanation_id | applications, interviews | 機微 | PostgreSQL |
| kyc_records | kyc_id | candidates, referral_partners | 機密 | PostgreSQL |
| media_sync_logs | sync_id | job_postings | 公開 | PostgreSQL |
| communication_threads | thread_id | applications, referral_partners, channel_subscriptions | 機密 | PostgreSQL |
| communication_messages | message_id | communication_threads | 機微 | PostgreSQL |
| channel_subscriptions | subscription_id | communication_threads, notification_templates | 機密 | PostgreSQL |
| line_contacts | line_user_id | candidates, communication_threads | 機密 | PostgreSQL |
| agent_orders | order_id | referral_partners, job_postings | 機密 | PostgreSQL |
| agent_metrics | metric_id | agent_orders | 機密 | PostgreSQL |
| performance_reviews | review_id | applications, candidates | 機密 | PostgreSQL |
| model_metrics | metric_id | performance_reviews | 機密 | PostgreSQL |
| team_cultures | culture_id | organizations | 機密 | PostgreSQL |
| culture_matches | match_id | candidates, team_cultures | 機密 | PostgreSQL |
| skill_gaps | gap_id | candidates | 機密 | PostgreSQL |
| learning_paths | path_id | skill_gaps | 機密 | PostgreSQL |
| interviewer_metrics | metric_id | users, interviews | 機密 | PostgreSQL |
| training_programs | program_id | interviewer_metrics | 機密 | PostgreSQL |

### 3.2 テーブル定義（抜粋）
```
CREATE TABLE job_postings (
  job_id UUID PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  title TEXT NOT NULL,
  employment_type TEXT NOT NULL CHECK (employment_type IN ('FULL_TIME','PART_TIME','CONTRACT','INTERN')),
  location JSONB NOT NULL, -- {country, prefecture, remote}
  salary_range JSONB, -- {min, max, currency, type}
  description_markdown TEXT NOT NULL,
  ai_score_profile JSONB, -- {skills:[], culture:[], suggestions:[]}
  status TEXT NOT NULL CHECK (status IN ('DRAFT','REVIEW','PUBLISHED','PAUSED','CLOSED')),
  publish_at TIMESTAMPTZ,
  expire_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(user_id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE candidates (
  candidate_id UUID PRIMARY KEY,
  full_name TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT,
  hashed_national_id TEXT,
  resume_url TEXT,
  persona_tags TEXT[],
  consent_status TEXT CHECK (consent_status IN ('ACCEPTED','REVOKED','EXPIRED')),
  pii_encryption_version INT DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE applications (
  application_id UUID PRIMARY KEY,
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  job_id UUID NOT NULL REFERENCES job_postings(job_id),
  source TEXT NOT NULL,
  current_stage TEXT NOT NULL,
  ai_scores JSONB, -- {fit, culture, salary_fit, attrition_risk}
  human_override JSONB,
  referral_partner_id UUID REFERENCES referral_partners(partner_id),
  escrow_status TEXT CHECK (escrow_status IN ('NONE','HELD','RELEASED','REFUNDED')),
  hired BOOLEAN DEFAULT false,
  hired_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE payment_orders (
  payment_id UUID PRIMARY KEY,
  application_id UUID NOT NULL REFERENCES applications(application_id),
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  amount_cents BIGINT NOT NULL,
  currency TEXT DEFAULT 'JPY',
  settlement_due_date DATE NOT NULL,
  stripe_transfer_id TEXT,
  status TEXT CHECK (status IN ('PENDING','IN_ESCROW','PAID','REFUNDED','FAILED')),
  kyc_status TEXT CHECK (kyc_status IN ('PENDING','APPROVED','REJECTED')),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

```
CREATE TABLE communication_threads (
  thread_id UUID PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  application_id UUID REFERENCES applications(application_id),
  referral_partner_id UUID REFERENCES referral_partners(partner_id),
  default_channel TEXT NOT NULL CHECK (default_channel IN ('EMAIL','SMS','LINE','PHONE','CHAT')),
  mute_until TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE communication_messages (
  message_id UUID PRIMARY KEY,
  thread_id UUID NOT NULL REFERENCES communication_threads(thread_id),
  direction TEXT NOT NULL CHECK (direction IN ('OUTBOUND','INBOUND')),
  channel TEXT NOT NULL CHECK (channel IN ('EMAIL','SMS','LINE','PHONE','CHAT')),
  subject TEXT,
  body TEXT,
  media_urls TEXT[],
  sent_at TIMESTAMPTZ,
  provider_message_id TEXT,
  status TEXT CHECK (status IN ('QUEUED','SENT','DELIVERED','FAILED','READ')),
  language TEXT DEFAULT 'ja-JP',
  meta JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE channel_subscriptions (
  subscription_id UUID PRIMARY KEY,
  thread_id UUID NOT NULL REFERENCES communication_threads(thread_id),
  channel TEXT NOT NULL,
  destination TEXT NOT NULL,
  opt_in BOOLEAN DEFAULT true,
  last_opt_in_at TIMESTAMPTZ,
  last_opt_out_at TIMESTAMPTZ,
  UNIQUE(thread_id, channel, destination)
);

CREATE TABLE line_contacts (
  line_user_id TEXT PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  candidate_id UUID REFERENCES candidates(candidate_id),
  display_name TEXT,
  picture_url TEXT,
  linked_thread_id UUID REFERENCES communication_threads(thread_id),
  consent BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE candidate_aliases (
  alias_id UUID PRIMARY KEY,
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  alias_type TEXT NOT NULL CHECK (alias_type IN ('EMAIL','PHONE','SOCIAL','AGENCY')),
  alias_value TEXT NOT NULL,
  verified BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(alias_type, alias_value)
);

CREATE TABLE candidate_tags (
  tag_id UUID PRIMARY KEY,
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  tag TEXT NOT NULL,
  source TEXT NOT NULL CHECK (source IN ('USER','AI','IMPORT')),
  created_by UUID REFERENCES users(user_id),
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE agent_orders (
  order_id UUID PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  referral_partner_id UUID NOT NULL REFERENCES referral_partners(partner_id),
  job_id UUID NOT NULL REFERENCES job_postings(job_id),
  brief_title TEXT NOT NULL,
  summary TEXT,
  fee_rate NUMERIC(5,2) NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('DRAFT','PUBLISHED','CLOSED')),
  sla_days INT,
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE agent_metrics (
  metric_id UUID PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES agent_orders(order_id),
  submitted_count INT DEFAULT 0,
  interview_count INT DEFAULT 0,
  hire_count INT DEFAULT 0,
  refund_count INT DEFAULT 0,
  avg_time_to_submit INTERVAL,
  last_submitted_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE performance_reviews (
  review_id UUID PRIMARY KEY,
  application_id UUID NOT NULL REFERENCES applications(application_id),
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  ai_scores_at_hire JSONB,
  actual_performance JSONB,
  accuracy_delta JSONB,
  review_period INTERVAL,
  reviewed_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE model_metrics (
  metric_id UUID PRIMARY KEY,
  model_version TEXT NOT NULL,
  accuracy_score NUMERIC(5,4),
  bias_indicators JSONB,
  last_reviewed_at TIMESTAMPTZ DEFAULT now(),
  next_review_due_at TIMESTAMPTZ
);

CREATE TABLE team_cultures (
  culture_id UUID PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES organizations(org_id),
  team_name TEXT NOT NULL,
  culture_profile JSONB, -- {values:[], work_style:[], communication:[], ...}
  diversity_index NUMERIC(5,4),
  turnover_rate NUMERIC(5,4),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE culture_matches (
  match_id UUID PRIMARY KEY,
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  team_culture_id UUID NOT NULL REFERENCES team_cultures(culture_id),
  match_score NUMERIC(5,4),
  compatibility_factors JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE skill_gaps (
  gap_id UUID PRIMARY KEY,
  candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
  missing_skills TEXT[],
  proficiency_levels JSONB,
  priority_score NUMERIC(5,4),
  identified_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE learning_paths (
  path_id UUID PRIMARY KEY,
  gap_id UUID NOT NULL REFERENCES skill_gaps(gap_id),
  recommended_courses JSONB,
  estimated_completion_time INTERVAL,
  learning_resources JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE interviewer_metrics (
  metric_id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(user_id),
  interview_count INT DEFAULT 0,
  avg_score_deviation NUMERIC(5,4),
  bias_indicators JSONB,
  calibration_score NUMERIC(5,4),
  last_evaluated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE training_programs (
  program_id UUID PRIMARY KEY,
  interviewer_metric_id UUID NOT NULL REFERENCES interviewer_metrics(metric_id),
  program_content JSONB,
  completion_status TEXT CHECK (completion_status IN ('NOT_STARTED','IN_PROGRESS','COMPLETED')),
  assigned_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ
);
```

- テーブル生成はDrizzle Kitによるマイグレーションで管理し、スキーマ定義（TypeScript）からDDLを自動生成する。PII列（email, phone, hashed_national_id）は`pgcrypto`拡張で暗号化。履歴は`application_history` テーブルでAudit（`updated_by`, `reason`）。
- BigQueryには`application_fact`, `payment_fact`, `feature_snapshots` として匿名ID化した形でロード。KeyマッピングはHashidsで不可逆。

### 3.3 個人情報・機微情報の取り扱い
- 区分: `機密(Confidential)`、`特定PII`、`金融` の3レベル。アプリケーション層で`Data Tagging`を実装し、GraphQLリゾルバでアクセス制御。
- 保存: S3バケットは`server-side encryption (SSE-KMS)`、`bucket policy`でVPN/VPCエンドポイント経由のみ許可。履歴書は30日経過後に自動削除（設定で延長可能）。
- 転送: OpenAIへ送信前に`pii-scrubber`ライブラリで固有名詞をトークナイズ。Stripe・KYCにはTLS1.2以上、mTLSは2025Q4までに導入。
- ログ: PIIを含むログは禁止。必要に応じて`redact`メソッドでマスク。監査ログは`audit_logs`にJSONBで保管し、7年保持。アクセスはAdminロールのみ。
- メッセージデータ: `communication_messages.body` は`pgcrypto`で暗号化し、検索用にN-gram索引用の`message_search_vector`を別列に保持。LINEユーザーIDはトークナイズし、連携解除で匿名化。
- 重複検知データ: `candidate_aliases.alias_value` はハッシュ＋トークナイズを併用し、辞書攻撃防止にソルト管理。タグ操作は監査ログに記録し、役割に応じた削除権限を制御。

## 4. API設計
### 4.1 GraphQLスキーマ（主要抜粋）
```
type Query {
  me: Viewer!
  job(id: ID!): Job!
  jobs(filter: JobFilter, pagination: PaginationInput!): JobConnection!
  candidate(id: ID!, includeHistory: Boolean = false): Candidate!
  candidateDuplicates(filter: DuplicateFilter!): [CandidateDuplicate!]!
  talentPool(filter: TalentPoolFilter!, pagination: PaginationInput!): TalentPoolConnection!
  pipeline(orgId: ID!): [PipelineStage!]!
  paymentOrder(id: ID!): PaymentOrder!
  conversations(filter: ConversationFilter!, pagination: PaginationInput!): ConversationConnection!
  conversation(id: ID!): Conversation!
  messageTemplates(orgId: ID!): [MessageTemplate!]!
  agentOrders(filter: AgentOrderFilter!, pagination: PaginationInput!): AgentOrderConnection!
  reports(input: ReportInput!): ReportPayload!
  performanceReview(id: ID!): PerformanceReview!
  teamCultureMatches(candidateId: ID!): [CultureMatch!]!
  learningPath(candidateId: ID!): LearningPath!
  interviewerMetrics(userId: ID!): InterviewerMetrics!
}

type Mutation {
  createJob(input: CreateJobInput!): Job!
  publishJob(id: ID!, channels: [JobChannelInput!]!, publishAt: DateTime): Job!
  ingestApplication(input: ApplicationInput!): Application!
  advanceStage(applicationId: ID!, nextStageId: ID!, feedback: FeedbackInput!): Application!
  triggerPayment(applicationId: ID!, escrowHoldTill: DateTime!): PaymentOrder!
  resendKycLink(targetId: ID!, targetType: KycTarget!): KycPayload!
  scheduleInterview(input: ScheduleInterviewInput!): Interview!
  sendMessage(input: SendMessageInput!): CommunicationMessage!
  scheduleBulkMessage(input: ScheduleBulkMessageInput!): BulkJob!
  upsertLineAutomation(input: LineAutomationInput!): LineAutomation!
  mergeCandidateDuplicates(input: MergeDuplicateInput!): Candidate!
  upsertCandidateTags(input: CandidateTagInput!): Candidate!
  createAgentOrder(input: AgentOrderInput!): AgentOrder!
  syncChatworkRoom(input: ChatworkSyncInput!): ChatworkRoomPayload!
  generatePerformanceReview(applicationId: ID!): PerformanceReview!
  updateTeamCulture(input: TeamCultureInput!): TeamCulture!
  generateLearningPath(candidateId: ID!): LearningPath!
  assignTrainingProgram(input: TrainingProgramInput!): TrainingProgram!
}
```
- Relay仕様に沿ったConnection/Edge構造を採用し、ページングは`first/after`。`JobFilter`には媒体、雇用形態、AIスコア閾値などを含む。
- Input validationはGraphQLレベルの`class-validator`とサービス層のビジネスルールで二重化。Audit対象のMutationは`@Audit`デコレータでログ記録。

### 4.2 REST / Webhook エンドポイント
| エンドポイント | メソッド | 用途 | リトライ | 備考 |
| --- | --- | --- | --- | --- |
| `/integrations/jobmedia/:channel/webhook` | POST | 媒体からの応募Webhook | 自動15分間隔（max 10回） | HMAC検証 |
| `/integrations/stripe/webhook` | POST | 決済イベント (`payment_intent.succeeded` 等) | Stripe標準（レジューム） | Secret署名 |
| `/integrations/kyc/webhook` | POST | KYC結果通知 | 15分間隔（max 6回） | mTLS対応予定 |
| `/integrations/line/webhook` | POST | LINE Messaging APIイベント受信（友だち追加、メッセージ、ブロック） | 429時は再送キューに積み5分後リトライ（max 5） | Verifyトークン＋署名検証 citeturn0open0 |
| `/integrations/twilio/webhook` | POST | SMS/音声通話イベント受信、配信結果更新 | 429時は指数バックオフ（max 5） | Twilio署名検証、冪等キー |
| `/integrations/chatwork/webhook` | POST | Chatworkからのメッセージ／既読イベント受信 | 429時は指数バックオフ（max 5） | Chatwork署名検証、部屋IDバリデーション citeturn0view0 |
| `/internal/airflow/report` | POST | Airflow→Reporting Serviceの完了通知 | 冪等ID必須 | VPC内通信 |
| `/public/referral/:token/apply` | POST | 紹介リンク経由応募 | 429制限 | reCAPTCHA v3 |
| `/admin/import/csv` | POST | 既存ATS/ExcelのCSVインポート（候補者・求人・エージェント） | 失敗時は検証レポート出力で再アップロードを促す | ファイルサイズ上限、カラムマッピング必須 citeturn0view0 |

- エラー設計: GraphQLは`extensions.code`でドメインエラー（例: `FORBIDDEN`, `PAYMENT_ESCROW_HELD`）、RESTは`application/problem+json`を返却。
- リトライ戦略: 外部API呼び出しは`exponential backoff (2^n, max 5)`、Stripe/KYCはIdempotency-Keyを付与。媒体APIの429超過時はジョブをリスケジュール。

## 5. 外部連携設計
| 連携 | シーケンス概要 | 失敗時のフォールバック | 監視指標 |
| --- | --- | --- | --- |
| Stripe Connect | (1) Payment Serviceが`payment_intent`作成 → (2) Connect AccountへTransfer + 手数料計算 → (3) Webhookで結果受信 → (4) 成功時に会計仕訳エクスポート | Stripe失敗時は請求書PDF生成＋銀行振込手動処理、エスクローは`FAILED`記録 | `payment_success_rate`, `escrow_time_to_release` |
| KYCベンダー | (1) Candidate Portalで本人確認依頼 → (2) ベンダーSDKで撮影 → (3) Webhook受信後、結果をCandidate Serviceへ反映 → (4) Stripe支払前に再確認 | 3回失敗でCSにタスク割当、メール手動確認フロー | `kyc_completed_percentage`, `kyc_avg_latency` |
| 求人媒体API | (1) Job Serviceが投稿JSONを媒体にPush → (2) 媒体IDを保存 → (3) 30分毎に応募Pull or Webhook受信 → (4) Status同期 | 投稿失敗時は担当者にSlack通知＋UIで再投稿ボタン。応募Pull失敗はジョブリトライ。 | `job_sync_error_rate`, `application_lag_minutes` |
| Slack通知 | (1) Notification ServiceがOrg設定に基づきメッセージ生成 → (2) Slack WebhookへPOST → (3) Responseをログ | 429時は指数バックオフ、3回失敗でメールにフォールバック | `slack_delivery_success`, `avg_notification_latency` |
| Chatwork連携 | (1) Communication ServiceがAPIトークン認証 → (2) 候補者・タスク情報をルームへ投稿 → (3) Webhookで返信・既読を受信 → (4) スレッドに反映 | Chatwork落ち時はSlack/SMSへフォールバック、再送キューで10分後リトライ | `chatwork_delivery_rate`, `chatwork_response_latency` citeturn0view0 |
| LINE公式アカウント | (1) Communication Serviceが認証 → (2) 友だち登録/メッセージWebhook受信 → (3) シナリオに応じて返信・応募フォームリンク配信 → (4) 既読・ブロック状況を更新 | 障害時はメール/SMSにフォールバックし、再接続後に未送信キューを再配信 | `line_friend_growth`, `line_delivered`, `line_block_rate` citeturn0open0 |
| SMS/音声（Twilio） | (1) 送信ジョブをキュー → (2) Twilio API呼び出し → (3) Webhookで送達/失敗更新 → (4) 失敗時は再送 or 別チャネル誘導 | 429/失敗時は指数バックオフ。業務時間外はメール案内に切替 | `sms_delivery_rate`, `call_connect_rate`, `sms_failures` citeturn0open0 |
| OpenAI API | (1) Feature抽出 → (2) Prompt生成 → (3) GPT-5呼び出し → (4) 返却をパース・検証 → (5) Explainability Engineへ送信 | API障害時は`ai_match`サービスが縮退モード（過去スコア再利用/閾値未達は「要人判断」） | `llm_latency_p95`, `llm_cost_per_application`, `fallback_rate` |
| BigQuery/Looker | (1) AirflowがPostgreSQL→BigQueryにELT → (2) Dataformでマート整形 → (3) Looker Studioでダッシュボード | BigQuery障害時はS3にParquetを書き出し後追いロード | `elt_job_success`, `report_staleness_hours` |

- 各連携はIntegration ServiceのFeature Flagで有効化制御。 Sandbox CredentialとProduction Credentialを環境変数で切替。
- SLA要件: Stripe/KYCは99.9% uptime。障害時にはStatusページを監視しFailoverポリシー（銀行振込手動処理など）を起動。

## 6. セキュリティ & コンプライアンス
### 6.1 RBAC権限マトリクス（抜粋）
| ロール | 対象ユーザー | 権限 | 備考 |
| --- | --- | --- | --- |
| `ADMIN` | オーナー/管理者 | Org設定、RBAC編集、請求閲覧、監査ログ閲覧 | 2FA必須 |
| `RECRUITER` | 採用担当 | 求人作成、候補者閲覧、AIスコア閲覧、決済リクエスト | 支払承認不可 |
| `HIRING_MANAGER` | 採用意思決定者 | 候補者評価承認、オファー発行 | 支払閲覧のみ |
| `INTERVIEWER` | 面接官 | 担当面接の議事録作成、AI根拠閲覧（限定） | 候補者検索不可 |
| `FINANCE` | 経理 | 成果報酬決済承認、請求書出力、KYCステータス閲覧 | 候補者詳細閲覧不可 |
| `AGENT` | 紹介会社 | 自社紹介候補のみ閲覧、進捗更新、請求金額閲覧 | AIスコア閲覧制限 |
| `CANDIDATE` | 候補者 | 自身の応募状況確認、書類更新 |  |

- 権限チェックはGraphQL Resolverに`@Scopes()`デコレータを適用し、`org_memberships.role_scopes`に基づく。行レベルセキュリティ（RLS）はPostgreSQLポリシーを有効化し、`current_setting('app.user_id')`を利用。

### 6.2 データ保護と暗号化
- **保存時**: PostgreSQLはAWS KMSカスタマーマスターキーで暗号化。S3はバケットレベルSSE-KMS + バージョニング。RedisはTLS + AUTH。
- **通信時**: すべてTLS1.3。内部通信はAWS PrivateLink/VPC Peering。Stripe・KYCはmTLS化。WebSocketはJWT検証＋Origin制限。
- **鍵管理**: KMSキーは分離したアカウントで管理し、アクセスはLeast Privilege。キー回転は90日周期、自動化をTerraformで。
- **秘密情報**: Secrets Managerで管理し、CI/CDはOIDCフェデレーションで一時トークンを取得。ローカル開発用にSOPSで暗号化。
- **アクセス制御**: Cloudflare Zero TrustでIP/CIDR・地理・端末指紋を評価し、許可されたネットワークのみ管理画面へアクセス。出張時は時間制限付きワンタイムトークンを発行。 citeturn0open0
- **多要素認証**: Better AuthでTOTP/FIDO2、Cloudflare Accessでデバイス信頼スコアを検証し、SAML/OIDC SSOと組み合わせて条件付きアクセスを強制。 citeturn0view0

### 6.3 監査・コンプライアンス
- モニタリング: 監査ログ（操作/認証/設定変更）を`audit_logs`に保存し、週次でSecurity Leadがレビュー。Lookerでダッシュボード化。
- 規制対応: 労働関連法の求人審査フローをJob Serviceに組み込み（禁止表現チェックリスト）。StripeのAML/KYCポリシーと整合。
- 認証: SOC2 Type1を2026Q1申請、ISMSは2026Q2目標。`docs/13_risk_register.md`に対応状況を追記。
- データ保持: 候補者データは採用完了後2年、KYCデータは法令に従い7年保存。削除要求は`DeleteWorkflow`で受理後30日以内に実行。

## 7. インフラ / DevOps
### 7.1 環境構成
- **Development**: AWS ap-northeast-1, VPC共有。RDS (PostgreSQL dev), ElastiCache, S3 dev。Feature BranchごとにEphemeral環境をPulumi Stackで生成し、Cloudflare Workers KV/QueuesはWranglerでローカルバインドして検証。
- **Staging**: Prodに近い構成。Stripe/KYCサンドボックス、OpenAIテストキー。Cloudflare Pages/Workersは専用アカウント（staging zone）で検証し、Zone Rulesを本番と同期。負荷テストは月次。Terraformの`workspace`をstagingで分離。
- **Production**: Multi-AZ Aurora PostgreSQL (Serverless v2)、ECS Fargateでサービスを稼働。フロントエンドはCloudflare Pagesで配信し、Cloudflare CDN + WAF（Bot Management, DDoS保護）経由でエッジ配信。Turnstile・Zero Trustポリシーで管理者アクセスを保護し、Slack GuardDutyアラートとCloudflare LogpushをSecOpsに送付。
- **ネットワーク**: Public/Private Subnet分離、NAT Gateway冗長化。PrivateLinkでStripe/KYC接続。Cloudflare Tunnelで管理者VPNをAWSへ接続し、ゼロトラストアクセスを強制。

### 7.2 IaC / CI/CD
- TerraformでVPC、RDS、ECS、Cloudflare Zone/Workers/Accessポリシー、Secretsを管理。Terraform Cloud + Sentinelでポリシーガードレール（PIIストレージ暗号化必須）。
- GitHub Actions:
  1. Lint/Test → 2. Build Container（kaniko）→ 3. OPA Security Scan → 4. Terraform Plan（自動レビュー）→ 5. ArgoCD/Cloudflare Pagesへデプロイトリガ。
- バージョニング: コンテナはSemVer（`service:1.3.0`）。データベースマイグレーションは`drizzle-kit migrate`をCIで実行し、失敗時は自動ロールバック。

### 7.3 Observability & Reliability
- 監視: Prometheus + Grafana。主要メトリクス（API latency p95, error rate, queue depth, LLM latency, Stripe failures, Cloudflare Cache hit率）。
- ログ: OpenSearch + Fluent Bit。PIIマスクフィルタをFluentで実装。GraphQLリクエストはSampling 10%。Cloudflare LogpushをS3→OpenSearchに取り込み、WAF/Botイベントを相関分析。
- トレース: OpenTelemetry → AWS X-Ray。主要フロー（応募→採用→決済）をService Mapで可視化。
- アラート: PagerDutyで優先度設定 (Sev1: 決済停止, Sev2: 応募処理遅延, Sev3: Slack通知遅延)。SLO: API成功率99.5%, LLM応答90%<5秒。
- DR: RTO 4時間 / RPO 1時間達成のため、Auroraクロスリージョンレプリカ（ap-southeast-1）を用意。年1回DRテストをOpsが主導。
- 通信監視: LINE/Twilio送達率、配信失敗、ブロック率、応答時間をGrafanaで可視化し、異常時はチャネル自動切替。 citeturn0open0

### 7.4 バックアップ & データマネジメント
- RDSスナップショット: 15分間隔の自動バックアップ + 35日保持。週次でPoint-in-timeリストア演習。
- S3: ライフサイクルポリシーで30日後にGlacier、1年後削除。重要ログは永年保管。
- BigQuery: 自動スナップショット＋Looker Studioで可視化。AccessログをCloud Loggingに転送。
- データ移行ログ: CSVインポートは検証レポートをS3に保存し、ハッシュ付きで改ざん検知。重複・欠損の統計をGrafanaに可視化。 citeturn0view0

## 8. 非機能要件トレーサビリティ
| NFRカテゴリ (`docs/11_nonfunctional_requirements.md`) | 目標値 | 設計上の対策 | 計測方法 |
| --- | --- | --- | --- |
| パフォーマンス | Web応答<1.5s p95, AIスコア<5s, 決済API<800ms | CDNキャッシュ、GraphQLキャッシュ、LLMバッチ化、Stripe連携を非同期化 | Grafanaダッシュボード、Cloudflare Web Analytics、Synthetic Monitoring |
| スケーラビリティ | 応募処理 5000件/日, 決済 100件/日 | ECS Fargateオートスケール、Queueベースの非同期処理、Read Replica | Kinesisメトリクス、Temporal queue depth |
| 可用性/信頼性 | 稼働率99.5%, DR RTO 4h/RPO 1h | Multi-AZ, Circuit Breaker, Retryポリシー, DR Runbook | CloudWatch SLO, DRドリル結果 |
| セキュリティ/プライバシー | PII暗号化, RBAC, KYC完全性 | SSE-KMS, RLS, Secrets管理, GuardDuty, Cloudflare Zero Trust | Security Hub, Cloudflare Security Analytics, 監査ログレビュー |
| コンプライアンス | 職業安定法準拠, KYC記録7年保存 | 求人審査ワークフロー, 保管ポリシー, 監査証跡 | Ops Checklist, 監査報告書 |
| 観測性 | MTTR < 30分, インシデント検知<5分 | PagerDuty, Error Budgetポリシー | Incident Postmortem, Alert履歴 |
| UX/アクセシビリティ | SUS>=80, WCAG2.1AA | デザインシステム、アクセシビリティLint、ユーザーテスト | Playwright AXE, UX調査 |
| サポートSLA | 応答SLA: 電話5分/チャット10分/メール4時間、解決SLA: プラン別（Pro 1営業日, Growth 8時間） | サポートBot＋Slack Connect＋専任CSマネが対応、LINE/SMS自動通知 | Zendesk/Ticketダッシュボード, Slack統計, サポートCSAT | citeturn0open0 |
| モバイル体験 | モバイル利用時の主要操作成功率95%、オフラインで10分以内の再同期 | PWA＋Workbox、IndexedDBキャッシュ、オフライン検証E2E | Playwright Mobile、Lighthouse PWA、Runtime同期ログ | citeturn0view0 |

## 9. 実装計画とリスク
### 9.1 フェーズ別ロードマップ（MVP期）
| スプリント(2週) | 期間 | 主な成果物 | 依存関係 |
| --- | --- | --- | --- |
| SP1-2 (〜2025-12-14) | Auth/RBAC基盤、Job Draft UI、PostgreSQLスキーマ初版、CSVインポートPoC | Terraform基盤完了、既存データ受領 | インフラチーム |
| SP3-4 (〜2026-01-25) | 媒体連携PoC、Candidate Import、AIスコアMVP、候補者重複検知＆タレントプール、LINE/Twilio連携PoC | OpenAI契約、KYCベンダーSandbox、LINE開発者審査 | 法務承認 |
| SP5-6 (〜2026-02-22) | Stripeエスクロー、決済UI、Slack/LINE/SMS/Chatwork通知、マルチチャネルUI、エージェント発注ポータル | Stripeアカウント審査完了、通信キャリア・Chatwork審査 | 財務 |
| SP7 (〜2026-03-08) | DRテスト、性能テスト、監査ログ、Staging→Prod準備 | QAリソース確保 | Ops/Legal |
| SP8 (〜2026-03-22) | Betaローンチ、サクセスハンドブック、運用Runbook | ベータ顧客オンボード | CS |

### 9.2 技術的依存関係・オープン課題
- BigQueryとのクロスクラウド転送におけるネットワークコスト最適化 → AWS Private Service Connect + BigQuery Data Transfer検討。
- Temporal vs. Step Functionsのコスト比較 → 2025-11-15までにPoC結果を`docs/12_architecture_overview.md`へ反映。
- 社内Distilledモデル開発（LLMコスト削減）→ Growthフェーズ、MVPではOpenAI依存を継続。
- 新しいAIサービス（Performance Review、Culture Matching、Learning Path、Interviewer Training）のモデル開発と統合。

### 9.3 リスクとMitigation
| リスク | 影響 | 対応策 | トリガー |
| --- | --- | --- | --- |
| Stripe審査遅延 | 決済機能リリース遅れ | 代替決済（銀行振込）バックアップ、事前書類準備 | 2026-01-15までに承認未完 |
| KYC精度不足 | 成果報酬支払停滞 | 二段階KYC（AI＋人手）、複数ベンダー冗長化 | KYC失敗率>5% |
| LLMコスト超過 | 利益率低下 | 条件分岐でBatch化、プロンプト最適化、Embedding再利用 | 1応募あたり>¥20 |
| データドリフト | AI精度低下 | Monthly再学習、Explainabilityレビュー、Human override | 精度乖離>5% |
| 媒体API変更 | 応募データ欠損 | 監視アラート、早期アップデート、Fallback CSV Import | Sync失敗連続3回 |
| LINE/Twilio/Chatwork審査・API制限 | 候補者通知が停止、機能出荷遅延 | 事前審査書類整備、代替チャネル（メール/Slack）フォールバック、レート制御、キュー監視 | 審査未完が30日超 or レート制限発生回数>3/月 | citeturn0open0turn0view0 |
| 新AIサービス開発遅延 | 新機能リリース遅れ | MVPではコア機能に集中、追加機能はGrowthフェーズに延期 | 2026-02-15までにPerformance Review MVP未完 |

## 10. 付録・次アクション
- **シーケンス図**: 求人配信フロー、選考→決済フローをMermaidで作成し、`docs/06_appendices.md` に格納（担当: EM、期限: 2025-11-10）。
- **ER図**: dbdiagram.ioで生成し、PNG/PDFを追加（担当: BE Lead、期限: 2025-11-05）。
- **設定テンプレート**: Slack通知テンプレ、Stripe料金表、KYCチェックリストを `docs/06_appendices.md` に追記（担当: Ops、期限: 2025-11-12）。
- **マルチチャネルRunbook**: LINE/Twilio/メール/SMSのシナリオ図、禁止ワードリスト、手動フォールバック手順を作成し `docs/06_appendices.md` に追加（担当: CX Lead、期限: 2025-11-15）。 citeturn0open0
- **重複検知＆タレントプール運用ガイド**: データクレンジング手順、タグ命名規則、エージェント発注のレビュー項目を整備し、`docs/06_appendices.md` にテンプレを追加（担当: Data Lead、期限: 2025-11-18）。 citeturn0view0
- **新サービス設計ドキュメント**: Performance Review、Culture Matching、Learning Path、Interviewer Trainingサービスの詳細設計を`docs/14_detailed_design.md`に追加（担当: BE Lead、期限: 2025-11-25）。
- **テスト計画リンク**: QAが`docs/05_operational_plan.md` セクション2.2に詳細テストケースを追加。

> 本書の疑問点・変更提案はGitHub Issuesで `Design` ラベルを付与し、週次アーキレビュー（火曜15:00 JST）で承認を得ること。
