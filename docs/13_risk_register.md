# Tsunagu Risk Register

## 1. Risk Log
| ID | Category | Description | Likelihood | Impact | Owner | Mitigation | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R-001 | Compliance | KYCベンダー審査遅延によりMVPスケジュールが遅延する | Medium | High | Legal Lead | 代替ベンダーと手動KYCプロセスを事前準備 | Open |
| R-002 | Technical | AIスコアの説明性不足で顧客承認が得られない | Medium | High | Lead Data Scientist | UI改善とナレッジ作成を先行実施 | Monitoring |
| R-003 | Financial | Stripe決済手数料変動による粗利悪化 | Low | Medium | CFO | 手数料交渉と代替決済手段の調査 | Open |

## 2. Risk Heatmap (To Fill)
- [ ] Likelihood × Impactマトリクスを作成し、主要リスクを配置する。

## 3. Emerging Risks & Watchlist
- [ ] 新しい規制動向（例: 生成AI指針）
- [ ] ベンダーロードマップ変更
- [ ] 国内労務・紹介ビジネスの法改正

## 4. Issue Log (Realized Risks)
| Issue ID | 関連Risk | 発生日 | 影響範囲 | 対応状況 | Lessons Learned |
| --- | --- | --- | --- | --- | --- |

## 5. Review Cadence
- [ ] Weekly Ops Syncで新規リスクの確認
- [ ] Monthly Exec Reviewでステータス更新
- [ ] Quarterly RetroでRisk Appetite見直し

> リスク更新時は `docs/05_operational_plan.md` §5.3 と同期し、重大リスクは24時間以内にオーナーをアサインしてください。
