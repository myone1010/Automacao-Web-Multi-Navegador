# Automacao-Web-Multi-Navegador
Desenvolvimento de Ferramenta para Automação Web Multi-Navegador

## O que construir 

### Objetivo funcional (assistido, não invasivo)

1. Você seleciona o mercado/seleção no Site A e define a stake.
2. A extensão captura **evento, mercado, seleção, odds alvo e stake**.
3. Para os Sites B/C:

   * Abre o mercado (via deeplink do próprio site quando existir) **ou** te guia até ele;
   * **Pré-preenche** o campo de stake;
   * **Realça** a seleção equivalente;
   * Mostra um **checklist** (odds mínima, linha/handicap, confirmação de jogo);
   * Você revisa e clica no botão do próprio site (“Place/Confirm”).

> Sem automação de login, sem cliques automáticos em “Place Bet”, sem alterar IP/localização, sem contornar limites.

---

## Arquitetura da extensão (Manifest V3)

**Componentes**

* `manifest.json`: permissões mínimas e `host_permissions` apenas para domínios alvo.
* **Content scripts (um por site)**: conhecem o DOM daqueles sites; extraem/definem valores.
* **Service worker (background)**: coordena mensagens, armazena o “parent bet” e envia para as abas dos outros sites.
* **Popup/Options UI**: configurações (ligas favoritas, slippage permitido, mapeamentos de mercados entre casas).

**Fluxo técnico**

1. No Site A, o content script detecta o slip aberto (via `MutationObserver`), lê seleção/linha/odds, lê a stake digitada e envia ao background:
   `chrome.runtime.sendMessage({type: 'PARENT_BET_CREATED', payload})`.
2. O background salva em `chrome.storage.session` e envia broadcast para outras abas/domínios suportados.
3. Nos Sites B/C, os content scripts recebem a mensagem, procuram o mercado/seleção equivalente (usando seu **mapeamento de mercados**), preenchem **apenas** a stake, realçam a seleção e exibem um pequeno **overlay** com:

   * seleção/linha/odds alvo;
   * botão “Copiar odds alvo”;
   * checklist de conferência;
   * **nenhum auto-click** em “Place Bet”.
4. Se a odd ao vivo divergir além do seu limite de slippage, o overlay alerta e **não** habilita o “ok” local — você decide manualmente.

**Permissões recomendadas**

* `"host_permissions": ["https://*.betano.*/*", "https://*.bet365.*/*", "https://*.superbet.*/*"]` (ajuste às TLDs relevantes).
* Nada de `"tabs"` amplo se você puder usar `scripting` e `activeTab`.
* Sem `cookies`/`identity` a menos que estritamente necessário para seu overlay local.

---

## Como lidar com diferenças entre sites

* **Mapeamento de mercados**: mantenha uma pequena tabela normalizada, por ex.:

  ```
  { normalized: "FT_1X2_HOME",
    betano: { marketId: "...", selector: "..."},
    bet365: { ... },
    superbet: { ... } }
  ```
* **Seletores frágeis**: use seletores robustos (atributos data-\* quando existirem).
* **Tolerância a mudanças**: envolva tudo em “try/catch” e exiba um fallback: “Abra o mercado X manualmente; clique aqui para copiar stake e odds alvo”.

---

## Multi-aba vs multi-navegador vs multi-PC

* **Mais prático:** **múltiplos perfis do Chrome** no **mesmo navegador/mesma máquina** (cada perfil = uma sessão). Você abre uma janela por perfil.
* **Várias abas no mesmo perfil**: útil apenas se todas forem a **mesma conta** — para contas diferentes, use **perfis diferentes**.
* **Múltiplos navegadores** (Chrome + Edge + Firefox): só se você já usa perfis neles; manutenção dos content scripts duplica.
* **Múltiplos PCs/VMs**: só quando houver requisito operacional (por exemplo, políticas internas de um operador). Aumenta custo e orquestração.

---

## Esqueleto mínimo (ilustrativo, assistivo)

**manifest.json**

```json
{
  "manifest_version": 3,
  "name": "Bet Replicator (Assistive)",
  "version": "0.1.0",
  "permissions": ["storage", "scripting"],
  "host_permissions": [
    "https://*.betano.*/*",
    "https://*.bet365.*/*",
    "https://*.superbet.*/*"
  ],
  "background": { "service_worker": "background.js" },
  "action": { "default_popup": "popup.html" },
  "content_scripts": [
    {
      "matches": ["https://*.betano.*/*"],
      "js": ["sites/betano.js"],
      "run_at": "document_idle"
    },
    {
      "matches": ["https://*.bet365.*/*"],
      "js": ["sites/bet365.js"],
      "run_at": "document_idle"
    },
    {
      "matches": ["https://*.superbet.*/*"],
      "js": ["sites/superbet.js"],
      "run_at": "document_idle"
    }
  ]
}
```

**background.js (resumo)**

```js
let parentBet = null;

chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  if (msg?.type === "PARENT_BET_CREATED") {
    parentBet = msg.payload; // {sport, league, event, market, selection, line, targetOdds, stake}
    chrome.storage.session.set({ parentBet });
    // broadcast para todos os content scripts
    chrome.tabs.query({}, tabs => {
      for (const t of tabs) {
        chrome.tabs.sendMessage(t.id, { type: "SYNC_PARENT_BET", payload: parentBet });
      }
    });
    sendResponse({ ok: true });
  }
});
```

**sites/betano.js (padrão assistivo)**

```js
function extractCurrentSlip() {
  // LER dados do slip aberto de forma não-invasiva (DOM do site)
  // retornar {sport, league, event, market, selection, line, targetOdds, stake}
}

function assistFill(payload) {
  // Encontrar mercado/seleção equivalentes neste site (mapeamento seu)
  // Preencher apenas o campo de stake e destacar a seleção
  // Renderizar overlay com checklist e NUNCA clicar automaticamente em "Place Bet"
}

chrome.runtime.onMessage.addListener((msg) => {
  if (msg?.type === "SYNC_PARENT_BET") {
    assistFill(msg.payload);
  }
});

// Se este é o "site mãe", quando detectar o slip pronto:
const observer = new MutationObserver(() => {
  const pb = extractCurrentSlip();
  if (pb?.stake && pb?.selection) {
    chrome.runtime.sendMessage({ type: "PARENT_BET_CREATED", payload: pb });
  }
});
observer.observe(document.documentElement, { childList: true, subtree: true });
```

> Observação: **não** inclua auto-submit/auto-click. Mantenha o usuário no comando.

---

## Limitações práticas (importantes)

* DOM de casas de aposta muda com frequência; sua extensão precisa de **feature flags** por site e **failsafe**.
* Muitas casas têm CSP forte e elementos shadow DOM; às vezes será preciso **inserir** um `iframe`/shadow host só para overlay.
* Odds ao vivo variam; implemente **slippage** mínimo e destaque mudança antes da confirmação.
* Evite qualquer interação com captchas, 2FA ou fluxos de login — isso **não** será automatizado.

---

## Próximos passos que eu recomendo

1. **Definir escopo inicial**: 1 esporte (futebol), 3 mercados (FT 1x2, AH, OU), 3 casas (Betano, Bet365, Superbet).
2. **Mapeamentos de mercado** (como identificar “FT 1x2 – Home/Draw/Away” em cada site).
3. **Protótipo** do content script em **modo assistivo**: preencher stake, realçar seleção e abrir deeplink quando existir.
4. **Testes de usabilidade**: medir quantos cliques a menos você dá por casa e quanto tempo economiza, mantendo conformidade.
