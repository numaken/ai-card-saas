SaaS版「AI名刺」— 完全仕様（最小構成でローンチ）
1. スタック / 前提

Next.js 14 App Router + TypeScript

UIkit 3（CDN）

Supabase（Postgres/RLS、Auth、SQLは sql/schema.sql）

Stripe Billing（Checkout + Webhook）

OpenAI（gpt-4o-mini）

デプロイ: Vercel

2. URL/ルーティング

公開名刺: /{orgHandle}/c/{cardSlug} → app/[orgHandle]/c/[cardSlug]/page.tsx

API:

POST /api/chat … OpenAI応答 + CTA案内

POST /api/estimate … 料金JSONで概算

POST /api/lead … リード保存

POST /api/stripe/webhook … サブスク更新

3. 環境変数

.env.sample に定義済み。VercelのProject Envに設定（.env.localはGit追跡しない）。

OPENAI_API_KEY, OPENAI_MODEL=gpt-4o-mini, OPENAI_TIMEOUT_MS=20000

NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY

STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, NEXT_PUBLIC_STRIPE_PRICE_PRO

NEXT_PUBLIC_UIKIT_CSS, NEXT_PUBLIC_UIKIT_JS, NEXT_PUBLIC_APP_URL

4. DBスキーマ

sql/schema.sql を Supabase SQL エディタで実行

RLS有効、匿名は active card の select と leads/eventsのinsertのみ許可

create_organization(handle,name) 関数で org + owner + free plan を一括作成可

5. 画面（公開ページ）

UIkit 3 でチャットUI、CTA（見積/相談/Stripeリンク）

入力 → /api/chat へPOST

追加CTA → /api/estimate or /api/lead

6. API設計（詳細）

/api/chat

入: { orgHandle, cardSlug, messages:[{role,content}], lang?, sessionId? }

処理: Supabaseから cards, pricing_rules をRLS下で取得 → OpenAIへ（systemにナレッジ埋め込み）

出: { ok, reply, nextActions:[{type,label,url?}] }

エラー: { ok:false, code, message }、429/500対応

/api/estimate

入: { orgHandle, cardSlug, items:[{key, options[]?}] }

処理: pricing_rules.rules（card専用優先→orgデフォルト）で合計算出

出: { ok, currency:"JPY", breakdown:[{key,base,options[],subtotal}], total, paymentLink? }

/api/lead

入: { orgHandle, cardSlug, name, email, message?, estimate_total? }

処理: leads へ insert（org_id はトリガで自動）

出: { ok, leadId }

/api/stripe/webhook

署名検証 → billing の plan/status/current_period_end 更新

7. OpenAI プロンプト（system）
あなたは {orgName} の営業アシスタントAIです。口調は {persona}。
目的：利用者の文脈に合わせ、最短で理解→行動（見積/リード/決済/予約）へ導く。
制約：与えられたナレッジ（bio/services/pricing_rules/disclaimer）内で回答。不明は推測せず質問。
出力：1〜3文＋必要に応じ箇条書き。価格は参考レンジ。誇大表現を避ける。

8. 非機能 / セキュリティ

Rate Limit（IP×org×card×window、ENVで設定）

CORS：同一オリジン前提

RLS：匿名select/insertの範囲最小化

監査ログ: events に非同期 insert（将来）

9. 受け入れ基準（MVP）

公開ページで Q&A/見積/リード が一連で動作

Stripe free→pro のアップグレードが反映（Webhook経由）

環境変数漏洩なし（.env* 無視、.env.sample のみ共有）

10. 実装TODO（Claude Codeにやらせる範囲）

lib/supabase.ts 実装（Server Component/Route用Client、RLS前提）

/api/chat のモックを置換：cards と pricing_rules 取得 → OpenAI 呼び出し

/api/estimate：rules 取得ロジック & breakdown 合計の堅牢化

/api/lead：リード insert、バリデーション、失敗時の理由返却

/api/stripe/webhook：billing 更新ロジック

公開ページの orgHandle/cardSlug の実値連携（Server Components で card メタ読込）

簡易RateLimit（Upstash or Supabase）

E2E：公開ページ→chat→estimate→lead の一連確認

</details>