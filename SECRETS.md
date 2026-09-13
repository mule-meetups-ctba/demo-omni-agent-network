# Segredos — como este repositório trata credenciais

## O modelo

Este projeto **não guarda nenhum segredo**, nem cifrado. Todo valor sensível é declarado como
**variável** em `exchange.json` e preenchido no momento do deploy da instância da rede:

```json
"metadata": {
  "variables": {
    "azureOpenAi": {
      "url":    { "description": "...", "default": "", "secret": false },
      "apiKey": { "description": "...", "default": "", "secret": true }
    }
  }
}
```

O flag `"secret": true` faz a plataforma tratar o valor como sensível — mascarado na UI e fora
dos logs. O `agent-network.yaml` referencia as variáveis por `${...}`, nunca o valor.

## Variáveis deste projeto

| Variável | `secret` | O que é |
|---|---|---|
| `demoSupportAgent.url` | não | URL pública do agente A2A via ingress Omni Gateway |
| `demoSupportAgent.clientId` | não | `client_id` do contrato `broker-to-agent` |
| `demoSupportAgent.clientSecret` | **sim** | `client_secret` do mesmo contrato |
| `mcpServer.url` | não | URL do MCP server via gateway |
| `mcpServer.clientId` | não | `client_id` do contrato `broker-to-mcp-server` |
| `mcpServer.clientSecret` | **sim** | `client_secret` do mesmo contrato |
| `azureOpenAi.url` | não | base URL do recurso Azure OpenAI, terminando em `/openai/v1` |
| `azureOpenAi.apiKey` | **sim** | API key do Azure OpenAI |

> O broker não tem variável de URL: o card de um broker não declara `url`/`supportedInterfaces` —
> ele **é** o endpoint e a URL é atribuída no deploy.

Se você clonar este repositório, os `default` vazios são o comportamento correto — preencha no
deploy, não no arquivo.

## ⚠️ Contrato das URLs — cada `.url` é a BASE, sem o path

As três variáveis de URL viram `upstreamUrl` **literal** da connection (confira em
`target/network/demo-omni-agent-network/connections/<conn>/connection.json` depois de um
`project build`). O path final é montado em outro lugar, por camada — então incluir o path na
variável **duplica** (`/mcp/mcp`, `/rpc/rpc`) e o upstream responde 404.

| Variável | Formato | **Não** pode conter | Quem acrescenta o path |
|---|---|---|---|
| `demoSupportAgent.url` | `https://<ingress-gw>/techwave-order-support-agent` | `/rpc` | o próprio card: `supportedInterfaces[0].url` é `${demoSupportAgent.url}/rpc` no `agent-network.yaml` |
| `mcpServer.url` | `https://<egress-gw>/techwave-support-mcp-server` | `/mcp` | `registry.mcps.mcpServer.metadata.transport.path: /mcp` |
| `azureOpenAi.url` | `https://<recurso>.services.ai.azure.com/openai/v1` | `/responses`, barra final | a policy `openai-transcoding-policy`, aplicada automaticamente na connection LLM |

Duas notas de topologia:

- **Agente = ingress (público), MCP = egress (interno).** O agente é chamado tanto pelo broker
  quanto por clientes externos na demo, então vive atrás do Public Ingress GW
  (`<ps-dns>.usa-e1.cloudhub.io`). O MCP server só é chamado de dentro da rede — Private Egress GW
  (`internal-<ps-dns>.usa-e1.cloudhub.io`), que **não resolve de fora do private space**. Testar o
  egress do seu laptop dá timeout; isso é o esperado, não uma falha.
- **O `/rpc` não é escolha arbitrária.** Ele tem que bater com
  `<a2a:interface protocol="JSONRPC" path="/rpc"/>` do `global-config.xml` do `demo-support-agent`.
  Mudou lá, muda aqui.

## ⚠️ Rotação — checklist

1. Gere as credenciais novas: **Azure OpenAI** (nova key) e **API Manager** (reset das
   credenciais dos contratos `broker-to-agent` e `broker-to-mcp-server`).
2. Atualize os valores das variáveis na **instância deployada** da rede.
3. Revogue as credenciais antigas nos sistemas de origem.
4. Lembre que a key do Azure OpenAI é consumida em **dois lugares independentes**: aqui (a
   connection LLM do broker) e no app `demo-support-agent` (`secure::openai.api.key`).
   Rotacionou? Atualize os dois, ou um dos caminhos para de funcionar.
