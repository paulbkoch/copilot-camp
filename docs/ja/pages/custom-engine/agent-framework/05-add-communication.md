---
search:
  exclude: true
---
# ラボ BAF5 - コミュニケーション機能の追加

このラボでは、Zava Insurance エージェントにプロフェッショナルなコミュニケーション機能を追加します。Microsoft Graph を使用して詳細な請求レポートをメール送信し、ビジョン解析結果や不正検知データを含む包括的な調査レポートを生成できるようにします。

???+ info "CommunicationPlugin を理解する"
    **CommunicationPlugin** によりエージェントは次のことが可能になります。
    
    - **プロフェッショナルなメール送信**: Microsoft Graph Mail API を使用し、HTML 形式の請求詳細を送信  
    - **調査レポートの生成**: ClaimsPlugin、VisionPlugin、PolicyPlugin のデータを統合した包括的レポートを作成  
    - **ナレッジベースの集約**: ビジョン解析、不正検知、ポリシー詳細を含むすべての請求データを取得  
    - **OAuth トークン管理**: キャッシュされた On-Behalf-Of (OBO) トークンで安全に Graph API を呼び出し
    
    これにより、コミュニケーションとレポート機能が加わり、請求処理ワークフローが完成します。

## Exercise 1: CommunicationPlugin の作成

請求検索、ビジョン解析、ポリシー検索機能がそろったので、レポート送信とメール送信のコミュニケーション機能を追加しましょう。

### Step 1: CommunicationPlugin を完成させる

??? note "このプラグインの役割"
    `CommunicationPlugin` は 2 つの主要機能を提供します。
    
    **SendClaimDetailsByEmail**:
    
    - Knowledge Base Service から包括的な請求データを取得
    - プロフェッショナルな HTML 形式のメールを作成
    - Microsoft Graph API で送信
    - 認証にはキャッシュ済み OAuth トークンを使用
    - 宛先が指定されていない場合は現在のユーザーのメールアドレスを既定値として使用
    
    **GenerateInvestigationReport**:
    
    - ビジョン解析と不正検知を含むすべての請求データを収集
    - 推奨事項を含む包括的な調査レポートをフォーマット
    - プロフェッショナルな構成の markdown でレポートを返却
    
    どちらのメソッドも **KnowledgeBaseService** をカスタム指示とともに利用し、請求データを取得・統合します。

1️⃣ `src/Plugins/CommunicationPlugin.cs` に新しいファイルを作成し、以下の完全実装を追加します。

```csharp
using Microsoft.Agents.Builder;
using Microsoft.Agents.Core.Models;
using System.ComponentModel;
using System.Text.Json;
using InsuranceAgent.Services;
using InsuranceAgent;
using Microsoft.Agents.Builder.State;

namespace ZavaInsurance.Plugins
{
    /// <summary>
    /// Communication Plugin for Zava Insurance
    /// Provides tools for generating professional policyholder communications
    /// Ensures consistent, compliant messaging across all customer touchpoints
    /// </summary>
    public class CommunicationPlugin
    {
        private readonly ITurnContext _turnContext;
        private readonly ITurnState _turnState;
        private readonly KnowledgeBaseService _knowledgeBaseService;
        private readonly HttpClient _httpClient;

        public CommunicationPlugin(ITurnContext turnContext, ITurnState turnState, KnowledgeBaseService knowledgeBaseService, HttpClient httpClient)
        {
            _turnContext = turnContext ?? throw new ArgumentNullException(nameof(turnContext));
            _turnState = turnState ?? throw new ArgumentNullException(nameof(turnState));
            _knowledgeBaseService = knowledgeBaseService ?? throw new ArgumentNullException(nameof(knowledgeBaseService));
            _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
        }

        /// <summary>
        /// Sends detailed claim information via email using Microsoft Graph
        /// </summary>
        /// <param name="claimId">The unique claim identifier</param>
        /// <param name="recipientEmail">Email address to send the claim details to (optional, uses policyholder email if not provided)</param>
        /// <returns>Success or failure message indicating email delivery status</returns>
        [Description("Sends a well-formatted email with comprehensive claim details including policyholder info, documentation status, timeline, and recommendations via Microsoft Graph.")]
        public async Task<string> SendClaimDetailsByEmail(string claimId, string recipientEmail = null)
        {
            await NotifyUserAsync($"Retrieving details for claim {claimId}...");

            // Read the user profile
            var userProfile = _turnState.Conversation.GetCachedUserProfile();
            var accessToken = _turnState.Conversation.GetCachedOBOAccessToken();

            // Use Knowledge Base with instructions for email-ready claim summary
            var instructions = @"You are preparing a professional claim summary for email. Include:
                - Claim Number, Status, and Date Filed
                - Policyholder Information
                - Claim Details (type, amount, location, description)
                - Current Status and Next Steps
                - Document Status and Requirements
                - Key Timeline Events
                Format clearly for email readability. Cite sources with [ref_id:X].";
            
            var query = $"complete details for claim {claimId}";
            var claimData = await _knowledgeBaseService.RetrieveAsync(query, instructions, topResults: 5);

            // Use provided email or default to the current user's email
            var toEmail = string.IsNullOrEmpty(recipientEmail) ? userProfile.Mail : recipientEmail;
            
            await NotifyUserAsync($"Preparing email for {toEmail}...");

            // Create well-formatted HTML email content using KB data
            var emailContent = CreateClaimDetailsEmailContentFromKB(claimId, claimData.ToString() ?? "");
            
            try
            {
                // Send email via Microsoft Graph
                var emailPayload = new
                {
                    message = new
                    {
                        subject = $"Zava Insurance - Claim Details Report ({claimId})",
                        body = new
                        {
                            contentType = "HTML",
                            content = emailContent
                        },
                        toRecipients = new[]
                        {
                            new
                            {
                                emailAddress = new
                                {
                                    address = toEmail
                                }
                            }
                        },
                        from = new
                        {
                            emailAddress = new
                            {
                                address = userProfile.Mail
                            }
                        }
                    },
                    saveToSentItems = true
                };

                var jsonContent = JsonSerializer.Serialize(emailPayload);
                var httpContent = new StringContent(jsonContent, System.Text.Encoding.UTF8, "application/json");
                
                _httpClient.DefaultRequestHeaders.Clear();
                _httpClient.DefaultRequestHeaders.Add("Authorization", $"Bearer {accessToken}");
                _httpClient.DefaultRequestHeaders.Add("Accept", "application/json");

                await NotifyUserAsync($"Sending email via Microsoft Graph...");

                var response = await _httpClient.PostAsync("https://graph.microsoft.com/v1.0/me/sendMail", httpContent);

                if (response.IsSuccessStatusCode)
                {
                    await NotifyUserAsync($"✅ Email sent successfully!");
                    return $"✅ Success: Claim details for {claimId} have been successfully sent to {toEmail}. " +
                           $"The email includes comprehensive claim information, documentation status, timeline, and recommendations.";
                }
                else
                {
                    var errorContent = await response.Content.ReadAsStringAsync();
                    await NotifyUserAsync($"❌ Failed to send email: {response.StatusCode}");
                    return $"❌ Error: Failed to send email for claim {claimId}. " +
                           $"Status: {response.StatusCode}, Details: {errorContent}";
                }
            }
            catch (Exception ex)
            {
                await NotifyUserAsync($"❌ Exception occurred while sending email");
                return $"❌ Error: Exception occurred while sending claim details email: {ex.Message}";
            }
        }

        /// <summary>
        /// Creates well-formatted HTML email content with claim details
        /// </summary>
        private string CreateClaimDetailsEmailContentFromKB(string claimId, string knowledgeBaseData)
        {
            return $@"
<!DOCTYPE html>
<html>
<head>
    <style>
        body {{ font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 0; padding: 20px; background-color: #f8f9fa; }}
        .container {{ max-width: 800px; margin: 0 auto; background-color: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }}
        .header {{ background: linear-gradient(135deg, #007bff, #0056b3); color: white; padding: 30px; border-radius: 8px 8px 0 0; }}
        .content {{ padding: 30px; }}
        .section {{ margin-bottom: 25px; }}
        .footer {{ background-color: #f8f9fa; padding: 20px; text-align: center; color: #6c757d; border-radius: 0 0 8px 8px; }}
    </style>
</head>
<body>
    <div class='container'>
        <div class='header'>
            <h1>🏢 Zava Insurance - Claim Details Report</h1>
            <h2>Claim #{claimId}</h2>
        </div>
        
        <div class='content'>
            <div class='section'>
                <h3>📋 Claim Information</h3>
                <pre style='background-color: #f8f9fa; padding: 15px; border-radius: 5px; white-space: pre-wrap;'>{knowledgeBaseData}</pre>
            </div>
        </div>
        
        <div class='footer'>
            <p><strong>Zava Insurance</strong> | Professional Claims Management</p>
            <p>This is an automated report generated from our AI-powered system.</p>
        </div>
    </div>
</body>
</html>";
        }

        /// <summary>
        /// Generates a comprehensive investigation report for a claim including vision analysis and fraud assessment
        /// </summary>
        /// <param name="claimNumber">The claim number to generate report for</param>
        /// <returns>A formatted investigation report with findings and recommendations</returns>
        [Description("Generates a comprehensive investigation report for a claim that includes damage assessment from vision analysis, fraud risk evaluation, and actionable recommendations for claim resolution.")]
        public async Task<string> GenerateInvestigationReport(string claimNumber)
        {
            if (string.IsNullOrWhiteSpace(claimNumber))
                return "❌ Error: Claim number cannot be empty.";

            await NotifyUserAsync($"📊 Generating investigation report for {claimNumber}...");

            try
            {
                // Use Knowledge Base to gather comprehensive claim data including vision and fraud analysis
                var instructions = @"You are an insurance claims investigator preparing a comprehensive investigation report. 
                    Include ALL available information:
                    
                    **Claim Overview:**
                    - Claim Number, Type, Status
                    - Policyholder Name and Policy Number
                    - Date Filed, Estimated Cost
                    - Location and Description of Incident
                    
                    **Vision Analysis Findings:**
                    - Damage assessment results from AI vision analysis
                    - Estimated repair costs from damage photos
                    - Approval status of vision analysis
                    - Key damage observations
                    
                    **Fraud Risk Assessment:**
                    - Fraud risk score and level
                    - Key fraud indicators identified
                    - Comparison to normal claim patterns
                    - Risk assessment explanation
                    
                    **Documentation Review:**
                    - Completeness of documentation
                    - Missing items (if any)
                    - Quality of submitted evidence
                    
                    **Recommendations:**
                    - Approval/denial recommendation with justification
                    - Required next steps
                    - Any additional investigation needed
                    - Suggested claim resolution path
                    
                    Format professionally with clear sections, bullet points, and evidence-based conclusions.
                    Cite all sources with [ref_id:X].";
                
                var query = $"Complete investigation details for claim {claimNumber} including vision analysis, fraud assessment, and all supporting evidence";
                var reportData = await _knowledgeBaseService.RetrieveAsync(query, instructions, topResults: 10);

                // Format the final report
                var report = new System.Text.StringBuilder();
                report.AppendLine($"# 📋 Investigation Report: {claimNumber}");
                report.AppendLine($"**Generated:** {DateTime.UtcNow:yyyy-MM-dd HH:mm:ss} UTC");
                report.AppendLine($"**Status:** Investigation Complete");
                report.AppendLine();
                report.AppendLine("---");
                report.AppendLine();
                report.AppendLine(reportData);
                report.AppendLine();
                report.AppendLine("---");
                report.AppendLine();
                report.AppendLine("**Report Prepared By:** Zava Insurance Claims Investigation System");
                report.AppendLine("**Next Steps:** Review findings and proceed with recommended claim resolution action.");

                await NotifyUserAsync("✅ Investigation report generated successfully!");
                
                return report.ToString();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error generating investigation report: {ex.Message}");
                return $"❌ Error generating investigation report: {ex.Message}";
            }
        }

        // Helper methods
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

<cc-end-step lab="baf5" exercise="1" step="1" />

## Exercise 2: Agent への CommunicationPlugin の登録

次に、ZavaInsuranceAgent に CommunicationPlugin を組み込みます。

### Step 1: CommunicationPlugin のインスタンス化

1️⃣ `src/Agent/ZavaInsuranceAgent.cs` を開きます。

2️⃣ `GetClientAgent` メソッド（約 159 行目）を探します。

3️⃣ プラグインがインスタンス化されている箇所（`PolicyPlugin policyPlugin = ...` の後）を見つけます。

4️⃣ CommunicationPlugin をインスタンス化するコードを追加します。

```csharp

// Get HttpClient for API calls
var httpClientFactory = scope.ServiceProvider.GetRequiredService<IHttpClientFactory>();
var httpClient = httpClientFactory.CreateClient();
// Create CommunicationPlugin with required dependencies
CommunicationPlugin communicationPlugin = new(context, turnState, knowledgeBaseService, httpClient);
```

<cc-end-step lab="baf5" exercise="2" step="1" />

### Step 2: Communication ツールの登録

同じ `GetClientAgent` メソッド内で、`toolOptions.Tools` にツールを追加している場所（約 180 行目）までスクロールします。

Policy ツールのセクションを見つけ、その直後に Communication ツールを追加します。

```csharp
// Register Communication tools
toolOptions.Tools.Add(AIFunctionFactory.Create(communicationPlugin.SendClaimDetailsByEmail));
toolOptions.Tools.Add(AIFunctionFactory.Create(communicationPlugin.GenerateInvestigationReport));
```

??? note "ツール登録のパターン"
    エージェントは **AIFunctionFactory** を使ってプラグインメソッドを AI ツールとして登録します。各メソッドの `[Description]` 属性がツールの説明となり、LLM がいつ呼び出すべきかを判断する手がかりになります。

<cc-end-step lab="baf5" exercise="2" step="2" />

## Exercise 3: Agent の指示とセキュリティの更新

エージェントの指示にコミュニケーション機能を追加し、ユーザー認証と OBO トークンをサポートするためのセキュリティ設定を更新します。

### Step 1: 指示に Communication ツールを追加

1️⃣ `src/Agent/ZavaInsuranceAgent.cs` の `AgentInstructions` フィールド（約 32 行目）を探します。

2️⃣ ツールリストのセクションを見つけ、コミュニケーションツールを含めた全ツールのリストに更新します。

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
For policy coverage questions and terms, use {{PolicyPlugin.SearchPolicyDocuments}}.
For sending investigation reports and claim details via email, use {{CommunicationPlugin.GenerateInvestigationReport}} and {{CommunicationPlugin.SendClaimDetailsByEmail}}.

**IMPORTANT**: When user asks to "check policy for this claim", first use GetClaimDetails to get the claim's policy number, then use GetPolicyDetails with that policy number.

Stick to the scenario above and use only the information from the tools when answering questions.
Be concise and professional in your responses.
""";
```

??? note "指示を更新する理由"
    エージェント指示は LLM に各ツールの使用タイミングを伝えます。コミュニケーションツールを追加することで、エージェントがメール送信やレポート生成を認識し、必要時に呼び出せるようになります。

<cc-end-step lab="baf5" exercise="3" step="1" />

### Step 2: OBO 設定の構成

1️⃣ `m365agents.local.yml` で `file/createOrUpdateJsonFile` アクション（約 47 行目）を探します。

2️⃣ `UserAuthorization` グループの設定で `me` 設定をアンコメントし、`OBOConnectionName`、`OBOScopes`、`Title`、`Text` を有効にします。コードは次のようになります。

```yaml
          UserAuthorization:
            DefaultHandlerName: me
            AutoSignin: true
            Handlers:
              me:
                Settings:
                  AzureBotOAuthConnectionName: "Microsoft Graph"
                  OBOConnectionName: "BotServiceConnection"
                  OBOScopes:
                    - "https://graph.microsoft.com/.default"
                  Title: "Sign in"
                  Text: "Sign in to Microsoft Graph"
```

??? note "このコードの役割"
    このコードにより、エージェントを支える Azure Bot で On-Behalf-Of (OBO) フローをサポートする設定が有効になります。
 
<cc-end-step lab="baf5" exercise="3" step="2" />

### Step 3: ユーザー認証と OBO の実装

1️⃣ `src/Agent/ZavaInsuranceAgent.cs` で `OnMessageAsync` メソッド（約 87 行目）を探します。

2️⃣ メソッドの最初の行 `await turnContext.StreamingResponse.QueueInformativeUpdateAsync( ...` の直後に次のコードを追加します。

```csharp
            // Check if user profile is already cached, if not fetch and cache it
            var userProfile = turnState.Conversation.GetCachedUserProfile();
            if (userProfile == null)
            {
                try
                {
                    // Get the access token and store it in the conversation state
                    var accessToken = await UserAuthorization.ExchangeTurnTokenAsync(turnContext, UserAuthorization.DefaultHandlerName, exchangeScopes: new[] { "https://graph.microsoft.com/.default" }, cancellationToken: cancellationToken);
                    turnState.Conversation.SetCachedOBOAccessToken(accessToken);

                    // Get the user profile and store it in the conversation state
                    userProfile = await GetUserProfile(accessToken, cancellationToken);
                    turnState.Conversation.SetCachedUserProfile(userProfile);

                    // Show current user profile information to let clients that support streaming know that we are processing the request for the current user.
                    await turnContext.StreamingResponse.QueueInformativeUpdateAsync($"⚒️ Working on your request {userProfile.DisplayName} ...", cancellationToken).ConfigureAwait(false);
                }
                catch (InvalidOperationException ex)
                {
                    System.Diagnostics.Trace.WriteLine($"Exception occurred: {ex.Message}");
                    // User is not signed in, proceed as anonymous and inform the user
                    await turnContext.StreamingResponse.QueueInformativeUpdateAsync("⚠️ Please sign in if you want to use authenticated features.", cancellationToken).ConfigureAwait(false);
                }
            }
```

??? note "このコードの役割"
    追加したコードは次の処理を担当します。
    
    - 現在のユーザープロファイルを会話から取得
    - ユーザープロファイルがキャッシュに無い場合
        - 現在のユーザーの OBO トークンを取得
        - OBO トークンを現在の会話にキャッシュ
        - Microsoft Graph でユーザープロファイルを取得
        - ユーザープロファイルを会話にキャッシュ
        - エージェントがユーザーのために作業していることを通知
    
    ユーザープロファイル取得に失敗した場合、エージェントはストリーミングでサインインを促します。
    
<cc-end-step lab="baf5" exercise="3" step="3" />

## Exercise 4: StartConversationPlugin の更新

ウェルカムメッセージを更新し、コミュニケーション機能を含むすべての機能をユーザーに案内します。

### Step 1: ウェルカムメッセージの更新

1️⃣ `src/Plugins/StartConversationPlugin.cs` を開きます。

2️⃣ `StartConversation` メソッドを探します。

3️⃣ `welcomeMessage` を完全なワークフローに置き換えます。

```csharp
var welcomeMessage = "👋 Welcome to Zava Insurance Claims Assistant!\n\n" +
                    "I'm your AI-powered insurance claims specialist. I help adjusters and investigators streamline the entire claims process - from initial assessment to final approval.\n\n" +
                    "**What I can do:**\n\n" +
                    "- Analyze claims for fraud indicators and risk patterns\n" +
                    "- Validate policy coverage and check expiration dates\n" +
                    "- Search policy documentation and claims procedures\n" +
                    "- Use Mistral AI to analyze damage photos instantly\n" +
                    "- Generate investigation reports\n" +
                    "- Send detailed claim information via email\n" +
                    "- Track claim timelines and identify processing bottlenecks\n\n" +
                    "🎯 Try this complete investigation workflow:\n" +
                    "1. \"Get details for claim CLM-2025-001007\"\n" +
                    "2. \"Check policy for this claim\"\n" +
                    "3. \"What coverage does auto insurance include?\"\n" +
                    "4. \"Analyze fraud risk for this claim\"\n" +
                    "5. \"Show damage photo for this claim\"\n" +
                    "6. \"Analyze this damage photo\"\n" +
                    "7. \"What's the claims filing procedure?\"\n" +
                    "8. \"Check compliance for this claim\"\n" +
                    "9. \"Generate investigation report for claim CLM-2025-001007\"\n" +
                    "10. \"Send the report by email\"\n\n" +
                    "Ready to complete a full claims investigation? What would you like to start with?";
```

??? note "完全なワークフロー"
    手順 1-10 は、最初の検索から最終レポート配布までの請求調査ワークフロー全体を示しています。これは実際のアジャスターが行うプロセスを反映しています。

<cc-end-step lab="baf5" exercise="4" step="1" />

## Exercise 5: コミュニケーション機能のテスト

いよいよコミュニケーション機能をテストしましょう！

### Step 1: 実行と確認

1️⃣ VS Code で **F5** を押してデバッグを開始します。

2️⃣ プロンプトが表示されたら **(Preview) Debug in Copilot (Edge)** を選択します。

3️⃣ ターミナルには通常の初期化メッセージが表示されます（新しいインデックスは作成されません）。

4️⃣ ブラウザーが開き、Microsoft 365 Copilot が表示されます。

<cc-end-step lab="baf5" exercise="5" step="1" />

### Step 2: 調査レポート生成のテスト

1️⃣ Microsoft 365 Copilot で次のように入力します。 

```text
Generate investigation report for CLM-2025-001007
```

エージェントは以下を実行します。

- `CommunicationPlugin.GenerateInvestigationReport` を使用
- ナレッジベースから包括的な請求データを収集
- ビジョン解析結果（写真が解析済みの場合）を含める
- 不正リスク評価を含める
- プロフェッショナルな markdown でレポートをフォーマット
- 推奨事項付きの構造化レポートを返却

**期待される応答:**

```
📊 Generating investigation report for CLM-2025-001007...

# 📋 Investigation Report: CLM-2025-001007
**Generated:** 2025-01-15 10:30:00 UTC
**Status:** Investigation Complete

[Comprehensive claim details including:]
- Claim overview
- Vision analysis findings
- Fraud risk assessment
- Documentation review
- Recommendations

**Report Prepared By:** Zava Insurance Claims Investigation System
**Next Steps:** Review findings and proceed with recommended claim resolution action.
```

!!! warning "OBO トークン構成エラー"
    メッセージ送信後に次のエラーメッセージが表示された場合:
    
    ```
    Sign in for 'me' completed without a token. Status=Exception/OBO for 'BotServiceConnection' is not setup for exchangeable tokens. For Token Service handlers, the 'Scopes' field on the Azure Bot OAuth Connection should be in the format of 'api://{appid_uri}/{scopeName}'.
    ```
    
    次の手順で修正してください。
    
    1. **Azure Portal** にアクセスし、**Azure Bot** を含む **Resource Group** を探す  
    2. Azure Bot リソースを開き、**Configuration** タブへ  
    3. **Microsoft Graph** Azure Active Directory v2 サービスプロバイダーをクリック  
    4. 既存の **Token Exchange URL** と **Scopes** の値をメモ（後で必要）  
    5. **Token Exchange URL** を `https://graph.microsoft.com` に変更  
    6. **Scopes** を `email openid profile User.Read Mail.Send` に変更  
    7. **Save** をクリック  
    8. 再度 **Microsoft Graph** サービスプロバイダーを選択し **Test Connection** を実行  
    9. 表示される権限要求で **Grant all consent** を選択  
    10. 設定画面に戻り、元の **Token Exchange URL** と **Scopes** を復元  
    11. もう一度 **Save**  
    12. エージェントに戻り、**ページを更新** して再テスト  

<cc-end-step lab="baf5" exercise="5" step="2" />

### Step 3: メール機能のテスト

1️⃣ 次を試してください。 

```text
Send claim details for CLM-2025-001007 by email
```

エージェントは以下を実行します。

- ナレッジベースから請求詳細を取得
- HTML 形式のメールを作成
- Microsoft Graph API で送信
- 既定であなたのメールアドレスを宛先に使用

**期待される応答:**

```
Retrieving details for claim CLM-2025-001007...
Preparing email for [your-email]@[domain].com...
Sending email via Microsoft Graph...
✅ Email sent successfully!

✅ Success: Claim details for CLM-2025-001007 have been successfully sent to [your-email]@[domain].com. 
The email includes comprehensive claim information, documentation status, timeline, and recommendations.
```

2️⃣ メール受信箱を確認し、プロフェッショナルな HTML 形式の請求詳細メールが届いていることを確認します。

3️⃣ 特定の宛先を指定して送信: 

```text
Send claim details for CLM-2025-001001 to john.doe@contoso.com
```

<cc-end-step lab="baf5" exercise="5" step="3" />

### Step 4: 完全なエンドツーエンド ワークフローのテスト

ウェルカムメッセージにある 10 段階のワークフローをテストします。

```
1. Get details for claim CLM-2025-001007
2. Check policy for this claim
3. What coverage does auto insurance include?
4. Analyze fraud risk for this claim
5. Show damage photo for this claim
6. Analyze this damage photo
7. What's the claims filing procedure?
8. Check compliance for this claim
9. Generate investigation report for claim CLM-2025-001007
10. Send the report by email
```

エージェントはすべてのプラグイン (ClaimsPlugin → PolicyPlugin → VisionPlugin → CommunicationPlugin) をシームレスに使用し、調査ワークフローを完了するはずです。

<cc-end-step lab="baf5" exercise="5" step="4" />

---8<--- "ja/b-congratulations.md"

ラボ BAF5 - コミュニケーション機能の追加 が完了しました！

学習したこと:

- ✅ メール送信とレポート生成を行う CommunicationPlugin の作成
- ✅ Microsoft Graph Mail API を統合してプロフェッショナルなメールを送信
- ✅ データを集約した包括的な調査レポートを生成
- ✅ エージェント指示を更新し、コミュニケーションワークフローを追加
- ✅ 請求調査のエンドツーエンド ワークフローをテスト

これで Zava Insurance エージェントは次の機能を備えた完全な請求処理ソリューションになりました。

- **検索**: Azure AI Search による請求およびポリシー検索  
- **分析**: 損害評価のための Mistral を用いた AI ビジョン  
- **検証**: SharePoint ポリシードキュメント検索  
- **コミュニケーション**: メールレポートと調査サマリー  

🎉 **おめでとうございます！** 本番運用可能な AI エージェントを構築しました！ 🎊

<cc-next url="../06-add-copilot-api" />

<img src="https://m365-visitor-stats.azurewebsites.net/copilot-camp/custom-engine/agent-framework/05-add-communication--ja" />