<div class="cc-lab-toc e-path">
    <img src="/copilot-camp/assets/images/path-icons/E-path-heading.png"></img>
    <div>
        <p>Microsoft 365 が AI モデルとオーケストレーションを提供する宣言型エージェントを構築したい場合は、次のラボを実施してください。</p>
        <ul id="lab-toc">
            <li><strong><a href="/copilot-camp/pages/extend-m365-copilot/index">🏁 ようこそ</a></strong></li>
            <li><strong>🔧 セットアップ</strong>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/00-prerequisites">ラボ E0 - セットアップ</a></li>
                </ul>
            </li>
            <li><strong>🧰 宣言型エージェントの基礎</strong>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/01-typespec-declarative-agent">ラボ E1 - TypeSpec を使用して宣言型エージェントを構築する</a>
                    </li>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/01a-geolocator">ラボ E1a - 位置情報ゲーム</a></li>
                </ul>
            </li>
            <li><strong>🛠️ API のゼロからの構築と統合</strong>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/02-build-the-api">ラボ E2 - API を構築する</a></li>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/03-add-declarative-agent">ラボ E3 - 宣言型エージェントと API を追加する</a></li>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/04-enhance-api-plugin">ラボ E4 - API とプラグインを強化する</a></li>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/05-add-adaptive-card">ラボ E5 - アダプティブカードを追加する</a></li>
                     <li><a href="/copilot-camp/pages/extend-m365-copilot/06a-add-authentication-ttk">ラボ E6a - 認証を追加する</a></li>
                </ul>
            </li>       
            <li><strong>🔌 統合</strong>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/07-add-graphconnector">ラボ E7 - Copilot コネクタを追加する</a></li>
                </ul>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/08-mcp-server">ラボ E8 - 宣言型エージェントを MCP サーバーに接続する</a></li>
                </ul>
                <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/09-connected-agent">ラボ E9 - 接続されたエージェント</a></li>
                </ul>
                 <ul>
                    <li><a href="/copilot-camp/pages/extend-m365-copilot/10-mcp-auth">ラボ 10 - 宣言型エージェントを OAuth で保護された MCP サーバーに接続する</a></li>
                </ul>
            </li>
        </ul>
    </div>
</div>

<script>
(() => {

// This script decorates the table of contents with a "you are here" indicator.
const toc = document.getElementsByClassName('cc-lab-toc');
for (const div of toc) {
    const lis = div.querySelectorAll('li');
    for (const li of lis) {
        const anchor = li.querySelector('a');
        if (anchor) {            // Get the last segment of the current URL path
            const currentPath = window.location.pathname.slice(0, -1).split('/').pop();

            // Get the last segment of the link path
            const linkPath = anchor.getAttribute('href').split('/').pop().replace('.md', '');

            // Compare the last segments
            if (currentPath === linkPath) {
                const existingSpan = document.querySelector('span.you-are-here');
                if (existingSpan) {
                    existingSpan.remove();
                }
                const span = document.createElement("span");
                span.innerHTML = "YOU&nbsp;ARE&nbsp;HERE";
                span.className = "you-are-here";
                li.appendChild(span);
            }
        }
    }
}
})();
</script>