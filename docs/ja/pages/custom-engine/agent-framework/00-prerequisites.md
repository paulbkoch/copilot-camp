---
search:
  exclude: true
---
# ラボ BAF0 - 前提条件

このラボでは **Microsoft 365 Agents SDK** と **Agent Framework** を使用して開発するカスタム エンジン エージェントをビルド、テスト、デプロイするための開発環境を構築します。

このラボで学習する内容:

- Microsoft 365 環境のセットアップ
- Visual Studio Code と Microsoft 365 Agents Toolkit のインストールと構成
- 必要な Azure リソースを作成するための Azure 環境の準備
- 必要な開発ツールのインストール

!!! pied-piper "注意事項"
    これらのサンプルおよびラボは教育およびデモンストレーション目的で提供されています。本番環境で使用する場合は必ず本番レベルにアップグレードしてください。

!!! note "注"
    カスタム エンジン エージェントをインストールして実行するには、管理者権限を持つ Microsoft 365 テナントが必要です。カスタム エンジン エージェントのテストに Microsoft 365 Copilot ライセンスは不要です。

## Exercise 1 : Microsoft Teams のセットアップ

### Step 1: Teams でカスタム アプリのアップロードを有効化

既定では、エンド ユーザーはアプリを直接アップロードできず、Teams 管理者がエンタープライズ アプリ カタログにアップロードする必要があります。この手順では、M365 Agents Toolkit が直接アップロードできるようにテナントを設定します。

1️⃣ [https://admin.microsoft.com/](https://admin.microsoft.com/){target=_blank} にアクセスし、Microsoft 365 Admin Center を開きます。  

2️⃣ 管理センターの左側パネルで **Show all** を選択し、すべてのナビゲーションを表示します。次に **Teams** を選択して Microsoft Teams admin center を開きます。  

3️⃣ Microsoft Teams admin center の左側で **Teams apps** を展開し、**Setup Policies** を選択します。App setup policy の一覧が表示されるので **Global (Org-wide default) policy** を選択します。  

4️⃣ **Upload custom apps** のスイッチが **On** になっていることを確認します。  

5️⃣ ページ下部の **Save** ボタンを選択して変更を保存します。  

> 変更が反映されるまで最大 24 時間かかる場合がありますが、通常はもっと早く完了します。

<cc-end-step lab="baf0" exercise="1" step="1" />

## Exercise 2: 開発環境のセットアップ

Windows、macOS、Linux のいずれのマシンでもラボを進められますが、前提条件をインストールできる権限が必要です。権限がない場合は、別のマシンまたは仮想マシンを使用してください。

### Step 1: Visual Studio Code をインストール

1️⃣ [https://code.visualstudio.com/](https://code.visualstudio.com/){target=_blank} から Visual Studio Code をダウンロードしてインストールします。  

2️⃣ インストール後、Visual Studio Code を起動します。  

<cc-end-step lab="baf0" exercise="2" step="1" />

### Step 2: .NET 9 SDK をインストール

Microsoft 365 Agents SDK と Agent Framework のビルドおよび実行には .NET 9 SDK が必要です。

1️⃣ [https://dotnet.microsoft.com/download/dotnet/9.0](https://dotnet.microsoft.com/download/dotnet/9.0){target=_blank} から .NET 9 SDK をダウンロードしてインストールします。  

2️⃣ ターミナルを開き、次のコマンドでインストールを確認します:  

```bash
dotnet --version
```

9.0.x 以上のバージョンが表示されれば成功です。  

<cc-end-step lab="baf0" exercise="2" step="2" />

### Step 3: C# Dev Kit 拡張機能をインストール

1️⃣ Visual Studio Code で、サイドバーの **Extensions** アイコンをクリックするか `Ctrl+Shift+X` (Windows/Linux) または `Cmd+Shift+X` (Mac) を押して Extensions ビューを開きます。  

2️⃣ **C# Dev Kit** を検索し、**Install** をクリックします。  

<cc-end-step lab="baf0" exercise="2" step="3" />

### Step 4: Microsoft 365 Agents Toolkit 拡張機能をインストール

1️⃣ Extensions ビューで **Microsoft 365 Agents Toolkit** を検索し、**Install** をクリックします。  

2️⃣ インストールが完了すると、サイドバーに Microsoft 365 Agents Toolkit のアイコンが表示されます。  

<cc-end-step lab="baf0" exercise="2" step="4" />

### Step 5: Azure CLI をインストール

Azure CLI は Azure リソースのプロビジョニングと管理に必要です。

1️⃣ [https://learn.microsoft.com/cli/azure/install-azure-cli](https://learn.microsoft.com/cli/azure/install-azure-cli){target=_blank} から Azure CLI をダウンロードしてインストールします。  

2️⃣ ターミナルを開き、次のコマンドでインストールを確認します:  

```bash
az --version
```

3️⃣ 続いて Azure にサインインします:  

```bash
az login
```

<cc-end-step lab="baf0" exercise="2" step="5" />

### Step 6: DevTunnel をインストール

DevTunnel はローカル開発とデバッグのため、インターネットからローカル マシンへのセキュアなトンネルを作成します。

**Windows:**

```bash
winget install Microsoft.DevTunnel
```

**macOS/Linux:**

```bash
curl -sL https://aka.ms/DevTunnelCliInstall | bash
```

インストール確認:

```bash
devtunnel --version
```

!!! tip "DevTunnel の代替"
    Visual Studio 2022 には DevTunnel が同梱されています。すでに Visual Studio 2022 をお使いの場合、追加インストールは不要です。

<cc-end-step lab="baf0" exercise="2" step="6" />

### Step 7: Azure Functions Core Tools をインストール

ローカルで Azure Functions を実行するには Azure Functions Core Tools が必要です。

1️⃣ Azure Functions Core Tools をインストールします:

**Windows:**

```bash
winget install Microsoft.Azure.FunctionsCoreTools
```

**macOS:**

```bash
brew tap azure/functions
brew install azure-functions-core-tools@4
```

2️⃣ 次のコマンドでインストールを確認します:

```bash
func --version
```

4.x 以上のバージョンが表示されれば成功です。  

<cc-end-step lab="baf0" exercise="2" step="7" />

## Exercise 3: Azure 環境のセットアップ

このコースを完了するには、Microsoft Foundry リソースを作成し AI モデルをデプロイするための Azure サブスクリプションが必要です。

### Step 1: Azure サブスクリプションを取得

まだ Azure サブスクリプションをお持ちでない場合は、$200 のクレジットが 30 日間利用できる [Azure free account](https://azure.microsoft.com/en-us/pricing/offers/ms-azr-0044p){target=_blank} を有効化できます。

手順:

1️⃣ [Azure free account](https://azure.microsoft.com/en-us/pricing/offers/ms-azr-0044p){target=_blank} ページで **Activate** を選択します。  
2️⃣ 利用するアカウントでサインインします (演習で使用する Microsoft 365 テナント アカウントを推奨)。  
3️⃣ **Privacy Statement** にチェックを入れ、**Next** を選択します。  
4️⃣ 本人確認のため携帯電話番号を入力します。  
5️⃣ 一時的な認証のため支払い情報を入力し、**Sign up** を選択します。課金は発生しません (従量課金に移行しない限り)。  

!!! tip "ヒント: 30 日後のリソース管理"
    Azure free account は 30 日間のみ有効です。期限までに無料サブスクリプションで実行中のサービスがないか確認してください。30 日以降も利用する場合は、支出制限を解除して従量課金制サブスクリプションへアップグレードしてください。

<cc-end-step lab="baf0" exercise="3" step="1" />

### Step 2: Microsoft Foundry プロジェクトを作成しモデルをデプロイ

このラボでは、複数のモデルがデプロイされた Microsoft Foundry プロジェクトが必要です。

1️⃣ [Microsoft Foundry](https://ai.azure.com){target=_blank} にアクセスし、Azure アカウントでサインインします。  
2️⃣ **+ Create new** → **Microsoft Foundry resource** → **Next** を選択します。  
3️⃣ 推奨されたプロジェクト名のまま **Create** を選択します (3～5 分ほどで作成されます)。  

!!! tip "リージョンの選択"
    すべてのラボで必要なモデルをサポートする **France Central** リージョンを選択してください。

4️⃣ プロジェクトが作成されたら、左側の **Deployments** を開きます。  
5️⃣ **+ Deploy model** をクリックし **Deploy base model** を選択します。  
6️⃣ **gpt-4.1** を検索して選択し、**Confirm** → **Deploy** をクリックします。  

!!! important "モデルの選択"
    スムーズな体験のため **gpt-4.1** を使用してください。本ラボでは knowledge base answer synthesis を利用しており、gpt-4.1 に最適化されています。他のモデルでは予期しない動作が起こる場合があります。

7️⃣ 続いて **text-embedding-ada-002** を検索して選択し、**Confirm** → **Deploy** をクリックします。  

!!! tip "認証情報の保存"
    次のラボで必要になるため、以下の情報を安全な場所に控えておいてください。  

    - **Endpoint URL**: Project settings → Properties (例: `https://your-resource.cognitiveservices.azure.com/`)  
    - **API Key**: 「Keys and Endpoint」セクション  
    - **Model Deployment Name**: gpt-4.1 デプロイ時に付けた名前  

!!! note "追加モデル"
    Embeddings や Vision 分析用などの追加モデルや **Azure AI Search** などの他の Azure サービスは、後続のラボで必要になった際にデプロイします。

<cc-end-step lab="baf0" exercise="3" step="2" />

### Step 3: コンテンツ セーフティ フィルターを構成

保険業務では "injury"、"collision"、"damage" などの用語がデフォルトのコンテンツ フィルターに引っかかる場合があります。低いしきい値のカスタム コンテンツ フィルターを作成します。

1️⃣ Microsoft Foundry でプロジェクトを開きます。  
2️⃣ 左サイドバーで **Guardrails + Controls** → **Content filters** を選択します。  
3️⃣ **+ Create content filter** をクリックします。  
4️⃣ フィルター名に **InsuranceLowFilter** と入力します。  
5️⃣ **Input filters** を次のとおり設定します:  
   - **Violence**: Low  
   - **Hate**: Low  
   - **Sexual**: Low  
   - **Self-harm**: Low  
   - Prompt shields for jailbreak attacks: Off  
   - Prompt shields for indirect attacks: Off  

6️⃣ **Next** を選択し、**Output filters** も同様に設定します:  
   - **Violence**: Low  
   - **Hate**: Low  
   - **Sexual**: Low  
   - **Self-harm**: Low  
   - Protected material for text: Off  
   - Protected material for code: Off  
   - Groundedness (Preview): Off  

7️⃣ **Next** をクリックします。  
8️⃣ 「Apply filter to deployments」で **gpt-4.1** デプロイメントを選択します。  
9️⃣ **Replace** を選択して新しいフィルターを適用します。  
🔟 **Create filter** をクリックして完了します。  

!!! warning "必要な理由"
    保険請求には "injury"、"accident"、"collision"、"bodily harm" などの用語が含まれますが、これらはデフォルトのコンテンツ フィルターでブロックされる場合があります。しきい値を **Low** に設定することで、通常の保険用語を許可しつつ、過度なコンテンツのみをブロックできます。

!!! tip "本番環境でのデプロイ"
    本番環境では、組織のコンテンツ セーフティ ポリシーを確認し、フィルター設定を適切に調整してください。ここでの設定は開発およびテスト環境向けです。

<cc-end-step lab="baf0" exercise="3" step="3" />

---8<--- "ja/b-congratulations.md"

Lab BAF0 - Prerequisites が完了しました！

次は Lab BAF1 - Build and Run Your First Agent に進みます。**Next** を選択してください。

<cc-next url="../01-build-and-run" />

<img src="https://m365-visitor-stats.azurewebsites.net/copilot-camp/custom-engine/agent-framework/00-prerequisites--ja" />