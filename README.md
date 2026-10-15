# Zeitgeist

Go言語で実装された、マルチソース・コンテンツ＆メディアフェッチャーCLI。

登録された複数メディア（RSS/Atomフィード、将来的なスクレイピング・Web API等）からコンテンツを高速並行フェッチし、HTMLタグや広告ノイズを除去・サニタイズしたクリーンな構造化JSONを標準出力（`stdout`）に出力します。

ニュース、技術情報、エンタメ、カルチャー、学術フィードなど、設定ファイル（YAML）に登録したソースに応じて任意のトピックを柔軟に集約可能です。

---

## 主な特徴

- **Adapter Pattern**: 媒体定義（Source）と取得実装（Driver）を疎結合化。同一の取得方式（RSS等）を複数媒体で使い回し可能。
- **超高速並行フェッチ**: Goroutineとセマフォにより、複数媒体を1秒未満（実測300〜500ms）で安全に並行取得。
- **耐障害性 (Graceful Degradation)**: 一部サイトが404やタイムアウトで落ちても、CLI全体は停止せず取得できた媒体のみを返却。
- **設定駆動 (Config-Driven)**: 収集対象の媒体リストはYAMLファイルで完全に外部管理（コードの再コンパイル不要）。
- **トークン最適化**: HTMLタグ除去、実体参照デコード、スニペットの200文字制限により、AI入力時のトークン消費を最小化。

---

## 使用感・ユースケース

### 1. CLI 単体での利用イメージ

ターミナルでコマンドを1回実行するだけで、複数メディアの最新記事が整形されたJSONとして一括出力されます。`jq` などの標準ツールと組み合わせることで、欲しい情報だけを即座に抽出できます。

```bash
# 全媒体のトピックを一括取得
./zeitgeist -config configs/sources.yaml

# 映画カテゴリのみ抽出し、タイトルとURLを一覧表示
./zeitgeist -config configs/sources.yaml -category movie | jq -r '.results[].items[] | "[\(.title)](\(.url))"'
```

**出力イメージ (stdout)**:
```json
{
  "fetched_at": "2026-10-16T00:46:30+09:00",
  "total_sources": 3,
  "succeeded_sources": 3,
  "results": [
    {
      "source": {
        "id": "natalie_eiga",
        "name": "映画ナタリー",
        "description": "映画業界の最新ニュース、新作公開情報、キャストインタビューのまとめ。",
        "category": "movie"
      },
      "items": [
        {
          "title": "堂本剛主演「まる」Netflixで配信、日常が◯に浸食されていく男の物語",
          "url": "https://natalie.mu/eiga/news/693825",
          "published_at": "2026-10-15T12:00:00Z",
          "snippet": "堂本剛が約27年ぶりに映画主演を務めた「まる」が、10月16日にNetflixで配信される。"
        }
      ]
    }
  ]
}
```

---

### 2. AI（LLM）と組み合わせる場合の利用イメージ

人間がニュースサイトを何件も巡回して探す代わりに、**CLIが「ノイズのないクリーンなJSON」を0.5秒で生成し、AIに渡す**ことで、最小限のトークン消費で会話や業務に使える要約・フックを自動生成できます。

```
[Webメディア群 (映画・音楽・マンガ・芸能)]
       │
       ▼ (高速並行フェッチ / 広告・HTMLタグ除去 / 300ms)
  [Zeitgeist CLI]
       │
       ▼ stdout (軽量・構造化JSON)
  [AI / LLM エージェント]
       │
       ▼ 自然言語への意味解釈・要約
  「本日の話題トピック3選 ＆ 会話フック」
```

**AIへのプロンプト連携例**:
> 「以下のJSON（Zeitgeistの出力）から、今日の雑談・コミュニケーションで使える話題を3つピックアップして要約して」

**AIによる出力結果イメージ**:
- **映画/配信**: 「堂本剛主演映画『まる』が本日よりNetflixで配信開始。日常が◯に浸食される奇妙な物語で話題です」
- **音楽**: 「米津玄師が過去最大規模のワールドツアー（全11都市）を発表しました」
- **漫画**: 「本日10/16は『名探偵コナン』単行本109巻の発売日です」

---

## インストールとビルド

```bash
git clone https://github.com/AobaIwaki123/zeitgeist.git
cd zeitgeist

# ビルド
go build -o zeitgeist ./cmd/zeitgeist

# 検証スクリプトの実行（フォーマット・vet・ビルド・スモークテスト）
./scripts/verify-all.sh
```

## オプション一覧

```
  -config string
        設定ファイル（YAML）へのパス (デフォルト: "configs/sources.sample.yaml")
  -category string
        指定したカテゴリのみ取得 (例: "movie", "music", "tech")
  -source string
        指定したソースIDのみ取得 (例: "natalie_eiga")
  -timeout int
        タイムアウト秒数の上書き (0 = 設定ファイルのデフォルトを使用)
  -verbose
        詳細なデバッグログを stderr に出力
```
