# セットアップガイド

このガイドでは、開発環境のセットアップからデバイスの初回起動までの手順を説明します。

---

## 🖥️ 1. 開発環境のセットアップ

### 1.1 Arduino IDEのインストール

1. [Arduino IDE 2.x](https://www.arduino.cc/en/software)をダウンロード
2. インストーラーを実行してインストール

### 1.2 ESP32ボードサポートの追加

1. Arduino IDEを起動
2. `ファイル` → `環境設定`
3. `追加のボードマネージャーのURL`に以下を追加:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
4. `ツール` → `ボード` → `ボードマネージャー`
5. "esp32"で検索してインストール

### 1.3 必要なライブラリのインストール

`ツール` → `ライブラリを管理`から以下をインストール:

- **TinyGSM** (SIM7080G通信用)
- **ArduinoJson** (JSONパース用)
- **HTTPClient** (HTTP通信用)

---

## 🔌 2. ハードウェアのセットアップ（Phase 1）

### 2.1 ブレッドボード配線

```
ESP32          SIM7080G
-------        ---------
TX (GPIO17) -> RX
RX (GPIO16) -> TX
GND         -> GND
3.3V        -> VCC

ESP32          LED
-------        ---
GPIO2       -> アノード（長い方）
               カソード（短い方）-> 330Ω抵抗 -> GND

ESP32          ボタン
-------        ------
GPIO4       -> 一方の端子
GND         <- もう一方の端子
GPIO4       <- 10kΩプルアップ抵抗 -> 3.3V
```

### 2.2 SIMカードの挿入

1. SIM7080Gモジュールの電源をOFF
2. SIMスロットを開ける
3. SIMカードを挿入（切り欠き方向に注意）
4. SIMスロットを閉じる

---

## 📱 3. IIJmio SIMの設定

### 3.1 SIMカードの有効化

1. [IIJmio会員ページ](https://www.iijmio.jp/)にログイン
2. SIMカードを登録
3. 開通手続きを実行
4. 開通完了を確認（通常15-30分）

### 3.2 APN設定（コード内で設定）

```cpp
const char apn[]  = "iijmio.jp";
const char user[] = "mio@iij";
const char pass[] = "iij";
```

---

## ☁️ 4. サーバー環境のセットアップ

### 4.1 Supabaseプロジェクト作成

1. [Supabase](https://supabase.com/)にサインアップ
2. 新規プロジェクトを作成
3. データベースパスワードを設定
4. リージョンを選択（Tokyo推奨）

### 4.2 データベーステーブル作成

SQL Editorで以下を実行:

```sql
-- GPS位置情報ログテーブル
CREATE TABLE gps_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  device_id TEXT NOT NULL,
  latitude DECIMAL(10, 8) NOT NULL,
  longitude DECIMAL(11, 8) NOT NULL,
  accuracy INTEGER,
  battery_level INTEGER,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- インデックス作成
CREATE INDEX idx_device_timestamp ON gps_logs(device_id, timestamp DESC);
CREATE INDEX idx_timestamp ON gps_logs(timestamp DESC);

-- Row Level Security (RLS) 有効化
ALTER TABLE gps_logs ENABLE ROW LEVEL SECURITY;

-- ポリシー作成（APIキー経由のアクセスを許可）
CREATE POLICY "Enable insert for service role" ON gps_logs
  FOR INSERT
  WITH CHECK (true);

CREATE POLICY "Enable read for service role" ON gps_logs
  FOR SELECT
  USING (true);
```

### 4.3 Supabase接続情報の取得

`Settings` → `API`から以下をコピー:
- Project URL
- anon public key
- service_role key（サーバー側で使用）

---

## 🚀 5. Vercelプロジェクトのセットアップ

### 5.1 Next.jsプロジェクト作成

```bash
npx create-next-app@latest kids-gps-server
cd kids-gps-server
npm install @supabase/supabase-js
```

### 5.2 Vercelへデプロイ

```bash
# Vercel CLIのインストール
npm i -g vercel

# ログイン
vercel login

# デプロイ
vercel
```

### 5.3 環境変数の設定

Vercelダッシュボードで以下を設定:

```
SUPABASE_URL=your-project-url
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
LINE_CHANNEL_ACCESS_TOKEN=your-line-token
```

---

## 📲 6. LINE Bot のセットアップ

### 6.1 LINE Developersコンソール

1. [LINE Developers](https://developers.line.biz/)にログイン
2. プロバイダーを作成
3. Messaging APIチャネルを作成
4. Channel Access Tokenを取得

### 6.2 Webhook URLの設定

1. Messaging API設定画面
2. Webhook URL: `https://your-app.vercel.app/api/line-webhook`
3. Webhookの利用: ON
4. 応答メッセージ: OFF（重要）

### 6.3 友だち追加

LINEアプリでBotのQRコードをスキャンして友だち追加

---

## 🔧 7. デバイスコードの書き込み

### 7.1 設定ファイルの作成

`device/config.h`を作成:

```cpp
#ifndef CONFIG_H
#define CONFIG_H

// WiFi設定（開発時のみ使用）
#define WIFI_SSID "your-wifi-ssid"
#define WIFI_PASSWORD "your-wifi-password"

// APN設定
#define APN "iijmio.jp"
#define APN_USER "mio@iij"
#define APN_PASS "iij"

// サーバー設定
#define SERVER_URL "https://your-app.vercel.app"
#define DEVICE_ID "device_001"

// ピン設定
#define SIM7080_TX 17
#define SIM7080_RX 16
#define LED_PIN 2
#define BUTTON_PIN 4

#endif
```

### 7.2 コードのアップロード

1. Arduino IDEでデバイスコードを開く
2. `ツール` → `ボード` → `ESP32 Dev Module`
3. `ツール` → `シリアルポート`でESP32を選択
4. アップロードボタンをクリック

### 7.3 シリアルモニターで確認

1. `ツール` → `シリアルモニター`
2. ボーレート: 115200
3. 起動ログを確認

---

## ✅ 8. 動作確認

### 8.1 GPS測位確認

1. デバイスを屋外に持ち出す
2. シリアルモニターでGPS座標を確認
3. 測位成功: `GPS: 35.xxxxx, 139.xxxxx`

### 8.2 通信確認

1. ボタンを押す
2. シリアルモニターで送信ログ確認
3. LINEに通知が届くことを確認

### 8.3 Supabaseデータ確認

1. Supabase Table Editorを開く
2. `gps_logs`テーブルにデータが追加されているか確認

---

## 🐛 9. トラブルシューティング

### GPS測位できない

- **症状:** GPS座標が0.000000のまま
- **原因:** 屋内または衛星が見えない
- **対策:** 
  - 屋外で空が見える場所に移動
  - 初回測位は5-10分かかる場合がある
  - GPSアンテナの接続を確認

### LTE接続失敗

- **症状:** `GPRS connection failed`
- **原因:** SIMカード未開通またはAPN設定誤り
- **対策:**
  - IIJmioの開通状況を確認
  - APN設定を再確認
  - SIMカードの挿入を確認

### サーバーに送信できない

- **症状:** `HTTP POST failed`
- **原因:** サーバーURLが間違っているまたはVercelがダウン
- **対策:**
  - `SERVER_URL`を確認
  - Vercelのデプロイ状況を確認
  - 手動でAPIエンドポイントにアクセステスト

### バッテリーがすぐ切れる

- **症状:** 数時間で電池切れ
- **原因:** Deep Sleepが動作していない
- **対策:**
  - Deep Sleep実装コードを確認
  - シリアルモニターでSleep動作を確認
  - LTEモジュールの完全切断を確認

---

## 📚 次のステップ

Phase 1が完了したら:

1. [Phase 2: 音声機能の実装](./phase2-guide.md)
2. バッテリー駆動時間の最適化
3. ケースの設計

---

## 💬 サポート

質問や問題があれば、GitHubのIssueで報告してください。
