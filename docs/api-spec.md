# API仕様書

サーバーAPIのエンドポイント仕様

---

## 🌐 ベースURL

```
Production: https://your-app.vercel.app
Development: http://localhost:3000
```

---

## 📍 1. GPS位置情報送信

### エンドポイント
```
POST /api/gps
```

### リクエスト

**Headers:**
```
Content-Type: application/json
```

**Body:**
```json
{
  "device_id": "device_001",
  "latitude": 35.6812,
  "longitude": 139.7671,
  "accuracy": 10,
  "battery_level": 85,
  "timestamp": "2026-01-21T10:30:00Z"
}
```

**パラメータ:**

| フィールド | 型 | 必須 | 説明 |
|-----------|---|------|------|
| device_id | string | ○ | デバイス識別ID |
| latitude | number | ○ | 緯度（-90〜90） |
| longitude | number | ○ | 経度（-180〜180） |
| accuracy | number | × | 測位精度（メートル） |
| battery_level | number | × | バッテリー残量（0-100%） |
| timestamp | string | ○ | 測位時刻（ISO 8601形式） |

### レスポンス

**成功（200 OK）:**
```json
{
  "success": true,
  "message": "Location saved successfully",
  "data": {
    "id": "uuid-here",
    "device_id": "device_001",
    "latitude": 35.6812,
    "longitude": 139.7671,
    "timestamp": "2026-01-21T10:30:00Z"
  }
}
```

**エラー（400 Bad Request）:**
```json
{
  "success": false,
  "error": "Invalid latitude or longitude"
}
```

**エラー（500 Internal Server Error）:**
```json
{
  "success": false,
  "error": "Database error"
}
```

---

## 🎤 2. 音声メッセージ送信（Phase 2）

### エンドポイント
```
POST /api/voice
```

### リクエスト

**Headers:**
```
Content-Type: multipart/form-data
```

**Body:**
```
device_id: device_001
audio: [binary WAV file]
duration: 10
timestamp: 2026-01-21T10:30:00Z
```

**パラメータ:**

| フィールド | 型 | 必須 | 説明 |
|-----------|---|------|------|
| device_id | string | ○ | デバイス識別ID |
| audio | file | ○ | 音声ファイル（WAV形式） |
| duration | number | ○ | 録音時間（秒） |
| timestamp | string | ○ | 録音時刻（ISO 8601形式） |

### レスポンス

**成功（200 OK）:**
```json
{
  "success": true,
  "message": "Voice message processed",
  "data": {
    "transcribed_text": "お母さん、お腹すいた",
    "line_sent": true
  }
}
```

**エラー（400 Bad Request）:**
```json
{
  "success": false,
  "error": "Invalid audio format"
}
```

---

## 📨 3. LINE Webhook（Phase 2）

### エンドポイント
```
POST /api/line-webhook
```

### リクエスト

LINE Messaging APIから自動的に送信されます。

**Headers:**
```
X-Line-Signature: [signature]
```

**Body:**
```json
{
  "events": [
    {
      "type": "message",
      "message": {
        "type": "text",
        "text": "わかったよ、帰ったらおやつ食べようね"
      },
      "source": {
        "userId": "U1234567890"
      }
    }
  ]
}
```

### レスポンス

**成功（200 OK）:**
```json
{
  "success": true
}
```

### 処理フロー

1. LINE Messaging APIから返信メッセージを受信
2. テキストをTTS（Text-to-Speech）で音声化
3. デバイスにPush通知
4. デバイスが音声を再生

---

## 📤 4. デバイスへメッセージ送信（Phase 2）

### エンドポイント
```
GET /api/device-message?device_id=device_001
```

### リクエスト

**Query Parameters:**

| パラメータ | 型 | 必須 | 説明 |
|-----------|---|------|------|
| device_id | string | ○ | デバイス識別ID |

### レスポンス

**成功（200 OK）:**
```json
{
  "success": true,
  "has_message": true,
  "message": {
    "id": "msg_001",
    "text": "わかったよ、帰ったらおやつ食べようね",
    "audio_url": "https://storage.supabase.co/...",
    "timestamp": "2026-01-21T10:35:00Z"
  }
}
```

**メッセージなし（200 OK）:**
```json
{
  "success": true,
  "has_message": false
}
```

---

## 📊 5. 位置情報履歴取得

### エンドポイント
```
GET /api/gps/history?device_id=device_001&limit=50
```

### リクエスト

**Query Parameters:**

| パラメータ | 型 | 必須 | 説明 |
|-----------|---|------|------|
| device_id | string | ○ | デバイス識別ID |
| limit | number | × | 取得件数（デフォルト: 50） |
| from | string | × | 開始日時（ISO 8601形式） |
| to | string | × | 終了日時（ISO 8601形式） |

### レスポンス

**成功（200 OK）:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid-1",
      "latitude": 35.6812,
      "longitude": 139.7671,
      "accuracy": 10,
      "battery_level": 85,
      "timestamp": "2026-01-21T10:30:00Z"
    },
    {
      "id": "uuid-2",
      "latitude": 35.6815,
      "longitude": 139.7675,
      "accuracy": 12,
      "battery_level": 84,
      "timestamp": "2026-01-21T10:25:00Z"
    }
  ],
  "count": 2
}
```

---

## 🔒 6. 認証

### デバイス認証

現在はデバイスIDのみで認証。

**将来の実装（Phase 3以降）:**
- JWTトークン認証
- デバイス登録API
- トークンリフレッシュ

### セキュリティ

- すべてのエンドポイントはHTTPS必須
- Supabase RLSでデータアクセス制御
- LINE Webhook署名検証

---

## 📝 エラーコード

| コード | 説明 |
|-------|------|
| 200 | 成功 |
| 400 | リクエストパラメータエラー |
| 401 | 認証エラー |
| 403 | アクセス権限なし |
| 404 | リソースが見つからない |
| 500 | サーバーエラー |
| 503 | サービス利用不可 |

---

## 🧪 テスト

### curlでのテスト例

**GPS送信:**
```bash
curl -X POST https://your-app.vercel.app/api/gps \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "device_001",
    "latitude": 35.6812,
    "longitude": 139.7671,
    "accuracy": 10,
    "battery_level": 85,
    "timestamp": "2026-01-21T10:30:00Z"
  }'
```

**履歴取得:**
```bash
curl "https://your-app.vercel.app/api/gps/history?device_id=device_001&limit=10"
```

---

## 📚 関連ドキュメント

- [Supabase API Documentation](https://supabase.com/docs/reference/javascript/introduction)
- [LINE Messaging API Reference](https://developers.line.biz/ja/reference/messaging-api/)
- [OpenAI Whisper API](https://platform.openai.com/docs/guides/speech-to-text)
