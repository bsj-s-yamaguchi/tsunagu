# Tsunagu エコシステム連携優先度マトリクス（2025-10-20 時点）

## 1. 評価概要
- 評価軸: **ビジネスインパクト（市場浸透・収益寄与）** × **実行容易性（API公開度・契約ハードル・法令対応）**。
- カテゴリ: 国内HRIS/給与SaaS、国内会計/ファイナンスSaaS、グローバルEOR/Payroll。
- 競合調査で確認した外部APIや新機能リリース情報を反映し、パートナー交渉の優先順位とPoCテーマを設定。

## 2. 優先度マトリクス
| 優先度 | 推奨パートナー | 主要ユースケース | ビジネスインパクト | 実行容易性 | 注釈 |
| --- | --- | --- | --- | --- | --- |
| Tier 1（即交渉） | **SmartHR（SmartHR Payroll + API）** | 勤怠/給与データ同期、雇用契約と成果報酬決済の自動照合 | Series EでARR1億ドルに到達した国内大手との連携でSMB導入障壁を大幅低減。citeturn2search0 | APIリクエスト元IP制限など最新の接続ガードが提供されており、連携ガイドが整備済み。citeturn3search0 | 要件定義書の成果報酬決済（Sec.2.4）とKYCワークフロー（Sec.2.2/10）の雇用データ起点。 |
|  | **Money Forward Cloud（Payroll/請求・AP）** | 成果報酬請求書自動発行、会計仕訳連携、AP自動化 | AI Invoice Download Agentで請求処理を自律化しつつ、クラウドERP全体でAIエージェントを強化中。citeturn6search1turn6search2 | PayrollのAPIキー発行手順が公開され、外部SaaSとの接続実績が豊富。citeturn1search1 | 要件Sec.2.4/4.2の自動請求書・仕訳と連携PonCを最短化。 |
| Tier 2（3カ月以内に打診） | **freee人事労務 / freee会計** | 給与・年末調整・会計同期、SMBクロスセル | TOKIUMなどAI経理サービスとのAPI連携を拡充し、SMBの業務自動化を支援。citeturn5search1turn5search2 | API連携の事例とパートナー施策が増加しており、実装ナレッジが蓄積。citeturn5search1turn5search2 | 要件Sec.2.4の自動決済/返金と会計処理をSMB層に浸透させる布石。 |
|  | **Yayoi（弥生）クラウド会計・給与** | 法定帳票・年末調整・請求データ連携 | IT導入補助金2025対象として中小企業向け導入支援を強化し、クラウド版への移行を後押し。citeturn7search0 | 製品刷新（弥生25シリーズ）でオンラインアップデートと法令対応を強化、今後のAPI公開余地を事前調査。citeturn7search2 | 要件Sec.4.1のSMB/個人事業主ターゲット獲得に寄与。 |
|  | **QuickBooks Online** | 海外子会社の会計同期、ドル建て請求 | AIエージェント群とEnterprise Suite強化で国際展開と高度分析ニーズに対応。citeturn2search0turn2news12 | Premium/Advancedで高度自動化とAPI連携が提供され、海外PoCの土台になる。citeturn2search0turn2news12 | Networkフェーズの海外展開（Sec.6）向け布石。 |
| Tier 3（探索） | **Remote（EOR/Contractor-of-Record）** | 国際雇用・支払い連携、越境採用サポート | Contractor of Recordでグローバル契約リスクと支払いを肩代わりし、柔軟な人材調達を実現。citeturn5search0turn5search4 | API公開は限定的だが価格・運用ガイドが整備されており、共同PoCの余地がある。citeturn5search4turn5search5 | Networkフェーズでのグローバル採用・成果報酬決済の拡張に適合。 |
|  | **Money Forward NextPay / 決済FinTech連携** | 立替・BNPLで採用成果報酬の早期資金化 | NextPayは送金スケジュール機能を拡張し、B2B決済のスピード改善に注力。citeturn8search0 | HR・会計連携を視野にした開発ロードマップが公開されており、共同開発余地がある。citeturn8search3 | 成果報酬エスクロー構想（Sec.2,ユニーク機能）との相性が高い。 |
|  | **Papaya Global / 他EOR** | 多通貨給与、グローバルKYC | グローバル給与アグリゲーションとコンプライアンス支援で大企業向け価値を拡大。citeturn4search4 | 競合連携やベータAPIが非公開のため、情報収集段階。 | Networkフェーズ候補。 |

## 3. 連携テーマと技術要件整理
### 3.1 SmartHR
- **ゴール**: SmartHR Payrollの給与計算結果や雇用契約データを受け取り、Tsunaguの成果報酬決済・KYCワークフローと自動同期。
- **必須API**: SmartHR API v1（従業員・給与・勤怠）、Webhook通知。APIアクセス制限が厳格化されたためIP制御とOAuthクライアント管理が必須。citeturn0search0
- **交渉ポイント**: 共同プレスで「採用→雇用→給与」一気通貫を訴求。SmartHRの導入企業に対して成果報酬型決済をアップセル。要件Sec.2.4/5.2/10に紐付く。

### 3.2 Money Forward Cloud
- **ゴール**: 成果報酬決済で発生する請求書・領収書を自動生成し、会計仕訳・AI債務処理に連携。
- **必須API**: Money Forward Cloud API（請求書・経費・会計）、APIキー発行とWebhook受信。citeturn1search1
- **交渉ポイント**: AI Accounts Payable Agentと連携し、請求承認～支払いまでの工数削減を共同PR。要件Sec.2.4/4.2/9に紐付く。

### 3.3 freee
- **ゴール**: SMBユーザーがfreee人事労務/会計を利用している前提で、成果報酬決済データをfreeeに自動送信。
- **必須API**: freee API（勤怠・給与・会計）。他社連携実績（TOKIUM）を参考にスコープ整理。citeturn4search1turn4search4
- **交渉ポイント**: 「採用から会計までの自動化」ソリューションを共同セミナーで展開。要件Sec.4.1のSMB市場に直結。

### 3.4 Yayoi
- **ゴール**: 弥生クラウド移行ユーザー向けに、成果報酬データをクラウド会計・給与へ同期し、法定帳票を自動生成。
- **必須API**: 弥生クラウドの外部連携（現時点ではパートナー限定）。連携可能性調査が必要。citeturn6search3
- **交渉ポイント**: 弥生の「業務の一気通貫・AI活用」ロードマップと合致する利点を提案。要件Sec.4.1/6で個人事業主層を獲得。

### 3.5 QuickBooks
- **ゴール**: 米国子会社/クライアントの採用成果報酬をQuickBooksで自動記帳し、ドル建て請求をStripeと同期。
- **必須API**: QuickBooks Online API（Premium/Advanced推奨）。citeturn2search0turn2news12
- **交渉ポイント**: Networkフェーズ（Sec.6）に向けた海外展開PoCを共同で実施。

### 3.6 Remote
- **ゴール**: RemoteのContractor of Recordと接続し、Tsunagu経由で紹介された契約者をEOR雇用化→給与支払いまで自動化。
- **必須API**: Remoteパートナー向けAPI（現状限定公開）。QuickBooks連携事例からAPI連携パターンを調査。citeturn5search0turn5search4
- **交渉ポイント**: グローバル採用・成果報酬決済のストーリーを共同で構築。要件Sec.6/Networkフェーズで活用。

## 4. パートナー交渉ロードマップ
| 期間 | アクション | オーナー | 成果物 |
| --- | --- | --- | --- |
| 2025-10 | SmartHR/Money Forwardとの情報交換ミーティング設定。 | BizDev | 連携要件メモ、共同価値提案ドラフト |
| 2025-11 | SmartHR APIアクセス申請、Money Forwardデベロッパー登録。 | Engineering | APIアクセスキー、試験環境接続レポート |
| 2025-12 | freee/Yayoiパートナープログラム申請、連携ユースケース資料作成。 | BizDev | 提携申請書、共催ウェビナー構成案 |
| 2026-01 | QuickBooks/Remote向けPoC企画書作成、英語Pitch資料整備。 | International Biz | PoC要件書、コスト試算 |
| 2026-02 | 連携PoC（SmartHR/Money Forward）リリース、導入先β顧客選定。 | Product | 成功指標レポート、次期ロードマップ反映 |

## 5. 次のステップ
1. 2025-10-24までにSmartHR・Money Forwardへの打診メール草稿を作成し、承認を得る。
2. 2025-10-28までにPoC対象顧客（SmartHR＋Money Forward利用企業）候補リストを営業と共通化。
3. 2025-11-05までにfreee/Yayoiのパートナープログラム要件を調査し、参加条件と費用を整理。
