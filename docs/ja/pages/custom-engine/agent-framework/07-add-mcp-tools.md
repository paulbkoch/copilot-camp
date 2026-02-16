---
search:
  exclude: true
---
# ラボ BAF7 - MCP ツール統合の追加

このラボでは、Zava Insurance エージェントに Model Context Protocol (MCP) ツールを拡張します。Azure Functions を使用してクレーム アジャスター管理機能を提供する MCP サーバーを作成し、そのツールを Custom Engine エージェントのカスタム プラグインから利用します。

???+ info "MCP 統合を理解する"
    **Model Context Protocol (MCP)** により、エージェントは次のことが可能になります:
    
    - **外部ツールへの接続**: 標準化されたプロトコルを使用して MCP サーバーのツールへアクセス  
    - **クレーム アジャスターの管理**: 専門分野および国別にアジャスターを一覧表示  
    - **アジャスターのクレーム割り当て**: クレーム タイプに基づき適切なアジャスターを自動割り当て  
    - **Azure Functions の活用**: スケーラビリティのために MCP ツールをサーバーレス関数としてホスト  
    
    この統合により、MCP エコシステムを使用してエージェントの機能を拡張する方法を示します。

<hr />

## 概要

前回までのラボでは、クレーム検索、ビジョン分析、ポリシー検索、コミュニケーション機能を追加しました。今回は MCP ツールを用いてクレーム アジャスターを管理できるようにエージェントを拡張します。保険業務では、クレームを専門のアジャスターにルーティングする必要があるため、よくある要件です。

**Model Context Protocol (MCP)** は、AI アプリケーションが外部データソースやツールに接続できるようにするオープン スタンダードです。Azure Functions で MCP サーバーを作成することで、ビジネス ロジックを MCP 互換エージェントが利用できるツールとして公開できます。

???+ note "作成するもの"
    - **MCP サーバー**: クレーム アジャスター ツールを MCP 経由で公開する Azure Function アプリ  
    - **ClaimsAdjustersPlugin**: MCP ツールを利用してアジャスターの一覧および割り当てを行うプラグイン  
    - **エージェント統合**: プラグインを組み込み、会話内でアジャスター管理を実現  

## Exercise 1: MCP サーバーのセットアップ

この演習では、クレーム アジャスター管理機能を提供する事前構築済み MCP サーバーをセットアップします。サーバーは TypeScript で書かれた Azure Functions で、Azure Table Storage にクレーム アジャスターのレコードを保存します。

### Step 1: MCP サーバーと前提条件の理解

Insurance MCP サーバーは、Model Context Protocol 経由でクレーム アジャスター管理ツールを公開する、すぐに使用できる Azure Functions アプリケーションです。次のツールを提供します:

- **get_claims_adjusters**: 国と専門分野でオプション フィルターをかけて、すべてのクレーム アジャスターを取得  
- **get_claims_adjuster**: ID で特定のクレーム アジャスターを取得  
- **assign_claim_adjuster**: 保険クレームにクレーム アジャスターを割り当て  

??? note "MCP サーバーの仕組み"
    MCP サーバーは MCP クライアントから呼び出せるツールを公開します。各ツールには次の要素があります:
    
    - **ツール名**: 一意の識別子 (例: `get_claims_adjusters`)  
    - **説明**: LLM がツールをいつ使用するか理解するための説明  
    - **プロパティ**: 型と説明を含む入力パラメーター  
    - **ハンドラー**: ツールが呼び出された際に実行される関数  
    
    Azure Functions は、ネイティブ MCP プロトコル バインディングを介して MCP サーバーをホストする便利なモデルを提供します。

MCP サーバーのアーキテクチャは次のとおりです:

1. **データ ストレージ**: クレーム アジャスター レコード用の Azure Table Storage  
2. **HTTP ハンドラー**: 直接 API アクセス用の REST エンドポイント  
3. **MCP ツール ハンドラー**: エージェントが利用する MCP ツールとして登録された関数  

開始前に以下を用意してください:

- [Node.js v22 以上](https://nodejs.org/en){target=_blank}  
- [Azure Functions Core Tools v4](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local){target=_blank}  
- [Visual Studio Code](https://code.visualstudio.com/){target=_blank}  
- [Dev tunnel](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started){target=_blank}  
- [Azurite](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite){target=_blank} - Azure Storage エミュレーター ( `npm install -g azurite` でインストール)  

<cc-end-step lab="baf7" exercise="1" step="1" />

### Step 2: MCP サーバーのダウンロードと確認

このラボでは、事前構築済みの Insurance MCP サーバーを使用します。サーバー ファイルを [こちらからダウンロード](https://download-directory.github.io/?url=https://github.com/microsoft/copilot-camp/tree/main/src/agent-framework/insurance-mcp&filename=insurance-mcp){target=_blank} してください。

1️⃣ ZIP を展開し、ローカル フォルダーに保存します。

2️⃣ 展開したフォルダーを Visual Studio Code で開きます:

```bash
cd insurance-mcp
code .
```

3️⃣ Visual Studio Code でプロジェクト構造を確認します。主な構成要素は次のとおりです:

- `src/functions`: MCP ツール実装を含む Azure Functions  
- `data`: 初期化用サンプル クレーム アジャスター データ  
- `env`: 環境設定ファイル  
- `package.json`: プロジェクト依存関係と npm スクリプト  
- `host.json`: Azure Functions ホスト設定  

!!! info
    MCP サーバーには、ビルド、dev tunnel の起動、Azure Functions のローカル実行を自動化する事前設定済み VS Code タスクが含まれています。 **F5** キー 1 回で全てを開始でき、開発体験が簡素化されます。

<cc-end-step lab="baf7" exercise="1" step="2" />

### Step 3: 環境の構成

MCP サーバーを実行する前に、Azure Table Storage 接続を構成する必要があります。

1️⃣ `env` フォルダーで `.env.local.sample` をコピーし、 `.env.local` という新しいファイルを作成します:

**Windows PowerShell:**
```powershell
Copy-Item env/.env.local.sample env/.env.local
```

**macOS/Linux:**
```bash
cp env/.env.local.sample env/.env.local
```

2️⃣ `env/.env.local` ファイルを編集し、Azure Table Storage の詳細を入力します:

```bash
# Azure Table Storage Configuration
AZURE_STORAGE_ACCOUNT=your_storage_account_name
AZURE_TABLE_ENDPOINT=https://your_storage_account_name.table.core.windows.net
TABLE_NAME=ClaimAdjusters
ALLOW_INSECURE_CONNECTION=false

# DevTunnel Configuration
TUNNEL_ID=
```

3️⃣ `.env.local.user.sample` をコピーし、 `.env.local.user` という新しいファイルを作成します:

**Windows PowerShell:**
```powershell
Copy-Item env/.env.local.user.sample env/.env.local.user
```

**macOS/Linux:**
```bash
cp env/.env.local.user.sample env/.env.local.user
```

4️⃣ `env/.env.local.user` ファイルを編集し、Azure Storage アカウント キーを入力します:

```bash
# Azure Table Storage Configuration
SECRET_AZURE_STORAGE_KEY=your_storage_account_key
```

!!! warning "機密情報の保護"
    `.env.local.user` ファイルには機密資格情報が含まれており、ソース管理にコミットしてはいけません。 `.gitignore` に既に含まれているため、誤ってコミットされるのを防ぎます。

<cc-end-step lab="baf7" exercise="1" step="3" />

### Step 4: 依存関係のインストールとデータの初期化

1️⃣ プロジェクトの依存関係をインストールします:

```bash
npm install
```

2️⃣ Azure Table Storage にクレーム アジャスター データを初期化します:

```bash
npm run init-data
```

このスクリプトは Azure Storage アカウントに `ClaimAdjusters` テーブルを作成し、サンプル クレーム アジャスター レコードを投入します。

<cc-end-step lab="baf7" exercise="1" step="4" />

### Step 5: Dev Tunnel で MCP サーバーを実行

プロジェクトには、dev tunnel を起動し Azure Functions をローカルで実行する VS Code タスクが事前設定されています。

1️⃣ まだ dev tunnel をインストールしていない場合は、[こちらの手順](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started){target=_blank} に従ってインストールします。

2️⃣ Visual Studio Code で **F5** を押して MCP サーバーを起動します。

事前設定済みの VS Code タスクにより次が自動的に実行されます:

- Azurite (Azure Storage エミュレーター) の起動  
- TypeScript プロジェクトのビルド  
- dev tunnel の作成と起動  
- Azure Functions ランタイムの起動  
- MCP Inspector のインストール、起動、およびブラウザーでの開放 (ツールのテストとデバッグ用)  

3️⃣ サーバーが起動したら、ターミナルに表示される dev tunnel URL (`https://your_devtunnel_id.devtunnels.ms` など) を確認します。MCP サーバーのエンドポイント URL は次のようになります:

```text
https://your_devtunnel_id.devtunnels.ms/runtime/webhooks/mcp
```

!!! tip "サーバーは起動したままにしておく"
    このラボ全体を通じて MCP サーバーと dev tunnel を起動したままにしてください。VS Code でデバッグ セッションを実行中はトンネルが有効です。再起動が必要な場合は **F5** を再度押すだけで構いません。

<cc-end-step lab="baf7" exercise="1" step="5" />

## Exercise 2: エージェントで MCP クライアントを構成

次に、Custom Engine エージェントを設定して MCP サーバーへ接続します。

### Step 1: MCP クライアント設定を追加

1️⃣ エージェント プロジェクトの `.env.local` ファイルを開きます。

2️⃣ Exercise 1 で取得した dev tunnel URL を使用して MCP サーバー設定を追加します:

```bash
# MCP Server Configuration
MCP_SERVER_URL=https://your_devtunnel_id.devtunnels.ms/runtime/webhooks/mcp
```

!!! note
    `your_devtunnel_id` を Exercise 1 で MCP サーバーを起動した際にターミナルに表示された実際のトンネル ID に置き換えてください。

<cc-end-step lab="baf7" exercise="2" step="1" />

### Step 2: 依存性注入に MCP クライアントを登録

1️⃣ エージェント プロジェクトのルートで新しいターミナルを開き、次のスクリプトを実行して ModelContextProtocol NuGet パッケージをインストールします:

```bash
dotnet add package ModelContextProtocol --version 0.4.1-preview.1
```

2️⃣ エージェント プロジェクトの `src/Program.cs` を開きます。

3️⃣ ファイルの先頭に必要な using ステートメントを追加します:

```csharp
using ModelContextProtocol.Client;
using ModelContextProtocol.Protocol;
```

4️⃣ 他のサービス (VisionService、LanguageModelService など) を登録している場所を見つけ、MCP クライアントの登録を追加します:

```csharp
// Register MCP Client for claims adjusters
builder.Services.AddSingleton<McpClient>(sp =>
{
    var configuration = sp.GetRequiredService<IConfiguration>();
    var mcpServerUrl = configuration["MCP_SERVER_URL"] 
        ?? throw new InvalidOperationException("MCP_SERVER_URL is not configured");

    var clientTransport = new HttpClientTransport(new HttpClientTransportOptions {
        Endpoint = new Uri(mcpServerUrl)});

    return McpClient.CreateAsync(clientTransport).GetAwaiter().GetResult();
});
```

<cc-end-step lab="baf7" exercise="2" step="2" />

## Exercise 3: ClaimsAdjustersPlugin の作成

次に、MCP ツールを使用してクレーム アジャスターを管理するプラグインを作成します。

### Step 1: ClaimsAdjustersPlugin の作成

??? note "このプラグインが行うこと"
    `ClaimsAdjustersPlugin` は主に 2 つの機能を提供します:
    
    **ListClaimsAdjustersAsync**:
    
    - クレーム タイプと国でフィルターしたクレーム アジャスターを取得  
    - クレーム タイプを検証 ( "Auto" と "Homeowners" のみサポート)  
    - MCP サーバーの `get_claims_adjusters` ツールを呼び出す  
    
    **AssignClaimAdjusterAsync**:
    
    - 特定のアジャスターをクレームに割り当て  
    - 割り当て ID を含む確認メッセージを返す  
    - MCP サーバーの `assign_claim_adjuster` ツールを呼び出す  

1️⃣ `src/Plugins/ClaimsAdjustersPlugin.cs` という新しいファイルを作成し、次の実装を追加します:

```csharp
using Microsoft.Agents.Builder;
using Microsoft.Agents.Core.Models;
using System.ComponentModel;
using System.Text.Json;
using InsuranceAgent;
using Microsoft.Agents.Builder.State;
using ModelContextProtocol.Client;

namespace ZavaInsurance.Plugins
{
    /// <summary>
    /// Claims Adjusters Plugin for Zava Insurance
    /// Provides tools for managing and retrieving claims adjuster information via MCP.
    /// </summary>
    public class ClaimsAdjustersPlugin
    {
        private readonly ITurnContext _turnContext;
        private readonly McpClient _mcpClient;
        private readonly IConfiguration _configuration;

        public ClaimsAdjustersPlugin(ITurnContext turnContext,
            McpClient mcpClient, 
            IConfiguration configuration)
        {
            _turnContext = turnContext ?? throw new ArgumentNullException(nameof(turnContext));
            _mcpClient = mcpClient ?? throw new ArgumentNullException(nameof(mcpClient));
            _configuration = configuration ?? throw new ArgumentNullException(nameof(configuration));
        }

        /// <summary>
        /// Retrieves claims adjusters based on claim type and country.
        /// </summary>
        /// <param name="claimType">The claim type to filter claims adjusters (Auto or Homeowners)</param>
        /// <param name="country">The country to filter claims adjusters</param>
        /// <returns>A list of claims adjusters matching the criteria</returns>
        [Description("Retrieves claims adjusters based on area and country")]
        public async Task<string> ListClaimsAdjustersAsync(string claimType, string country)
        {
            await NotifyUserAsync($"Retrieving claims adjusters for area {claimType} and country {country}...");

            // Validate claim type - only "Auto" and "Homeowners" are supported
            if (claimType != "Auto" && claimType != "Homeowners")
            {
                claimType = null;
            }

            // Validate country
            if (country == "All")
            {
                country = null;
            }

            var result = await _mcpClient.CallToolAsync("get_claims_adjusters", 
                new Dictionary<string, object?> {                 
                    ["area"] = claimType, 
                    ["country"] = country
                }
            );

            if (!result.IsError.HasValue || result.IsError.HasValue && !result.IsError.Value)
            {
                var adjusters = result.Content;
                return JsonSerializer.Serialize(adjusters, new JsonSerializerOptions { WriteIndented = true });
            }
            else
            {
                return $"Error retrieving claims adjusters!";
            }
        }

        /// <summary>
        /// Assigns a claims adjuster to a specific claim.
        /// </summary>
        /// <param name="claimId">The ID of the claim</param>
        /// <param name="adjusterId">The ID of the claims adjuster</param>
        /// <returns>Confirmation message of assignment</returns>
        [Description("Assigns a claims adjuster to a specific claim")]
        public async Task<string> AssignClaimAdjusterAsync(string claimId, string adjusterId)
        {
            await NotifyUserAsync($"Assigning claims adjuster {adjusterId} to claim {claimId}...");

            var result = await _mcpClient.CallToolAsync("assign_claim_adjuster", 
                new Dictionary<string, object?> {                 
                    ["claimId"] = claimId, 
                    ["adjusterId"] = adjusterId
                }
            );

            if (!result.IsError.HasValue || result.IsError.HasValue && !result.IsError.Value)
            {
                var adjusters = result.Content;
                return JsonSerializer.Serialize(adjusters, new JsonSerializerOptions { WriteIndented = true });
            }
            else
            {
                return $"Error assigning claims adjuster!";
            }
        }

        private async Task NotifyUserAsync(string message)
        {
            if (!_turnContext.Activity.ChannelId.Channel!.Contains(Channels.Webchat))
            {
                await _turnContext.StreamingResponse.QueueInformativeUpdateAsync(message);
            }
            else
            {
                await _turnContext.StreamingResponse.QueueInformativeUpdateAsync(message).ConfigureAwait(false);
            }
        }
    }
}
```

<cc-end-step lab="baf7" exercise="3" step="1" />

## Exercise 4: エージェントに ClaimsAdjustersPlugin を登録

次に、ZavaInsuranceAgent に ClaimsAdjustersPlugin を組み込みます。

### Step 1: エージェント コンストラクターの更新

1️⃣ `src/Agent/ZavaInsuranceAgent.cs` を開きます。

2️⃣ ファイルの先頭に必要な using ステートメントを追加します:

```csharp
using ModelContextProtocol.Client;
```

3️⃣ クラス フィールド セクションを見つけ、MCP クライアントのフィールドを追加します:

```csharp
private readonly McpClient _mcpClient = null;
```

4️⃣ コンストラクターを更新し、MCP クライアントを受け取り保存します:

```csharp
public ZavaInsuranceAgent(AgentApplicationOptions options, IChatClient chatClient, IConfiguration configuration, IServiceProvider serviceProvider, IHttpClientFactory httpClientFactory, McpClient mcpClient) : base(options)
{
    _chatClient = chatClient;
    _configuration = configuration;
    _serviceProvider = serviceProvider;
    _httpClient = httpClientFactory.CreateClient() ?? throw new ArgumentNullException(nameof(httpClientFactory));
    _mcpClient = mcpClient;

    // Greet when members are added to the conversation
    OnConversationUpdate(ConversationUpdateEvents.MembersAdded, WelcomeMessageAsync);

    // Listen for ANY message to be received
    OnActivity(ActivityTypes.Message, OnMessageAsync, autoSignInHandlers: [UserAuthorization.DefaultHandlerName]);
}
```

<cc-end-step lab="baf7" exercise="4" step="1" />

### Step 2: ClaimsAdjustersPlugin のインスタンス化

1️⃣ `GetClientAgent` メソッド (他のプラグインをインスタンス化している場所) を見つけます。

2️⃣ 既存のプラグインの後に ClaimsAdjustersPlugin のインスタンス化を追加します:

```csharp
// Create ClaimsAdjustersPlugin with MCP client
ClaimsAdjustersPlugin claimsAdjustersPlugin = new(context, _mcpClient, _configuration);
```

<cc-end-step lab="baf7" exercise="4" step="2" />

### Step 3: ClaimsAdjusters ツールの登録

同じ `GetClientAgent` メソッドで、 `toolOptions.Tools` にツールを追加している箇所を見つけ、クレーム アジャスター ツールを追加します:

```csharp
// Register Claims Adjusters MCP tools
toolOptions.Tools.Add(AIFunctionFactory.Create(claimsAdjustersPlugin.ListClaimsAdjustersAsync));
toolOptions.Tools.Add(AIFunctionFactory.Create(claimsAdjustersPlugin.AssignClaimAdjusterAsync));
```

<cc-end-step lab="baf7" exercise="4" step="3" />

### Step 4: エージェント インストラクションの更新

エージェント インストラクションを更新して、クレーム アジャスター機能を含めます。

1️⃣ `ZavaInsuranceAgent.cs` の `AgentInstructions` フィールドを見つけます。

2️⃣ クレーム アジャスター ツールに関する説明を追加します:

```csharp
private readonly string AgentInstructions = """
        You are a professional insurance claims assistant for Zava Insurance.

        Whenever the user starts a new conversation or provides a prompt to start a new conversation like "start over", "restart", 
        "new conversation", "what can you do?", "how can you help me?", etc. use {{StartConversationPlugin.StartConversation}} and 
        provide to the user exactly the message you get back from the plugin.

        **Available Tools:**
        Use {{DateTimeFunctionTool.getDate}} to get the current date and time.
        For claims search, use {{ClaimsPlugin.SearchClaims}} and {{ClaimsPlugin.GetClaimDetails}}.
        For damage photo viewing, use {{VisionPlugin.ShowDamagePhoto}}.
        For AI vision damage analysis, use {{VisionPlugin.AnalyzeAndShowDamagePhoto}} and require approval via {{VisionPlugin.ApproveAnalysis}}.
        For policy search, use {{PolicyPlugin.SearchPolicies}} and {{PolicyPlugin.GetPolicyDetails}}.
        For sending investigation reports and claim details via email, use {{CommunicationPlugin.GenerateInvestigationReport}} and {{CommunicationPlugin.SendClaimDetailsByEmail}}.
        For claims compliance analysis, use {{ClaimsPoliciesPlugin.AnalyzeClaimCompliance}}.

        To list claim adjusters use {{ClaimsAdjustersPlugin.ListClaimsAdjusters}}. When listing claim adjusters:
        - Always try to use the country of the current claim, if any. Otherwise, if no country is specified by the user, set country value to 'All'.
        - Always try to use the claim type of the current claim, if any.
        - Always retrieve id, firstName, lastName, email, country, phone, and area for each claim adjuster.
        - Only "Auto" and "Homeowners" are valid claim types. If the user provides any other claim type, set area value to null.

        To assign a claim adjuster to a claim use {{ClaimsAdjustersPlugin.AssignClaimAdjuster}}.

        **IMPORTANT**: When user asks to "check policy for this claim", first use GetClaimDetails to get the claim's policy number, then use GetPolicyDetails with that policy number.

        **IMPORTANT**: If in the response there are references to citations like [1], [2], etc., make sure to include those citations in the response so that M365 Copilot can render them properly.

        Stick to the scenario above and use only the information from the tools when answering questions.
        Be concise and professional in your responses.
        """;
```

??? note "これらのインストラクションが重要な理由"
    インストラクションは、LLM がクレーム アジャスター ツールを効果的に使用できるように導きます:
    
    - **国の推論**: クレームに国情報がある場合、それを使用  
    - **クレーム タイプの検証**: "Auto" と "Homeowners" のみ有効  
    - **コンテキスト認識**: 既存のクレーム コンテキストを活用し、関連するアジャスターを提供  

<cc-end-step lab="baf7" exercise="4" step="4" />

## Exercise 5: MCP ツール統合のテスト

それでは、MCP ツール統合全体をテストしましょう！

### Step 1: 実行と確認

1️⃣ MCP サーバーが dev tunnel と共に実行中であることを確認します。実行していない場合は、MCP サーバー プロジェクトを VS Code で開き、 **F5** を押して起動します。

2️⃣ 別の VS Code ウィンドウでエージェント プロジェクトを開き、 **F5** を押してエージェントのデバッグを開始します。

3️⃣ プロンプトが表示されたら **(Preview) Debug in Copilot (Edge)** を選択します。

4️⃣ ターミナルに通常の初期化メッセージが表示されることを確認します。

5️⃣ Microsoft 365 Copilot がブラウザーで開きます。

!!! tip "2 つのプロジェクトを同時実行"
    MCP サーバー用とエージェント用に 2 つの VS Code ウィンドウを同時に開いて実行する必要があります。テストを行う前に両方が稼働していることを確認してください。

<cc-end-step lab="baf7" exercise="5" step="1" />

### Step 2: クレーム アジャスターの一覧取得をテスト

1️⃣ まず Microsoft 365 Copilot で、クレームのコンテキストを確立するためにクレームを取得します:

```text
Get details for claim CLM-2025-001007
```

2️⃣ 次にアジャスターを尋ねます:

```text
List available claims adjusters for this claim
```

エージェントは次を行うはずです:

- クレームのタイプ (Auto) とクレーム詳細に含まれる国を使用  
- `get_claims_adjusters` MCP ツールを呼び出す  
- 条件に一致するアジャスターの一覧を返す  

<cc-end-step lab="baf7" exercise="5" step="2" />

### Step 3: アジャスターの割り当てをテスト

1️⃣ アジャスター一覧を取得した後、次のようにして割り当てます:

```text
Assign adjuster ADJ-EE-0001 to this claim
```

エージェントは次を行うはずです:

- `assign_claim_adjuster` MCP ツールを呼び出す  
- 割り当て ID 付きの確認メッセージを返す  
- アジャスターの名前を確認する  

<cc-end-step lab="baf7" exercise="5" step="3" />

### Step 4: フィルター付きでテスト

1️⃣ 直接フィルターを指定してテストします:

```text
Show me all Auto adjusters in the United States
```

エージェントは専門分野と国でアジャスターをフィルターするはずです。

2️⃣ "All" の国指定でテストします:

```text
List all Homeowners adjusters
```

エージェントは国に関係なく Homeowners アジャスターをすべて返すはずです。

<cc-end-step lab="baf7" exercise="5" step="4" />

!!! success "おめでとうございます！"
    MCP ツールを Custom Engine エージェントに正常に統合できました。これでエージェントは次のことが可能です:
    
    ✅ 外部 MCP サーバーへの接続  
    ✅ 専門分野と国でフィルターしたクレーム アジャスターの一覧取得  
    ✅ クレームへのアジャスター割り当てと確認  
    ✅ クレーム コンテキストを利用した適切なアジャスター候補の提示  
    
    このパターンを拡張することで、MCP 互換サービスとの統合を行い、豊富なツールと機能をエージェントに活用させることができます。