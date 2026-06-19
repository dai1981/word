# /word 英単語辞書ページ生成キット

このリポジトリは、英単語の詳細ページ（`word/word/{単語}.html`）を、決まったテンプレートで量産するためのものです。

## 使い方（Claude への指示例）

> 「s- の次の15語を作って、コミットして」

このとき Claude が行うこと:

1. 既存の `data/*.json` と `word/word/*.html` を見て、**まだ作っていない単語**を辞書順で決める
2. 新しいバッチを `data/s-batch2.json`（次は batch3…）として、**下記スキーマで**作成する
3. `python3 generate.py` を実行して `word/word/*.html` を生成する
4. 生成された HTML と `data/*.json` を **commit** する（必要なら PR）

## 生成スクリプト

```bash
python3 generate.py
```

`data/` 内のすべての `*.json`（単語データの配列）を読み、`word/word/{単語}.html` を出力します。外部依存なし（標準ライブラリのみ）。

## データのスキーマ（1単語 = 1オブジェクト）

`data/s-batch1.json` が実例です。各単語は次のキーを持ちます。

```json
{
  "english": "見出し語(小文字)",
  "pron": "発音記号(IPA, スラッシュ無し)",
  "kana": "カナ発音",
  "hinshi": "品詞表示 (例: 動詞 / 名詞)",
  "ld_pos": "JSON-LD用の品詞",
  "ld_etym": "JSON-LD用の語源(短く)",
  "ld_level": "英検レベル (例: 4級)",
  "badges": ["対象(例:小学生〜)", "英検X級", "CEFR Xx", "特徴"],
  "meaning_main": "主な意味をスラッシュ区切り",
  "description": "meta description (70字程度)",
  "keywords": ["xxx 意味", "xxx 使い方", "..."],
  "meanings": [{ "hinshi": "【品詞】意味", "note": "例や補足" }],
  "usages":   [{ "title": "① 見出し", "desc": "説明", "example_en": "英語例文", "example_ja": "和訳" }],
  "warn":     ["よくある間違い・注意点"],
  "etym_note": "語源の解説文",
  "etym_chain": [{ "node": "語形", "sub": "意味", "era": "時代" }],
  "phrases":  [{ "en": "フレーズ", "ja": "意味" }],
  "related":  [{ "en": "関連語", "ja": "意味" }],
  "faq":      [{ "q": "質問", "a": "回答" }],
  "quiz":     [{ "q": "問題文", "A": "選択肢A", "B": "選択肢B", "C": "選択肢C", "D": "選択肢D", "correct": "B" }]
}
```

### 推奨の分量
meanings 3〜5 / usages 2〜3 / phrases 6〜8 / related 3〜4 / faq 2〜3 / quiz 2 / etym_chain 2〜3段（最後が現代英語）。

### 出力先・固定値
- 出力: `word/word/{english}.html`（実サイトの `/word/word/` にそのまま対応）
- canonical: `/word/word/{english}.html`
- GA4: `G-MKNGEYPKNJ`、AdSense: `ca-pub-3234684892462480`、JSON-LD set URL: `https://eigo-duke.com/word/word.html`
  （変更する場合は `generate.py` 冒頭の定数 `GA` / `LD_SET_URL` を編集）

## 注意
- 内容は学習者向けの正確さを優先。意味・語源・例文・クイズは公開前に目視確認を推奨。
- `data/*.json` は追記式。過去バッチは消さない（重複チェックの基準になります）。
