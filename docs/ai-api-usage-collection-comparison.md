# OpenAI・Claude・Gemini API 利用量自動収集の比較

- 調査日：2026年11月6日
- 対象：API管理者が、既存のAPIキーを使った利用履歴を後から取得する運用
- 想定構成：管理者がGAS（Google Apps Script）で収集し、Googleスプレッドシートに保存
- 対象外：社員の個別アプリに、各リクエスト時の独自ログ保存処理を追加する方式

## 1. 結論

**OpenAI APIとClaude APIは、管理者向けの公式APIで複数APIキーの利用実績を事後集計できる。Gemini API（Google AI Studio／Gemini Developer API）は、Google Cloud側のAPIキー一覧やプロジェクト単位のメトリクス・課金情報は取得できる一方、既存の複数キーごとのトークン数・料金をOpenAIと同じ粒度で一括取得できる公式管理APIは確認できない。**

「取得できない」は、Googleの全サービス・全構成で技術的に不可能との断定ではなく、今回の対象である既存のGemini Developer APIキーに対し、公式管理APIでキー別トークン数・費用を事後取得する標準手段が確認できないという意味。

## 2. 3社の自動収集機能比較

| 比較項目 | OpenAI API | Claude API（Claude Console） | Gemini API（Google AI Studio） |
| --- | --- | --- | --- |
| GASから管理API等を呼び出す | 可能 | 可能 | 一部可能 |
| 既存の複数APIキーの利用履歴を管理者側で一括収集 | 可能 | 可能 | キー別実績の一括取得は未確認 |
| APIキー別入力・出力トークン数 | 可能（対応するUsage API） | 可能（Analytics Usage） | 標準の管理APIでの取得は未確認 |
| APIキー別リクエスト回数 | 可能（対応するUsage API） | 可能（Analytics Usage） | 標準の管理APIでの取得は未確認 |
| APIキー別の実費 | 可能（Costs API、期間などの制限あり） | 可能（Analytics Cost、適用条件あり） | 標準の管理APIでの取得は未確認 |
| モデル別集計 | 可能 | 可能 | Cloud Monitoringで対応メトリクスのモデルラベルを利用可能 |
| プロジェクト／ワークスペース別集計 | プロジェクト別 | ワークスペース別 | Google Cloudプロジェクト別 |
| 過去履歴の取得 | 取得可能な履歴・集計条件に依存 | Analytics APIの期間制限に依存 | Monitoringの保存期間・事前のBillingエクスポート設定等に依存 |
| 既存の呼び出し元アプリを変更しない運用 | 可能 | 可能 | プロジェクト単位の監視は可能だが、キー単位の詳細集計は保証できない |

**注意：** 「GASから実行できる」と「APIキー別に正確なトークン数・料金が返る」は別の要件。GASはHTTPリクエストの実行基盤であり、取得可能な粒度は提供元のAPI・メトリクス仕様に依存する。

### 2.1 OpenAI

- Usage API例：`GET https://api.openai.com/v1/organization/usage/completions`
- Costs API：`GET https://api.openai.com/v1/organization/costs`
- 管理用Admin APIキーで認証し、`group_by=api_key_id`、`group_by=project_id` 等で集計する。
- Completions Usageでは、入力・出力・キャッシュトークン、リクエスト数、モデルなどを取得できる。
- 画像生成、Embeddings、音声等は対応する別のUsage APIを確認・追加する。
- **UsageとCostsは集計軸が異なる**ため、モデル別のUsage行にキー別料金を単純結合すると二重計上になり得る。
- APIキー別のCosts集計には取得開始日の制限が適用される場合がある。実際に「`group_by=api_key_id is not available for this time range`」が返る場合、エラーで示された開始日以降に変更するか、キー別集計を外して取得する。取得可能範囲は組織・仕様による。

### 2.2 Claude

- 推奨：**Usage and Cost Analytics API（ベータ版）**
- Usage：`GET https://api.anthropic.com/v1/organizations/analytics/usage`
- Cost：`GET https://api.anthropic.com/v1/organizations/analytics/cost`
- Claude Console組織ではAdmin APIキー等の管理用認証情報を使う。Claude Enterprise組織はAnalytics API用認証など条件が異なる。
- `api_key_id`、モデル、ワークスペース等の単位でトークン数・リクエスト数・費用を集計できる。
- リクエストにはベータヘッダー `anthropic-beta: analytics-usage-cost-2026-08-24` と `anthropic-version` が必要。
- **期間制限：** 2026年1月1日以降かつ直近365日以内、1回の要求期間は最大31日。1年分取得時には期間分割とページネーションが必要。
- キーに帰属しない利用は`api_key_id=null`になり得る。Claude EnterpriseなどではAPIキーIDの次元が適用されないケースもある。
- Cost APIの単位（最小通貨単位とUSD換算）を仕様に従い確認してから保存する。

## 3. Gemini：管理者側から取得できる情報

対象は **Google AI Studioで発行・管理するGemini Developer APIキー**。Vertex AI経由のGemini利用は認証・監視・課金体系が異なるため、別途評価が必要。

| 管理者が取得したい情報 | 判定 | 代表的な取得手段／注意点 |
| --- | --- | --- |
| Google Cloudプロジェクト内のAPIキー一覧 | 可能 | API Keys API `projects.locations.keys.list`（権限要） |
| APIキーの名前・ID・作成日時・所属プロジェクト等 | 可能 | API Keys APIのキー情報取得 |
| プロジェクト単位の利用メトリクス | 条件付きで可能 | Cloud MonitoringのGemini API関連メトリクス。利用するモデル・提供段階・メトリクスの有無に依存 |
| モデル別のトークン数 | 条件付きで可能 | 例：`generativelanguage.googleapis.com/generate_content_usage_output_token_count` のモデルラベル |
| プロジェクト別課金額 | 条件付きで可能 | Cloud Billing／BigQueryエクスポート。課金設定・SKU等を確認 |
| APIキー別のリクエスト回数 | 公式の汎用的な取得手段を確認できず | 対象メトリクスにキー識別ラベルが存在するか要検証。プロジェクト別値をキー別値と同一視しない |
| APIキー別の入力・出力トークン数 | 公式の汎用的な取得手段を確認できず | Cloud MonitoringにトークンメトリクスがあってもキーIDで分割できるとは限らない |
| APIキー別の実際の料金 | 公式の汎用的な取得手段を確認できず | Cloud Billingのプロジェクト／SKU粒度とAPIキー粒度は別 |
| 管理者による既存利用履歴の一括バックフィル | キー単位では保証できず | 保存期間・事前設定・APIの仕様に依存 |

### 3.1 取得できるものとできないものの区別

1. **API Keys APIはキーの管理情報を返すAPI**であり、各キーが消費したトークン数・請求料金を返すUsage APIではない。
2. **Cloud Monitoringはプロジェクト等の監視メトリクスを取得する仕組み**。Gemini APIのトークン数関連メトリクスはあるが、すべてのメトリクスにAPIキーIDのラベルがあるとは限らない。取得したメトリクスの定義とラベルを確認する。
3. **Cloud Billingはプロジェクト／サービス／SKU等に基づく課金データ**。APIキーID別料金を直接返す仕組みとは限らない。BigQueryエクスポートは設定前の全履歴が必ずバックフィルされるわけではない。
4. **Google AI Studioの画面に表示できる情報が、そのまま管理用REST APIで取得できるとは限らない。**

### 3.2 管理者側だけでの運用案

**案A：プロジェクト別集計で妥協する（最も現実的）**

- 社員・用途・アプリ単位にGoogle Cloudプロジェクトを分けて利用する。
- 管理者がプロジェクト別のMonitoring／Billingを収集し、プロジェクトIDと社内管理台帳を突き合わせる。
- **1プロジェクト=1キー=単一の利用元**を徹底できる場合、プロジェクトの利用量をそのキーの利用量の近似値として扱える。ただし他のキー・認証方式・サービスからの消費が混在した場合は成立しない。
- プロジェクトが増えるため運用負担が大きく、キー別の実測値を保証する方式ではない。

**案B：APIキー別の正確な集計を必須にする**

- 呼び出し元アプリでレスポンスの`usageMetadata`をキーIDとともに記録するか、共通プロキシで計測する。
- これは「既存アプリ無改修で管理者が後から集計」という今回の要件を満たさないため、別案として扱う。

## 4. GAS一元管理への適用判断

- **OpenAI：採用可能。** Admin APIキーをGASのスクリプトプロパティ等で安全に保持し、Usage／Costsの集計を取得する。
- **Claude：採用可能。** Anthropic側の管理用認証情報とベータヘッダーを用意し、期間分割・ページネーションに対応する。
- **Gemini：条件付き。** 既存キー別の使用量・費用を管理APIから後追いで取得するという要件は、確認できた公式情報では満たすと断言できない。まずプロジェクト別データの取得可否とキー別メトリクスのラベル有無を実環境で検証する。

### 推奨する共通データ形式

`date_utc, provider, api_key_id, project_or_workspace_id, model, requests, input_tokens, output_tokens, cached_tokens, cost_amount, currency, aggregation_level, source_api`

- `aggregation_level`（APIキー別／プロジェクト別／組織全体）を必須にし、異なる集計粒度を混同しない。
- キー別に帰属できないものは空欄として残し、推定値を確定実績として扱わない。
- Usage行とCost行の集計軸が異なるときは、データ種別または指標名で分離して料金の二重計上を防ぐ。
- 管理用キーは利用者に配布せず、GASのサーバー側で保管し、Webアプリは原則「自分のみ」アクセスとする。

## 5. 公式一次資料

### OpenAI
1. [Organization Usage API](https://developers.openai.com/api/reference/resources/admin/subresources/organization/subresources/usage)
2. [Costs API](https://developers.openai.com/api/reference/resources/admin/subresources/organization/subresources/usage/methods/costs)

### Anthropic Claude
3. [Usage and Cost Analytics API（推奨・ベータ版）](https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api)
4. [Usage and Cost API（従来版）](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)
5. [Admin API](https://platform.claude.com/docs/en/manage-claude/admin-api)

### Google Gemini / Google Cloud
6. [Gemini API 課金・利用量](https://ai.google.dev/gemini-api/docs/billing)
7. [API Keys API：キーの一覧と情報取得](https://docs.cloud.google.com/api-keys/docs/get-info-api-keys)
8. [Google Cloud Monitoring：Gemini APIのメトリクス](https://docs.cloud.google.com/monitoring/api/metrics_gcp_d_h)
9. [Cloud Billing：BigQueryへの料金エクスポート](https://docs.cloud.google.com/billing/docs/how-to/export-data-bigquery-setup)

---
本資料は2026年11月6日時点の公式仕様に基づく。APIの提供段階、取得可能期間、利用可能な集計次元は変更され得るため、実装時に公式リファレンスと実環境で再確認すること。
