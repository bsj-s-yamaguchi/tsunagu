# Tsunagu データベース設計書（MVPドラフト）

## 0. ドキュメント概要
- **目的**: 採用SaaS「Tsunagu」のMVP期（〜2026-03-31）におけるデータベース構成・スキーマ・運用ルールを定義し、開発・運用チーム間の共通認識を形成する。
- **スコープ**: プライマリOLTP（Aurora PostgreSQL）、補助ストア（Redis/S3）、データ連携（BigQuery, Feature Store）に関する論理・物理設計、命名規約、アクセス制御、ライフサイクル管理を記載。アプリケーションコードの実装詳細やBIダッシュボード定義は対象外。
- **参照資料**: `docs/14_detailed_design.md` §3, `docs/07_feature_catalog.md`, `docs/08_requirements.md` §13, `docs/11_nonfunctional_requirements.md`, `docs/05_operational_plan.md` §3。
- **更新ルール**: テーブル追加／重大な制約変更／データ保持ポリシー更新時に改訂。変更履歴は `docs/00_cover.md` へ反映。

## 1. データアーキテクチャ概要
| レイヤ | ストア | 役割 | 備考 |
| --- | --- | --- | --- |
| OLTP | **Aurora PostgreSQL Serverless v2** (`ap-northeast-1`) | トランザクション、整合性、RBAC, RLS | スキーマ: `core`, `payments`, `compliance`, `analytics_staging` |
| キャッシュ | Redis（ElastiCache） | セッション（Better Auth）、短期キャッシュ、ジョブキュー、マルチチャネル未送信メッセージ | データ保持は最大7日。重要データはPostgreSQLを唯一の真実源とする。 citeturn0open0 |
| ファイル | S3 (`tsunagu-prod`) | レジュメ等バイナリ、暗号化（SSE-KMS）、ライフサイクル管理 | バケット: `resume`, `kyc`, `exports`, `log-archive` |
| 分析 | BigQuery | KPI/レポート用スター/スノーフレークスキーマ | Airflow経由でELT。個人識別情報はHash/Tokens化。 |
| AI Feature | Feast（GKE） + BigQuery | モデル学習特徴量、推論結果のスナップショット | `feature_store` テーブル群をPostgreSQLからETLで反映。 |

## 2. 命名規約 & 全体ポリシー
- **スキーマ**: `core`（採用ドメイン）、`payments`（成果報酬・請求）、`compliance`（KYC・監査）、`communication`（マルチチャネル履歴）、`ai`（スコア・根拠）、`infra`（設定・マイグレーション管理）。 citeturn0open0
- **テーブル名**: スネークケース、単数形（例: `core.job_posting`）。ジョインテーブルは `{左}_{右}`（例: `core.job_channel`）。
- **主キー**: 全テーブルでUUID v7（時間順）を使用。カラム名は `<entity>_id`。
- **共通監査列**: `created_at`, `created_by`, `updated_at`, `updated_by`。`updated_by`は非NULL、システム操作は`SYSTEM`.
- **論理削除**: 原則サポートしない。ビジネス面で必要な場合は `status` カラム＋履歴テーブルで管理。
- **タイムゾーン**: `TIMESTAMPTZ` で保存、UTC。UIでJST表示はアプリ側で変換。
- **マイグレーション**: Drizzle ORM + `drizzle-kit` で生成し、Git管理。`infra.schema_migrations` で適用履歴を保持。

## 3. 論理モデル（ER概観）
### 3.1 採用ドメイン（core）
- `organization` 1---n `org_membership` ---n `user`
- `organization` 1---n `job_posting` 1---n `job_channel`
- `job_posting` 1---n `application` n---1 `candidate`
- `application` 1---n `pipeline_stage_snapshot`
- `application` 1---n `interview` 1---1 `interview_note`
- `candidate` 1---n `kyc_record`（`compliance`）

### 3.2 決済ドメイン（payments）
- `application` 1---1 `payment_order` 1---n `payout`
- `payment_order` 1---n `invoice`、`stripe_event`
- `referral_partner` 1---n `payout`

### 3.3 コンプライアンス（compliance）
- `kyc_record`（候補者/紹介者対象）、`kyc_attempt`
- `audit_log`（全操作追跡）＋`access_token_audit`
- `consent_history`（候補者同意）

### 3.4 AI/分析補助（ai / analytics_staging）
- `ai_score_snapshot`（fit/culture/salary/attrition）
- `ai_explanation`（特徴寄与度・根拠テキスト）
- `feature_snapshot`（Feast同期用）
- `report_job_status`（ETL進捗）

### 3.5 コミュニケーション（communication）
- `communication_thread` 1---n `communication_message`
- `communication_thread` 1---n `channel_subscription`
- `line_contact` 1---1 `communication_thread`（LINEユーザーとスレッド紐付け）
- `communication_thread` n---1 `application`（紐づかない汎用スレッドも許容）
- 各スレッドは言語設定、チャネル禁止設定、IP制限解除トークンなどを保持し、ジョブカン同等のマルチチャネル履歴をカバー。 citeturn0open0

## 4. テーブル詳細（抜粋）
以下は主要テーブルのカラム定義サマリ。完全なDDLは `apps/backend/db/schema.ts` にて管理。

### 4.1 core.organization
| カラム | 型 | 制約 / 説明 |
| --- | --- | --- |
| `organization_id` | UUID PK | v7 |
| `name` | TEXT NOT NULL | 企業名 |
| `plan` | TEXT NOT NULL | `FREE/PRO/GROWTH/ENTERPRISE` |
| `billing_email` | CITEXT NOT NULL | 請求連絡先 |
| `primary_timezone` | TEXT DEFAULT 'Asia/Tokyo' | IANA TZ |
| `stripe_customer_id` | TEXT | UNIQUE |
| `brand_theme` | JSONB | ロゴ、カラー等 |
| `status` | TEXT NOT NULL | `ACTIVE/SUSPENDED/TRIAL` |
| `created_at`/`updated_at` | TIMESTAMPTZ | DEFAULT `now()` |
| `created_by`/`updated_by` | UUID | FK `user.user_id` |

### 4.2 core.user & org_membership
- `user` はメールアドレス（CITEXT）にUNIQUE制約。Better Authの外部ID(`auth_provider_id`)を保持。
- `org_membership` は `(org_id,user_id)` 複合PK。`role` は `ADMIN/RECRUITER/...`。RLSポリシーは `current_setting('app.org_id')` に基づきアクセス制御。

### 4.3 core.job_posting
| カラム | 型 | 備考 |
| --- | --- | --- |
| `job_posting_id` | UUID PK | |
| `organization_id` | UUID FK | ON DELETE CASCADE |
| `title` | TEXT | フルテキスト検索対象 (`GIN` index) |
| `employment_type` | TEXT CHECK | `FULL_TIME`等 |
| `location` | JSONB | {country, prefecture, remote} |
| `salary_range` | NUMRANGE | `CHECK (lower>=0)` |
| `description_markdown` | TEXT | |
| `ai_score_profile` | JSONB | RAG結果 |
| `status` | TEXT | `DRAFT/REVIEW/PUBLISHED/...` |
| `publish_at`, `expire_at` | TIMESTAMPTZ | |
| `meta` | JSONB | カスタム属性 |
| 監査列 |  | |

索引: `(organization_id,status) btree`, `GIN (to_tsvector('simple', title || ' ' || description_markdown))`.

### 4.4 core.candidate
- PII列（`full_name`, `email`, `phone`）は `pgcrypto` (`pgp_sym_encrypt`) で暗号化。検索用に正規化・ハッシュ列（`email_hash`）を別途保持。
- `consent_status` + `consent_updated_at` で同意トラッキング。
- `resume_url` はS3署名URL、30日以上は`resume_archive`へ移動。

### 4.5 core.application
| カラム | 型 | 備考 |
| --- | --- | --- |
| `application_id` | UUID PK |
| `candidate_id` | UUID FK `candidate` |
| `job_posting_id` | UUID FK `job_posting` |
| `source` | TEXT | 媒体 / 紹介 |
| `pipeline_stage_key` | TEXT | 現在ステージ |
| `ai_scores` | JSONB | {fit,culture,salary_fit,attrition_risk} |
| `human_override` | JSONB | 手動調整 |
| `referral_partner_id` | UUID FK | NULL許容 |
| `escrow_status` | TEXT | `NONE/HELD/RELEASED/REFUNDED` |
| `hired` | BOOLEAN | |
| `hired_at` | TIMESTAMPTZ | |
| 監査列 |  | |

索引: `(job_posting_id,pipeline_stage_key)`, `(candidate_id)`, `GIN(ai_scores jsonb_path_ops)`.

### 4.6 payments.payment_order
| カラム | 型 | 備考 |
| --- | --- | --- |
| `payment_order_id` | UUID PK |
| `application_id` | UUID UNIQUE | 採用1件につき1オーダー |
| `organization_id` | UUID | |
| `amount_cents` | BIGINT | CHECK >0 |
| `currency` | TEXT DEFAULT 'JPY' | |
| `settlement_due_date` | DATE | |
| `stripe_transfer_id` | TEXT | NULL許容 |
| `status` | TEXT | `PENDING/IN_ESCROW/PAID/REFUNDED/FAILED` |
| `kyc_status` | TEXT | `PENDING/APPROVED/REJECTED` |
| `metadata` | JSONB | |
| 監査列 |  | |

索引: `(status, settlement_due_date)`, `(organization_id, status)`.

### 4.7 compliance.kyc_record
- `target_type`: `CANDIDATE`/`REFERRAL_PARTNER`.
- `provider` (Liquid/Prove 等)。
- `result` JSONB（スコア、リスク指標）。`status` (`PENDING/APPROVED/RETRY/REJECTED`).
- `expires_at` で再確認タイミングを制御。
- `last_checked_at` + `attempt_counter`。失敗時は `kyc_attempt` に詳細を保存。

### 4.8 compliance.audit_log
- 一意ID + `occurred_at`.
- `actor_type` (`USER/SYSTEM/API`), `actor_id`.
- `resource_type`, `resource_id`, `action`, `metadata(jsonb)`.
- `ip_address`, `user_agent`, `success`.
- GINインデックス `metadata`、BTREE `(resource_type, resource_id)`.

### 4.9 ai.ai_score_snapshot
- `snapshot_id` PK, `application_id`, `model_version`, `prompt_template_version`.
- 数値スコア列（NUMERIC(5,2)）。
- `explanation_id` FK（ai_explanation）。
- `evaluated_at`.

### 4.10 analytics_staging.stage_application_fact
- ETL用中間テーブル。`org_id`, `job_id`, `candidate_dim_id`, `pipeline_stage_dim_id`, `status`, `hired_flag`, `llm_cost_jpy`, `ingested_at`.
- BigQuery `analytics.application_fact` のソース。30日でパージ。

### 4.11 communication.*
| テーブル | 主キー | 主要カラム | 備考 |
| --- | --- | --- | --- |
| `communication_thread` | `thread_id` | `org_id`, `application_id`, `referral_partner_id`, `default_channel`, `mute_until`, `locale`, `snooze_reason` | 1候補者につき1+α（部門別）想定。LINE/Twilio審査対応のためチャネルフラグを保持。 citeturn0open0 |
| `communication_message` | `message_id` | `thread_id`, `direction`, `channel`, `subject`, `body`, `media_urls`, `status`, `sent_at`, `provider_message_id`, `language`, `meta` | 本文は`pgp_sym_encrypt`で暗号化。`status`により未送信/配信済/既読を管理。 |
| `channel_subscription` | `subscription_id` | `thread_id`, `channel`, `destination`, `opt_in`, `last_opt_in_at`, `last_opt_out_at` | Do-Not-Contactリストとしても利用。 |
| `line_contact` | `line_user_id` | `org_id`, `candidate_id`, `display_name`, `picture_url`, `linked_thread_id`, `consent`, `created_at` | LINE UIDはHashidsでマスクし、削除要求で匿名化。 |

DDL例:
```sql
CREATE TABLE communication_thread (
  thread_id UUID PRIMARY KEY,
  org_id UUID NOT NULL REFERENCES core.organization(organization_id),
  application_id UUID REFERENCES core.application(application_id),
  referral_partner_id UUID REFERENCES core.referral_partner(partner_id),
  default_channel TEXT NOT NULL CHECK (default_channel IN ('EMAIL','SMS','LINE','PHONE','CHAT')),
  locale TEXT NOT NULL DEFAULT 'ja-JP',
  snooze_reason TEXT,
  mute_until TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE communication_message (
  message_id UUID PRIMARY KEY,
  thread_id UUID NOT NULL REFERENCES communication_thread(thread_id),
  direction TEXT NOT NULL CHECK (direction IN ('INBOUND','OUTBOUND')),
  channel TEXT NOT NULL CHECK (channel IN ('EMAIL','SMS','LINE','PHONE','CHAT')),
  subject TEXT,
  body BYTEA NOT NULL,
  media_urls TEXT[],
  status TEXT NOT NULL CHECK (status IN ('QUEUED','SENT','DELIVERED','FAILED','READ')),
  provider_message_id TEXT,
  language TEXT DEFAULT 'ja-JP',
  sent_at TIMESTAMPTZ,
  meta JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX communication_message_thread_idx ON communication_message(thread_id, sent_at DESC);
CREATE INDEX communication_message_channel_status_idx ON communication_message(channel, status);
```

## 5. インデックス・パフォーマンス戦略
- **共通**: UUID v7で挿入順クラスタリングを維持。高頻度読み出しテーブル（`application`, `job_posting`）は `CLUSTER` コマンドを定期実行。
- **全文検索**: 求人検索用に `tsvector` 列を持ち、`GIN` インデックス。同期はDBトリガーで更新、バッチ再構築は週次。
- **行レベルロック緩和**: `application` のステージ更新は`SELECT FOR UPDATE SKIP LOCKED`で処理。Temporalワークフローからのジョブはバッチ単位（50件）。
- **パーティショニング**: 初年度は必要なし。データ量が増加した場合、`payments.payment_order` を `settlement_due_date` 月単位で`LIST`パーティション検討。
- **メッセージ索引**: `communication_message` は `(thread_id, sent_at DESC)`、`(channel, status)`、全文検索用`message_search_vector`（N-gram）を作成し、既読/未読やチャネル別リトライを高速化。 citeturn0open0

## 6. データフロー & 同期
- **媒体同期**: Job Service -> PostgreSQL -> Kafka `application.created` → Temporal Workflow → Candidate Service insert。失敗時は再試行ジョブ。
- **Stripe Webhook**: `payments.stripe_event` へ暫定保存 → `payment_order` 更新 → Airflowへ通知。
- **KYC**: ベンダーWebhook → `compliance.kyc_record` 更新 → `payment_order.kyc_status` へ反映。
- **LINE公式アカウント**: LINE Messaging API → `communication_message`（INBOUND）挿入 → 即時返信 or ボット応答 → 既読/ブロック情報更新。失敗時はメール/SMS代替。 citeturn0open0
- **Twilio SMS/音声**: Webhook → `communication_message` ステータス更新 → キューリトライ or 別チャネル誘導。エラーはOpsにSlack通知。 citeturn0open0
- **ETL (Airflow)**: 30分毎に変更分を `analytics_staging` にロード → BigQuery へMerge → Looker更新。
- **Feature Store**: 夜間バッチで `ai.feature_snapshot` に集約し、Feastで取り込み。

## 7. データライフサイクル & 保持
- `candidate` PII列: オブジェクト削除要求から30日以内に削除（`DeleteWorkflow`）。履歴は匿名化して保持。
- `resume` バイナリ: 30日でGlacier、2年で削除。法的保全要求時はLegal承認で延長。
- `payments` データ: 税務要件により7年保持。状態`REFUNDED`でも記録維持。
- `audit_log`: 7年保持。S3へLogpushし、PostgreSQLにはRolling 18ヶ月＋アーカイブ参照。
- `analytics_staging` テーブル: BigQuery反映後30日でパージ。
- `communication_message`: OUTBOUNDは2年保持、INBOUNDは1年保持。LINE未読ログは90日で匿名化。削除要求時はスレッド全体をトークナイズ。 citeturn0open0

## 8. セキュリティ・アクセス制御
- **接続**: アプリはAWS IAMベースのIAM Auth。ローカル開発は短命Pass。
- **RLS**: `core` テーブルは `org_membership` に紐づく行のみアクセス可能。`ADMIN` のみRLSバイパス。
- **暗号化**: `pgcrypto` を用いた透過暗号。鍵はAWS KMS管理、Drizzleで`transformer`実装。
- **監査**: `SET app.user_id`, `SET app.org_id` をミドルウェアで実行し、トリガーで監査列更新。
- **バックアップ検証**: 週次で`pg_dump`をテスト環境にリストアして整合性確認。
- **IP・端末制御**: Cloudflare Zero Trustのポリシー決定結果を`compliance.access_token_audit`に書き込み、許可されたIP/CIDRのみ管理画面アクセスを許容。 citeturn0open0
- **メッセージ保護**: `communication_message.body` は`pgp_sym_encrypt`で暗号化し、検索用にN-gramトークナイズした`message_search_vector`を別列に格納。メディアはS3(署名URL, SSE-KMS)に保存。 citeturn0open0

## 9. 運用・マイグレーション
- マイグレーションは `pnpm db:migrate`（Drizzle Kit）で実行。PRで自動`drizzle-kit push`（dry-run）を行い、スキーマ差分をレビュー。
- リリース前に`stg`環境でマイグレーション適用→Smoke Test→`prod`。失敗時は`drizzle-kit rollback`＋DBスナップショットから復旧。
- `schema_version` の更新はFeature Flagと連動し、互換性のない変更は2スプリント前にDeprecation告知。

## 10. 監視・メトリクス
- **PostgreSQL**: CPU/Memory/Connection/Replica Lag、`deadlocks`, `long_running_queries`。Grafanaダッシュボードで可視化。
- **Slow Query**: `log_min_duration_statement = 300ms`。週次で分析し、必要に応じてインデックス調整。
- **Drizzle Migration Lag**: `infra.schema_migrations` との差分をOpsがモニタ。未適用が24時間超過でPagerDuty。
- **ETL整合性**: BigQueryとPostgreSQLの件数差分をAirflowで検証。閾値±1%でアラート。
- **チャンネルKPI**: `communication_message` からチャネル別送達率・既読率・ブロック率を集計し、ラインやSMSのSLAを監視。Airflowで日次スナップショットをBigQueryへ送信。 citeturn0open0

## 11. 今後の課題・TODO
- [ ] ER図（dbdiagram.io）を生成し、`docs/06_appendices.md` にリンク（担当: BE Lead, due 2025-11-05）。
- [ ] `candidate` PII暗号化カラムの部分一致検索要件の精査（営業要望）。Bloom index等の調査。
- [ ] `payments` テーブルの将来パーティショニング検証（データ規模30万件到達時）。
- [ ] GDPR向け Data Processing Agreement に伴うRetention設定のレビュー。
- [ ] Data Catalog（Collibra等）導入検討。MVPではConfluence＋本書で代替。
- [ ] `communication_message` テンプレートの多言語対応とLINE/Twilio審査用ログエクスポートの自動化を設計（担当: CX/BE、due 2025-11-12）。 citeturn0open0

## 3. サービス基盤テーブル

### 3.1 ユーザー・認証関連テーブル
```sql
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
  failed_login_attempts INTEGER DEFAULT 0,
  locked_until TIMESTAMP,
  last_login_at TIMESTAMP,
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

-- 権限テーブル
CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- ロール権限関連テーブル
CREATE TABLE role_permissions (
  role_id UUID REFERENCES roles(id),
  permission_id UUID REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);

-- ユーザーロール関連テーブル
CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id),
  role_id UUID REFERENCES roles(id),
  created_at TIMESTAMP DEFAULT NOW(),
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

-- SSOプロバイダーテーブル
CREATE TABLE sso_providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) NOT NULL,
  client_id VARCHAR(255),
  client_secret VARCHAR(255),
  redirect_uri VARCHAR(255),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- ユーザーSSO関連テーブル
CREATE TABLE user_sso_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  provider_id UUID REFERENCES sso_providers(id),
  provider_user_id VARCHAR(255),
  access_token VARCHAR(500),
  refresh_token VARCHAR(500),
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(provider_id, provider_user_id)
);
```

### 3.2 サブスクリプション・請求関連テーブル
```sql
-- プランテーブル
CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  price_cents INTEGER NOT NULL,
  billing_interval VARCHAR(20) NOT NULL, -- 'month', 'year'
  max_jobs INTEGER, -- NULL = 無制限
  max_candidates INTEGER, -- NULL = 無制限
  max_ai_inference INTEGER, -- NULL = 無制限
  features JSONB, -- 有効な機能リスト
  is_active BOOLEAN DEFAULT true,
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- サブスクリプションテーブル
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  plan_id UUID REFERENCES plans(id),
  status VARCHAR(20) NOT NULL, -- 'active', 'cancelled', 'paused'
  current_period_start TIMESTAMP NOT NULL,
  current_period_end TIMESTAMP NOT NULL,
  trial_ends_at TIMESTAMP,
  cancel_at_period_end BOOLEAN DEFAULT false,
  canceled_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 支払方法テーブル
CREATE TABLE payment_methods (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  stripe_payment_method_id VARCHAR(255),
  type VARCHAR(20) NOT NULL, -- 'card', 'bank_account'
  card_brand VARCHAR(20), -- 'visa', 'mastercard' etc.
  card_last4 VARCHAR(4),
  card_exp_month INTEGER,
  card_exp_year INTEGER,
  is_default BOOLEAN DEFAULT false,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 請求テーブル
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  subscription_id UUID REFERENCES subscriptions(id),
  stripe_invoice_id VARCHAR(255),
  amount_cents INTEGER NOT NULL,
  currency VARCHAR(3) DEFAULT 'jpy',
  status VARCHAR(20) NOT NULL, -- 'draft', 'open', 'paid', 'void', 'uncollectible'
  issued_at TIMESTAMP NOT NULL,
  due_at TIMESTAMP NOT NULL,
  paid_at TIMESTAMP,
  pdf_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 利用状況テーブル
CREATE TABLE usage_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  subscription_id UUID REFERENCES subscriptions(id),
  metric_name VARCHAR(100) NOT NULL, -- 'jobs_posted', 'candidates_processed', 'ai_inference'
  quantity INTEGER NOT NULL,
  recorded_at TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 3.3 サポート・ヘルプ関連テーブル
```sql
-- サポートチケットテーブル
CREATE TABLE support_tickets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  user_id UUID REFERENCES users(id),
  subject VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  status VARCHAR(20) NOT NULL, -- 'open', 'in_progress', 'resolved', 'closed'
  priority VARCHAR(20) NOT NULL, -- 'low', 'medium', 'high', 'urgent'
  category VARCHAR(50), -- 'billing', 'technical', 'feature_request'
  assigned_to UUID REFERENCES users(id), -- サポート担当者
  resolved_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- チケットコメントテーブル
CREATE TABLE ticket_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_id UUID REFERENCES support_tickets(id),
  user_id UUID REFERENCES users(id),
  content TEXT NOT NULL,
  is_internal BOOLEAN DEFAULT false, -- 内部コメントかどうか
  created_at TIMESTAMP DEFAULT NOW()
);

-- ナレッジベース記事テーブル
CREATE TABLE knowledge_articles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  category VARCHAR(100),
  tags VARCHAR(255)[], -- 配列形式のタグ
  is_published BOOLEAN DEFAULT false,
  view_count INTEGER DEFAULT 0,
  helpful_votes INTEGER DEFAULT 0,
  not_helpful_votes INTEGER DEFAULT 0,
  created_by UUID REFERENCES users(id),
  updated_by UUID REFERENCES users(id),
  published_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- リリースノートテーブル
CREATE TABLE release_notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version VARCHAR(20) NOT NULL,
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  release_date TIMESTAMP NOT NULL,
  is_published BOOLEAN DEFAULT false,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW()
);

-- ユーザーリリースノート閲覧テーブル
CREATE TABLE user_release_note_views (
  user_id UUID REFERENCES users(id),
  release_note_id UUID REFERENCES release_notes(id),
  viewed_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, release_note_id)
);
```

### 3.4 セキュリティ・コンプライアンス関連テーブル
```sql
-- 監査ログテーブル
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  company_id UUID REFERENCES companies(id),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id UUID,
  metadata JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- コンプライアンスリクエストテーブル
CREATE TABLE compliance_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  user_id UUID REFERENCES users(id),
  request_type VARCHAR(50) NOT NULL, -- 'data_access', 'data_portability', 'data_deletion'
  status VARCHAR(20) NOT NULL, -- 'pending', 'processing', 'completed', 'rejected'
  details JSONB, -- リクエスト詳細
  requested_at TIMESTAMP DEFAULT NOW(),
  processed_at TIMESTAMP,
  processed_by UUID REFERENCES users(id)
);

-- データ保持ポリシーテーブル
CREATE TABLE data_retention_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id UUID REFERENCES companies(id),
  resource_type VARCHAR(50) NOT NULL, -- 'candidates', 'jobs', 'applications'
  retention_period_days INTEGER NOT NULL,
  auto_archive BOOLEAN DEFAULT false,
  auto_delete BOOLEAN DEFAULT false,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

