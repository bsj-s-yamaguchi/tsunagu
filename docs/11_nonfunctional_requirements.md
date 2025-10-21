# Tsunagu Non-Functional Requirements (NFR)

## 1. Performance & Scalability
- [ ] 応答時間目標（Web/API）と測定方法
- [ ] 処理スループット（応募/支払/AI推論）
- [ ] スケーリング戦略（オートスケール、バッチ処理枠）

## 2. Availability & Reliability
- [ ] 稼働率SLOとSLA
- [ ] フェイルオーバー/DR計画（Primary/Secondary）
- [ ] バックアップ/リストア手順

## 3. Security & Privacy
- [ ] アクセス制御/RBAC要件（`docs/05_operational_plan.md`連動）
- [ ] データ暗号化・鍵管理ポリシー
- [ ] 監査ログ保持期間と可視化

## 4. Compliance & Legal
- [ ] KYC/AML規制対応（`docs/05_operational_plan.md` §3.2参照）
- [ ] 職業安定法・特商法に関する運用要件
- [ ] 個人情報保護対応（ISO/ISMS、プライバシーポリシー）

## 5. Observability & Monitoring
- [ ] ログ／メトリクス／トレースの標準化
- [ ] SLA/SLO監視ダッシュボード要件
- [ ] アラート設計と運用フロー

## 6. UX & Accessibility
- [ ] WCAG準拠レベル、色覚対応
- [ ] 多言語サポートとローカリゼーション
- [ ] モバイル/レスポンシブ要件

## 7. Maintainability & Operations
- [ ] コード品質基準（Lint/Testing/CI）
- [ ] リリース手順と環境構成管理
- [ ] サードパーティ依存関係の更新ポリシー

> 各項目は `docs/08_requirements.md` と差分を明確化し、数値目標・計測方法を記入後チェックを完了に変更してください。
