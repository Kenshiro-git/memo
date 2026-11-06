# ChatGPT・Claude・Gemini 契約別の利用量・費用集計可否

- 調査基準日：2026-11-06
- 対象：各社の一般向けチャット製品（個人契約／チーム・法人契約）
- 調査方針：公式ドキュメントで明示された機能のみ「可能」とする。確認できない機能は「公式資料で未確認」と記載し、「不可能」と断定しない。
- **注意：ChatGPT／Claude.ai／Gemini アプリの契約と、OpenAI API／Claude API／Gemini API の従量課金契約は別**。本書の主表はチャット製品についての比較である。

## 1. 判定対象と用語

1. **利用者別トークン数**：ユーザー単位で入力・出力トークン数が確認できるか。
2. **利用者別費用**：ユーザー単位で使用量に対応する支出額、または消費クレジットを確認できるか。定額のシート代とトークン換算額は同義ではない。
3. **管理画面／CSV集計**：管理画面で確認できるか、CSVをダウンロードできるか。
4. **API自動集計**：契約の管理者が正式なAPIを通じて定期的・プログラム的に取得できるか。

記号：「○」公式資料で対応確認、「△」一部項目／条件付き、「—」該当機能を公式資料で確認できず。

## 2. チーム・法人契約比較

| 契約 | 利用者別トークン | 利用者別費用・クレジット | 管理画面・CSV | API自動取得 | 条件・制約 |
|---|---|---|---|---|---|
| ChatGPT Business | — | ○ 消費クレジット数（USD換算とは異なる） | ○ Workspace analytics（メンバー表） | —（利用者別の全チャットトークン・金額APIは未確認） | メンバー別メッセージ数・シート種類・消費クレジット数。期間は7日、1か月、6か月、12か月、カスタム。 |
| ChatGPT Enterprise | △ 対象のトークンベース請求契約 | △ 契約の請求方式による | ○ Workspace analytics・対象契約の使用量レポート | △ 用途別APIの公開範囲・契約による。利用者別トークン／費用の一括取得は別途確認 | トークンベース請求は適格なEnterprise契約のみ。請求対象の単位は契約のrate cardに依存。 |
| Claude Team | ○ CSV | ○ CSVのUSD支出額 | ○ CSVエクスポート | —（Claude Team自体のEnterprise Analytics APIは対象外） | 過去90日以内の期間指定。最新データは前日まで。 |
| Claude Enterprise | ○ CSV・利用量レポート | ○ CSV・支出レポート | ○ | ○ Analytics API | Primary OwnerによるAPI有効化とread:analyticsキーが必要。使用量ベース・シートベースの契約で費用レポートの対象が異なる。 |
| Gemini for Google Workspace | — | —（利用者別トークン費用） | ○ 管理コンソールのGemini利用状況レポートとエクスポート | △ Workspaceの利用状況管理は可能だが、利用者別トークン・費用を返す公式APIは未確認 | 利用回数、活動日数、利用レベル等の活動指標とトークン数は異なる。 |

**判定上の注意**：「—」は「絶対にできない」ではなく、今回確認した公式資料に当該取得手段が明示されていないことを表す。

### 2-1. ChatGPT Business

Workspace analytics でワークスペース全体のアクティブユーザー数、メッセージ数、使用クレジット数を表示。メンバーごとのシート種類、クレジット消費数、送信メッセージ数を確認できる。入力／出力トークン総数やUSD建て利用者別の全チャット原価を意味しない。

公式：[ChatGPT Business release notes（2026-04-16 Workspace analytics）](https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes)

### 2-2. ChatGPT Enterprise

適格なEnterprise契約ではChat、Work、Codex等の使用量を契約のレートカードに応じてトークン等の単位で計測し、USDで請求するトークンベース請求がある。この仕組みをすべてのEnterprise契約に一般化しない。Workspace analyticsは生のメッセージログではなく集計レポートであり、CSVエクスポート機能がある。APIによる利用者別費用とトークンの取得は、各APIの提供項目と契約条件を別途精査する必要がある。

公式：
- [Token-based billing for ChatGPT Enterprise](https://help.openai.com/en/articles/20001520-token-based-billing-for-chatgpt-enterprise)
- [Workspace analytics for ChatGPT Enterprise and Edu](https://help.openai.com/en/articles/10875114-workspace-analytics-for-chatgpt-enterprise-and-edu)

### 2-3. Claude Team・Enterprise

組織の利用状況分析からダウンロードするCSVの公式掲載項目：

| CSVフィールド | 内容 |
|---|---|
| User's email / Account UUID | 利用者の識別 |
| Product | Chat / Claude Code / Coworkなど |
| Model / model family | 利用モデル |
| total_requests | リクエスト回数 |
| total_prompt_tokens | 入力トークン数（会話履歴、システム指示等を含む） |
| total_completion_tokens | 出力トークン数（拡張思考トークンを含む） |
| total_net_spend_usd | 割引・クレジット・契約条件適用後の支出額 |
| total_gross_spend_usd | 割引・クレジット適用前の金額 |

**制約**：
- CSVの対象期間は直近90日以内。最新のデータは前日分まで。
- データ更新は日次で、約1日の遅延。
- 使用量ベースのEnterpriseでは組織全体の使用量を対象とする。
- シートベースEnterpriseで使用量クレジットを有効化している場合、費用エクスポートはシート付帯枠を超える支出のみ反映。定額枠内の費用をすべて「支出額」として取得できるわけではない。
- Analytics APIは**Claude Enterprise向け**で、Primary Ownerが有効化して `read:analytics` スコープのキーを作成する。
- APIのコスト／使用量エンドポイントは使用量ベースEnterpriseでは対象。シートベースEnterpriseでは使用量クレジットの対象分に限定される。

公式：
- [View usage analytics for Team and Enterprise plans](https://support.claude.com/en/articles/12883420-view-usage-analytics-for-team-and-enterprise-plans)
- [Claude Analytics APIs](https://platform.claude.com/docs/en/manage-claude/analytics-api)
- [Usage and Cost Analytics API](https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api)

※ 最後の Usage and Cost Analytics API は、**Claude Console（API契約）とClaude Enterpriseの両方**に対応する。Claude ConsoleのデータをClaude Teamのチャット利用データと取り違えない。

### 2-4. Google Workspace版Gemini

Google Workspace管理者はGeminiの組織・利用者別の導入状況や利用状況をレポートできる。利用状況・活動日数等と、モデル別の入力／出力トークン数、USD換算料金の集計は別の指標。今回確認した公式資料では**ユーザー別の全トークン数・相当料金を取得する機能は確認できない**。Gemini APIの請求・ログもWorkspace Geminiの利用記録を直接表すものではない。

公式：[Review Gemini usage in your organization](https://knowledge.workspace.google.com/admin/generative-ai/review-gemini-usage-in-your-organization)

## 3. 個人契約比較

| 契約 | 全チャットの自分の入力・出力トークン合計 | 使用枠・支出確認 | 別ユーザーの利用データを管理者が一括取得 |
|---|---|---|---|
| ChatGPT Free / Go / Plus / Pro | — | △ プラン・機能によって使用枠と購入クレジット残高、最近の利用状況を確認 | — |
| Claude Free / Pro / Max | — | △ 使用制限や追加使用に関する利用状況・支出を確認 | — |
| Gemini 無料 / Google AI各プラン | — | △ 機能・モデルごとの使用制限を確認 | — |

**留意点**：定額契約では、月額契約金と「チャット処理が使ったトークン数から算出した仮想的なAPI料金」は同じではない。個人契約の使用制限・クレジット残高確認が可能でも、すべての会話の入力／出力トークン数が取得できるとは限らない。複数社員の個人契約を管理者が横断的に集計する公式機能も今回の公式資料では確認できない。

参考：
- [OpenAI: Using Credits for Flexible Usage in ChatGPT (Personal plans)](https://help.openai.com/en/articles/12642688-using-credits-for-flexible-usage-in-chatgpt-personal-plans)
- [Google: Gemini apps limits and upgrades](https://support.google.com/gemini/answer/16275805?hl=ja-JP)
- [Anthropic: Claude Help Center](https://support.claude.com/)

## 4. 外部システムでの自動集計とAPI契約の区別

| 対象 | 公式に確認された自動取得 | 注意 |
|---|---|---|
| ChatGPT Business（チャット契約） | 利用者別トークン・USD費用の正式な取得APIは未確認 | Workspace analyticsの表示とAPI提供を混同しない |
| ChatGPT Enterprise（チャット契約） | 分析／コンプライアンス関連APIはあるが、利用者別トークン・費用の組み合わせは契約とAPI仕様に依存 | 導入前に取得フィールド・契約方式を照合 |
| Claude Team（チャット契約） | CSVダウンロードは確認。Enterprise向けAnalytics APIは対象外 | 手動CSVをAPI自動取得と表記しない |
| Claude Enterprise（チャット契約） | ○ Analytics APIで対象データの自動取得 | 料金レポートは使用量ベース／シートベースで対象が異なる |
| Gemini in Workspace（チャット契約） | 管理者向け利用状況レポート。トークン・USD費用APIは未確認 | Workspaceの利用状況とGemini APIを混同しない |
| OpenAI API / Claude Console API / Gemini API（別契約） | API別の使用量／費用機能が別途存在 | これらでチャット個人契約の利用分が自動的に集計されるわけではない |

### Claudeの最新Analytics APIに関する補足（API契約との混同防止）

Claudeの `GET /v1/organizations/analytics/usage` および `/cost` は、Claude Console組織またはClaude Enterprise組織の分析データを取得する**ベータAPI**。Claude ConsoleではAdmin API資格情報、Claude Enterpriseでは `read:analytics` キーが必要。レスポンス更新は通常4時間以内、最大24時間の遅れがあり得る。検索期間には最大365日の遡及や1リクエスト31日といった制約がある。仕様は公式ドキュメントを参照。

公式：[Usage and Cost Analytics API](https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api)

## 5. 事実ベースの結論

1. **Claude Team**：ユーザー別トークン数・支出額をCSVで取得可能。ただし期間90日・前日までの更新・API対象外などの制限がある。
2. **Claude Enterprise**：CSVに加えてAnalytics APIに対応。ただし利用者別の支出対象は契約方式によって変わる。
3. **ChatGPT Business**：利用者別のメッセージ数・消費クレジット数は確認可能。全チャットの利用者別トークン数・USD費用は公式資料で未確認。
4. **ChatGPT Enterprise**：対象契約でトークンベース請求および使用量レポートに対応。すべてのEnterprise契約で同一の機能が使えるわけではない。
5. **Gemini（Google Workspace）**：組織・利用者別の利用状況レポートは提供。利用者別の全トークン数・トークン費用は公式資料で未確認。
6. **3社の個人向けチャット契約**：使用枠や追加使用の情報は一部確認可能。ただし全チャットの正確なトークン数と推定料金、および個人契約を横断した管理者集計は公式資料で未確認。

## 6. 一次資料一覧

- OpenAI Business: https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes
- OpenAI Enterprise請求: https://help.openai.com/en/articles/20001520-token-based-billing-for-chatgpt-enterprise
- OpenAI Enterprise分析: https://help.openai.com/en/articles/10875114-workspace-analytics-for-chatgpt-enterprise-and-edu
- OpenAI 個人: https://help.openai.com/en/articles/12642688-using-credits-for-flexible-usage-in-chatgpt-personal-plans
- Claude Team/Enterprise CSV: https://support.claude.com/en/articles/12883420-view-usage-analytics-for-team-and-enterprise-plans
- Claude Enterprise Analytics API: https://platform.claude.com/docs/en/manage-claude/analytics-api
- Claude Unified Usage/Cost Analytics API: https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api
- Google Workspace Gemini: https://knowledge.workspace.google.com/admin/generative-ai/review-gemini-usage-in-your-organization
- Gemini個人: https://support.google.com/gemini/answer/16275805?hl=ja-JP
