# Trellis — 技術設計書

## 1. データベーススキーマ

### 1.1 気象データ
**テーブル: `HourlyWeather`**
- `timestamp` (PK): DateTime
- `weatherCode`: Int
- `temperature`: Float
- `pressure`: Float
- `humidity`: Float

*設計方針:* 入力時スナップショットは廃止。定期取得データから近傍結合で対応。

### 1.2 睡眠データ
**テーブル: `SleepSessions`**
- `id` (PK)
- `startTime`: DateTime
- `endTime`: DateTime
- `totalDurationMinutes`: Int
- `deepSleepMinutes`: Int
- `lightSleepMinutes`: Int
- `remMinutes`: Int
- `awakeMinutes`: Int

**テーブル: `SleepStages`**
- `id` (PK)
- `sessionId` (FK → SleepSessions)
- `startTime`: DateTime
- `endTime`: DateTime
- `stageType`: Enum [DEEP, LIGHT, REM, AWAKE]

*設計方針:* ステージ遷移の時系列を保持し、視覚化と相関分析を可能にする。

### 1.3 位置情報
**テーブル: `LocationCaptures`**
- `id` (PK)
- `timestamp`: DateTime
- `latitude`: Double
- `longitude`: Double

**テーブル: `MoodRecords`**
- `id` (PK)
- `timestamp`: DateTime
- `mood`: Enum [GOOD, BAD]
- `source`: Enum [WIDGET, APP_HOME]
- `locationId` (FK → LocationCaptures, nullable)

**テーブル: `TextLogs`**
- `id` (PK)
- `timestamp`: DateTime
- `content`: String
- `locationId` (FK → LocationCaptures, nullable)

*設計方針:* 位置は独立テーブル＋外部キー方式。同一時刻の気分・テキスト記録は同一Locationレコードを参照可能。

### 1.4 生体データ（活動系）
**テーブル: `DailyBiometricSummaries`**（MVPで活用）
- `targetDate` (PK, Date)
- `steps`: Int
- `heartRateMin`: Int
- `heartRateAvg`: Int
- `heartRateMax`: Int

**テーブル: `HourlyBiometrics`**（MVPで蓄積のみ）
- `startTime` (PK)
- `endTime`: DateTime
- `steps`: Int
- `heartRateMin`: Int
- `heartRateAvg`: Int
- `heartRateMax`: Int

*設計方針:* Health Connectの履歴データ消滅を防ぐため、詳細粒度は即座に保存。日次は集計実体として保持。

---

## 2. 外部連携詳細

### 2.1 Open-Meteo API
**採用理由:** APIキー不要、非商用無料、CC BY 4.0

**エンドポイント使い分け:**

| 用途 | エンドポイント | パラメータ |
|------|---------------|------------|
| 定期取得 | `/v1/forecast` | `current` |
| 欠損補填 | `/v1/forecast` | `past_days` |
| 長期分析（将来） | `/v1/archive` | - |

**取得項目:** 天気コード、気温、気圧、湿度（1時間粒度）

### 2.2 Health Connect
**取得データ型:**
- `SleepSessionRecord` → SleepSessions/SleepStages
- `StepsRecord` → Daily/Hourly集計
- `HeartRateRecord` → Daily/Hourly集計（min/avg/max）

**同期戦略:**
- 初回：全期間データを一括取得
- 定期：前回同期時刻以降の差分を取得
- 欠損補填：アプリ起動時に直近7日間のデータ整合性をチェック

---

## 3. バックグラウンド処理

### 3.1 WorkManager構成
**定期タスク:**
- `WeatherSyncWorker`: 1時間ごと（気象データ取得）
- `HealthSyncWorker`: 6時間ごと（生体データ同期）

**制約:**
- ネットワーク接続必須（気象）
- バッテリー最適化無視は要求しない

### 3.2 欠損補填ロジック
1. アプリ起動時に各テーブルの最新タイムスタンプを確認
2. 現在時刻との差分が閾値（気象：2時間、生体：24時間）を超えた場合
3. 対象期間をバッチで取得しINSERT（既存レコードはUPSERT）

---

## 4. 位置情報取得実装

**方式:** FusedLocationProviderClient（フォアグラウンドのみ）

**フロー:**
1. ユーザーが記録ボタンをタップ
2. `MoodRecords`/`TextLogs` に一時レコードをINSERT（locationId=null）
3. `getCurrentLocation()` を非同期実行（タイムアウト：5秒）
4. 取得成功： `LocationCaptures` にINSERT → 外部キーをUPDATE
5. 取得失敗/タイムアウト： そのまま完了（locationId remains null）

**精度:** `PRIORITY_BALANCED_POWER_ACCURACY`（街区レベルで十分）