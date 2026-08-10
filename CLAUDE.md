# /word 英単語辞書ページ生成キット

このリポジトリは、英単語の詳細ページ（`word/word/{単語}.html`）を、決まったテンプレートで量産するためのものです。

---

## 404防止の必須ルール（最重要）

2026年7月、Search Console に約1万件の 404 が発生しました。原因は
**JSON-LD に出力していた `"value": "/ˈweðər/"` 形式の発音記号**です。
Googlebot は JSON-LD 内の「引用符で囲まれた `/` 始まりの文字列」をルート相対URLと解釈し、
`https://www.eigo-duke.com/ˈweðər/` をクロールして全件 404 になりました。対処済みです。

### JSON-LD の発音をスラッシュで囲まない

```json
{ "@type": "PropertyValue", "name": "発音", "value": "ˈweðər" }
```

- `data/*.json` の `"pron"` にスラッシュを書いてしまっても、`generate.py` の
  `strip_slashes()` が自動で除去するので安全です
- **`generate.py` の該当箇所を変更しないでください。** 変更すると再発します
- 本文の `発音：/ˈweðər/（ウェザー）` は地の文なので影響が小さく、従来どおりでよい
  （実際の404 約250件のうち本文由来は4〜5件）

### ファイル名とリンクは必ず小文字

本番は Linux で大文字小文字を区別します。`Doha.html` へのリンクは
`doha.html` が存在しても 404 になります。ローカルの Windows/XAMPP では再現しません。

- `generate.py` は `english` を `.lower()` してファイル名にする
- `related` の `en` も `.lower()` して href にする
- **固有名詞も見出し語は小文字で登録すること**（`"english": "japan"`、h1 の表示だけ
  大文字にしたい場合は要相談）

### スペースを含む語はリンクにしない

`give up` のような複数語は `/word/word/give up.html` という存在しないページへの
デッドリンクになります。`generate.py` が自動で `<span class="word-chip">` にします。

### 関連語の実在チェックはしない

`related` に挙げた語のページがまだ無くても、リンクは張られます（`<a>` のまま）。
これは意図的です。データ元が Git・VSCode・ローカルの XAMPP と複数あり、
リポジトリの `word/word/` が本番の全ファイルを持っていないため、
生成時の自動チェックは誤判定を起こします。

不足分は定期的に棚卸しして埋めます。手順は「不足リストの棚卸し」を参照。

---

## リポジトリとローカルの役割分担（重要）

| 場所 | 内容 |
|---|---|
| **Git（このリポジトリ）** | `generate.py` で新規生成した分の `data/*.json` と `word/word/*.html` |
| **ローカル `C:\xampp\htdocs\word\word`** | 本番と同じ全ファイル（約17,000）。旧HTMLもここにある |
| **本番（ロリポップ）** | 全ファイル |

**Git は「新規生成分の管理台帳」**であり、本番の完全なコピーではありません。
したがって次の点に注意してください。

- `python3 generate.py`（引数なし）が出力するのは `data/*.json` にある語だけです。
  ローカルの `word/word/` とは別物なので、混同しないこと
- 不足リストの棚卸しは**ローカル**で行います（Git 側では正しい結果が出ません）
- デプロイはバッチ単位なので、Git に全ファイルが無くても問題ありません

### スラングページは対象外

`slang-*.html` は別テンプレート（安全度メーター、会話形式、フォーマル度別の
言い換えなど）で、別途管理されています。`generate.py` では生成できません。
**不足リストからも除外済みです。** もし `slang-` で始まる語が現れたら飛ばしてください。

---

## 使い方（Claude への指示例）

> 「不足リストの続きから20語作って、コミットして」
> 「不足リストの上から30語作って、コミットして」
> 「s- の次の15語を作って、コミットして」

このとき Claude が行うこと:

1. 既存の `data/*.json` と `word/word/*.html` を見て、**まだ作っていない単語**を決める
2. 新しいバッチを `data/{名前}-batch{N}.json` として、下記スキーマで作成する
3. `python3 generate.py` を実行して `word/word/*.html` を生成する
4. 生成された HTML と `data/*.json` を **main へ直接コミット**する

---

## 不足単語リストからの生成

`data/_missing_words.tsv` が生成の優先順位リストです。1行目はヘッダ、
タブ区切りで4列あります。

```
word	links	searches	score
border	17	0	17
plateau	15	0	15
sentimental	14	0	14
comprehensive	13	0	13
```

| 列 | 意味 |
|---|---|
| `word` | 見出し語（小文字） |
| `links` | サイト内から張られている被リンク数。多いほどハブ語 |
| `searches` | サイト内検索で入力されたが結果が無かった回数（`logmiss.php` の記録） |
| `score` | `links + searches × 5` の優先度。**この降順で並んでいる** |

### 手順

1. `data/_missing_words.tsv` を上から読む
2. 既存の `data/*.json` と `word/word/*.html` に無い語だけを対象にする
3. `data/missing-batch{N}.json` を作成（N は連番）
4. `python3 generate.py` を実行し、main へ直接コミット
   - 例: `Add missing-batch2 (20 words: border...tight)`
5. TSV からは削除しない（進捗は `data/*.json` で判断する）

### 現在の状況（2026年8月）

- 全2,801語（スラング除外済み）
- うち被リンク2本以上が511語 → ここまでで切れているリンクの37%が塞がる
- 残り2,290語は被リンク1本のみ（ある1ページの関連語に1回出ただけ）

被リンク2本以上を作り終えた時点で、一度 Search Console のインデックス率を
確認してから残りに進む方針です。

### 単語ではないがその意味を調べて追加してほしい語

`searches` 列が 1 以上の語はユーザーの検索ログ由来なので、
**英単語として成立しないものが混ざっています。**

- サービス名・サイト名（`yahoo`, `poki`, `youtube`, `geforcenow` など）
- 試験名・略語（`teap` など）
- 打ち間違い・入力途中（`repuire`（require）, `reslut`（result）,
  `jamany`（Germany）, `sophisti`, `millitally` など）
- 一般的な英単語として辞書に載らないもの

できる限りユーザーに分かりやすいように、それが何なのかの説明を入れてください。
関連などが分からないものは飛ばしてOKです。

---

## 不足リストの棚卸し

新しく作ったページの `related` から、また新しい未生成語が生まれます。
月に一度など定期的に、**ローカル**（`C:\xampp\htdocs`）の PowerShell で
リストを作り直し、`data/_missing_words.tsv` に上書きコミットしてください。

```powershell
$root='C:\xampp\htdocs\word\word'; $e=[Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase); foreach($f in [IO.Directory]::GetFiles($root,'*.html')){[void]$e.Add([IO.Path]::GetFileNameWithoutExtension($f))}; $rx=[regex]::new('href="/word/word/([^"]+)\.html"','Compiled'); $d=@{}; foreach($f in [IO.Directory]::GetFiles($root,'*.html')){foreach($m in $rx.Matches([IO.File]::ReadAllText($f,[Text.Encoding]::UTF8))){$s=[uri]::UnescapeDataString($m.Groups[1].Value); if($s -match '\s'){continue}; $l=$s.ToLower(); if($l -like 'slang-*'){continue}; if(-not $e.Contains($l)){$d[$l]=[int]$d[$l]+1}}}; $out=[Environment]::GetFolderPath('Desktop')+'\_missing_links.tsv'; "word`tlinks"|Set-Content -Encoding UTF8 $out; $d.GetEnumerator()|Sort @{e={$_.Value};Descending=$true},@{e={$_.Key}}|%{"{0}`t{1}" -f $_.Key,$_.Value}|Add-Content -Encoding UTF8 $out; "不足 $($d.Count) 語 -> $out"
```

検索ログ（`missing_words.log`）も合わせる場合は、`word` をキーに
`links` と `searches` を突き合わせ、`score = links + searches × 5` で並べ直します。

---

## 生成スクリプト

```bash
python3 generate.py
```

`data/` 内のすべての `*.json`（単語データの配列）を読み、`/word/word/{単語}.html` を出力します。
外部依存なし（標準ライブラリのみ）。

バッチを指定して生成することもできます（デプロイで使用）。

```bash
python3 generate.py --batch missing-batch2            # そのバッチだけ生成
python3 generate.py --batch missing-batch2 --out _deploy   # 出力先を変える
```

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

meanings 3〜5 / usages 2〜3 / phrases 6〜8 / related 3〜4 / faq 2〜3 / quiz 2 /
etym_chain 2〜3段（最後が現代英語）。

### related を選ぶときのコツ

既にページがありそうな一般的な語を優先してください。
珍しい語ばかり挙げると未生成リンクが増え、次の棚卸しで不足リストが膨らみます。
同じバッチで作る語を相互に挙げるのも有効です。

### 出力先・固定値

- 出力: `/word/word/{english}.html`（実サイトの `/word/word/` にそのまま対応）
- canonical: `/word/word/{english}.html`（ルート相対で出力される）
- GA4: `G-MKNGEYPKNJ`、AdSense: `ca-pub-3234684892462480`
- JSON-LD set URL: `https://www.eigo-duke.com/word/word.html`
  （変更する場合は `generate.py` 冒頭の定数 `GA` / `LD_SET_URL` を編集）

---

## コミット方針（重要）

- **main ブランチに直接コミットしてください。**（PR・ブランチ作成は不要）
  - Claude Code のセッションで作業ブランチが指定されている場合はそちらへコミットし、
    あとで GitHub 上で PR をマージしてください
- 1バッチごとに、生成した `/word/word/*.html` と追加した `data/*.json` をまとめて1コミットにする
- コミットメッセージ例: `Add missing-batch2 (20 words: border...tight)`

## デプロイ（手動・バッチ単位）

**コミットしてもサーバーには送られません。**

GitHub の Actions タブ →「Deploy to Lolipop」→「Run workflow」で、
`batch` 欄に送りたいバッチ名（例: `missing-batch2`）を入力して実行します。
複数まとめるならカンマ区切り（`missing-batch2,missing-batch3`）。

指定したバッチのページだけが `_deploy/` に生成されて転送されるため、
**過去バッチには一切触れません。** 削除も構造的に起こりません。
詳細は `DEPLOY.md` を参照。

## 投入ペースについて

本番には既に約17,000ページあります。短期間に数千ページを一括追加すると、
Google の「大規模生成されたコンテンツの不正使用」ポリシーの検知対象になり得ます。
判定基準は生成手段ではなく**独自の価値があるか**なので、テンプレートの水準を保つことが前提ですが、
**1バッチ15〜30語、score の高い順**で段階的に投入し、
Search Console のインデックス率を確認しながら進めてください。

## 今後の予定

- 不足リスト2,801語の消化（被リンク2本以上の511語をまず優先）
- 旧HTML（Homepage Builder 製、`name="GENERATOR"` を含む）2,381語の作り直し。
  中身をJSで描画する作りのため検索エンジンから中身が見えていない。
  ただし `babycarriage` `bro._Bro.` のような整理対象も含まれるため要精査
- 動詞の活用形（-s / -ed / -ing / 過去分詞）と形容詞比較級のページ化。
  約3,000ページ規模。原形ページに canonical を向ける設計で重複を回避する

## 注意

- 内容は学習者向けの正確さを優先。意味・語源・例文・名詞なら可算名詞か不可算名詞か表示
- `data/*.json` は追記式。過去バッチは消さない（重複チェックの基準になります）
