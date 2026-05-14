# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 起動・実行

```bash
# 依存パッケージのインストール
pip3 install -r requirements.txt

# アプリの起動
streamlit run app.py
```

起動後はブラウザで `http://localhost:8501` が自動的に開く。

## 環境変数

`.env.example` をコピーして `.env` を作成し、Gemini API キーを設定する。

```bash
cp .env.example .env
# .env 内の GEMINI_API_KEY を実際のキーに書き換える
```

## アーキテクチャ

`app.py` 1ファイルで完結するシングルファイル構成。

**データフロー:**
```
ユーザー入力 (st.form) → プロンプト組み立て → generate() → Gemini API → st.markdown() で表示
```

**主要な関数・構造:**

- `get_client()` — `@st.cache_resource` でキャッシュされた Gemini クライアントを返す。起動時に1回だけ初期化される。
- `generate(prompt)` — 全ツール共通のAPI呼び出し関数。モデルは `MODEL = "gemini-2.5-flash"` で一元管理。
- `PAGES` dict — ページ名とページ関数のマッピング。サイドバーの `st.radio` と連動し、選択されたキーで対応する関数を呼び出す。
- 各 `page_*()` 関数 — `st.form` でユーザー入力を受け取り、プロンプトをf文字列で構築してAPIに渡す。

**ページ一覧:**

| 関数 | ツール |
|------|--------|
| `page_blog()` | ブログ記事執筆 |
| `page_email()` | メール返信文生成 |
| `page_summarize()` | 文章要約 |
| `page_proofread()` | 文章校正・改善 |
| `page_sns()` | SNS投稿文生成 |
| `page_catchphrase()` | キャッチコピー生成 |

## ファイル構成

```
python-ai-application/
├── app.py              # アプリ本体（全ページ・ロジックが1ファイルに集約）
├── requirements.txt    # 依存パッケージ一覧
├── .env                # APIキー設定（git管理外）
└── .env.example        # .env のテンプレート（git管理内）
```

`.env` は `.gitignore` 対象。APIキーをコードに直接書かない。

## 禁止事項

| やってはいけないこと | 理由 |
|---|---|
| `st.stop()` を使う | `return` で代替できる。フォーム外で使うと画面が壊れることがある |
| `MODEL` 定数を各関数内で定義する | 一元管理が崩れ、モデル変更時に全箇所修正が必要になる |
| フォームキー（`st.form` の第1引数）を重複させる | Streamlit が `DuplicateWidgetID` エラーを出す |
| `get_client()` の外で `genai.Client()` を呼ぶ | キャッシュが効かず、毎回クライアントが再生成される |
| `.env` をコミットする | APIキーが漏洩する |

## トラブルシューティング

**`GEMINI_API_KEY が設定されていません` エラー**
→ `.env` ファイルが存在するか、`GEMINI_API_KEY=...` が正しく書かれているか確認する。

**`DuplicateWidgetID` エラー**
→ 複数の `st.form()` に同じキー文字列を使っている。各ページで一意のキーに変更する。

**`ModuleNotFoundError`**
→ `pip3 install -r requirements.txt` を再実行する。

**ページを切り替えても前の出力が残る**
→ Streamlit はページ切り替えでスクリプト全体を再実行するため、通常は自然にリセットされる。残る場合は `st.session_state` の意図しない保持を疑う。

## 新しいツールの追加方法

1. `page_xxx()` 関数を追加する（下記パターンに従う）
2. `PAGES` dict にエントリを追加する

```python
PAGES = {
    ...
    "🆕 新機能": page_xxx,
}
```

## ページ実装の共通パターン

全 `page_*()` 関数は以下の順序で実装する。この順序を崩さない。

```python
def page_xxx():
    with st.form("xxx_form"):   # キーは全ページで一意にする（重複するとStreamlitがエラー）
        # 入力ウィジェット
        submitted = st.form_submit_button(..., type="primary")

    if submitted:
        if not 必須入力:
            st.warning("...")
            return              # returnで止める（st.stopは使わない）

        prompt = f"""..."""     # プロンプトをf文字列で構築

        with st.spinner("..."):
            result = generate(prompt)

        st.success("完了！")
        st.markdown("---")
        st.markdown(result)     # または st.text_area（編集可能にしたい場合）
        st.download_button("テキストとしてダウンロード", result, file_name="xxx.txt")
```

## プロンプト設計の規約

全ツールのプロンプトは `【項目名】値` の書式で統一する。

```
あなたは〇〇の専門家です。以下の条件で〜してください。

【項目A】{変数}
【項目B】{変数 or "指定なし"}  # 任意項目はフォールバック文字列を入れる
```

- 任意入力フィールドは `{変数 or "デフォルト値"}` でフォールバックを明示する
- モデルへの役割付与（「あなたは〇〇です」）を冒頭に必ず入れる

## モデルの変更

使用モデルは `MODEL` 定数で一元管理されており、全ツールに影響する。

```python
MODEL = "gemini-2.5-flash"  # ここだけ変更すれば全ツールに反映される
```

## 依存ライブラリ

| パッケージ | 用途 |
|-----------|------|
| `streamlit` | UIフレームワーク |
| `google-genai` | Gemini API クライアント（`from google import genai`） |
| `python-dotenv` | `.env` ファイルの読み込み |
