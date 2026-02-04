# SNS自動投稿システム 設計書

## 1. 概要

### 目的
AI（Claude API + 画像生成AI）でコンテンツを自動生成し、Facebook・Instagram・X (Twitter) に完全自動で投稿するシステムを構築する。

### 対象SNS

| SNS | ターゲット | 投稿頻度 | コンテンツ形式 |
|-----|-----------|----------|--------------|
| X (Twitter) | 介護関係者全般 | 毎日 | テキスト + 画像 |
| Facebook | 経営者・管理者 | 週3回 | テキスト + 画像 |
| Instagram | 若手介護職員 | 週3回 | 画像 + キャプション |

### 月間投稿数
- X: 約30投稿/月
- Facebook: 約12投稿/月
- Instagram: 約12投稿/月
- **合計: 約54投稿/月**

---

## 2. コスト見積もり

### 月額コスト

| 項目 | 費用 | 備考 |
|------|------|------|
| X API | **$0** | Free tier（月1,500投稿まで。投稿のみなら十分） |
| Meta Graph API | **$0** | Facebook + Instagram。無料 |
| Claude API (テキスト生成) | **約$2〜5** | 54投稿 × 約$0.05/投稿 |
| 画像生成 (GPT Image 1 Mini) | **約$0.50〜2** | 54枚 × 約$0.01〜0.04/枚 |
| GCP Cloud Run | **約$3〜10** | 低トラフィック。min-instances=0 |
| GCP Cloud Scheduler | **約$0.10** | 3ジョブ（各SNS用） |
| GCP Cloud Storage | **約$0.50** | 画像一時保存 |
| GCP Secret Manager | **約$0.06** | APIキー保管 |

### **月額合計: 約$6〜18 (約900円〜2,700円)**

X を Free tier で運用すれば、非常に低コストで実現可能。

### 初期費用
- なし（GCPの既存プロジェクトを使用）

---

## 3. システムアーキテクチャ

```
┌──────────────────────────────────────────────────────────────┐
│                    GCP Cloud Platform                        │
│                                                              │
│  ┌─────────────┐     ┌──────────────────────────────────┐   │
│  │ Cloud        │     │ Cloud Run: sns-auto-poster       │   │
│  │ Scheduler    │────▶│                                  │   │
│  │              │     │  1. コンテンツプラン選択           │   │
│  │ - X: 毎日8時 │     │  2. Claude API でテキスト生成     │   │
│  │ - FB: 月水金 │     │  3. 画像生成API で画像作成        │   │
│  │ - IG: 火木土 │     │  4. 各SNS APIで投稿              │   │
│  └─────────────┘     │  5. 結果をログ出力               │   │
│                       └──────────┬───────────────────────┘   │
│                                  │                           │
│  ┌─────────────┐    ┌───────────┴──────────┐               │
│  │ Secret       │    │ Cloud Storage         │               │
│  │ Manager      │    │ (生成画像の一時保存)    │               │
│  │              │    └──────────────────────┘               │
│  │ - X API Keys │                                            │
│  │ - Meta Token │    ┌──────────────────────┐               │
│  │ - Claude Key │    │ Cloud Logging         │               │
│  │ - OpenAI Key │    │ (投稿ログ・エラー監視)  │               │
│  └─────────────┘    └──────────────────────┘               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              ┌─────────┐ ┌──────┐ ┌──────────┐
              │ X API   │ │ Meta │ │ OpenAI   │
              │ (POST)  │ │ Graph│ │ Image API│
              └─────────┘ │ API  │ └──────────┘
                          └──────┘
```

---

## 4. コンテンツ生成戦略

### 4.1 コンテンツカテゴリー (既存戦略ベース)

| カテゴリー | 割合 | 内容例 |
|-----------|------|--------|
| AI活用Tips | 40% | ChatGPTで議事録作成、報告書のAI時短術 |
| 助成金情報 | 25% | 75%オフの活用法、申請チェックリスト |
| 導入事例 | 20% | 施設の成功事例、受講者インタビュー風 |
| 業界トレンド | 15% | 介護DX動向、AI最新ニュース |

### 4.2 プラットフォーム別コンテンツ方針

**X (Twitter):**
- 短文テキスト（280文字以内）
- ハッシュタグ: #介護AI #介護DX #ホリエモンAI学校
- 画像: 1枚（OGP風のテキスト入り画像）
- 配信時間: 朝8時（経営者のSNS利用ピーク）

**Facebook:**
- 長文テキスト（300〜500文字）
- ビジネストーン、経営者向け
- 画像: 1枚（実績数字やグラフ風のビジュアル）
- 配信時間: 朝7時

**Instagram:**
- ビジュアル重視
- キャプション + ハッシュタグ（最大30個）
- 画像: スクエア (1080x1080) のインフォグラフィック風
- 配信時間: 朝6時 or 夜21時

### 4.3 AI生成プロンプト設計

各投稿生成時に Claude API に渡すシステムプロンプト:

```
あなたは「ホリエモンAI学校 介護校」のSNSマーケティング担当です。
以下の情報をもとに、{platform}向けの投稿を1件作成してください。

【サービス概要】
- 介護業界特化のAI研修サービス
- 堀江貴文監修、240以上の講座
- 助成金で最大75%オフ（年間31万円→実質7万円）
- 受講者の95%がAI未経験
- 業務改善まで伴走サポート

【今日のカテゴリー】{category}
【トーン】{tone}
【文字数制限】{char_limit}
【ハッシュタグ】{hashtags}

【過去投稿との重複を避ける】
最近の投稿: {recent_posts}
```

### 4.4 画像生成プロンプト設計

```
Create a professional social media graphic for a Japanese elderly care (介護)
AI training service. Style: clean, modern, business-oriented.

Theme: {theme}
Text overlay (in Japanese): {overlay_text}
Color palette: Blue (#1a73e8) and white, professional
Aspect ratio: {aspect_ratio}
Do NOT include any human faces or photos of real people.
```

---

## 5. 技術仕様

### 5.1 ディレクトリ構成

```
sns-auto-poster/
├── src/
│   ├── index.ts              # エントリーポイント (Cloud Run HTTPハンドラ)
│   ├── scheduler.ts          # スケジュール管理
│   ├── content/
│   │   ├── generator.ts      # Claude APIでテキスト生成
│   │   ├── image-generator.ts # OpenAI APIで画像生成
│   │   ├── templates.ts      # 投稿テンプレート定義
│   │   └── categories.ts     # カテゴリーとトピック管理
│   ├── platforms/
│   │   ├── x-client.ts       # X (Twitter) API v2 クライアント
│   │   ├── facebook-client.ts # Meta Graph API (Facebook)
│   │   ├── instagram-client.ts # Meta Graph API (Instagram)
│   │   └── types.ts          # 共通型定義
│   ├── storage/
│   │   └── gcs-client.ts     # Cloud Storage操作
│   └── utils/
│       ├── logger.ts         # Cloud Logging連携
│       └── secrets.ts        # Secret Manager連携
├── Dockerfile
├── package.json
├── tsconfig.json
└── .env.example
```

### 5.2 主要依存パッケージ

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "openai": "^4.70.0",
    "twitter-api-v2": "^1.17.0",
    "@google-cloud/secret-manager": "^5.0.0",
    "@google-cloud/storage": "^7.0.0",
    "@google-cloud/logging": "^11.0.0",
    "express": "^4.18.0",
    "axios": "^1.7.0"
  }
}
```

### 5.3 API連携仕様

#### X (Twitter) API v2 - Free Tier

```typescript
// 認証: OAuth 2.0 (User Context)
// エンドポイント:
// - POST /2/tweets (テキスト投稿)
// - POST /2/media/upload (画像アップロード)

// 制限:
// - 月間1,500投稿 (30投稿/月なら余裕)
// - メディアアップロード可能
```

#### Meta Graph API v22.0

```typescript
// 認証: System User Access Token (長期トークン)
// Facebook投稿:
// - POST /{page-id}/photos (画像付き投稿)
// - POST /{page-id}/feed (テキスト投稿)

// Instagram投稿 (2ステップ):
// 1. POST /{ig-user-id}/media (メディアコンテナ作成)
//    params: image_url, caption
// 2. POST /{ig-user-id}/media_publish (公開)
//    params: creation_id

// 制限:
// - Instagram: 200コール/時間
// - Facebook: エンドポイント毎のレート制限
```

#### 画像生成 (OpenAI GPT Image 1 Mini)

```typescript
// エンドポイント: POST /v1/images/generations
// パラメータ:
//   model: "gpt-image-1-mini"
//   prompt: 画像生成プロンプト
//   size: "1024x1024" (Instagram) / "1792x1024" (X, Facebook)
//   quality: "low" or "medium"
// コスト: $0.005〜0.02/枚
```

### 5.4 投稿フロー (シーケンス)

```
Cloud Scheduler (cron)
    │
    ▼ HTTP POST /api/post
Cloud Run (sns-auto-poster)
    │
    ├─ 1. カテゴリー決定 (ローテーション)
    │     - 前回の投稿カテゴリーを確認
    │     - 40/25/20/15 の比率に基づいて次のカテゴリーを選択
    │
    ├─ 2. テキスト生成 (Claude API)
    │     - プラットフォーム別プロンプトを構築
    │     - 最近の投稿を参照して重複回避
    │     - テキスト + 画像プロンプトを同時生成
    │
    ├─ 3. 画像生成 (OpenAI API)
    │     - Claude が生成した画像プロンプトを使用
    │     - プラットフォーム別サイズで生成
    │     - Cloud Storage に一時保存
    │
    ├─ 4. SNS投稿 (各プラットフォームAPI)
    │     - X: テキスト + 画像を投稿
    │     - Facebook: ページに画像付き投稿
    │     - Instagram: メディアコンテナ作成 → 公開
    │
    ├─ 5. ログ記録 (Cloud Logging)
    │     - 投稿内容、投稿ID、ステータス
    │     - エラー時はアラート通知
    │
    └─ 6. レスポンス返却
          - 成功: 200 + 投稿ID
          - 失敗: 500 + エラー詳細
```

### 5.5 スケジュール設定 (Cloud Scheduler)

| ジョブ名 | cron式 | 説明 |
|----------|--------|------|
| `post-x-daily` | `0 8 * * *` | X: 毎日8:00 JST |
| `post-facebook` | `0 7 * * 1,3,5` | Facebook: 月水金 7:00 JST |
| `post-instagram` | `0 21 * * 2,4,6` | Instagram: 火木土 21:00 JST |

---

## 6. セットアップ手順

### 6.1 事前準備 (手動作業が必要)

#### X (Twitter) Developer Account
1. https://developer.x.com でDeveloper Accountを作成
2. Projectを作成し、Free tierのAppを作成
3. OAuth 2.0 User Contextの設定
4. `media.write`, `tweet.write` スコープを有効化
5. API Key, API Secret, Access Token, Access Secret を取得

#### Meta (Facebook / Instagram)
1. https://developers.facebook.com でDeveloper Accountを作成
2. Business タイプのAppを作成
3. 必要な権限を追加:
   - `pages_manage_posts`
   - `pages_read_engagement`
   - `instagram_basic`
   - `instagram_content_publish`
4. **App Review を申請** (数週間〜数ヶ月かかる)
5. System User Access Token を生成（長期トークン）
6. Facebook Page ID、Instagram Business Account ID を取得

#### OpenAI API
1. https://platform.openai.com でAPIキーを取得
2. 請求情報を設定

#### Claude API (Anthropic)
1. https://console.anthropic.com でAPIキーを取得
2. 請求情報を設定

### 6.2 GCP リソース作成

```bash
# Secret Manager にAPIキーを保存
echo -n "YOUR_X_API_KEY" | gcloud secrets create x-api-key --data-file=-
echo -n "YOUR_X_API_SECRET" | gcloud secrets create x-api-secret --data-file=-
echo -n "YOUR_X_ACCESS_TOKEN" | gcloud secrets create x-access-token --data-file=-
echo -n "YOUR_X_ACCESS_SECRET" | gcloud secrets create x-access-secret --data-file=-
echo -n "YOUR_META_ACCESS_TOKEN" | gcloud secrets create meta-access-token --data-file=-
echo -n "YOUR_OPENAI_API_KEY" | gcloud secrets create openai-api-key --data-file=-
echo -n "YOUR_ANTHROPIC_API_KEY" | gcloud secrets create anthropic-api-key --data-file=-

# Cloud Storage バケット作成
gsutil mb -l asia-northeast1 gs://PROJECT_ID-sns-images/

# Cloud Schedulerジョブ作成
gcloud scheduler jobs create http post-x-daily \
  --schedule="0 8 * * *" \
  --time-zone="Asia/Tokyo" \
  --uri="https://sns-auto-poster-XXXXX.run.app/api/post" \
  --http-method=POST \
  --body='{"platform":"x"}' \
  --oidc-service-account-email=SCHEDULER_SA@PROJECT_ID.iam.gserviceaccount.com

gcloud scheduler jobs create http post-facebook \
  --schedule="0 7 * * 1,3,5" \
  --time-zone="Asia/Tokyo" \
  --uri="https://sns-auto-poster-XXXXX.run.app/api/post" \
  --http-method=POST \
  --body='{"platform":"facebook"}' \
  --oidc-service-account-email=SCHEDULER_SA@PROJECT_ID.iam.gserviceaccount.com

gcloud scheduler jobs create http post-instagram \
  --schedule="0 21 * * 2,4,6" \
  --time-zone="Asia/Tokyo" \
  --uri="https://sns-auto-poster-XXXXX.run.app/api/post" \
  --http-method=POST \
  --body='{"platform":"instagram"}' \
  --oidc-service-account-email=SCHEDULER_SA@PROJECT_ID.iam.gserviceaccount.com
```

### 6.3 デプロイ

```bash
cd sns-auto-poster

# Docker ビルド & Cloud Run デプロイ
gcloud run deploy sns-auto-poster \
  --source . \
  --region asia-northeast1 \
  --platform managed \
  --no-allow-unauthenticated \
  --set-secrets "X_API_KEY=x-api-key:latest,\
X_API_SECRET=x-api-secret:latest,\
X_ACCESS_TOKEN=x-access-token:latest,\
X_ACCESS_SECRET=x-access-secret:latest,\
META_ACCESS_TOKEN=meta-access-token:latest,\
OPENAI_API_KEY=openai-api-key:latest,\
ANTHROPIC_API_KEY=anthropic-api-key:latest" \
  --set-env-vars "FACEBOOK_PAGE_ID=YOUR_PAGE_ID,\
INSTAGRAM_ACCOUNT_ID=YOUR_IG_ID,\
GCS_BUCKET=PROJECT_ID-sns-images" \
  --cpu 1 \
  --memory 512Mi \
  --min-instances 0 \
  --max-instances 3 \
  --timeout 120
```

---

## 7. リスクと対策

| リスク | 影響 | 対策 |
|--------|------|------|
| Meta App Review が通らない | FB/IG投稿不可 | 早期申請。まずXのみで開始 |
| AI生成コンテンツの品質 | ブランド毀損 | プロンプトの精緻化、NGワードフィルター |
| API障害 | 投稿漏れ | リトライ処理、Cloud Loggingでアラート |
| レート制限超過 | 一時的にブロック | 投稿間隔を十分に確保 |
| X Free tierの廃止・変更 | コスト増 | Basic($100/月)への移行を想定 |
| Meta トークン期限切れ | 投稿失敗 | System User Token使用（長期有効）、期限監視 |

---

## 8. 今後の拡張

| 段階 | 内容 |
|------|------|
| Phase 1 | X のみで自動投稿開始（Meta App Review待ち） |
| Phase 2 | Facebook / Instagram 追加 |
| Phase 3 | 投稿パフォーマンス分析（エンゲージメント取得） |
| Phase 4 | A/Bテスト機能（同一テーマで訴求軸を変えて効果測定） |
| Phase 5 | 管理ダッシュボード（投稿履歴・分析の可視化） |

---

## 9. まとめ

- **月額コスト**: 約$6〜18（約900〜2,700円）
- **対応SNS**: X, Facebook, Instagram
- **コンテンツ**: AI自動生成（テキスト + 画像）
- **運用**: 完全自動（Cloud Scheduler + Cloud Run）
- **最大のボトルネック**: Meta App Review（数週間〜数ヶ月）
- **推奨**: まずXのみで開始し、Meta承認後にFB/IGを追加
