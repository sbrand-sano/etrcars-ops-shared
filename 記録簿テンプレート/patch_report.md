---
date: YYYY-MM-DD
type: patch
target: EC2          # EC2 | ECS | RDS | ALB | Cloudflare | その他
status: scheduled    # scheduled | in-progress | completed | pending
ticket:
reporter: ETR
related_docs:
  - ja/procedures/03_patch_management.md
  - ja/references/appxE5_patch_list.xlsx
---

# パッチ適用記録: <対象> <YYYY-MM-DD>

## 1. 概要
- 対象システム:
- パッチ種別: （セキュリティ / 機能 / バグ修正）
- 適用バージョン: 旧 X.Y → 新 X.Z
- 影響範囲: （停止有無、影響ユーザー、所要時間）

## 2. 事前準備
- [ ] Trello 承認: チケットURL
- [ ] バックアップ取得日時:
- [ ] 関係者連絡: （日時・宛先）
- [ ] ロールバック手順確認:

## 3. 実施内容
| 時刻 | 作業 | 担当 | 結果 |
|---|---|---|---|
| HH:MM | パッチ取得 | | |
| HH:MM | メンテナンス開始通知 | | |
| HH:MM | 適用 | | |
| HH:MM | 動作確認 | | |
| HH:MM | 完了通知 | | |

## 4. 結果・確認事項
- 適用結果: （成功 / 失敗）
- 動作確認項目:
  - [ ] サービス起動
  - [ ] ヘルスチェック
  - [ ] ログ確認
  - [ ] 監視アラート確認
- エビデンス:
  - スクリーンショット: ja/references/images/...

## 5. 残課題・次回予定
-

## 6. 関連リンク
- 手順書: [03_patch_management](../../../ja/procedures/03_patch_management.md)
- パッチ一覧: ja/references/appxE5_patch_list.xlsx
- raw 実施報告: ./同階層の xlsx ファイル
