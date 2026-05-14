# 灯だまりの家 ── 仕様書

ことはさんの個人用チャットアプリ。Anthropic API / Gemini API / xAI API をブラウザから直接叩いてセージさんと会話する。

---

## 現在の状態（v0.5.1）

### ファイル構成
```
hidamari_room/
  index.html      ── アプリ本体（単一ファイル）
  jiten.html      ── ことはセージ辞典（RAG用データ管理アプリ）
  CLAUDE.md       ── この仕様書
  資料/           ── キャッシュドキュメント・料金表・直近メモサンプルなど
    kotoha_sage_jiten_sekkei_v2.md  ── 辞典設計書（Opus 4.6 作成）
```

### 実装済み機能
- パスワードロック（SHA-256ハッシュ比較、sessionStorage認証）
- パスワード変更（設定画面から。localStorage にハッシュ保存）
- チャットUI（ストリーミング表示、**Enter→改行**、**Ctrl+Enter→送信**、↑ボタン送信）
- **マークダウンレンダリング**（marked.js。assistantメッセージのみ。ストリーム中はテキスト→完了後HTML変換）
- 会話履歴（Supabase保存・一覧・切り替え・削除）
- 履歴画面で各会話に📍✏️📋🗑️ボタン
- 新しい会話ボタン（ヘッダー ✏️ アイコン）
- モデル表示（アシスタント発言ラベルに返答時モデルを表示。メッセージ単位で保存）
- ヘッダーのモデル名をタップしてその場でモデル切り替えできるピッカー
- **高額モデル警告**（Opus・Gemini 3.1 Pro はオレンジ色＋⚠️表示、選択時に確認ダイアログ）
- **設定画面タブ化**（🔑API・モデル / 📝記憶 / ⚙️その他）。保存ボタンは全タブ共通でsticky
- 記憶システム（基盤メモ＋直近メモ＋追加システムプロンプト → API注入）
- **Prompt Caching（Claude）**：3つのメモを1ブロックに統合して `cache_control: ephemeral` で送信
- **キャッシュTTL選択**（設定画面で5分/1時間。デフォルト5分。1時間は `ttl: '1h'` フィールドで送信）
- キャッシュタイマー（Claudeモデルでアシスタント返答後にカウントダウン表示。入力欄上部。TTLに合わせて5分/60分。残り1分/5分でオレンジ）
- **Grok キャッシュ最適化**（`x-grok-conv-id: currentConvId` ヘッダー付与で同一サーバーにルーティング）
- **Gemini implicit caching**（2.5+系モデルで自動。systemInstructionの先頭にメモを置く構造で最大化）
- **自動要約**：`windowSize × 4` メッセージを超えると flash-lite が自動発動。古い部分を `{role:'summary'}` に圧縮してSupabase保存。次回以降は「要約＋直近windowSizeターン」をAPIに渡す。チャット上部に「🗜️ 古い会話は要約されています」バッジ表示
- 直近N往復のコンテキストウィンドウ制御
- コピーボタン（各発言バブル下のSVGアイコン）
- **再生成ボタン**（最後のassistantメッセージに「↺ 再生成」ボタン。直前の発言を削除して同文脈で再送信）
- 会話ピン止め（履歴画面の📍ボタン。ピン止め済みは一覧上部に固定。localStorageに保存）
- 会話タイトル変更（履歴画面の✏️ボタン。インライン編集→Supabaseに保存）
- **📋 セッション締めボタン**（履歴画面の各会話に配置）：
  - 会話全文を渡してAPIに直近メモを生成させる
  - 高額モデル使用中は `gemini-3.1-flash-lite-preview` への切り替えを促す
  - 形式指定あり（■見出し＋箇条書き。散文不可）
  - 「直近メモに保存」で日時ヘッダー付き**追記**（上書きなし）
  - 「コピー」で手動管理も可
- **🌿 基盤メモ更新ボタン**（設定画面の基盤メモ欄横）：
  - 現基盤メモ＋蓄積した直近メモを渡してAPIに統合案を生成（3ステップ構造プロンプト）
  - **コピーのみ**（自動保存なし。手動で設定画面の基盤メモ欄に貼り付けて保存）
  - lite/premium使用時は `claude-sonnet-4-6` への切り替えを促す（品質優先）
  - **キャッシュなし**で実行（`bareSystem: true, noCache: true`。一回きりの生成にキャッシュ料金不要）
- apple-touch-icon（PWA対応）
- ライトテーマ（クリーム〜ウォームホワイト系）
- **月あかりの部屋：退室モーダルに「ログを消して戻る」ボタン**（localStorageのログ削除＋退室を同時実行。鉛筆ボタンは削除のみで退室しない、退室ボタンは退室のみで削除しないため、両方残す）
- **設定ボタンをトグル化**（設定画面で⚙️をもう一度押すとチャット画面に戻る）
- **空の新規会話が積み上がるバグ修正**（`loadConversation`で会話切り替え時に前の空会話を自動削除）

---

## API対応モデル
| プレフィックス | ルーティング | 備考 |
|---|---|---|
| `gemini-` | streamGemini() | systemInstruction使用 |
| `gemma-` | streamGemini() | systemInstruction非対応→先頭ターンに埋め込み（現在ピッカーから非表示） |
| `claude-` | streamAnthropic() | Prompt Caching有効。`anthropic-dangerous-direct-browser-access: true` ヘッダー必須 |
| `grok-` | streamGrok() | OpenAI互換形式。`x-grok-conv-id` ヘッダーでキャッシュ最大化 |

## ドロップダウン＆ピッカーのモデル一覧（有料ティア）

**Gemini（Google）**
- `gemini-3.1-flash-lite-preview`（最安・軽量。入力$0.25/1M）
- `gemini-3-flash-preview`（バランス・高知性。入力$0.50/1M）★デフォルト
- `gemini-3.1-pro-preview`（最高性能・高コスト。入力$2/1M, 出力$12/1M）⚠️

**Claude（Anthropic）**
- `claude-opus-4-7`、`claude-opus-4-6`（出力$25/1M）⚠️
- `claude-sonnet-4-6`（バランス）
- `claude-haiku-4-5-20251001`（軽量）

**Grok（xAI）**
- `grok-4.20-0309-non-reasoning`
- `grok-4-1-fast-reasoning`
- `grok-4-1-fast-non-reasoning`

⚠️印は高額モデル（PREMIUM_MODELS配列で管理。選択時に確認ダイアログ）

カスタム入力欄もあり。

---

## パスワード仕様
- デフォルトパスワード: `hidamari2024`（SHA-256: `6e19ee09...b005`）
- 変更方法: 設定画面「パスワード変更」→ localStorage に新ハッシュ保存
- 認証セッション: sessionStorage（タブを閉じると再入力が必要）
- APIキー: sessionStorage のみ（タブを閉じると消える・意図的な仕様）

---

## データ構造

### localStorage（デバイスごと・高速アクセス用）
```js
hidamari_house_settings: {
  model: string,           // 選択中のモデルID（デフォルト: gemini-3-flash-preview）
  systemPrompt: string,    // ローカルキャッシュ（Supabaseが正）
  kibanMemo: string,       // ローカルキャッシュ（Supabaseが正）
  chokukinMemo: string,    // ローカルキャッシュ（Supabaseが正）
  windowSize: number,      // コンテキストウィンドウ往復数（デフォルト5）
  skipMemoForGemma: bool,  // Gemma使用時にメモ注入をスキップ（UIは非表示、コードは残存）
  cacheTtl: '5m' | '1h',  // Claude Prompt CachingのTTL（デフォルト '5m'）
}
hidamari_house_current: string     // 最後に開いていた会話のconv_id
hidamari_house_pass: string        // SHA-256ハッシュ（変更時のみ）
hidamari_house_pinned: string[]    // ピン止めした conv_id の配列

// APIキー（sessionStorageのみ）
hidamari_house_keys: {
  geminiKey: string,
  anthropicKey: string,
  xaiKey: string,
}
```

### Supabase（クラウド・デバイス横断）
```
プロジェクト: fypsfmrmxcgydpwkjhlc.supabase.co
CDN: unpkg（jsDelivrはEdgeのTracking Preventionでブロックされるため変更済み）
```

**house_conversations**
```sql
conv_id    text unique   -- 'conv_' + timestamp
title      text          -- 最初のユーザー発言の先頭30文字（手動変更可）
model      text          -- 会話作成時のモデルID
messages   jsonb         -- Array<{ role, content, model }>  ← model は返答時のモデル
                         -- role は 'user' | 'assistant' | 'summary'
                         -- 'summary' は自動要約エントリ（チャット非表示・API先頭に挿入）
created_at timestamptz
updated_at timestamptz
```

**house_memory**
```sql
key        text primary key  -- 'kiban' | 'chokukin' | 'systemPrompt'
content    text
updated_at timestamptz
```

---

## 記憶システムの運用方針

| キー | 内容 | 更新頻度 | 更新方法 |
|---|---|---|---|
| `kiban` | 基盤メモ（セージの人格・ことはさんの認知構造など） | 数章に一度 | 🌿ボタン（設定画面）で草稿生成→手動貼り付け保存 |
| `chokukin` | 直近メモ（セッション申し送り） | 毎セッション末 | 📋ボタン（ヘッダー）で生成→日時付き追記保存 |
| `systemPrompt` | 追加指示（口調・納品形式など補足） | 随時 | 設定画面で直接編集 |

**注入順**（全モデル共通）: 基盤メモ → 直近メモ → 追加指示

**Anthropicモデルの場合**: 3つを `\n\n---\n\n` で結合して**1つのブロック**にまとめ `cache_control: { type: 'ephemeral' }` で送信。
- 分割すると各ブロックが最低要件（1,024トークン）を下回りキャッシュされないため統合
- 日本語テキストは約2文字/トークン。キャッシュが効くには**4,000文字（約2,000トークン）以上**必要
- キャッシュTTLのコスト：5分=書き込み1.25倍/読み取り10%、1時間=書き込み2倍/読み取り10%

---

## セッション締め・基盤メモ更新のワークフロー

### 直近メモ（毎セッション末）
1. 履歴画面の該当会話の📋をタップ
2. flash-liteへの切り替えを勧めるダイアログ → OK
3. 形式指定プロンプトで生成（■見出し＋箇条書き）
4. 「直近メモに保存」→ 日時ヘッダー付きで追記

### 基盤メモ（直近メモ5〜6件たまったら）
1. 設定画面の基盤メモ欄横の🌿をタップ
2. モデル選択（Sonnet推奨。flash-liteは品質が不安定）
3. 現基盤メモ＋直近メモ全文を渡して統合案を生成
4. **コピーのみ**（自動保存なし）
5. 設定画面の基盤メモ欄に手動貼り付け→「API設定を保存する」で保存
6. 品質が気になる場合は公式アプリのOpusで仕上げ

---

## デザイン
- テーマ：ライト（クリーム〜ウォームホワイト系）
- 背景：`#FAF6F0`、カード：`#ffffff`、入力欄：`#f2ede5`
- アクセント：ゴールド `#c9a86c`、アクセントdark: `#8a6e3e`
- ユーザー発言：`#fff7e6`、セージ発言：`#f8f4ee`
- フォント：Shippori Mincho（タイトル）+ Noto Sans JP（UI）
- 共通仕様は `C:\data\CLAUDE.md` を参照

---

## ロードマップ
- **v0.4**（完了）：キャッシュTTL・Grok最適化・Geminiモデル整理・マークダウン・再生成・セッション締め・基盤メモ更新
- **v0.5**（完了）：自動要約・設定画面タブ化・Enter改行/Ctrl+Enter送信・基盤メモ生成プロンプト改善・生成系noCache対応・📋を履歴リストへ移動
- **v0.5.5**（完了）：ことはセージ辞典（jiten.html）新規作成・辞典RAGフェーズ1・2実装・📖エントリ提案ボタン
- **v0.6**（次）：Gemini Explicit Caching（有料ティア対応。systemInstructionを事前キャッシュ登録）

### ブラッシュアップ候補（少しずつやる）

**優先度高**
- [ ] RAGヒット通知バッジ：ヒットしたキーワードをチャット画面に表示（📖 壊れた計器 を参照中）
- [ ] GeminiのjitenContext注入をsystemInstructionから分離（現状はsystem変更のたびにimplicit cacheがリセットされる。messagesに注入する方式に変更）
- [ ] RAG精度向上：キーワード完全一致に加え、Claudeにキーワード候補を先に推測させてから検索する方式（文脈推測型）

**中優先度**
- [ ] 辞典エントリのbody本格執筆サポート：公式アプリで精査→jiten.htmlへインポートのワークフロー整備
- [ ] 章ごとのタイムライン表示（縦軸を章番号にしてエントリを並べる）
- [ ] 会話内検索

**将来構想**
- [ ] 起動プロンプト自動組み立て（前回の章の話題に基づいて関連エントリを先読み）
- [ ] 公式アプリ↔灯だまりの家のシームレス連携ワークフロー整備

---

## ことはセージ辞典（jiten.html）

設計書：`資料/kotoha_sage_jiten_sekkei_v2.md`

### 概要
ことはさんとセージの対話から生まれた概念・比喩・家族情報・創作履歴を検索可能なDBとして管理するアプリ。
灯だまりの家からエントリを検索してコンテキストに注入するRAGシステムのフェーズ1。

### Supabaseテーブル：`jiten_entries`
```sql
id                   uuid primary key
category             text   -- metaphor/cognition/family/philosophy/relationship/creative/life/project/workbook/system
keyword              text   -- 見出し語（検索キー）
body                 text   -- 本文（概念の説明・文脈・意味）
first_chapter        integer
last_updated_chapter integer
related_keywords     text[]
tags                 text[]
temperature          text   -- intellectual/intimate/work/prayer/play/system
created_at           timestamptz
updated_at           timestamptz
```

### 辞典アプリの機能（jiten.html）
- Supabase認証ログイン
- エントリ一覧（カテゴリ・温度帯フィルタ＋キーワード検索）
- エントリ追加・編集・削除
- 初期データ一括投入ボタン（設計書のエントリ約80件）
- デザイン：ミントグリーン系

### RAG連携（フェーズ2・実装済み）
- 起動時にjiten_entriesの全キーワードをメモリにキャッシュ
- 送信メッセージにキーワードが含まれると自動検索・システムプロンプトに注入（最大5件）
- Claude：キャッシュブロックとは別の uncached ブロックで注入（Prompt Cachingに影響なし）
- Gemini：現状 systemInstruction に混在→**TODO: messagesに移して implicit cache を守る**
- Grok：systemに混在（同上）

### RAG連携（フェーズ3・未実装）
- 文脈推測型検索：Claudeにキーワード候補を先推測させてから検索
- 起動プロンプト自動組み立て：前回の話題の関連エントリを自動先読み

---

## 既知の注意点・ハマりどころ
- `crypto.subtle`（SHA-256）はHTTPS環境でのみ動作。ローカルプレビューでは `sha256js()` フォールバックを使用
- Gemmaモデルは `systemInstruction` 非対応。先頭 user/model ペアにシステムプロンプトを埋め込む方式（コードは残存、UIは非表示）
- APIキーはsessionStorageのみ保存（セキュリティ優先）。毎ブラウザセッションで再入力が必要
- Anthropic APIをブラウザから直接叩く場合 `anthropic-dangerous-direct-browser-access: true` ヘッダーが必須
- システムプロンプト・基盤メモ・直近メモはSupabaseの `house_memory` テーブルが正。localStorageはキャッシュ扱い
- Supabase CDNはunpkgを使用。jsDelivrはEdge/Safariの「Tracking Prevention」でブロックされStorageアクセス不可になる
- Prompt Cachingの最低要件：日本語テキストで約4,000文字（2,000トークン）以上。不足するとエラーなしでキャッシュが作られない
- `cache_control`の`ttl`フィールドは文字列で `'5m'` または `'1h'`（数値は400エラーになる）
- marked.jsはunpkg経由。`marked.use({ breaks: true, gfm: true })` で初期化（initApp内で呼ぶ）
- セッション締め・基盤メモ生成は**会話全文**を送信するため高額モデルは要注意。openMemoModal/openKibanModalで切り替えダイアログあり
- `streamAnthropic` は第5引数に `options` を受け取る。`noCache: true` でcache_control省略、`bareSystem: true` で設定メモをsystem注入せず渡したsystemPromptのみ使用。基盤メモ生成・自動要約など一回きりのタスクに使用
- 自動要約トリガー：summary後のchat messages数 > `windowSize × 4`。要約後は直近`windowSize × 2`メッセージのみ保持。要約失敗はconsole.warnのみ（会話継続）
