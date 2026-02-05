---
search:
  exclude: true
---
# ラボ BAF1 - はじめてのエージェントの構築と実行

このラボでは、 **Microsoft 365 Agents SDK** と **Agent Framework** を使用してカスタムのエンジン エージェントを構築し、実行します。スタータープロジェクトを探索し、コアコンポーネントを理解し、エージェントが Microsoft 365 Copilot で動作する様子を確認します。

Zava Insurance Agent は、保険査定担当者が請求処理を効率化するために設計されています。この最初のラボでは、ユーザーに挨拶し、AI を活用した回答を提供できる基本的な会話エージェントから始めます。

???+ info "Microsoft 365 Agents SDK と Agent Framework とは？"
    **Microsoft 365 Agents SDK** は、Microsoft 365 の各チャネル (Teams、Copilot など) にエージェントをデプロイするためのコンテナーとスキャフォールディングを提供し、アクティビティ、イベント、通信を処理します。AI 非依存で、任意の AI サービスを利用できます。
    
    **Agent Framework** は、LLM、ツール呼び出し、マルチエージェント ワークフローを使用して AI エージェントを構築するためのオープンソース開発キットです。Semantic Kernel と AutoGen の後継であり、AI 機能とエージェント ロジックを提供します。
    
    これらを組み合わせることで、Agent Framework でインテリジェントなエージェントを構築し、Agents SDK を使って Microsoft 365 にデプロイできます。

## 演習 1: プロジェクトのクローンと探索

この演習では、Copilot Camp リポジトリをクローンし、スタータープロジェクトの構成を確認してエージェントがどのように整理されているかを理解します。

### 手順 1: リポジトリのクローン

まず Copilot Camp リポジトリをクローンし、Agent Framework のスタータープロジェクトに移動します。

1️⃣ ターミナルまたはコマンド プロンプトを開きます。

2️⃣ リポジトリをクローンします:

```bash
git clone https://github.com/microsoft/copilot-camp.git
cd copilot-camp/src/agent-framework/begin
```

3️⃣ Visual Studio Code でプロジェクトを開きます:

```bash
code .
```

<cc-end-step lab="baf1" exercise="1" step="1" />

### 手順 2: プロジェクト構成の確認

エージェント プロジェクトの構成を理解しましょう。

1️⃣ Visual Studio Code で Explorer ビューのフォルダーを展開します。以下のような構成が表示されるはずです:

```
begin/
├── src/
│   ├── Agent/
│   │   └── ZavaInsuranceAgent.cs       # Main agent implementation
│   ├── Plugins/                        # Custom plugins (tools) for the agent
│   │   ├── StartConversationPlugin.cs  # Welcome message plugin
│   │   └── DateTimeFunctionTool.cs     # Date/time utility
├── appPackage/                         # Teams app manifest and icons
├── env/                                # Environment configuration files (API keys, endpoints)
├── infra/                              # All required scripts, data and templates for the agent's infrastructure
├── Program.cs                          # Application entry point - configures services and starts web app
├── InsuranceAgent.csproj               # Project file
└── m365agents.local.yml                # M365 Agents provisioning config
```

<cc-end-step lab="baf1" exercise="1" step="2" />

### 手順 3: エージェント実装の理解

メインのエージェント ファイルを確認し、動作を把握します。

1️⃣ `src/Agent/ZavaInsuranceAgent.cs` を Visual Studio Code で開きます。

2️⃣ クラス冒頭付近にある `AgentInstructions` プロパティを探します。これらの指示が AI モデルの **システムプロンプト** として機能していることに注目してください。

- エージェントの役割を定義しています: "You are a professional insurance claims assistant for Zava Insurance..."
- `{{PluginName.FunctionName}}` 構文を使用して利用可能なツールを列挙しています
- `{{StartConversationPlugin.StartConversation}}` と `{{DateTimeFunctionTool.getDate}}` を含んでいます

これらの指示により、AI に行動指針と使用可能なツールが伝えられます。

3️⃣ 下にスクロールして **コンストラクター** メソッド `ZavaInsuranceAgent(...)` を見つけます。イベント ハンドラーが設定されていることを確認します。

- `OnConversationUpdate(ConversationUpdateEvents.MembersAdded, WelcomeMessageAsync)` - ユーザーが参加した際にウェルカム メッセージを送信
- `OnActivity(ActivityTypes.Message, OnMessageAsync)` - 受信メッセージを処理

4️⃣ `GetClientAgent` メソッドを探します。`toolOptions` を作成し、プラグインを登録している箇所を確認します。

- `ChatOptions` オブジェクトを作成し、`Tools` リストを設定
- `AIFunctionFactory.Create` を用いて `startConversationPlugin.StartConversation` を追加
- 同様に `DateTimeFunctionTool.getDate` を追加

ここで、会話中に AI が呼び出す **プラグイン** (ツール) を登録しています。

<cc-end-step lab="baf1" exercise="1" step="3" />

### 手順 4: プラグインの確認

プラグインがどのように機能するか見てみましょう。

1️⃣ `src/Plugins/StartConversationPlugin.cs` を開きます。

2️⃣ プラグインの構造を確認します:

```csharp
public class StartConversationPlugin
{
    [Description("Starts a new conversation suggesting a conversation flow.")]
    public async Task<string> StartConversation()
    {
        var welcomeMessage = "👋 Welcome to Zava Insurance Claims Assistant!...";
        return welcomeMessage;
    }
}
```

注目ポイント:

- `[Description]` 属性は AI に **いつこのツールを使用するか** を伝えます
- メソッドはフォーマット済みのウェルカム メッセージを返します
- 引数のないシンプルなプラグインです

3️⃣ `src/Plugins/DateTimeFunctionTool.cs` を開きます。

4️⃣ 現在の日付と時刻を提供する方法を確認します。

- `[Description]` に "Gets the current date and time" と記載
- `getDate()` メソッドは static で、`DateTime.Now` をフォーマットして返します

このプラグインは、エージェントがシステム情報へアクセスしてユーザーの質問に答える方法を示しています。

<cc-end-step lab="baf1" exercise="1" step="4" />

### 手順 5: アプリ マニフェストと会話スターターの確認

アプリ マニフェストを確認し、Microsoft 365 Copilot でエージェントがどのように表示されるか見てみましょう。

1️⃣ `appPackage/manifest.json` を開きます。

2️⃣ `name` セクションでエージェントの表示名を確認します。

```json
"name": {
    "short": "Zava Insurance Agent",
    "full": "Zava Insurance Claims Assistant"
}
```

3️⃣ `conversationStarters` 配列までスクロールします。これは、ユーザーがエージェントと最初に対話するときに表示される推奨プロンプトです。

```json
"conversationStarters": [
    {
        "title": "Instructions",
        "description": "What can you do?"
    },
    {
        "title": "Today's Date",
        "description": "What's today's date?"
    },
    {
        "title": "About Insurance",
        "description": "Tell me about insurance claims"
    },
    {
        "title": "Claims Process",
        "description": "Explain how claims processing works"
    }
]
```

これらの会話スターターは、ユーザーがエージェントとどのように対話すればよいかをガイドします。エージェントの機能に合わせてカスタマイズできます。

4️⃣ `copilotAgents.declarativeAgent` セクションでは、カスタム エンジン エージェントとしての機能が定義されています。

<cc-end-step lab="baf1" exercise="1" step="5" />

### 手順 6: アプリケーション エントリ ポイントの確認

Program.cs で全体がどのように組み合わさっているかを見てみましょう。

1️⃣ `Program.cs` を開きます。

2️⃣ 理解しておくべき主なセクション:

**構成の読み込み**: `builder.Configuration` が設定を読み込む箇所を探します。複数のソースから読み込んでいることに注目してください。

- `.env` ファイル (環境別設定) - `AddEnvFile`
- 機密データ (API キーなど) のための User secrets - `AddUserSecrets`
- 環境変数 - `AddEnvironmentVariables`

**サービス登録**: `builder.Services` にサービスを登録している箇所を確認します。

- `AddSingleton<IStorage, MemoryStorage>()` - 会話状態用にメモリ ストレージを登録
- `AddAgentApplicationOptions()` - エージェント構成を登録
- `AddAgent<ZavaInsuranceAgent>()` - エージェント本体をサービスとして登録

**チャット クライアント構成**: `IChatClient` がシングルトンとして登録される箇所を確認します。以下のように動作します。

- エンドポイント、API キー、デプロイ名を設定から取得
- `AzureOpenAIClient` をエンドポイントと資格情報で作成
- 指定されたデプロイメント (gpt-4.1) 用のチャット クライアントを返却

これにより、エージェントの AI 機能を支える Azure OpenAI との接続が確立されます。

<cc-end-step lab="baf1" exercise="1" step="6" />

## 演習 2: エージェントの構成

エージェントを実行する前に、Azure AI の資格情報を設定する必要があります。

### 手順 1: 環境ファイルの設定

エージェントは環境ファイルを使用して構成を管理します。設定を行いましょう。

1️⃣ Visual Studio Code で `env/` フォルダーに移動します。

2️⃣ 2 つのサンプル ファイルがあることを確認します。

- `.env.local.sample`
- `.env.local.user.sample`

3️⃣ `.env.local.sample` を `.env.local` にコピーします。

**Windows PowerShell:**
```powershell
Copy-Item env/.env.local.sample env/.env.local
```

**macOS/Linux:**
```bash
cp env/.env.local.sample env/.env.local
```

4️⃣ `.env.local.user.sample` を `.env.local.user` にコピーします。

**Windows PowerShell:**
```powershell
Copy-Item env/.env.local.user.sample env/.env.local.user
```

**macOS/Linux:**
```bash
cp env/.env.local.user.sample env/.env.local.user
```

<cc-end-step lab="baf1" exercise="2" step="1" />

### 手順 2: Azure AI 資格情報の追加

次に、Microsoft Foundry のデプロイメントを使用するようエージェントを構成します。

1️⃣ `env/.env.local` を Visual Studio Code で開きます。

2️⃣ `MODELS_ENDPOINT` 変数を見つけ、Lab BAF0 で取得した Azure AI エンドポイントに更新します。

```bash
MODELS_ENDPOINT=https://your-resource.services.ai.azure.com/
```

!!! tip "エンドポイントの確認方法"
    エンドポイントがわからない場合は、以下の手順で確認できます。

    1. [Microsoft Foundry](https://ai.azure.com) にアクセス
    2. プロジェクトを選択
    3. **Settings** → **Properties** を開く
    4. **Endpoint** の URL をコピー

3️⃣ `env/.env.local.user` を Visual Studio Code で開きます。

4️⃣ `SECRET_MODELS_API_KEY` 変数を見つけ、API キーを入力します。

```bash
SECRET_MODELS_API_KEY=your-api-key-here
```

!!! warning "API キーの取り扱いに注意"
    `.env.local.user` ファイルには機密情報が含まれており、既に `.gitignore` に追加されています。このファイルをソース管理にコミットしないでください。

<cc-end-step lab="baf1" exercise="2" step="2" />

### 手順 3: Microsoft 365 と Azure へのサインイン

Microsoft 365 Agents Toolkit は、Microsoft 365 と Azure の両方への認証が必要です。

1️⃣ Visual Studio Code で、アクティビティ バー (左側) の **Microsoft 365 Agents Toolkit** アイコンをクリックします。

2️⃣ ツールキット パネルで **ACCOUNTS** セクションを探します。

3️⃣ **Sign in to Microsoft 365** をクリックし、サインイン フローを完了します。

4️⃣ **Sign in to Azure** をクリックし、サインイン フローを完了します。

!!! note "初回サインイン"
    初回サインイン時には、Microsoft 365 Agents Toolkit 拡張機能に権限を付与する必要があります。

<cc-end-step lab="baf1" exercise="2" step="3" />

## 演習 3: エージェントの実行とテスト

いよいよエージェントを実行し、その動きを確認しましょう！

### 手順 1: エージェントの起動

F5 デバッグ機能を使ってエージェントを実行します。

1️⃣ Visual Studio Code で既定のデバッグ構成 `(Preview) Debug in Copilot (Edge)` を選択し、 **F5** を押すか、メニューの **Run → Start Debugging** を選択します。

2️⃣ デバッグ ターゲットを選択するよう求められたら、 **(Preview) Debug in Copilot (Edge)** を選択します。

!!! tip "デバッグ ターゲットの選択肢"
    "Debug in Teams (Edge)" や "Debug in Teams (Chrome)" など複数のオプションが表示される場合があります。Microsoft 365 Copilot でテストするには、必ず **(Preview) Debug in Copilot (Edge)** を選択してください。

3️⃣ エージェントを初めて実行する際、Microsoft 365 Agents Toolkit は次の操作を行います。

- **Azure subscription** を選択するように求められます
- 新しい **resource group** を作成するか既存のものを選択します
- リソースの **region** を選択します (Microsoft Foundry プロジェクトに近いリージョンを推奨)
- Azure リソース (Azure Bot Service、App Registration) をプロビジョニングします

このプロビジョニング プロセスには通常 2～3 分かかります。

!!! tip "Azure リソースのプロビジョニング"
    初回実行時、ツールキットは以下を作成します。

    - **Azure Bot Service** - メッセージのルーティングを処理
    - **App Registration** - 認証を管理
    - **Dev Tunnel** - ローカルマシンへの安全なトンネルを作成

4️⃣ Visual Studio Code の **Terminal** 出力を確認します。次のようなログが表示されるはずです。

```
🌍 Environment: local
🏢 Starting Zava Insurance Agent...
🤖 Main agent using model: gpt-4.1
✅ Agent initialized successfully!
```

5️⃣ Microsoft 365 Copilot がブラウザーで開きます。

6️⃣ Zava Insurance Agent の **インストール ダイアログ** が表示されます。 **Add** をクリックします。

7️⃣ インストール後、 **Open in Copilot** または **Chat** をクリックします。

<cc-end-step lab="baf1" exercise="3" step="1" />

### 手順 2: 基本的な会話のテスト

エージェントと対話してみましょう！

1️⃣ Microsoft 365 Copilot で、チャット ウィンドウにエージェントと会話スターターが表示されているはずです。

![Conversation starters in Microsoft 365 Copilot](../../../assets/images/agent-framework/BAF1-test1.png)

2️⃣ 「What can you do?」を選択してウェルカム メッセージを表示します。

![Conversation starters in Microsoft 365 Copilot](../../../assets/images/agent-framework/BAF1-test2.png)

3️⃣ **"What's today's date?"** と尋ねてみましょう。

エージェントは `DateTimeFunctionTool` を呼び出し、現在の日付と時刻を返すはずです。

4️⃣ **"What can you do?"** または **"Start over"** と尋ねてみましょう。

エージェントは `StartConversationPlugin` を呼び出し、再度ウェルカム メッセージを表示します。

5️⃣ 一般的な質問を試してください: **"Tell me about insurance claims"**

エージェントは保険金請求について有用な説明を提供するはずです。

6️⃣ 範囲外の質問を試してください: **"What's the weather today?"**

エージェントは保険アシスタントとしての範囲外である旨を丁寧に伝えるはずです。

<cc-end-step lab="baf1" exercise="3" step="2" />

### 手順 3: デバッグ出力の確認

1️⃣ Visual Studio Code に戻り、 **Debug Console** を確認します。

2️⃣ プラグイン呼び出し、AI 応答、メッセージ処理がリアルタイムでログに表示されることを確認します。

<cc-end-step lab="baf1" exercise="3" step="3" />

## 演習 4: エージェントのカスタマイズ

エージェントを簡単に変更して、パーソナライズしてみましょう。

### 手順 1: ウェルカム メッセージの更新

1️⃣ デバッガーを停止します (Shift+F5)。

2️⃣ `src/Plugins/StartConversationPlugin.cs` を開き、`welcomeMessage` 変数を探します。

3️⃣ 1 行目を変更して自分の名前を追加します。例: `"👋 Welcome! I'm [Your Name]'s Agent!\n\n"`

4️⃣ 保存して **F5** を押して再起動し、Copilot に **"start over"** と入力して変更を確認します。

<cc-end-step lab="baf1" exercise="4" step="1" />

---8<--- "ja/b-congratulations.md"

ラボ BAF1 - はじめてのエージェントの構築と実行 を完了しました！

このラボで学んだこと:

- ✅ Agent Framework プロジェクトのクローンと探索
- ✅ Azure AI 資格情報によるエージェントの構成
- ✅ ローカルでのエージェントの実行とデバッグ
- ✅ Microsoft 365 Copilot でのエージェントのテスト
- ✅ コアコンポーネント (エージェント、プラグイン、インストラクション) の理解
- ✅ 動作をカスタマイズする簡単な変更の方法

次のラボでは、Azure AI Search と gpt-4.1 を用いたドキュメント検索の統合により、さらに強力な機能を追加します！

<cc-next url="../02-add-claim-search" />

<img src="https://m365-visitor-stats.azurewebsites.net/copilot-camp/custom-engine/agent-framework/01-build-and-run--ja" />