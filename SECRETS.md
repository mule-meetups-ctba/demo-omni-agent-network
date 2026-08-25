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
| `demoOmniBroker.url` | não | URL pública do broker via ingress, anunciada no agent card |

Se você clonar este repositório, os `default` vazios são o comportamento correto — preencha no
deploy, não no arquivo.

## ⚠️ Rotação — checklist

1. Gere as credenciais novas: **Azure OpenAI** (nova key) e **API Manager** (reset das
   credenciais dos contratos `broker-to-agent` e `broker-to-mcp-server`).
2. Atualize os valores das variáveis na **instância deployada** da rede.
3. Revogue as credenciais antigas nos sistemas de origem.
4. Lembre que a key do Azure OpenAI é consumida em **dois lugares independentes**: aqui (a
   connection LLM do broker) e no app `demo-support-agent` (`secure::openai.api.key`).
   Rotacionou? Atualize os dois, ou um dos caminhos para de funcionar.
