# deepme-crawler — Mapa de Integrações

## Visão Geral

```mermaid
flowchart LR
  A[Seeds / arquivos locais<br/>(crawled.*, seeds.* e CSV)] <--> B[deepme-crawler<br/>app.js]
  B --> C[github.com<br/>HTML de repositórios/usuários]
```

## Dependências de Saída (Outbound)

> O que este serviço chama (integrações externas) e o que ele persiste/consulta fora do processo.

| Serviço / Sistema | Tipo | Finalidade | Crítico? |
|---|---|---|---|
| github.com | HTTP | Buscar páginas HTML de **usuários (stars)** e **repositórios (stargazers)** para extrair links/nomes via **jsdom** | Sim |
| Sistema de arquivos local (disk) | DB/Storage (Arquivo) | Persistir **seeds**, **marcadores crawled** e **resultados CSV** (stars,repo) para retomada e deduplicação | Sim |
| `db.tar.gz` (pacote pré-gerado) | Storage (arquivo) | Carregar/fornecer base de estado/dataset inicial para o crawler (conteúdo exato: PREENCHER) | Não |

### Detalhes das integrações críticas

#### github.com (HTTP)
- **Endpoint/Tópico:** Páginas web do GitHub para:
  - **Usuários:** aba/tela de *stars* (coleta repos estrelados)
  - **Repositórios:** aba/tela de *stargazers* (coleta usuários)
  - (URLs exatas e parâmetros: PREENCHER)
- **Autenticação:** PREENCHER (não foi informado token/headers; presumível “público” via HTTP)
- **Comportamento em falha:** PREENCHER (não foram documentados retry/backoff/circuit breaker; há indício de loop contínuo com `crawlInterval=300`, mas o tratamento de erro não está especificado)

#### Persistência local (Seeds / Crawled / CSV)
- **Endpoint/Tópico:** Arquivos no runtime:
  - `seeds.*` (usuários e repos por primeira letra)
  - `crawled.*` (marcadores por primeira letra)
  - `RepoDBCsvFile` (CSV com `stars,repo` por partição)
  - (paths/nomes exatos: PREENCHER)
- **Autenticação:** N/A
- **Comportamento em falha:** PREENCHER (ex.: criação de diretórios/arquivos, tolerância a arquivo inexistente, locking concorrente)
  - Observação do estado acumulado: `mkdirSync` aparenta estar comentado; comportamento se diretórios/arquivos não existirem: PREENCHER

## Dependências de Entrada (Inbound)

> Quem chama ou consome este serviço (não há exposto de rede no estado acumulado; integração é principalmente via cron/execução local).

| Chamador | Tipo | O que usa |
|---|---|---|
| Runtime / gerenciador `forever` (processo local) | Execução (não-HTTP) | Inicia e mantém o `app.js` em execução contínua (scripts de start: PREENCHER) |

## Eventos

### Publica
| Evento | Tópico/Exchange | Consumidores conhecidos |
|---|---|---|
| PREENCHER | PREENCHER | PREENCHER |

### Consome
| Evento | Origem | Ação executada |
|---|---|---|
| PREENCHER | PREENCHER | PREENCHER |

---  
> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Valide os endpoints e tópicos com o código antes de mergear.