# deepme-crawler — Arquitetura

## Visão Geral
O **deepme-crawler** adota uma arquitetura de **script monolítico executável** (um único processo Node.js, tipicamente `app.js`) com **persistência em arquivos locais** (seeds, marcadores de “crawled” e métricas em CSV). O racional principal é **operacional**: permitir **retomada simples** após reinícios, evitar a necessidade de componentes externos (filas, workers, banco transacional) e reduzir custo de infraestrutura para um crawler que executa em ritmo controlado. A execução contínua é delegada a um gerenciador de processo (**`forever`**), enquanto o “estado” do crawling é reconstruído a partir do sistema de arquivos.

## Stack Tecnológico
| Camada | Tecnologia | Motivo |
|---|---|---|
| Runtime / execução | Node.js | Adequação ao ecossistema do stack atual do repositório e integração direta com escrita em disco via `fs`. |
| Extração HTML | jsdom | Permite parseamento de HTML retornado pelo GitHub e navegação/extração de links/valores sem browser real. |
| Controle de processo | forever | Mantém o script “online” (reinício automático) sem exigir orquestrador externo. |
| Qualidade de código | eslint | Padroniza e valida regras mínimas do projeto. |
| Persistência | Arquivos locais (texto/CSV) | Facilita retomada e evita banco/infra; estado e histórico ficam versionados/localmente. |

## Padrões Adotados
- **Checkpointing por arquivo (estilo “persistent state”)**: seeds e marcadores `crawled.*` em disco permitem retomar sem reprocessar itens já vistos.
- **Idempotência via marcadores (`crawled`)**: antes de adicionar seeds, o script verifica se já foi marcado como crawled; depois remove do seed e registra no marcador. Isso reduz duplicidade e evita loops.
- **Processamento dirigido por “frontier” (seeds)**: a “fronteira” do crawl (o que será processado) é o conjunto de seeds particionado; o crawler expande a fronteira (usuários ↔ repos) à medida que encontra relações.
- **Particionamento por letra (sharding local simples)**: arquivos `*.*` particionados por primeira letra do username/owner para distribuir tamanho e manter operações de leitura/escala mais previsíveis em disco.

## Decisões de Arquitetura (ADRs)

### ADR 1 — Persistência em arquivos locais em vez de banco/filas
- **Contexto:** O crawler depende de um estado incremental (quais seeds restam, quais itens já foram processados, e métricas por repo). Também precisa sobreviver a reinícios.
- **Decisão:** Persistir:
  - seeds (usuários e repos) em arquivos locais particionados por letra;
  - marcadores `crawled.*` para evitar recrawling;
  - métricas em CSV local (`RepoDBCsvFile` com formato `stars,repo`).
- **Consequências:**
  - **Prós:** retomada simples (reconstrução do estado via disco); menor custo operacional (sem banco/infra); idempotência “natural” com `crawled`.
  - **Contras:** maior risco de corrupção/inconsistência se o processo cair no meio da escrita; escalabilidade limitada (I/O de disco e tamanho dos arquivos); concorrência fica mais difícil (o modelo atual parece assumir execução single-thread/processo).

### ADR 2 — Arquitetura monolítica (um processo, uma lógica de execução)
- **Contexto:** O crawler é essencialmente um pipeline sequencial: selecionar seed → buscar HTML no GitHub → extrair links → filtrar/registar sementes → persistir e marcar crawled.
- **Decisão:** Implementar tudo em um único artefato executável (`app.js`), controlado por `forever`, com ritmo controlado internamente.
- **Consequências:**
  - **Prós:** simplicidade de implementação e depuração; menor latência operacional por eliminar comunicação entre serviços.
  - **Contras:** menor isolamento de falhas (uma exceção pode derrubar todo o processo); dificuldade de escalar horizontalmente; observabilidade e controle fino (rate-limit, retries, circuit breaker) ficam mais difíceis de padronizar e testar.

### ADR 3 — Idempotência via marcadores `crawled.*` e remoção do seed processado
- **Contexto:** Durante a expansão usuário↔repositório, os mesmos itens podem surgir múltiplas vezes a partir de diferentes páginas (estrelas e stargazers).
- **Decisão:** Aplicar estas regras:
  - só adiciona seed se **não estiver marcado** em `crawled`;
  - após processar um repo: **remove do seed** e **marca como crawled**;
  - após processar um usuário: **remove do seed** e **marca como crawled**.
- **Consequências:**
  - **Prós:** reduz duplicidade e evita loops; torna o processo mais previsível ao longo do tempo.
  - **Contras:** depende fortemente de consistência entre operações de leitura/escrita (seed removido vs crawled marcado); se houver escrita parcial, pode haver reprocessamento ou perda de itens.

### ADR 4 — Particionamento por primeira letra para seeds e marcadores
- **Contexto:** Seeds e marcadores podem crescer rapidamente; operações de leitura/escrita em arquivos únicos podem degradar desempenho e aumentar risco de concorrência.
- **Decisão:** Estruturar arquivos como `seeds/users/repos` e `crawled/users/repos` **por primeira letra** do username/owner.
- **Consequências:**
  - **Prós:** controle mais granular e distribuição do crescimento; operações e buscas podem ser mais baratas (dependendo da implementação real de `readSomeSeed`).
  - **Contras:** ainda há riscos de escalabilidade quando um “bucket” concentra muita massa; aumenta complexidade do código (gerenciar letra, caminhos e arquivos).

### ADR 5 — Extração por HTML (jsdom) a partir de requisições HTTP diretas ao GitHub
- **Contexto:** O crawler precisa obter “stargazers” e “stars” a partir das páginas do GitHub sem integrar API oficial (pelo que foi descrito).
- **Decisão:** Fazer HTTP outbound e parsear o HTML retornado com jsdom para extrair informações.
- **Consequências:**
  - **Prós:** reduz dependência de autenticação/SDK de API; flexível para extrair campos conforme a estrutura das páginas.
  - **Contras:** maior fragilidade a mudanças de HTML; potencial impacto com rate limits/anti-bot do GitHub; ausência de tratamento explícito (na análise) para retry/backoff/circuit breaker.

## Dependências Externas
| Dependência | Tipo | Propósito |
|---|---|---|
| github.com | HTTP (outbound) | Buscar HTML de páginas de **usuários** (abas de stars) e **repositórios** (abas de stargazers) para extração de links/dados via jsdom. |

## Pontos de Atenção
- **Inconsistência/fragmentação do código de inicialização:** o trecho do repositório citado termina abruptamente (“termina com `c`”) e o mecanismo real do loop contínuo não está totalmente documentado. Isso afeta entendimento de:
  - quando a execução ocorre (cron vs interval interno),
  - como reinicia após erro,
  - e qual é o fluxo exato de seleção/execução de seeds.
- **Função `readSomeSeed` com variáveis possivelmente indefinidas/inconsistentes:** há referência a `repoSeed` e logs/remoções fora do fluxo correto (conforme observado). Isso pode causar:
  - seed selection incorreta,
  - remoção/registro em arquivos errados,
  - e consequente duplicidade/perda de cobertura.
- **Criação de diretórios não garantida (mkdirSync comentado):** não está claro o comportamento quando pastas/arquivos não existem. Risco:
  - falha imediata no runtime por `ENOENT`,
  - ou lógica de fallback que ainda não foi confirmada.
- **Rate limiting e controle de ritmo:** embora exista indicação de `crawlInterval=300` e uso de `setTimeout`, não foi descrito tratamento de:
  - limites do GitHub (rate limit por IP/autenticação),
  - backoff em caso de erro,
  - retries com jitter,
  - diferenciação entre erros transitórios (429/5xx) e permanentes (404/410).
- **Concorrência implícita:** o desenho com arquivos locais e marcadores pressupõe execução **single-process**. Rodar instâncias paralelas pode causar corrida em escrita/remoção de seeds e `crawled` (sem locking/transactionalidade).
- **Idempotência do CSV (`stars,repo`):** existe regra para não duplicar linhas quando o repo já existe no arquivo, mas não está descrito o método (indexação/checagem por leitura completa do CSV, hash, etc.). Isso pode impactar performance e ainda assim falhar se houver escrita simultânea.

---
> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Valide com o time antes de mergear.