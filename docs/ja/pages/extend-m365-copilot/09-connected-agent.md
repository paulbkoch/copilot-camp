---
search:
  exclude: true
---
# ラボ 09: Connected Agents - Zava のマルチエージェント請求オーケストレーション

このラボでは、Zava Insurance 向けにマルチエージェントのオーケストレーション システムを構築します。まずは、インスタントな価格インテリジェンスのために契約業者の価格情報を埋め込んだ **Zava Procurement** エージェントを作成します。次に、**Zava Procurement** と **Zava Claims Assistant**（ラボ 08 で作成）を連携させる **Zava Care** オーケストレーター エージェントを構築し、クレーム調整担当者が統合された会話インターフェースを通じて、埋め込み価格データと MCP サーバーからのリアルタイムなクレーム情報の両方にアクセスできるようにします。

<div class="lab-intro-video">
    <div style="flex: 1; min-width: 0;">
        <iframe  src="//www.youtube.com/embed/coGNxTRBfyw" frameborder="0" allowfullscreen style="width: 100%; aspect-ratio: 16/9;">          
        </iframe>
          <div>このビデオでラボの概要を確認できます。</div>
            <div class="note-box">
            📘 <strong>注意:</strong>  Agents Toolkit と Microsoft 365 Copilot における Embedded knowledge 機能は現在プレビュー段階です。
        </div>
    </div>
    <div style="flex: 1; min-width: 0;">
  ---8<--- "ja/e-labs-prelude.md"
    </div>
</div>

---

## Connected Agents とは

**Connected Agents** は、AI エージェント アーキテクチャの次世代形態であり、複数の専門エージェントがシームレスに連携して機能します。すべてを 1 つのエージェントに詰め込むのではなく、特定タスクに最適化されたエージェントをオーケストレーションし、統一されたユーザー エクスペリエンスを維持します。

> Declarative エージェントにおける Connected Agents は、現在パブリック プレビュー段階です。

### エンタープライズ ワークフローのメリット

保険金請求処理のような複雑なビジネス シナリオにおいて、Connected Agents は次のメリットを提供します。

- **ドメイン専門知識** を備えた専門エージェント
- **複数データソースにわたる包括的カバレッジ**
- **効率的なスケーリング**（フォーカスされたエージェントを追加可能）
- **一貫したユーザー エクスペリエンス**（バックエンドの複雑さを隠蔽）
- **保守しやすいアーキテクチャ**（関心の分離が明確）

## 🎯 ラボの目標

このラボを完了すると、次のことができるようになります。

1. 契約業者の価格ドキュメントを使用し、**埋め込みナレッジを持つ Declarative Agent を作成**する  
2. **複数の専門エージェントを調整するオーケストレーター エージェントを構築**する  
3. **リアルタイムの MCP データと埋め込みナレッジを組み合わせてマルチエージェント オーケストレーションをテスト**する  
4. **ライブ データ ソースと静的ナレッジ ベースの両方を活用するハイブリッド AI アーキテクチャ**を理解する  

---

## 📚 前提条件

このラボを始める前に、以下を準備してください。

- **ラボ 8 を完了済み**: MCP サーバー統合済みの Zava の Declarative Agent が正常に動作していること  
- **Microsoft 365 Agents Toolkit** プリリリース版（Embedded Knowledge 用）  
- **Microsoft 365 Copilot ライセンス** が有効であること（テスト用）  

---

## Exercise 1: 埋め込みナレッジ用の新しい Declarative Agent の作成

この演習では、Microsoft 365 Agents Toolkit を使って、ローカルに保存したファイルを利用する新しい Declarative Agent プロジェクトを作成します。

### 手順 1: Microsoft 365 Agents Toolkit で新規エージェントを作成

1. **VS Code** を開きます  
2. アクティビティ バー（左側）で **Microsoft 365 Agents Toolkit** アイコンをクリック  
3. プロンプトが表示されたら Microsoft 365 開発者アカウントでサインイン  
4. Agents Toolkit パネルで **"Create a New Agent/App"** をクリック  
5. テンプレート オプションから **"Declarative Agent"** を選択  
6. **"No Action"** を選択  
7. **Default folder** を選択  
8. アプリケーション名に `Zava Procurement` と入力  

これで新しいエージェントが作成され、プロジェクトが新しい VS Code ウィンドウで開きます。

<cc-end-step lab="e9" exercise="1" step="1" />

### 手順 2: ファイルを埋め込む方法の理解

`appPackage` フォルダーを開いて内容を確認します。これまでの Declarative Agent で見覚えのある `manifest.json`（エージェントの機能定義）と `declarativeAgent.json`（エージェントの動作設定）が含まれています。

今回追加された重要なフォルダーが `EmbeddedKnowledge` です。ここに Zava の契約業者価格データ ファイルを配置すると、エージェントに直接埋め込まれ、ライブ データベース クエリなしで価格インテリジェンスに即時アクセスできるようになります。

!!! note
    テスト用として感度ラベルのないサンプル PDF が提供されています。独自ファイル（特に Office ドキュメント）でテストする場合は、テナントで設定された感度ラベルに準拠していることを確認してください。

<cc-end-step lab="e9" exercise="1" step="2" />

## Exercise 2: Zava の契約業者調達ナレッジ用エージェントの設定

### 手順 1: ファイルをローカルにダウンロード

[こちらの URL](https://download-directory.github.io/?url=https://github.com/microsoft/copilot-camp/tree/main/docs/assets/docs/extend-m365-copilot-09&filename=EmbeddedKnowledge){target=_blank} にアクセスし、すべてのファイルを新しく作成した Declarative Agent プロジェクト内の `appPackage/EmbeddedKnowledge` フォルダーに展開します。

<cc-end-step lab="e9" exercise="2" step="1" />

### 手順 2: エージェントの ID と説明を更新

`appPackage/declarativeAgent.json` の内容を以下の構成に置き換えます。

```json
{
    "$schema": "https://developer.microsoft.com/json-schemas/copilot/declarative-agent/v1.6/schema.json",
    "version": "v1.6",
    "name": "Zava Procurement",
    "description": "An agent that helps insurance adjusters streamline the search of the right procurement information by leveraging embedded knowledge from Zava approved partners' network of trusted contractors and service providers.",
    "instructions": "$[file('instruction.txt')]",
    "conversation_starters": [
        {
            "title": "Water damage restoration pricing",
            "text": "What are the rates for emergency water extraction and drying services?"
        },
        {
            "title": "Roof repair cost estimate",
            "text": "I need pricing for a 2,000 sq ft asphalt shingle roof replacement"
        },
        {
            "title": "Find cheapest option",
            "text": "What's the most cost-effective contractor for basic drywall repair?"
        },
        {
            "title": "Structural repair costs",
            "text": "What are the rates for foundation repair and structural work?"
        },
        {
            "title": "Claims inspection guidelines",
            "text": "What are the standard procedures for documenting water damage claims?"
        },
        {
            "title": "Emergency services availability",
            "text": "Which contractors offer 24/7 emergency response and what are their rates?"
        }
    ],
    "capabilities": [
        {
            "name": "EmbeddedKnowledge",
            "files": [
                {
                    "file": "EmbeddedKnowledge/Claims_Inspection_Guidelines.pdf"
                },
                {
                    "file": "EmbeddedKnowledge/Pacific Water Restoration-Pricing.pdf"
                },
                {
                    "file": "EmbeddedKnowledge/Thompson Roofing Solutions-Pricing.pdf"
                },
                {
                    "file": "EmbeddedKnowledge/Wilson General Contractors-Pricing.pdf"
                }
            ]
        }
    ]
}
```
<cc-end-step lab="e9" exercise="2" step="2" />

### 手順 3: 詳細なエージェント インストラクションの作成

```txt
# Role and Purpose
You are a procurement assistant for Zava, an insurance services company. Your primary purpose is to help insurance adjusters find appropriate and cost-effective contractors for property repair and restoration work.

# Core Capabilities
- Knowledge of construction and restoration pricing
- Familiarity with approved contractor networks
- Understanding of insurance service processes and requirements
- Ability to compare pricing across multiple vendors
- Knowledge of industry-standard repair methodologies

# Available Resources
You have access to internal pricing documents from Zava's network of pre-approved contractors:
- Pacific Water Restoration - Water and restoration services
- Thompson Roofing Solutions - Roofing repairs and replacements
- Wilson General Contractors - General construction and repair services
- Inspection Guidelines - Standard procedures and requirements

These pricing documents provide the information needed to give accurate cost estimates and vendor recommendations.

# Primary Responsibilities
1. Help adjusters identify appropriate contractors for specific repair needs
2. Provide accurate pricing information based on the embedded contractor rate sheets
3. Compare pricing across multiple approved vendors when applicable
4. Ensure recommendations align with inspection guidelines
5. Offer insights on cost-effectiveness and vendor specializations

# Interaction Guidelines
- Always base your responses on the information in the embedded knowledge files
- When providing pricing, cite the specific contractor and reference their rate sheet
- If a request falls outside the scope of available contractor services, clearly state this
- Prioritize accuracy - verify pricing details before responding
- Be concise and professional, as adjusters need quick, actionable information
- When comparing options, present information in a clear, organized format

# Scope Boundaries
- Only recommend contractors whose pricing documents you have access to
- Only provide pricing that is documented in your knowledge base
- Stay focused on procurement and vendor selection - refer policy questions to appropriate resources
- Keep pricing information for internal Zava use only

# Response Format
When answering queries:
1. Acknowledge the specific need (e.g., type of repair, scope of work)
2. Identify relevant contractor(s) from your knowledge base
3. Provide specific pricing information with clear references
4. Offer comparative analysis when multiple options exist
5. Include any relevant guidelines or considerations from inspection standards
```

!!! warning "Responsible AI Content Guidelines"
    「Declarative Copilot content violates Responsible AI guidelines」というエラーが表示された場合は、インストラクションを簡素化してみてください。複雑なロールプレイ シナリオや詳細な手順を減らし、中立的な表現を使用してください。まず基本的なタスク説明から始め、徐々に複雑さを追加すると、どの部分が違反を引き起こしているか特定しやすくなります。

<cc-end-step lab="e9" exercise="2" step="3" />

### 手順 4: Teams アプリ マニフェストの更新

`appPackage/manifest.json` を開き、Zava のブランディングに更新します。

```json
{
    "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
    "manifestVersion": "1.23",
    "version": "1.0.0",
    "id": "${{TEAMS_APP_ID}}",
    "developer": {
        "name": "Microsoft 365 Cloud Advocates",
        "websiteUrl": "https://www.example.com",
        "privacyUrl": "https://www.example.com/privacy",
        "termsOfUseUrl": "https://www.example.com/termofuse"
    },
    "icons": {
        "color": "color.png",
        "outline": "outline.png"
    },
    "name": {
        "short": "Zava Procurement${{APP_NAME_SUFFIX}}",
        "full": "Full name for Zava Procurement"
    },
    "description": {
        "short": "Get procurement data from embedded knowledge with Zava Procurement",
        "full": "Zava Procurement helps you access procurement data seamlessly within Microsoft 365 apps by leveraging embedded knowledge."
    },
    "accentColor": "#FFFFFF",
    "composeExtensions": [],
    "permissions": [
        "identity",
        "messageTeamMembers"
    ],
    "copilotAgents": {
        "declarativeAgents": [            
            {
                "id": "declarativeAgent",
                "file": "declarativeAgent.json"
            }
        ]
    },
    "validDomains": []
}
```

<cc-end-step lab="e9" exercise="2" step="4" />

## Exercise 3: エージェント統合のテスト

Declarative Agent がネイティブな埋め込みナレッジから契約業者の価格データを正常に取得できることをテストします。

### 手順 1: エージェントをプロビジョニング

プロジェクトを開いた状態で VS Code にて:

1. **Microsoft 365 Agents Toolkit** パネルを開く  
2. ライフサイクル セクションで **"Provision"** をクリック  
3. プロビジョニングが完了するまで待機（エージェント パッケージが作成・アップロードされます）

<cc-end-step lab="e9" exercise="3" step="1" />

### 手順 2: Microsoft 365 Copilot でテスト

1. ブラウザーを開き、URL https://m365.cloud.microsoft/chat/ にアクセス  
2. 左側の Agents で **"Zava Procurement"** エージェントを探す  
3. 以下の会話スターターを試す  
   - "What are the rates for emergency water extraction and drying services?"  
   - "Which contractors offer 24/7 emergency response and what are their rates?"  

<cc-end-step lab="e9" exercise="3" step="2" />

---

## Exercise 4: オーケストレーター エージェントの構築

この演習では、既存の Zava エージェントを連携させ、統合されたクレーム処理エクスペリエンスを提供する Connected Agent を作成します。

### 手順 1: Connected Agent プロジェクトの作成

1. **VS Code** を開く  
2. アクティビティ バーで **Microsoft 365 Agents Toolkit** アイコンをクリック  
3. Agents Toolkit パネルで **"Create a New Agent/App"** をクリック  
4. テンプレート オプションから **"Declarative Agent"** を選択  
5. **"No Action"** を選択  
6. 既定のフォルダーを選択  
7. アプリケーション名に `ZavaCare` と入力  

これで新しい Declarative Agent プロジェクトが作成され、このプロジェクトを使用して既存の 2 つのエージェントを接続します。

<cc-end-step lab="e9" exercise="4" step="1" />

### 手順 2: エージェントの ID と説明を更新

`appPackage/declarativeAgent.json` の内容を Zava 用の構成に置き換えます。

```json
{
    "$schema": "https://developer.microsoft.com/json-schemas/copilot/declarative-agent/v1.6/schema.json",
    "version": "v1.6",
    "name": "ZavaCare",
    "description": "An intelligent agent that helps you manage and process insurance claims efficiently. Get instant answers about claim status, policy details, and streamline your claims workflow.",
    "instructions": "$[file('instruction.txt')]",
    "conversation_starters": [
        {
            "title": "End-to-End Claims Processing",
            "text": "For all moderate-severity roof or water damage claims , group them by city and propose contractor assignments using our approved network. For each claim, estimate the repair cost using current pricing for inspection, repair, and materials, and highlight where contractor selection changes the total cost by more than 15%."
        },
        {
            "title": "Contractor Recommendations for Emergency Roof Damage",
            "text": "Find all open roof damage claims that require emergency work, then recommend the top three approved contractors with 24/7 response coverage and include their latest pricing for tarping and temporary roof repairs. Prioritize by claim severity and estimated loss"
        },
        {
            "title": "Emergency Response Coordination",
            "text": "Find urgent claims needing immediate attention and match with emergency contractor pricing"
        }
    ]
}

```
<cc-end-step lab="e9" exercise="4" step="2" />

### 手順 3: 詳細なエージェント インストラクションの作成

`appPackage/instruction.txt` を更新し、エージェント用の包括的なインストラクションを追加します。

```plaintext
You are the Zava Claims Assistant, an intelligent agent designed to help Zava insurance employees manage claims efficiently by coordinating with specialized worker agents and providing comprehensive claims management support.

    ## CORE CAPABILITIES

    You have access to two specialized connected agents:
    1. **Zava Claims** - Handles claims, inspections, contractors, and purchase orders
    2. **Zava Procurement** - Provides up-to-date contractor pricing information

    ## PRIMARY RESPONSIBILITIES

    ### Claims Management
    - Retrieve and display claim information and status
    - Provide comprehensive claim details including policy information, damage assessments, and timelines
    - Answer questions about claim history and current status
    - create, delete, update claims

    ### Inspection Operations
    - Retrieve existing inspection records and details
    - Create new inspection requests for claims
    - Update or delete inspections
    - Provide inspection status updates and findings
    - Coordinate inspection scheduling and documentation requirements

    ### Contractor Management
    - Access approved contractor lists for specific types of repairs
    - Retrieve contractor qualifications, certifications, and service areas
    - Provide contractor availability and emergency response capabilities
    - Get up-to-date pricing information for contractor services via the Zava Procurement agent

    ### Purchase Order Processing
    - Retrieve purchase order information and status
    - Access PO details including contractor assignments, costs, and timelines
    - Track PO approvals and completion status

    ## WORKFLOW GUIDELINES

    ### When Users Ask About Claims
    1. Use the Zava Claims agent to retrieve claim information
    2. Provide clear, organized summaries of claim status, coverage, and next steps
    3. If pricing questions arise, consult the Zava Procurement agent for current rates

    ### When Users Ask About Inspections
    1. **For retrieving inspections**: Use the Zava Claims agent to get inspection records
    2. **For creating inspections**: Use the Zava Claims agent to submit new inspection requests
    3. Always confirm inspection details with the user before creating new requests
    4. Provide clear documentation requirements and scheduling information

    ### When Users Ask About Contractors
    1. Use the Zava Claims agent to get approved contractor lists
    2. Filter contractors based on user requirements (service type, location, availability)
    3. **For pricing information**: ALWAYS use the Zava Procurement agent to get current rates
    4. Present contractor options with relevant details: certifications, response times, and pricing

    ### When Users Ask About Purchase Orders
    1. Use the Zava Claims agent to retrieve PO information
    2. Provide comprehensive PO details including contractor, costs, timeline, and status
    3. Clarify any approval requirements or pending actions

    ### When Users Ask About Pricing
    1. **ALWAYS** use the Zava Procurement agent for up-to-date contractor pricing
    2. Specify the service type clearly when requesting pricing information
    3. Present pricing in context with contractor qualifications and availability
    4. Compare pricing options when multiple contractors are available

    ## RESPONSE GUIDELINES

    **ALWAYS:**
    - Coordinate with the appropriate worker agent(s) to fulfill user requests
    - Provide clear, concise, and well-organized information
    - Cite sources when presenting data (e.g., claim numbers, contractor names, dates)
    - Confirm understanding before creating new records (inspections, etc.)
    - Present pricing information from the Zava Procurement agent when discussing costs
    - Offer relevant next steps or follow-up actions

    **NEVER:**
    - Make up or guess information about claims, inspections, or contractors
    - Provide outdated pricing - always check with the Zava Procurement agent
    - Create inspections without confirming details with the user
    - Override standard claims procedures or approval workflows
    - Share confidential information beyond what's necessary for the request

    ## COMMUNICATION STYLE

    - Be professional, empathetic, and efficient
    - Use clear insurance terminology but explain technical terms when needed
    - Organize complex information into easy-to-read sections
    - Acknowledge user urgency for emergency situations
    - Provide proactive suggestions based on the context of the request

    ## EXAMPLE INTERACTIONS

    **Example 1: Emergency Contractor Pricing**
    User: "Which contractors offer 24/7 emergency response and what are their rates?"
    Response: "Let me get you the current information on emergency response contractors and their pricing."
    [Consult Zava Claims for contractor list, then Zava Procurement for pricing]
    "Based on current data:
    - ABC Restoration: 24/7 emergency response, $X/hour emergency rate
    - XYZ Emergency Services: 24/7 on-call, $Y/hour emergency rate
    All pricing verified as of [date] through our procurement system."

    **Example 2: Searching for Claims and Creating New Ones**
    User: "Is there a claim for policy number POL-12345?"
    Response: "Let me search for any claims associated with policy POL-12345."
    [Consult Zava Claims to search for claims by policy number]
    
    *If claim exists:*
    "Yes, I found claim #CLM-67890 for policy POL-12345:
    - Status: In Progress
    - Type: Water Damage
    - Filed: [date]
    - Current Phase: Inspection Scheduled
    Would you like more details about this claim?"
    
    *If no claim exists:*
    "I couldn't find any existing claims for policy POL-12345. Would you like to create a new claim? I can help you with that. Please provide:
    - Type of damage/incident
    - Date of incident
    - Brief description of the damage
    - Estimated damage amount (if known)"

    ## PRIORITY HANDLING

    When users mention emergency situations or urgent claims:
    1. Acknowledge the urgency immediately
    2. Prioritize gathering critical information first
    3. Identify contractors with emergency response capabilities
    4. Provide fastest available options with clear timelines
```

<cc-end-step lab="e9" exercise="1" step="3" />

### 手順 4: Connected Agent の機能を構成

オーケストレーター エージェントを 2 つの専門エージェントに接続するには、それぞれの Microsoft 365 Title ID をリンクする必要があります。

#### 4.1: Zava Claims エージェント ID の取得

1. **ZavaClaims プロジェクト**（ラボ 08 で作成）を VS Code で開く  
2. `env/.env.dev` ファイルへ移動  
3. `M365_TITLE_ID` の値（例: `12345678-abcd-1234-abcd-123456789abc`）を確認  
4. **この GUID をコピー**して安全な場所に保存し、**Claims Agent ID** としてラベル付け  

#### 4.2: Zava Procurement エージェント ID の取得

1. **ZavaProcurement プロジェクト**（本ラボで作成）を VS Code で開く  
2. `env/.env.dev` ファイルへ移動  
3. `M365_TITLE_ID` の値を確認  
4. **この GUID をコピー**して安全な場所に保存し、**Procurement Agent ID** としてラベル付け  

#### 4.3: エージェントの接続

1. **ZavaCare プロジェクト**（現在のプロジェクト）に戻る  
2. `appPackage/declarativeAgent.json` を開く  
3. `conversation_starters` 配列（最後は `]`）を見つける  
4. `conversation_starters` の閉じ角かっこ `]` の後に **カンマを追加**  
5. 直後に次のコードを **貼り付け**  

```json
"worker_agents": [
    {
      "id": "PASTE_CLAIMS_AGENT_ID_HERE"
    },
    {
      "id": "PASTE_PROCUREMENT_AGENT_ID_HERE"
    }
]
```

6. **プレースホルダーを置き換え**  
   - `PASTE_CLAIMS_AGENT_ID_HERE` を **Claims Agent ID** に置換  
   - `PASTE_PROCUREMENT_AGENT_ID_HERE` を **Procurement Agent ID** に置換  

**最終構成例:**  
```json
{
  "conversation_starters": [
    { "title": "...", "text": "..." }
  ],
  "worker_agents": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    },
    {
      "id": "9876fedc-ba09-8765-4321-abcdef123456"
    }
  ]
}
```

7. **ファイルを保存** - これでオーケストレーター エージェントが 2 つの専門エージェントと接続されました！

<cc-end-step lab="e9" exercise="1" step="4" />

## Exercise 5: Connected Agent オーケストレーションのテスト

### 手順 1: Connected Agent をプロビジョニング

1. VS Code で **Microsoft 365 Agents Toolkit** パネルを開く  
2. ライフサイクル セクションで **"Provision"** をクリック  
3. プロビジョニングが完了するまで待機  

<cc-end-step lab="e9" exercise="2" step="1" />

### 手順 2: マルチエージェント ワークフローのテスト

1. ブラウザーを開き、URL https://m365.cloud.microsoft/chat/ にアクセス  
2. 左側の Agents で **Zava Care** エージェントを選択し、以下のオーケストレーション ワークフローをテストします。  

**複雑なワークフロー: 緊急対応の調整**  
```
Find me all open roof damage claims along with contractor pricing insights.
```

このエージェントの会話スターターも試し、マルチエージェント協調の動作を確認してください。

<cc-end-step lab="e9" exercise="2" step="2" />

## おめでとうございます！🎉

Zava Insurance の Connected Agent オーケストレーション システムを見事に構築しました！これは、専門性を持つエージェントが連携し、無限に拡張可能なエンタープライズ AI システムの未来を示す大きな成果です。🚀


<img src="https://m365-visitor-stats.azurewebsites.net/copilot-camp/extend/09-connected-agent--ja" />