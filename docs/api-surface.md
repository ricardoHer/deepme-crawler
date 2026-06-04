# deepme-crawler — Superfície de API

## Visão Geral
- **Tipo:** REST / GraphQL / gRPC: **PREENCHER** (não foi identificado no repositório um servidor HTTP/GraphQL/gRPC; o serviço parece ser um script Node.js)
- **Base URL:** **PREENCHER**
- **Autenticação:** **sem auth** (não há evidência de endpoint externo; o crawling é acionado localmente via `node app.js`)
- **Formato:** JSON / Protobuf: **PREENCHER**
  
> Observação: com base no “estado acumulado” fornecido, o `deepme-crawler` opera como **script monolítico** que faz chamadas **HTTP outbound** ao GitHub e persiste resultados em **arquivos locais** (seeds, markers `crawled.*` e CSV). Não há, até o momento, uma **API pública** documentada (endpoints/queries/methods).

## Endpoints
**Não aplicável / não encontrado.**  
Não foi identificado no contexto um servidor expondo rotas REST (ex.: `/health`, `/status`, `/start`) nem um schema GraphQL ou serviço gRPC.

## Autenticação e Autorização
- **PREENCHER**: como não há endpoints, não há controle por roles/scopes documentado.
- Se houver alguma forma de executar/controlar o crawler via HTTP, isso **não está presente** no contexto acumulado e deve ser validado no código / OpenAPI / spec.

## Rate Limiting
- **PREENCHER**: não há documentação no contexto sobre limites/controle de ritmo para as requisições ao GitHub.
- Indícios a validar no código:
  - `crawlInterval = 300` (rate/pausa entre ciclos) e uso de `setTimeout`
  - impacto de **GitHub rate limits** para listagens de repositórios/usuários

## Versionamento
- **PREENCHER**: não há estratégia de versionamento de API (pois não foi identificado um contrato HTTP).

---

> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Valide os contratos com o código/OpenAPI spec antes de mergear.