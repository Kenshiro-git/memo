# Google Cloud Vision 同期OCR レイアウト再構築設計書

## 1. 概要

本設計では、**Google Cloud Vision の同期APIで1ページPDFをOCRし、文字単位の座標情報から行と文字列グループを自前で再構築し、最終的にグループ単位の矩形座標を出力する処理**を実装する。

対象となる帳票では、以下のようなレイアウト上の課題がある。

- 「請　求　書」のように、見た目を整えるために文字間隔が広く設定されている文字列がある
- 「株式会社サンプル　〇〇支社」のように、同じ行でも人間から見ると別グループに見える文字列がある
- 「数　量」「単　価」「金　額」と、その下の明細行がVision側で誤って同じ行や同じまとまりとして扱われる場合がある
- Visionの `Word`、`Paragraph`、`Block` の認識結果だけでは、人間が見たレイアウトと一致しない場合がある

そのため、Google Cloud Visionには主に以下を担当させる。

- 文字認識
- 文字単位の座標取得
- 信頼度取得

一方、以下はPython側で再構築する。

- 行判定
- 文字列グループ判定
- グループ単位の矩形座標計算

---

## 2. 基本方針

本処理では、VisionのOCR結果をそのままレイアウト構造として信用しない。

特に重要な設計原則は次の3点である。

### 2.1 VisionのWord・Paragraph・Blockを最終レイアウト判定として信用しない

Visionが返す以下の階層は、OCR結果を取得するために辿るが、最終的な行・グループ判定には使用しない。

```text
Page
 ↓
Block
 ↓
Paragraph
 ↓
Word
 ↓
Symbol
```

最終的に利用する主な情報は `Symbol` 単位の文字と座標である。

### 2.2 Y方向のLine判定をX方向のGroup判定より先に行う

処理順序は必ず次とする。

```text
Symbol
  ↓
Line
  ↓
Group
```

X方向の文字間隔だけで先にグループを作ってはいけない。

### 2.3 異なるLineの文字は絶対に同一Groupへ含めない

一度別行と判定した文字同士は、その後どのような条件が成立しても同一Groupに含めない。

これはHard Constraintとする。

---

## 3. 処理概要

全体フローは以下の通り。

```text
1ページPDF
    ↓
Google Cloud Vision 同期API
DOCUMENT_TEXT_DETECTION
    ↓
fullTextAnnotation
    ↓
Symbol単位に展開
    │
    ├─ text
    ├─ boundingBox
    └─ confidence
    ↓
各Symbolの矩形情報を作成
    ↓
Y座標からLineを再構築
    ↓
異なるLineは完全分離
    ↓
各Line内をX座標順にソート
    ↓
文字間隔・文字サイズ・gapパターンを解析
    ↓
Visual Groupを構築
    ↓
Group内Symbolの外接矩形を計算
    ↓
最終結果

{
  line_id,
  group_id,
  text,
  x,
  y,
  width,
  height
}
```

---

# 4. 入力仕様

## 4.1 入力ファイル

- 形式: PDF
- MIME Type: `application/pdf`
- ページ数: 1ページ
- OCR方式: Google Cloud Vision `DOCUMENT_TEXT_DETECTION`
- API方式: 同期API

Python側では以下のような入力を想定する。

```python
pdf_path = "invoice.pdf"
```

---

# 5. Google Cloud Vision 同期API仕様

## 5.1 API呼び出し

```python
from google.cloud import vision_v1


def call_vision(pdf_path: str):
    client = vision_v1.ImageAnnotatorClient()

    with open(pdf_path, "rb") as f:
        content = f.read()

    request = vision_v1.AnnotateFileRequest(
        input_config=vision_v1.InputConfig(
            content=content,
            mime_type="application/pdf",
        ),
        features=[
            vision_v1.Feature(
                type_=vision_v1.Feature.Type.DOCUMENT_TEXT_DETECTION
            )
        ],
        pages=[1],
    )

    response = client.batch_annotate_files(
        requests=[request]
    )

    return (
        response.responses[0]
        .responses[0]
        .full_text_annotation
    )
```

---

## 5.2 Visionレスポンス概念

概念的には以下のようなレスポンスが返る。

```json
{
  "fullTextAnnotation": {
    "text": "...",
    "pages": [
      {
        "blocks": [
          {
            "paragraphs": [
              {
                "words": [
                  {
                    "symbols": [
                      {
                        "text": "数",
                        "confidence": 0.99,
                        "boundingBox": {
                          "vertices": [
                            {"x": 430, "y": 215},
                            {"x": 450, "y": 215},
                            {"x": 450, "y": 238},
                            {"x": 430, "y": 238}
                          ]
                        }
                      }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

---

## 5.3 利用するVisionフィールド

| Visionフィールド | 使用 | 用途 |
|---|---:|---|
| `Symbol.text` | ◎ | 文字 |
| `Symbol.bounding_box.vertices` | ◎ | 文字位置 |
| `Symbol.confidence` | ○ | OCR信頼度 |
| `Word` | △ | デバッグ・補助情報 |
| `Paragraph` | △ | デバッグ・補助情報 |
| `Block` | △ | デバッグ・補助情報 |
| `detected_break` | △ | 補助判定 |
| `full_text_annotation.text` | △ | OCR全文確認 |

最終レイアウト判定では、`Word`、`Paragraph`、`Block` を信用しすぎない。

---

# 6. 内部データ仕様

VisionのSymbolを以下の形式へ正規化する。

```python
{
    "symbol_id": 1,
    "text": "数",

    "left": 430,
    "top": 215,
    "right": 450,
    "bottom": 238,

    "width": 20,
    "height": 23,

    "center_x": 440.0,
    "center_y": 226.5,

    "confidence": 0.99
}
```

---

## 6.1 BoundingBoxの矩形化

Visionは4頂点のBoundingBoxを返す。

これを以下の形式へ変換する。

```python
def bbox_to_rect(vertices):
    xs = [v.x for v in vertices]
    ys = [v.y for v in vertices]

    left = min(xs)
    right = max(xs)
    top = min(ys)
    bottom = max(ys)

    return {
        "left": left,
        "top": top,
        "right": right,
        "bottom": bottom,
        "width": right - left,
        "height": bottom - top,
        "center_x": (left + right) / 2,
        "center_y": (top + bottom) / 2,
    }
```

---

# 7. Symbol抽出設計

Visionの階層構造は辿るが、最終的にはすべてのSymbolをフラット化する。

```python
def extract_symbols(document):
    symbols = []
    symbol_id = 0

    for page in document.pages:
        for block in page.blocks:
            for paragraph in block.paragraphs:
                for word in paragraph.words:
                    for symbol in word.symbols:
                        symbol_id += 1

                        rect = bbox_to_rect(
                            symbol.bounding_box.vertices
                        )

                        symbols.append({
                            "symbol_id": symbol_id,
                            "text": symbol.text,
                            **rect,
                            "confidence": symbol.confidence,
                        })

    return symbols
```

結果例:

```python
[
    {"text": "数", ...},
    {"text": "量", ...},
    {"text": "単", ...},
    {"text": "価", ...},
    {"text": "1", ...},
]
```

---

# 8. 行判定仕様

## 8.1 方針

文字Aと文字BがY方向に十分重なっている場合のみ、同じ行候補とする。

この段階ではX座標は考慮しない。

---

## 8.2 Vertical Overlap

```python
def vertical_overlap_ratio(a, b):
    overlap = max(
        0,
        min(a["bottom"], b["bottom"])
        - max(a["top"], b["top"])
    )

    min_height = min(
        a["height"],
        b["height"]
    )

    if min_height <= 0:
        return 0.0

    return overlap / min_height
```

### 同一行の例

```text
文字A
top    = 220
bottom = 240

文字B
top    = 222
bottom = 242
```

```text
overlap = 18
height  = 20

18 / 20 = 0.90
```

同じ行と判定できる。

### 別行の例

```text
ヘッダー「数量」
220 ～ 240

下段「12,345,678」
245 ～ 265
```

```text
overlap = 0
```

別行と判定する。

---

# 9. Lineクラスタリング設計

文字対文字だけでなく、Symbolと既存LineのY範囲を比較する。

Lineは以下の情報を持つ。

```python
{
    "line_id": 1,
    "top": 220,
    "bottom": 241,
    "symbols": [...]
}
```

---

## 9.1 SymbolとLineの重複率

```python
LINE_OVERLAP_THRESHOLD = 0.5


def symbol_line_overlap(symbol, line):
    overlap = max(
        0,
        min(symbol["bottom"], line["bottom"])
        - max(symbol["top"], line["top"])
    )

    line_height = line["bottom"] - line["top"]

    denominator = min(
        symbol["height"],
        line_height
    )

    if denominator <= 0:
        return 0.0

    return overlap / denominator
```

---

## 9.2 Line生成

```python
def build_lines(symbols):
    symbols = sorted(
        symbols,
        key=lambda s: (
            s["center_y"],
            s["left"]
        )
    )

    lines = []

    for symbol in symbols:
        best_line = None
        best_score = 0.0

        for line in lines:
            score = symbol_line_overlap(
                symbol,
                line
            )

            if score > best_score:
                best_line = line
                best_score = score

        if (
            best_line is not None
            and best_score >= LINE_OVERLAP_THRESHOLD
        ):
            best_line["symbols"].append(symbol)

            best_line["top"] = min(
                s["top"]
                for s in best_line["symbols"]
            )

            best_line["bottom"] = max(
                s["bottom"]
                for s in best_line["symbols"]
            )

        else:
            lines.append({
                "line_id": len(lines) + 1,
                "top": symbol["top"],
                "bottom": symbol["bottom"],
                "symbols": [symbol],
            })

    lines.sort(
        key=lambda line: line["top"]
    )

    for index, line in enumerate(lines, start=1):
        line["line_id"] = index

        line["symbols"].sort(
            key=lambda s: s["left"]
        )

    return lines
```

---

# 10. Hard Constraint

以下を仕様として固定する。

```text
異なるline_idを持つSymbolは、
いかなる条件でも同一Groupに含めない。
```

このルールにより、以下のような誤結合を防ぐ。

```text
数　量
12,345,678 個数
```

NG:

```text
数量12,345,678個数
```

OK:

```text
Line 1: 数量
Line 2: 12,345,678個数
```

---

# 11. X方向グルーピング仕様

Lineが確定した後、同一Line内だけでX方向のグルーピングを行う。

例:

```text
商品名／品名       数     量        単     価        金     額
```

最終的に以下へ分割する。

```text
[商品名／品名]
[数量]
[単価]
[金額]
```

---

# 12. Normalized Gap

固定pxではなく、文字サイズに対する相対距離を使う。

```python
def normalized_gap(a, b):
    gap = b["left"] - a["right"]

    avg_height = (
        a["height"]
        + b["height"]
    ) / 2

    if avg_height <= 0:
        return 0.0

    return gap / avg_height
```

---

## 12.1 例

文字高さが20pxの場合:

```text
数→量
gap = 18px
normalized = 0.9

量→単
gap = 45px
normalized = 2.25

単→価
gap = 19px
normalized = 0.95
```

```text
0.90   2.25   0.95
       ↑
     境界候補
```

---

# 13. 「請　求　書」のような均等字間への対応

単純に「gapが大きいから分割」としてはいけない。

```text
請       求       書
```

例えば:

```text
請→求 = 0.95
求→書 = 1.01
```

この場合、gapは大きいが均等である。

```text
0.95 ≒ 1.01
```

そのため、意図的なletter spacingとして扱い、以下の1グループにする。

```text
[請求書]
```

---

# 14. 「数量」「単価」の分割

例:

```text
数       量             単       価
```

```text
数→量 = 0.9
量→単 = 2.4
単→価 = 0.95
```

```text
0.9   2.4   0.95
      ↑
局所的に大きいgap
```

この場合:

```text
[数量]
[単価]
```

と分割する。

---

# 15. 境界判定ロジック

絶対gapではなく、前後との差を用いる。

```python
def is_group_boundary(gaps, index):
    current = gaps[index]

    neighbors = []

    if index > 0:
        neighbors.append(gaps[index - 1])

    if index < len(gaps) - 1:
        neighbors.append(gaps[index + 1])

    if not neighbors:
        return False

    neighbor_avg = (
        sum(neighbors)
        / len(neighbors)
    )

    if neighbor_avg <= 0:
        return False

    ratio = current / neighbor_avg

    return ratio >= 2.0
```

例:

```text
0.9  2.4  0.95
```

```text
2.4 / ((0.9 + 0.95) / 2)
≈ 2.59
```

境界候補として扱う。

---

# 16. Group矩形仕様

Groupが確定したら、そのGroupに含まれるSymbolの外接矩形を計算する。

```python
def make_group(
    symbols,
    line_id,
    group_id,
):
    left = min(
        s["left"]
        for s in symbols
    )

    right = max(
        s["right"]
        for s in symbols
    )

    top = min(
        s["top"]
        for s in symbols
    )

    bottom = max(
        s["bottom"]
        for s in symbols
    )

    return {
        "line_id": line_id,
        "group_id": group_id,

        "text": "".join(
            s["text"]
            for s in symbols
        ),

        "x": left,
        "y": top,

        "width": right - left,
        "height": bottom - top,

        "symbols": symbols,
    }
```

---

## 16.1 「数量」の例

```text
数
x = 430 ～ 450

量
x = 470 ～ 490
```

Group:

```text
left   = 430
right  = 490
top    = 215
bottom = 238
```

結果:

```json
{
  "text": "数量",
  "x": 430,
  "y": 215,
  "width": 60,
  "height": 23
}
```

---

# 17. 最終レスポンス仕様

推奨フォーマット:

```json
{
  "page": 1,
  "width": 1024,
  "height": 768,
  "lines": [
    {
      "line_id": 10,
      "top": 215,
      "bottom": 239,
      "groups": [
        {
          "group_id": 1,
          "text": "商品名／品名",
          "x": 150,
          "y": 215,
          "width": 200,
          "height": 24
        },
        {
          "group_id": 2,
          "text": "数量",
          "x": 430,
          "y": 215,
          "width": 60,
          "height": 23
        },
        {
          "group_id": 3,
          "text": "単価",
          "x": 535,
          "y": 215,
          "width": 60,
          "height": 23
        },
        {
          "group_id": 4,
          "text": "金額",
          "x": 630,
          "y": 215,
          "width": 65,
          "height": 23
        }
      ]
    }
  ]
}
```

---

# 18. コード構成設計

推奨Python構成:

```text
ocr/
├── __init__.py
│
├── vision_client.py
│   └── Google Vision同期API
│
├── symbol_extractor.py
│   └── Visionレスポンス → Symbol
│
├── geometry.py
│   ├── bbox_to_rect()
│   ├── vertical_overlap_ratio()
│   └── normalized_gap()
│
├── line_detector.py
│   └── Symbol → Line
│
├── group_detector.py
│   └── Line → Group
│
├── models.py
│   ├── Symbol
│   ├── Line
│   └── TextGroup
│
├── processor.py
│   └── 全体制御
│
└── config.py
    └── 閾値管理
```

---

# 19. データクラス設計

```python
from dataclasses import dataclass, field


@dataclass
class Symbol:
    symbol_id: int
    text: str

    left: float
    top: float
    right: float
    bottom: float

    width: float
    height: float

    center_x: float
    center_y: float

    confidence: float


@dataclass
class TextGroup:
    group_id: int
    line_id: int

    text: str

    x: float
    y: float
    width: float
    height: float

    symbols: list[Symbol] = field(
        default_factory=list
    )


@dataclass
class Line:
    line_id: int

    top: float
    bottom: float

    symbols: list[Symbol] = field(
        default_factory=list
    )

    groups: list[TextGroup] = field(
        default_factory=list
    )
```

---

# 20. 設定値設計

閾値はコードに直書きせず設定クラスで管理する。

```python
from dataclasses import dataclass


@dataclass
class OcrGroupingConfig:
    line_overlap_threshold: float = 0.5
    local_gap_ratio_threshold: float = 2.0
    min_confidence: float = 0.0
```

利用例:

```python
config = OcrGroupingConfig(
    line_overlap_threshold=0.5,
    local_gap_ratio_threshold=2.0,
)
```

---

# 21. Group Detector設計

```python
def build_groups(
    line,
    config,
):
    symbols = sorted(
        line.symbols,
        key=lambda s: s.left
    )

    if not symbols:
        return []

    if len(symbols) == 1:
        return [
            create_group(
                [symbols[0]],
                line.line_id,
                1
            )
        ]

    gaps = []

    for a, b in zip(
        symbols,
        symbols[1:]
    ):
        gap = (
            b.left - a.right
        )

        avg_height = (
            a.height + b.height
        ) / 2

        normalized = (
            gap / avg_height
            if avg_height > 0
            else 0
        )

        gaps.append(normalized)

    groups = []
    current = [symbols[0]]

    group_id = 1

    for index, gap in enumerate(gaps):

        boundary = is_group_boundary(
            gaps,
            index,
            config
        )

        next_symbol = symbols[index + 1]

        if boundary:
            groups.append(
                create_group(
                    current,
                    line.line_id,
                    group_id,
                )
            )

            group_id += 1

            current = [
                next_symbol
            ]

        else:
            current.append(
                next_symbol
            )

    if current:
        groups.append(
            create_group(
                current,
                line.line_id,
                group_id,
            )
        )

    return groups
```

---

# 22. Processor設計

外部からは内部ロジックを意識せず呼び出せるようにする。

```python
class OcrLayoutProcessor:

    def __init__(
        self,
        vision_client,
        config,
    ):
        self.vision_client = vision_client
        self.config = config

    def process(self, pdf_path: str):
        document = (
            self.vision_client
            .detect_pdf(pdf_path)
        )

        symbols = extract_symbols(
            document
        )

        lines = build_lines(
            symbols,
            self.config
        )

        for line in lines:
            line.groups = build_groups(
                line,
                self.config
            )

        return lines
```

利用例:

```python
processor = OcrLayoutProcessor(
    vision_client,
    config,
)

result = processor.process(
    "invoice.pdf"
)
```

---

# 23. API化する場合

FastAPI等で提供する場合:

```text
POST /ocr/layout
```

処理:

```text
PDF
 ↓
Vision
 ↓
Symbol抽出
 ↓
Line再構築
 ↓
Group再構築
 ↓
JSON
```

レスポンス例:

```json
{
  "page": 1,
  "lines": [
    {
      "line_id": 1,
      "groups": [
        {
          "text": "請求書",
          "x": 120,
          "y": 30,
          "width": 170,
          "height": 35
        }
      ]
    }
  ]
}
```

---

# 24. テスト仕様

最低限、以下のパターンを確認する。

| パターン | 入力 | 期待結果 |
|---|---|---|
| 普通の文字 | `請求書` | `[請求書]` |
| 均等字間 | `請　求　書` | `[請求書]` |
| グループ分離 | `数量　　単価` | `[数量][単価]` |
| 別行 | `数量` / `10` | 必ず別Line |
| 大きな空白 | `株式会社サンプル　〇〇支社` | 2Group |
| 数値 | `123,456,780` | 1Group |
| 記号混在 | `商品名／品名` | 1Group |
| 小文字・英数字混在 | `No. 12345` | 設計方針に従ってGroup化 |

---

## 24.1 重要な回帰テスト

以下が同じLineへ入らないことを必ず確認する。

```text
数　量
────────
12,345,678 個数
```

また、以下は1Groupになることを確認する。

```text
請　求　書
```

期待値:

```text
請求書
```

---

# 25. ログ仕様

判定理由を追跡できるようにログを残す。

例:

```text
[LINE]
symbol=数
y=215-238
assigned_line=10
overlap=0.93

[GROUP]
left=数
right=量
normalized_gap=0.91
boundary=False

[GROUP]
left=量
right=単
normalized_gap=2.34
neighbor_ratio=2.48
boundary=True
```

ログにより、なぜグループが分割されたかを追跡できる。

---

# 26. VisionとPythonの責任分担

## Google Cloud Vision

```text
文字を読む
  ↓
Symbol.text

位置を取得
  ↓
Symbol.boundingBox

信頼度を取得
  ↓
Symbol.confidence
```

## Python

```text
どの行か？
  ↓
Line

どの文字と同じ塊か？
  ↓
Group

塊はどこにあるか？
  ↓
x / y / width / height
```

つまり、

```text
Vision = OCR + Geometry取得
Python = Layout再構築
```

という責任分離とする。

---

# 27. 今後の拡張候補

本設計をベースに、以下の改善を追加できる。

- 文字の回転・傾きへの対応
- 罫線位置の検出
- 表構造の復元
- 列境界の推定
- `detected_break` を補助情報として利用
- OCR confidenceが低い文字の再判定
- 帳票テンプレートごとの閾値最適化
- DBSCANなどを利用した座標クラスタリング
- 文字列辞書を利用した補助判定
- OpenAI等による意味的グループ判定の補助
- PDF上へのハイライト描画
- グループ単位の座標を後続APIへ連携

---

# 28. まとめ

本処理では、Google Cloud Visionのレイアウト認識をそのまま採用せず、`Symbol.text` と `Symbol.bounding_box.vertices` を中心に使用する。

処理順序は必ず次とする。

```text
PDF
 ↓
Vision同期OCR
 ↓
Symbol
 ↓
Line
 ↓
Group
 ↓
Group矩形
 ↓
JSON
```

特に以下を重要仕様とする。

1. VisionのWord・Paragraph・Blockは最終判定として信用しない
2. Y方向のLine判定を必ず先に実施する
3. 異なるLineのSymbolは絶対に同一Groupへ結合しない
4. 同一Line内のみでX方向のグループ判定を実施する
5. 固定pxではなく、文字サイズに対する相対gapを使う
6. 「請　求　書」のような均等字間は1Groupとして扱えるようにする
7. 「数量」「単価」のように局所的にgapが大きい場合は別Groupとする
8. Group確定後、所属Symbolの外接矩形から `x / y / width / height` を作成する

この構成により、Visionの誤ったWord・Paragraph・行認識の影響を減らしつつ、人間が見た帳票レイアウトに近い文字列グループと座標情報を生成できる。
