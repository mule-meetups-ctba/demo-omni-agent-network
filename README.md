# meetup-omni-agent-network

Projeto **Agent Network 2.0** (MuleSoft Agent Fabric) da demo do MuleSoft Meetup. Define a rede
que coordena o agente A2A, as tools MCP e o LLM — e o **broker** que decide quando resolver algo
sozinho e quando delegar.

- **Schema:** `agentNetwork: 2.0.0` (YAML `registry` + `context` + `brokers`)
- **Broker:** AgentScript (`brokers/meetup-broker.agent`), dialeto `AGENTFABRIC=1.0`
- **LLM:** Azure OpenAI `gpt-5.4-mini` (endpoint `/responses`)
- **Protocolo entre agentes:** A2A 1.0

> Parte de uma demo com 4 repositórios. Arquitetura, walkthrough completo, políticas de gateway
> e roteiro de apresentação: **`meetup-omni-material`**.

## Estrutura

```
.
├── agent-network.yaml     # registry (assets) + context (conexões) + brokers
├── exchange.json          # metadados Exchange + variáveis (com flag secret)
└── brokers/
    └── meetup-broker.agent  # o broker, em AgentScript
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
- **URL da connection LLM é só a base.** Para Azure OpenAI, termina em `/openai/v1`, sem
  `/responses` e sem barra final — o broker monta o path sozinho. Sobra de path aí é a causa
  clássica de redirect/404.

E uma do broker: **cada tool MCP é uma `action` explícita** com o `tool_name` exato. É mais
verboso que referenciar o servidor inteiro, mas dá controle fino sobre o que o broker pode usar.

## Guided determinism

O broker separa o que exige julgamento probabilístico do que exige controle determinístico:

```
trigger (A2A)  →  orchestrator (LLM decide)  →  echo (resposta A2A)
```

O LLM decide *o quê*; o grafo garante *a ordem*.

## Variáveis

Todo `${...}` do YAML é declarado em `exchange.json` e preenchido no deploy — nunca no arquivo.
Ver [SECRETS.md](SECRETS.md) para a lista completa e o checklist de rotação.

## Publicando e deployando

1. Abra o projeto no **Anypoint Code Builder** (ou copie a estrutura para um projeto novo gerado
   pelo template Agent Network).
2. Preencha as variáveis de `exchange.json` com os valores reais pós-deploy dos apps (URLs via
   ingress GW + credenciais dos contratos).
3. Confira que `supportedInterfaces[0].url` do agente bate com `${meetupAgent.url}/rpc` — o path
   `/rpc` do binding JSON-RPC do `demo-support-agent`.
4. Publique os assets no **Exchange** e deploye a instância da rede.
5. Registre o broker como Agent Instance no ingress gateway e aplique as políticas
   (Client ID Enforcement + A2A PII Detector + Rate Limiting).
6. Abra o **Agent Visualizer** e confirme broker, agente e MCP server no mapa.

## Comportamento para demonstrar

Duas perguntas ao broker, dois caminhos diferentes no Visualizer:

| Pergunta | O que o broker faz |
|---|---|
| "Qual o status do pedido TW-1001?" | chama a tool MCP **direto** — trace broker → MCP |
| "Monitor do TW-1002 com defeito, quero reembolso" | **delega** ao agente — trace broker → agente → MCP → 4 tools |

A instrução de roteamento que produz isso está no `orchestrator` do AgentScript.
