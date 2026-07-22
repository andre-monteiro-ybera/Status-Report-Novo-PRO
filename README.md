# Painel de Andamento — Novo PRO

Dashboard de acompanhamento do projeto **Novo PRO** (desenvolvido pela Global, equipe terceira), voltado à visibilidade para stakeholders. Mostra status de features, PBIs, horas alocadas, decisões e um indicador de confiança na entrega até o prazo (16/10/2026).

---

## 1. Visão geral da arquitetura

```
Planilha MVP_PRO (Google Sheets)
        │
        ▼
   N8N (webhook)  ──lê as 3 abas, monta o JSON──▶  endpoint GET
        │
        ▼
  dashboard-novo-pro.html  ──fetch no endpoint──▶  renderiza as 4 abas
   (publicado no Vercel)        (botão "Atualizar")
```

- **Fonte de verdade:** a planilha `MVP_PRO` no Google Sheets. É o único lugar que se edita no dia a dia.
- **Transformação:** um workflow N8N lê as abas e devolve um JSON no formato que o dashboard espera.
- **Apresentação:** um HTML estático (sem backend) que busca esse JSON e desenha.

O dashboard **não** tem servidor próprio: é um arquivo HTML que roda no navegador e chama o endpoint do N8N.

---

## 2. Arquivos do projeto

| Arquivo | O que é |
|---|---|
| `dashboard-novo-pro.html` | O dashboard (casca). Lê os dados do endpoint N8N **ou** do `dados.js`. É o que vai pro Vercel. |
| `dados.js` | Cópia local dos dados (`window.DASHBOARD_DATA`). Serve de **fallback** se o endpoint estiver fora do ar. |
| `dashboard-novo-pro-REVISAO.html` | Versão única com dados embutidos, só para revisar offline. Não vai pro Vercel. |
| `n8n-code-node.js` | O código do nó "Code" do N8N que transforma a planilha no JSON. |
| `n8n-endpoint-guia.md` | Guia de montagem do workflow N8N. |

---

## 3. As 4 abas do dashboard

1. **Visão executiva** — linha do tempo até o prazo, indicador de confiança na entrega, cards por status de feature e funil de PBIs.
2. **Escopo & features** — tabela por persona (Distribuidor, Administrador, Representante, Profissional) com status, refinamento e PBIs. No celular, vira blocos empilhados.
3. **Horas alocadas** — total, por mês, por papel e por pessoa (dados da Global).
4. **Decisões** — log cronológico das decisões do projeto.

---

## 4. Como funciona a fonte de dados (2 modos)

No topo do `<script>` do `dashboard-novo-pro.html`:

```js
const ENDPOINT = "";
```

- **`ENDPOINT` vazio** → usa os dados embutidos no `dados.js` (modo offline/estático).
- **`ENDPOINT` com a URL do webhook** → busca ao vivo do N8N; o botão **Atualizar** recarrega.

```js
const ENDPOINT = "https://n8n-homolog.b2.club/webhook/novo-pro-dados";
```

**Fallback automático:** se o endpoint falhar (N8N fora do ar, sem rede), o dashboard exibe a última versão embutida com o aviso "sem conexão — exibindo última versão", em vez de quebrar.

---

## 5. Rotina de atualização semanal

O dado é atualizado **manualmente na planilha** (não há sincronização automática com o Azure DevOps neste momento).

1. Atualize a planilha **MVP_PRO** (status das features, PBIs em validação, horas, decisões).
2. Se estiver no **modo endpoint (ao vivo):** basta abrir o dashboard e clicar em **Atualizar** — ele relê a planilha via N8N.
3. Se estiver no **modo estático (`dados.js`):** gere um novo `dados.js` a partir da planilha e publique de novo no Vercel.

> Enquanto a cadência é semanal e a edição é manual, o modo ao vivo cobre bem. Ponto de atenção: no modo ao vivo, o stakeholder pode pegar a planilha no meio de uma edição. Se for um dia de mudança grande, edite numa cópia e só reflita quando fechar.

---

## 6. O workflow N8N

**Nome:** `GET - Novo PRO Status Report`

**Cadeia de nós:**

```
Webhook → Projeto Completo → Horas Alocadas → Decisões → Code in JavaScript → Respond to Webhook
```

- **Webhook** — método GET, path `novo-pro-dados`, Authentication **None**, Respond = "Using Respond to Webhook Node".
- **Projeto Completo / Horas Alocadas / Decisões** — três nós Google Sheets (Get Row(s)), lendo as três abas da MVP_PRO. Todos com **Execute Once ligado** e credencial Service Account (`n8n-metricas-fluxo`). Os nomes dos nós precisam ser exatamente esses (o Code os referencia por nome).
- **Code in JavaScript** — conteúdo do `n8n-code-node.js`. Monta o objeto `DASHBOARD_DATA`.
- **Respond to Webhook** — Respond With **JSON**, Response Body `{{ $json }}`, header **`Access-Control-Allow-Origin: *`** (obrigatório para o navegador não bloquear por CORS).

**Importante:**
- O workflow precisa estar **Active** para a Production URL responder.
- A planilha precisa estar **compartilhada com o e-mail da Service Account** (papel Leitor).

### Colunas que o Code espera

**Aba Projeto Completo:** `Perfil / Épico`, `Funcionalidade/ Feature`, `Status`, `Refinamento`, `PBIs Totais`, `PBIs em Validação (Tests)`, `PBIs Concluídos`.

**Aba Horas Alocadas:** `Colaborador`, `Tipo`, `Horas Alocadas`, `Data de Início`.

**Aba Decisões:** `Data`, `Descrição`.

> O Code tolera pequenas variações de grafia nas colunas de horas e datas no formato brasileiro (dd/mm/aaaa). Se renomear colunas na planilha, confira o bloco `COL` / `COLH` / `COLD` no início do `n8n-code-node.js`.

---

## 7. Deploy no Vercel

Publique **os dois arquivos juntos, na mesma pasta**: `dashboard-novo-pro.html` e `dados.js` (o HTML procura o `dados.js` ao lado dele).

- No modo endpoint, o `dados.js` funciona como fallback; mantê-lo atualizado de vez em quando é bom.
- Para atualizar, republique.

---

## 8. Indicador de confiança na entrega

Não é probabilidade — é um **sinal por regras auditáveis**, média de três fatores (0–100), cada um rastreável à planilha:

- **Ritmo** — progresso (% pronto p/ validação) vs. % do tempo decorrido.
- **Maturidade do escopo** — % de PBIs refinados (Realizado) vs. estimados.
- **Escopo iniciado** — % de features que saíram do "não iniciado".

Faixas: **≥70 Alta · 50–69 Média · <50 Baixa**. Pesos configuráveis no bloco `CONF` do HTML. A evolução natural é trocar por previsão probabilística (Monte Carlo) quando houver histórico de conclusão.

---

## 9. Troubleshooting (problemas já enfrentados)

| Sintoma | Causa | Solução |
|---|---|---|
| `horas` vem vazio no JSON | Nome de coluna errado no Code (`"Horas"` em vez de `"Horas Alocadas"`), ou versão antiga do Code colada | Colar a versão atual do `n8n-code-node.js`; conferir bloco `COLH` |
| Gráfico por mês vazio, mas por pessoa OK | Data BR (`27/04/2026`) não parseada pelo `new Date()` | Já resolvido com a função `parseData` na versão atual |
| Webhook retorna 404 / "not registered" | Workflow não está **Active** | Ligar o toggle Active; usar a **Production URL** (sem `-test`) |
| Dashboard mostra "sem conexão" | Endpoint fora do ar, CORS ausente, ou URL errada | Conferir header CORS no Respond; testar a URL direto no navegador |
| Cota do Google estourada ("too many requests") | Muitos Execute seguidos, ou nós rodando 1x por item | Esperar 1 min; garantir **Execute Once** em todos os Google Sheets |
| Planilha não aparece no N8N | Service Account sem acesso | Compartilhar a MVP_PRO com o e-mail da Service Account (Leitor); usar "By URL/ID" se o dropdown não listar |

---

## 10. Pendências / decisões em aberto

- **Ambiente do N8N:** hoje aponta para `n8n-homolog` (homologação). Definir se é estável para produção ou se deve migrar para um ambiente definitivo antes de fixar a URL.
- **Coluna "Finalizadas" (PBIs Concluídos):** hoje é 0 em tudo (nada em "Done"). Quando passar a ser preenchida na planilha, o dashboard já reflete sozinho (colunas e ordenação já preparadas).
- **Divergência de escopo:** confirmar se o perfil Profissional tem mesmo 5 features (a planilha ao vivo mostra 5; uma revisão anterior mostrava mais).
- **Segurança:** o dado é público por URL (decisão consciente). Se precisar restringir, exige um proxy para esconder credencial — o dashboard-HTML puro não protege segredo.
- **Evolução:** previsão probabilística (Monte Carlo) usando throughput real; possível ingestão direta do Azure DevOps via N8N, dispensando digitação manual de PBIs.

---

*Gerado como parte da construção do painel. Mantenha este README junto dos arquivos no repositório/Vercel.*
