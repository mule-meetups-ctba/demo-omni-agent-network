# demo-omni-agent-network

Projeto **Agent Network 2.0** (MuleSoft Agent Fabric) da demo do MuleSoft Meetup. Define a rede
que coordena o agente A2A, as tools MCP e o LLM — e o **broker** que decide quando resolver algo
sozinho e quando delegar.

- **Schema:** `agentNetwork: 2.0.0` (YAML `registry` + `context` + `brokers`)
- **Broker:** AgentScript (`brokers/demo-omni-broker.agent`), dialeto `AGENTFABRIC=1.0`
- **LLM:** Azure OpenAI `gpt-5.4-mini` (endpoint `/responses`)
- **Protocolo entre agentes:** A2A 1.0

> Parte de uma demo com quatro repositórios, que só faz sentido completa:
>
> - [`demo-order-support-api`](https://github.com/mule-meetups-ctba/demo-order-support-api) — API REST de pedidos + consulta do pagamento no Stripe
> - [`demo-support-mcp-server`](https://github.com/mule-meetups-ctba/demo-support-mcp-server) — MCP server com as quatro tools de suporte
> - [`demo-support-agent`](https://github.com/mule-meetups-ctba/demo-support-agent) — agente A2A com tool-calling
>
> A arquitetura, o passo a passo de deploy e as políticas de gateway estão
> descritos no artigo que acompanha a demo.

## Estrutura

```
.
├── agent-network.yaml     # registry (assets) + context (conexões) + brokers
├── exchange.json          # metadados Exchange + variáveis (com flag secret)
└── brokers/
    └── demo-omni-broker.agent  # o broker, em AgentScript
```

**`registry`** descreve *o que existe* (agentes, MCP servers, LLMs — que viram assets no
Exchange). **`context`** descreve *como se conectam* (URLs, autenticação, políticas). A separação
existe porque a mesma definição de agente pode ser conectada de formas diferentes em ambientes
diferentes.

## Detalhes que não se adivinham

Três coisas que só a documentação resolve — e que custam tempo se erradas:

- **Chave da interface do agente:** `a2a` para protocolo 1.x, `a2a_v03` para 0.3.x. Isso permite
  que a rede converse com agentes em versões diferentes do protocolo ao mesmo tempo — útil numa
  migração incremental.
- **`platform` do LLM:** os valores aceitos são `Gemini`, `OpenAI` e `AzureOpenai` — repare na
  grafia de `AzureOpenai`. No AgentScript, Azure OpenAI usa `kind: "OpenAI"`; só o Bedrock
  OpenAI difere, com prefixo `openai.` no nome do modelo.
- **`apikey-client-credentials` usa objetos, não strings.** Em `context.connections.<id>.authentication`,
  `clientId` e `clientSecret` são `{value, name}` — `value` é o `${...}`, `name` é o header enviado
  (`client_id`/`client_secret`, os defaults do Client ID Enforcement). Passar string crua dá ~30
  violações AMF em cascata (`must be object`, `must match a schema in anyOf`) e o `build` falha.
- **URL da connection LLM é só a base.** Para Azure OpenAI, termina em `/openai/v1`, sem
  `/responses` e sem barra final — o broker monta o path sozinho. Sobra de path aí é a causa
  clássica de redirect/404.

E uma do broker: **cada tool MCP é uma `action` explícita** com o `tool_name` exato. É mais
verboso que referenciar o servidor inteiro, mas dá controle fino sobre o que o broker pode usar.

## Guided determinism

O broker separa o que exige julgamento probabilístico do que exige controle determinístico:

```
trigger ──> generator classifyIntent          (1 chamada de LLM, sem actions)
                 │
            router intentRouter               (determinístico, zero tokens)
                 ├── order_status ──> subagent orderStatusLookup   (só a tool de leitura)
                 │                        └──> echo statusResponse
                 └── otherwise ────> orchestrator resolveSupportRequest
                                          └──> router refundRouter (determinístico)
                                                 ├── executor refundPayment ──┐
                                                 └───────────────────────────> executor notifyTeam
                                                                                 └──> echo actionResponse
```

O LLM decide *o quê*; o grafo garante *a ordem* e guarda o que é irreversível.

**Duas coisas o modelo não decide**, porque são nós do grafo e não frases do prompt:

| | Como era | Como é |
|---|---|---|
| Avisar o time | frase no system prompt — o modelo às vezes encerrava sem chamar | `executor notifyTeam`, sempre no caminho de ação |
| Mover dinheiro | `refund_payment` exposta ao LLM em `reasoning.actions`, guardrail em prosa | `executor refundPayment`, atrás do `refundRouter` |

O reembolso é o caso que mais paga essa diferença. O orchestrator **não tem** a tool de
reembolso: ele só preenche outputs estruturados (`refundDecision`, `alreadyRefunded`,
`paymentIntentId`, `refundReason`). Quem decide é o router; quem executa é o executor — um único
ponto no broker que fala com o Stripe, alcançável por uma rota só.

Antes havia **dois** caminhos para reembolsar: a tool MCP direta e a delegação ao
`order_support_agent`, que também reembolsa. Nada impedia o modelo de fazer os dois na mesma
rodada.

> **A ordem das rotas do `refundRouter` é a regra.** Elas são avaliadas de cima para baixo e a
> primeira que casar vence. A rota de bloqueio (`alreadyRefunded == "yes"`) vem **primeiro**, para
> ganhar de `refundDecision == "refund"`. Inverter as duas reintroduz o reembolso em duplicidade
> sem gerar nenhum erro de compilação.

### ⚠️ Pendente: as políticas ainda não estão no YAML

A política recomendada para o broker é **Client ID Enforcement + A2A PII Detector + Rate Limiting /
Spike Control** no ingress do broker. Hoje elas não estão declaradas aqui — aplicadas à mão no API
Manager, **somem no próximo redeploy ou undeploy**.

Tentei declarar e o `agent-network project build` recusou. Fica registrado o que já está resolvido
e onde exatamente trava, para não refazer a investigação:

**Resolvido.** Coordenadas no Exchange (org pública da MuleSoft,
`68ef9520-24e9-4cf2-b2f5-620025690913`) — Omni Gateway é Flex, então as variantes `-flex`:

| Política | assetId | version |
|---|---|---|
| Client ID Enforcement | `client-id-enforcement-flex` | 1.2.0 |
| A2A PII Detector | `a-two-a-pii-detector-flex` | 1.1.1 |
| Rate Limiting | `rate-limiting-flex` | 1.2.2 |
| Spike Control | `spike-control-flex` | 1.2.0 |

```bash
anypoint-cli-v4 exchange asset list "Client ID Enforcement" --limit 20 --output json
```

> O termo de busca é **argumento posicional**, não `--search`. E use `--output`, nunca `-o`: o
> short flag colide com `--offset` e o erro que aparece é `Expected an integer but received: json`.

**Resolvido.** Shape da declaração e do binding (doc oficial):

```yaml
context:
  policies:
    clientIdEnforcement:
      ref: { name: client-id-enforcement-flex }
      configuration: { }

brokers:
  demo-omni-broker:
    interfaces:
      a2a:
        policies:
          inbound:
            - ref: { name: clientIdEnforcement }
          outbound: []
```

O mesmo shape (`policies.inbound` / `policies.outbound`) vale em `context.connections.<id>`.

**Onde trava.** Com a política acima em `exchange.json.dependencies`, o build falha:

```
Asset 68ef9520-24e9-4cf2-b2f5-620025690913/client-id-enforcement-flex/1.2.0/ is unreachable
Cannot resolve reference client-id-enforcement-flex of kind policy
```

**Não é permissão.** Com as mesmas credenciais, o asset responde 200 no Exchange Maven de consumo:

```bash
curl -H "Authorization: Bearer $TOKEN"   https://maven.anypoint.mulesoft.com/api/v3/maven/68ef9520-24e9-4cf2-b2f5-620025690913/client-id-enforcement-flex/1.2.0/client-id-enforcement-flex-1.2.0.pom
```

Também não é o `classifier`: com `policy-implementation`, com `binary`, ou sem classifier nenhum, a
mensagem é idêntica — e a URI do erro sempre termina em `/`, como se o plugin montasse o
identificador por conta própria e ignorasse o que está na dependência.

**Hipótese mais provável:** `context.policies` serve para políticas publicadas na **sua própria
org** (custom policies), e as embutidas da MuleSoft entram por instância no API Manager. Reforça
isso o fato de o Omni Gateway já aplicar sozinho um conjunto delas a cada deploy (A2A Agent Card,
Tracing, Agent Connection Telemetry).

**Enquanto não fecha:** aplique as três no console e **reaplique depois de cada redeploy** — é
justamente o que o `CLAUDE.md` avisa que se perde. Vale testar no Anypoint Code Builder, que pode
resolver policies por outro caminho que o plugin da CLI.

## Variáveis

Todo `${...}` do YAML é declarado em `exchange.json` e preenchido no deploy — nunca no arquivo.
Ver [SECRETS.md](SECRETS.md) para a lista completa e o checklist de rotação.

## Publicando e deployando

Validação, publish e deploy são feitos pelo **Anypoint CLI Agent Fabric plugin**
(`npm i -g mulesoft-anypoint-cli-agent-fabric-plugin`). Autenticação é 100% por variável de
ambiente — nunca inline:

```bash
export ANYPOINT_CLIENT_ID='<connected app client_id>'
export ANYPOINT_CLIENT_SECRET='<connected app client_secret>'
export ANYPOINT_ORG='a75e4983-9a43-49c7-bfaa-b2e8d2b70a95'   # BG "Research"
export ANYPOINT_ENV='Prod'                                    # nome, NAO o UUID (ver nota abaixo)

anypoint-cli-agent-fabric-plugin agent-network project build   --path .
anypoint-cli-agent-fabric-plugin agent-network project publish --path .
```

> **`ANYPOINT_ORG` tem que ser igual ao `groupId`/`organizationId` do `exchange.json`.** Divergir
> dá `401`/`403` no publish.
>
> **`ANYPOINT_ENV` é o NOME do environment, não o UUID.** Passar o id dá
> **"Environment not found"** — exatamente o mesmo erro de um nome inexistente, o que manda você
> investigar o lado errado. A org tem dois: `Dev` (sandbox) e `Prod` (production, sufixo de DNS
> `-ab12cd`). Confira com `anypoint-cli-v4 account environment list --output json`.
>
> **O plugin do Agent Fabric não lê a config do `anypoint-cli-v4`** — só as variáveis de ambiente.
> O `anypoint-cli-v4` funciona com a config gravada; o plugin, não. Se o build falhar com
> *"No authentication mechanism was provided"*, é isso.
>
> **`publish` ignora o `version` do `exchange.json`** e incrementa a partir do último publicado.
> Em 13/set/2026 o repo dizia `1.0.0`, o Exchange já tinha `2.0.0`, e o publish gerou `2.0.1`.
> Realinhe o arquivo depois de publicar.

O `build` valida o `agent-network.yaml` (schema AMF) **e** o `.agent` (dialeto AGENTFABRIC), e
gera `target/`. O `publish` sobe 5 assets: os 3 do registry (`demoSupportAgent`, `mcpServer`,
`azureOpenAi`), o broker (`demo-omni-broker`) e a rede (`demo-omni-agent-network`).

### Valores de Prod (private space e DNS do seu ambiente)

O Omni Gateway de Prod é um app próprio (o sufixo do hostname muda por environment). As três
variáveis de URL vão para o **gateway**, não para os apps direto — é o gateway que valida o
`client_id`/`client_secret` que a connection injeta (policy `credential-injection-api-key`) e que
aplica PII Detector / Rate Limiting.

```bash
anypoint-cli-agent-fabric-plugin agent-network project deploy --path .   --environment Prod -g <nome-do-gateway>   --property demoSupportAgent.url:https://<ingress-gw-host>/techwave-order-support-agent   --property mcpServer.url:https://<egress-gw-host-interno>/techwave-support-mcp-server   --property azureOpenAi.url:https://<recurso>.services.ai.azure.com/openai/v1   --property demoSupportAgent.clientId:<contrato broker-to-agent>   --property demoSupportAgent.clientSecret:<...>   --property mcpServer.clientId:<contrato broker-to-mcp-server>   --property mcpServer.clientSecret:<...>   --property azureOpenAi.apiKey:<...>
```

Os apps por trás do gateway (DNS **interno**, só resolve dentro do private space) são o *backend*
de cada API instance — não vão no `agent-network.yaml`:

| API instance no gateway | Upstream (backend) |
|---|---|
| `/techwave-order-support-agent` | `https://<app-support-agent-host>.internal.cloudhub.io/techwave-order-support-agent/` |
| `/techwave-support-mcp-server` | `https://<app-support-mcp-server-host>.internal.cloudhub.io/` |
| `/techwave-order-support-api` | `https://<app-order-support-api-host>.internal.cloudhub.io/` |

> **Duas regras nessa tabela, as duas descobertas na marra.**
>
> 1. **O Upstream termina em `/`.** O gateway monta o destino como `Upstream (literal) + (path
>    público − base path, sem as barras iniciais)` — concatenação crua, sem separador. Sem a barra,
>    `/techwave-order-support-agent/rpc` vira `/techwave-order-support-agentrpc` e o app responde
>    `No listener for endpoint`.
> 2. **O agente não escuta mais em `/agents/order-support`.** O `agentPath` passou a ser
>    `/techwave-order-support-agent`, igual ao base path da instância, porque a validação de bijeção
>    do A2A Connector compara o *path* da URL anunciada no card com `agentPath` + path da interface —
>    ou seja, **o gateway não pode reescrever o path**.

> **Barra final quebra o card.** `supportedInterfaces[0].url` é `${demoSupportAgent.url}/rpc`, então
> um valor terminado em `/` vira `...//rpc`. **Nenhuma das três variáveis `--property *.url` pode
> terminar em `/`** — o oposto da regra do campo Upstream acima, que exige a barra. São coisas
> diferentes: uma é o que a rede anuncia, a outra é o que o gateway concatena.

Depois disso:

1. Preencha as variáveis de `exchange.json` com os valores reais pós-deploy dos apps (URLs via
   ingress GW + credenciais dos contratos) — ou passe-as no deploy com `--property k:v`.
2. Confira que `supportedInterfaces[0].url` do agente bate com `${demoSupportAgent.url}/rpc` — o path
   `/rpc` do binding JSON-RPC do `demo-support-agent`.
3. Deploye a instância da rede (`agent-network project deploy`).
4. Registre o broker como Agent Instance no ingress gateway e aplique as políticas
   (Client ID Enforcement + A2A PII Detector + Rate Limiting).
5. Abra o **Agent Visualizer** e confirme broker, agente e MCP server no mapa.

## Comportamento para demonstrar

Duas perguntas ao broker, dois caminhos diferentes no Visualizer:

| Pergunta | O que o broker faz |
|---|---|
| "Qual o status do pedido TW-1001?" | chama a tool MCP **direto** — trace broker → MCP |
| "Monitor do TW-1002 com defeito, quero reembolso" | **delega** ao agente — trace broker → agente → MCP → 4 tools |

Quem produz essa bifurcação é o `generator classifyIntent` + `router intentRouter` do AgentScript — não uma frase no prompt.
