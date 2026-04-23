# MemPalace セットアップ計画

## 1. 背景・目的

どのLLM（Claude / ローカルLLM / 他サービス）を使っても、**たつやの記憶・文脈を統一参照できる外部メモリ層**を構築する。使えば使うほど精度が上がるパーソナライズドRAGの実現。

---

## 2. 技術選定の結論

- **ベース：MemPalace**（2026年4月時点で最高精度のOSS Memory System）
- 理由：完全ローカル・無料・MCP対応・Claude連携済み・verbatim storage・時系列KG内蔵
- 方針：車輪の再発明なし。MemPalaceのギャップだけ上乗せする

---

## 3. MemPalaceが解決済みのこと

| 機能 | 状況 |
|------|------|
| 全文verbatim storage（ChromaDB + SQLite） | ✅ 完動 |
| Wing / Hall / Room構造による34%検索精度向上 | ✅ 完動 |
| MCP 19ツール（Claude Code / Claude Desktop連携） | ✅ 完動 |
| 会話エクスポートingest（5フォーマット対応） | ✅ 完動 |
| 時系列Knowledge Graph（Zep相当、SQLiteで無料・完全ローカル） | ✅ 完動 |
| Auto-save hooks（Claude Code専用） | ✅ 完動 |

---

## 4. たつや固有のギャップと対応

| 優先度 | ギャップ | 対応 |
|--------|---------|------|
| 🔴1 | 日本語Embedding精度が低い | `multilingual-e5-large`に差し替え |
| 🔴2 | OllamaからHTTPで記憶参照できない | FastAPI Memory APIラッパーを立てる |
| 🔴3 | Claude Web UI会話の自動ingestがない | 自動ingestスクリプト作成 |
| 🟡4 | Obsidian Vaultのwikilink/tag未解釈 | Vault前処理スクリプト作成 |
| 🟡5 | Windows/WSLのUnicode問題 | 動作確認＋パス修正 |

---

## 5. 実装順序

```
Step 1：MemPalaceインストール・動作確認（WSL）
Step 2：Embeddingモデル差し替え（multilingual-e5-large）← 今ここ
Step 3：FastAPI Memory APIラッパー構築
Step 4：Claude Web UI自動ingestスクリプト
Step 5：Obsidian Vault前処理スクリプト
Step 6：全LLMからMemory APIを叩く統合テスト
```

---

## 6. Claude Codeへの指示文

```
# MemPalace セットアップ & Embedding差し替えタスク

## 前提環境
- OS: Windows 11 + WSL2（Ubuntu）
- Python 3.9+
- GPU: RTX 5080（16GB VRAM）/ RTX 3060（12GB VRAM）
- Ollama常駐済み

## タスク概要
MemPalaceをWSL上にインストールし、デフォルトのEmbeddingモデルを
日本語対応の`multilingual-e5-large`に差し替える。

## Step 1：インストール・動作確認

pip install mempalace
mempalace init ~/mempalace-tatsuya
mempalace status

## Step 2：Embeddingバックエンドの差し替え

MemPalaceのsearcher.pyおよびconvo_miner.pyで使われている
EmbeddingFunctionを特定し、以下に差し替える。

### 差し替え先（優先順）
1. `multilingual-e5-large`（sentence-transformers経由）
2. 重い場合は`multilingual-e5-small`で代替

from sentence_transformers import SentenceTransformer
model = SentenceTransformer("intfloat/multilingual-e5-large")

ChromaDBのEmbeddingFunctionとして使えるよう
カスタムクラスをwrapして差し込む。

### 確認事項
- searcher.py / convo_miner.py / miner.py のEmbed呼び出し箇所をすべて洗い出す
- デフォルトのEmbeddingFunctionがどこで注入されているか確認
  （config.pyかmcp_server.pyにある可能性）
- 差し替え後、日本語テキストでsearch動作確認

## Step 3：Windows/WSLのUnicode問題確認

GitHub Issue #47（UnicodeEncodeError on Windows）を念頭に、
日本語ファイル名・日本語テキストのingest時にエラーが出ないか確認。
問題があればopen()のencoding='utf-8'追加などで修正。

## Step 4：動作テスト

以下のテストデータで一通り動作確認：

echo "面接のフィードバック：論理的な思考が評価された" > /tmp/test_ja.txt
mempalace mine /tmp/ --wing test
mempalace search "面接 フィードバック"

期待結果：日本語テキストが正しくヒットすること。

## 完了条件
- [ ] mempalace statusが正常に動く
- [ ] multilingual-e5-largeでembedが走る
- [ ] 日本語テキストのingest・searchが動く
- [ ] UnicodeエラーなしでWSL上で完走する

## 注意
- MemPalaceのバックエンドはpluggable設計なので、
  既存コードの変更は最小限にとどめる
- 差し替え箇所はconfig.jsonで切り替え可能にしておくと後で楽
```
