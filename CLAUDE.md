# /word 英単語辞書ページ生成キット

このリポジトリは、英単語の詳細ページ（`word/word/{単語}.html`）を、決まったテンプレートで量産するためのものです。

---

## リポジトリとローカルの役割分担（最重要）

| 場所 | 内容 | ファイル数 |
|---|---|---:|
| **Git（このリポジトリ）** | `generate.py` で新規生成した分の `data/*.json` と `word/word/*.html` | 一部のみ |
| **ローカル `C:\xampp\htdocs\word\word`** | 本番と同じ全ファイル。旧HTMLもここにある | 約21,600 |
| **本番（ロリポップ）** | 全ファイル | 約21,600 |

**Git は「新規生成分の管理台帳」**であり、本番の完全なコピーではありません。

### だから「ページが実在するか」を Git 上で判定してはいけない

`word/word/` に無いからといって、そのページが存在しないとは限りません。
実際、2026年8月に不足リストを Git 上の基準で作ったところ、
3,256語のうち**2,290語（70%）は既に本番に存在していました。**

- リンク先の実在確認、不足リストの棚卸しは**必ずローカルで**行う
- Claude Code は `data/*.json` に無い語を「未作成」と判断してよいが、
  `word/word/` の有無で判断してはいけない
- 不足リスト（`data/_missing_words.tsv`）はローカルで作られた確定版なので、
  **これを信頼して上から作ればよい**

### スラングページは対象外

`slang-*.html` は別テンプレート（安全度メーター、会話形式、フォーマル度別の
言い換えなど）で、別途管理されています。`generate.py` では生成できません。
不足リストからも除外済みです。もし `slang-` で始まる語が現れたら飛ばしてください。

---

## 404防止の必須ルール

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

### ファイル名とリンクは必ず小文字

本番は Linux で大文字小文字を区別します。`Doha.html` へのリンクは
`doha.html` が存在しても 404 になります。ローカルの Windows/XAMPP では再現しません。

- `generate.py` は `english` を `.lower()` してファイル名にする
- `related` の `en` も `.lower()` して href にする
- **固有名詞も見出し語は小文字で登録すること**（`"english": "japan"`）

### スペースを含む語はリンクにしない

`give up` のような複数語は `/word/word/give up.html` という存在しないページへの
デッドリンクになります。`generate.py` が自動で `<span class="word-chip">` にします。

### 関連語の実在チェックはしない

`related` に挙げた語のページがまだ無くても、リンクは張られます（`<a>` のまま）。
Git 側では実在判定ができないため、生成時の自動チェックは誤判定を起こします。
不足分は定期的にローカルで棚卸しして埋めます。

---

## 使い方（Claude への指示例）

> 「不足リストの続きから20語作って、コミットして」
> 「不足リストの上から30語作って、コミットして」

このとき Claude が行うこと:

1. `data/_missing_words.tsv` を上から読む
2. **`data/*.json` に無い語**を対象にする（`word/word/*.html` の有無では判断しない）
3. 新しいバッチを `data/missing-batch{N}.json` として、下記スキーマで作成する
4. `python3 generate.py` を実行して `word/word/*.html` を生成する
5. 生成された HTML と `data/*.json` を **main へ直接コミット**する

---

## 不足単語リスト

`data/_missing_words.tsv` が生成の優先順位リストです。1行目はヘッダ、
タブ区切りで4列あります。

```
word	links	searches	score
capital	11	0	11
flower	11	0	11
language	11	0	11
city	10	0	10
```

| 列 | 意味 |
|---|---|
| `word` | 見出し語（小文字） |
| `links` | サイト内から張られている被リンク数。多いほどハブ語 |
| `searches` | サイト内検索で入力されたが結果が無かった回数（`logmiss.php` の記録） |
| `score` | `links + searches × 5` の優先度。**この降順で並んでいる** |

### 現在の状況（2026年8月）

- **966語**（ローカル実ファイルと突合済みの確定版。スラング除外済み）
- 本番の総ページ数は約21,600

上から順に作れば、切れている内部リンクが効率よく塞がります。

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

## 不足リストの棚卸し（ローカルで実行）

新しく作ったページの `related` から、また新しい未生成語が生まれます。
月に一度など定期的に、**ローカルの PowerShell** でリストを作り直し、
`data/_missing_words.tsv` に上書きコミットしてください。

**ステップ1: リンク漏れを抽出**

```powershell
$root='C:\xampp\htdocs\word\word'; $e=[Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase); foreach($f in [IO.Directory]::GetFiles($root,'*.html')){[void]$e.Add([IO.Path]::GetFileNameWithoutExtension($f))}; $rx=[regex]::new('href="/word/word/([^"]+)\.html"','Compiled'); $d=@{}; foreach($f in [IO.Directory]::GetFiles($root,'*.html')){foreach($m in $rx.Matches([IO.File]::ReadAllText($f,[Text.Encoding]::UTF8))){$s=[uri]::UnescapeDataString($m.Groups[1].Value); if($s -match '\s'){continue}; $l=$s.ToLower(); if($l -like 'slang-*'){continue}; if(-not $e.Contains($l)){$d[$l]=[int]$d[$l]+1}}}; $out=[Environment]::GetFolderPath('Desktop')+'\_missing_links.tsv'; "word`tlinks"|Set-Content -Encoding UTF8 $out; $d.GetEnumerator()|Sort @{e={$_.Value};Descending=$true},@{e={$_.Key}}|%{"{0}`t{1}" -f $_.Key,$_.Value}|Add-Content -Encoding UTF8 $out; "実在 $($e.Count) / 不足 $($d.Count) 語 -> $out"
```

**ステップ2: 既存の TSV を実ファイルで検証（Git 側で作ったリストを持ち込んだ場合は必須）**

```powershell
$src=[Environment]::GetFolderPath('Desktop')+'\_missing_words.tsv'; $out=[Environment]::GetFolderPath('Desktop')+'\_missing_words_clean.tsv'; $e=[Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase); foreach($f in [IO.Directory]::GetFiles('C:\xampp\htdocs\word\word','*.html')){[void]$e.Add([IO.Path]::GetFileNameWithoutExtension($f))}; $l=Get-Content $src -Encoding UTF8; $k=$l[1..($l.Count-1)]|?{$_ -and -not $e.Contains(($_ -split "`t")[0]) -and ($_ -split "`t")[0] -notlike 'slang-*'}; $l[0]|Set-Content -Encoding UTF8 $out; $k|Add-Content -Encoding UTF8 $out; "実在 $($e.Count) / 入力 $($l.Count-1) 語 → 本当に不足 $($k.Count) 語 -> $out"
```

検索ログ（`missing_words.log`）も合わせる場合は、`word` をキーに
`links` と `searches` を突き合わせ、`score = links + searches × 5` で並べ直します。

---

## 生成スクリプト

```bash
python3 generate.py                                        # data/*.json 全件 → word/word/
python3 generate.py --batch missing-batch2                 # そのバッチだけ
python3 generate.py --batch missing-batch2 --out _deploy   # 出力先を変える（デプロイ用）
```

`data/` 内の `*.json`（単語データの配列）を読み、`{english}.html` を出力します。
外部依存なし（標準ライブラリのみ）。

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

本番には約21,600ページあるので、**一般的な語ならたいてい既に存在します。**
`run` `make` `light` のような基本語を素直に選べば、リンクはほぼ生きます。
逆に珍しい造語や専門用語ばかり挙げると未生成リンクが増えます。

### 出力先・固定値

- 出力: `/word/word/{english}.html`
- canonical: `/word/word/{english}.html`（ルート相対）
- GA4: `G-MKNGEYPKNJ`、AdSense: `ca-pub-3234684892462480`
- JSON-LD set URL: `https://www.eigo-duke.com/word/word.html`
  （変更する場合は `generate.py` 冒頭の定数 `GA` / `LD_SET_URL` を編集）

---

## コミット方針

- **main ブランチに直接コミットしてください。**（PR・ブランチ作成は不要）
  - Claude Code のセッションで作業ブランチが指定されている場合はそちらへコミットし、
    あとで GitHub 上で PR をマージしてください
- 1バッチごとに、生成した `/word/word/*.html` と追加した `data/*.json` をまとめて1コミットにする
- コミットメッセージ例: `Add missing-batch5 (20 words: capital...city)`

## デプロイ（手動・バッチ単位）

**コミットしてもサーバーには送られません。**

GitHub の Actions タブ →「Deploy to Lolipop」→「Run workflow」で、
`batch` 欄に送りたいバッチ名（例: `missing-batch5`）を入力して実行します。
複数まとめるならカンマ区切り（`missing-batch5,missing-batch6`）。

指定したバッチのページだけが `_deploy/` に生成されて転送されるため、
**過去バッチには一切触れません。** 削除も構造的に起こりません。
詳細は `DEPLOY.md` を参照。

## 投入ペースについて

本番には既に約21,600ページあります。短期間に数千ページを一括追加すると、
Google の「大規模生成されたコンテンツの不正使用」ポリシーの検知対象になり得ます。
判定基準は生成手段ではなく**独自の価値があるか**なので、テンプレートの水準を保つことが前提ですが、
**1バッチ15〜30語、score の高い順**で段階的に投入し、
Search Console のインデックス率を確認しながら進めてください。

## 今後の予定

- 不足リスト966語の消化
- 旧HTML（Homepage Builder 製、`name="GENERATOR"` を含む）2,381語の作り直し。
  中身をJSで描画する作りのため検索エンジンから本文が見えていない。
  ただし `babycarriage` `bro._Bro.` のような整理対象も含まれるため要精査
- 動詞の活用形（-s / -ed / -ing / 過去分詞）と形容詞比較級のページ化。
  約3,000ページ規模。原形ページに canonical を向ける設計で重複を回避する

## 注意

- 内容は学習者向けの正確さを優先。意味・語源・例文・名詞なら可算名詞か不可算名詞か表示
- `data/*.json` は追記式。過去バッチは消さない（重複チェックの基準になります）
