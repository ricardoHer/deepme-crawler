# deepme-crawler — Modelo de Dados

## Entidades Principais

### SeedUser
- **Propósito:** Representa um username agendado para ser crawleado (entrada em “seeds”).
- **Campos principais:**
  - **username** (`string`) — identificador GitHub (ex: `octocat`)
  - **primeiraLetra** (`string`) — partição derivada de `username[0].toLowerCase()` (ex: `o`)
  - **fonteDoSeed** (`string`) — `PREENCHER` (se vem de algum arquivo/rodada específica)
  - **criadoEm** (`string | datetime`) — `PREENCHER` (se existir)
- **Relacionamentos:**
  - Relaciona-se com **CrawledUserMarker** via `username` (seed só pode existir se ainda não estiver marcado como crawled).
  - Gera **SeedRepo** ao processar o crawler de usuário (stars → repos elegíveis).
- **Observações:**
  - Persistência por arquivo local particionado: `seeds.crawled.users.<primeiraLetra>` (ou equivalente conforme schema real do diretório).
  - **Regra de elegibilidade:** repositórios estrelados retornados devem obedecer `minStars = 10` e devem ignorar repo cujo `owner == username`.
  - Seed só é adicionado se **não estiver** em **CrawledUserMarker**.

---

### SeedRepo
- **Propósito:** Representa um repositório agendado para ser crawleado.
- **Campos principais:**
  - **repo** (`string`) — formato `owner/repo` (ex: `octocat/hello-world`)
  - **owner** (`string`) — `repo.split('/')[0]`
  - **nome** (`string`) — `repo.split('/')[1]`
  - **primeiraLetra** (`string`) — partição derivada de `owner[0].toLowerCase()`
  - **fonteDoSeed** (`string`) — `PREENCHER`
- **Relacionamentos:**
  - Relaciona-se com **CrawledRepoMarker** via `repo`.
  - Ao processar o crawler de repositório, gera **SeedUser** a partir dos stargazers elegíveis.
- **Observações:**
  - Persistência por arquivo local particionado: `seeds.crawled.repos.<primeiraLetra>` (ou equivalente conforme schema real).
  - **Regra de elegibilidade (crawler de repositório):** ignorar usuários cujo valor seja igual ao `owner` do repositório (`user == owner`).
  - Seed só é adicionado se **não estiver** em **CrawledRepoMarker**.
  - Pós-processamento: ao concluir crawl de um repo, o repo sai do seed e entra no marcador de crawled.

---

### CrawledUserMarker
- **Propósito:** Marca usernames já processados para impedir recrawl.
- **Campos principais:**
  - **username** (`string`)
  - **primeiraLetra** (`string`) — `username[0].toLowerCase()`
  - **status** (`string`) — `PREENCHER` (provavelmente binário/linhas no arquivo)
  - **marcadoEm** (`string | datetime`) — `PREENCHER`
- **Relacionamentos:**
  - Consome/valida contra **SeedUser** (seed deve ser ignorado se existir marcador).
- **Observações:**
  - Persistência em arquivo local particionado: `crawled.users.<primeiraLetra>` (nome exato pode variar conforme o projeto, usar `PREENCHER` se necessário no schema real).
  - Implementação típica: presença do username em arquivo = crawled.

---

### CrawledRepoMarker
- **Propósito:** Marca repositórios já processados para impedir recrawl.
- **Campos principais:**
  - **repo** (`string`) — `owner/repo`
  - **owner** (`string`)
  - **primeiraLetra** (`string`) — `owner[0].toLowerCase()`
  - **marcadoEm** (`string | datetime`) — `PREENCHER`
- **Relacionamentos:**
  - Consome/valida contra **SeedRepo**.
- **Observações:**
  - Persistência em arquivo local particionado: `crawled.repos.<primeiraLetra>` (ou equivalente).
  - Presença no arquivo = crawled.

---

### RepoMetricCsv
- **Propósito:** Persistir métricas mínimas coletadas por repo (ex.: `stars`), em CSV local por partição.
- **Campos principais (linha CSV):**
  - **stars** (`number`) — primeira coluna (conforme “linhas `stars,repo`”)
  - **repo** (`string`) — segunda coluna (formato `owner/repo`)
  - **owner** (`string`) — derivado de `repo` para uso de partição
  - **primeiraLetra** (`string`) — derivado de `owner[0].toLowerCase()`
- **Relacionamentos:**
  - Relaciona-se a **CrawledRepoMarker** via `repo` (registro em CSV deve ocorrer após crawl do repo).
- **Observações:**
  - Persistência em arquivo CSV local por partição: `repo.db.csv.<primeiraLetra>` (nome exato varia; usar `PREENCHER` no schema real).
  - **Regra de integridade de dados:** não adicionar linha duplicada se o repo já existir no arquivo.
  - Deduplicação: `PREENCHER` (provável varredura do arquivo ou index em memória).

---

## Diagrama de Relacionamentos

```mermaid
SeedUser --> CrawledUserMarker
SeedRepo --> CrawledRepoMarker

SeedUser --> SeedRepo
SeedRepo --> SeedUser

SeedRepo --> RepoMetricCsv
CrawledRepoMarker --> RepoMetricCsv
```

> Nota: Relacionamentos são controlados por chaves naturais (`username`, `repo`) e pela lógica do crawler (entrada em seed → validação contra marker → persistência em CSV → gravação de marker e remoção do seed).

## Fluxos de Dados Principais

### 1) Seleção e crawl de SeedUser (Crawler de Usuário)
1. **Origem:** ler “pedaços” do seed de usuários por partição (arquivo `seeds...users.<letra>`). O nome e método exato de leitura estão parcialmente indefinidos; comportamento de `readSomeSeed` tem inconsistências citadas.
2. **Transformação:**
   - Para cada `username`:
     - verificar elegibilidade via marcador: ignorar se existir em `crawled.users.<letra>`.
     - buscar páginas HTML no GitHub (abas de repos estrelados do usuário).
     - extrair links/repositórios via `jsdom`.
     - filtrar: `stars > minStars (10)` e ignorar repositório cujo `owner == username`.
3. **Destino/persistência:**
   - Adicionar repos elegíveis como **SeedRepo** (somente se não estiverem marcados como crawled em `crawled.repos.<letra>`).
   - Remover o `username` do seed (regra: “depois de crawlar um usuário, remover o usuário correspondente do seed de usuários”).
   - Registrar `username` em **CrawledUserMarker**.

---

### 2) Seleção e crawl de SeedRepo (Crawler de Repositório)
1. **Origem:** ler “pedaços” do seed de repos por partição (arquivo `seeds...repos.<letra>`).
2. **Transformação:**
   - Para cada `repo` (`owner/repo`):
     - verificar elegibilidade via marcador: ignorar se existir em `crawled.repos.<letra>`.
     - buscar HTML no GitHub (stargazers do repositório).
     - extrair usuários via `jsdom`.
     - filtrar: ignorar usuários cujo valor seja igual ao `owner` do repositório.
3. **Destino/persistência:**
   - Adicionar usuários elegíveis como **SeedUser** (somente se não estiverem marcados em `crawled.users.<letra>`).
   - Persistir métricas do repo em **RepoMetricCsv** (ex.: `stars,repo`):
     - regra: não adicionar duplicado se repo já existir no arquivo.
   - Remover `repo` do seed de repos.
   - Registrar `repo` em **CrawledRepoMarker**.

---

### 3) Deduplicação e retomada (Crawled Markers e Seeds)
1. **Origem:** arquivos locais persistidos em runtime, incluindo seeds e markers por partição.
2. **Transformação:** antes de processar um item, checar presença em marcador.
3. **Destino:** atualização atômica lógica em arquivos:
   - seed (remove item já processado)
   - marker (add item processado)
   - CSV (append só se não duplicar)

> Confiabilidade: comportamento exato de escrita/ordenação/atomicidade (ex.: rename temporário) está `PREENCHER` — validar no schema real.

## Estratégia de Persistência
- **Banco principal:** arquivos locais em filesystem (`fs`) como “fonte da verdade” para:
  - seeds (usuários/repos)
  - markers crawled (usuários/repos)
  - métricas CSV por repo
- **Cache:** `PREENCHER` (pode existir cache em memória de seeds/markers durante execução)
- **Event store:** não utilizado (arquitetura é script monolítico; sem event sourcing)

## Considerações de Performance
- **Particionamento por primeira letra** (`username[0]`, `owner[0]`):
  - reduz tamanho de listas e acelera checagens/leituras por ciclo.
- **Índices/deduplicação no CSV:**
  - crítico para evitar duplicatas (“não adicionar linha duplicada se o repo já existir no arquivo”).
  - `PREENCHER` se há índice em memória (Set) por partição; caso não haja, a deduplicação pode ser O(n) por inserção.
- **Concorrência/ritmo:**
  - `crawlInterval = 300` e uso de `setTimeout`/loop contínuo via `forever` foram citados; impacto em rate limits do GitHub está `PREENCHER`.
- **Integridade do filesystem:**
  - criação de diretórios/arquivos no runtime (ex.: `mkdirSync`) foi citada como comentada; comportamento se não existirem paths está `PREENCHER`.

---
> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Valide com o schema real do banco antes de mergear.