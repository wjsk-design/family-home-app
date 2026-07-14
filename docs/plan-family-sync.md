# 家族共有の実装計画（夫婦・複数端末）

## 目的（要件）

パパ（本人）とママが、**別々のスマホ・別々のLINEアカウント**から**同じ家庭のデータ**（予定・やること・買い物・植物・メモ）を見て編集できるようにする。現状は各端末の localStorage のみで、端末間で共有されない。

## いまの制約と、直す場所

- データは端末内 localStorage だけ → 共有には「1つの共有置き場（サーバー）」が要る。
- 幸い `js/store.js` の `load()` / `save()` に保存が集約されている（冒頭コメントにも「実データ運用時は load/save を差し替える」と明記）。**同期はこの2関数に差し込むだけ**で済む。

## 全体像

```
[パパ端末] ⇄  [共有置き場：GAS + スプレッドシート]  ⇄ [ママ端末]
  localStorage(キャッシュ)        householdId 配下に state を保存
```

各端末はローカルに持ちつつ、共有置き場と pull/push で同期する。

## 世帯（household）の考え方

- **householdId**（家庭ID）を1つ発行し、データはこのID配下に置く。
- パパが家庭を作成 → **招待コード**を発行 → ママが一度だけコード入力で参加。
- 以降、両者は同じ householdId のデータを読み書きする。

## 認証（ここが本題）

- LIFFで各自 **LINEログイン** → `liff.getProfile()` の userId を取得。
- household に登録された userId だけが読み書きできる（サーバー側で照合）。
- 書き込み時は `liff.getIDToken()` を**サーバー側でLINEに検証**してから受け付ける（なりすまし防止）。
- **秘密情報（チャネルアクセストークン等）はGAS側のみ**。公開リポジトリのフロントには絶対に置かない。

## 同期方式（まずは単純に）

- 起動時とデータ変更時に、state 全体を共有置き場へ push し、他端末の変更を pull。
- **競合**: 夫婦2人の同時編集は稀。初版は「**後勝ち（最終更新時刻）**」で単純化。将来 notes 等の項目 updatedAt を使った項目マージに拡張可能。
- **オフライン**: localStorage を正のキャッシュにして、オンライン時に同期。

## GAS 最小実装

- スプレッドシート（DB）: `households` シート … `householdId` / 参加 `userId` 一覧 / `data`(state の JSON) / `updatedAt`。
- `doGet`: 自 household の data を返す。
- `doPost`: data 更新（IDトークン検証 → household 所属チェック → 保存）。
- フロント: `store.load()` を「GASから pull → 無ければ localStorage」、`store.save()` を「localStorage 保存 → GAS push」に差し替え。

## 段階（小さく出す順）

1. **参加フロー**: household 作成＋招待コード → ママが同じIDに参加（2人が同じ household に紐づく）
2. **同期**: 手動「同期」ボタン → 起動時・変更時の自動同期
3. **競合**: 後勝ちで安定化
4. **通知と合流**: 毎朝ダイジェスト push（[plan-notifications.md](plan-notifications.md) と共通のサーバー基盤）

## 注意

- 個人情報は最小限（メッセージ・保存とも）。
- IDトークン検証を省略しない（省くと URL さえ知れば他人が書けてしまう）。
- notifications の算出ロジック等、フロントの仕様をサーバーに移植する際は**フロントを正**とする。

## 関連

- [plan-notifications.md](plan-notifications.md) … LINEプッシュ。サーバー基盤・認証を本計画と共有する。
