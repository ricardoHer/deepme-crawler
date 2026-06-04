# deepme-crawler — Glossário de Domínio

> Termos e conceitos usados neste serviço e seus significados no contexto do negócio.
> Use este glossário ao ler o código, escrever tickets ou comunicar com outros times.

## Termos de Negócio

### Seed (Seeds)
**Definição:** Lista “pendente” de usuários e/ou repositórios que devem ser processados no próximo ciclo de crawling. Serve como mecanismo de escalonamento e retomada após pausas/reinícios.  
**No código:** Aparece como arquivos de seeds particionados em disco, tipicamente `seeds.crawled.*` e/ou variações por tipo (usuários vs repositórios) e por letra (ex.: `seedUsers.<letra>`, `seedRepos.<letra>`). Também aparece em funções de leitura/seleção (`readSomeSeed`) e em rotinas que removem seeds ao concluir o processamento.  
**Não confundir com:** `Crawled Marker` (marcação de “já processado”); Seed é “a processar”, marker é “processado”.

---

### Crawler de Usuário
**Definição:** Processo que, dado um usuário (seed), busca repositórios relevantes do usuário e seleciona repositórios elegíveis para virar seed de repositórios.  
**No código:** Lógica que itera sobre seeds de usuários e faz coleta via HTTP/HTML com `jsdom` (renderização/extração) para obter links de repos (ex.: páginas de “stars”).  
**Regras associadas:**
- considerar apenas repositórios com `stars > minStars` (minStars = 10);
- ignorar repositório cujo `owner` (primeira parte do `owner/repo`) seja o próprio usuário.

**Não confundir com:** Crawler de Repositório (que começa de um repo e extrai usuários).

---

### Crawler de Repositório
**Definição:** Processo que, dado um repositório (seed), busca usuários relacionados (ex.: stargazers) e seleciona usuários elegíveis para virar seed de usuários.  
**No código:** Lógica que itera sobre seeds de repositórios e faz coleta via HTTP/HTML com `jsdom` para obter links de usuários (ex.: stargazers).  
**Regras associadas:**
- ignorar usuários cujo valor seja igual ao `owner` do repositório (`repo.split('/')[0]`).

**Não confundir com:** Crawler de Usuário.

---

### Crawled Marker
**Definição:** Registro persistente (em arquivos locais) que identifica quais itens (usuários e repositórios) já foram processados, evitando recrawling.  
**No código:** Arquivos particionados por letra (por exemplo `crawledUsers.<letra>` e `crawledRepos.<letra>`), usados em checagens antes de adicionar seeds e atualizados após concluir o crawling.  
**Não confundir com:** Persistência em CSV de métricas (CSV guarda dados do repo, não status de processamento).

---

### Persistência em Arquivo (Filesystem Persistence)
**Definição:** Persistência dos seeds, markers crawled e listas auxiliares diretamente em disco (texto/listas), para permitir retomada simples sem fila/banco de mensageria.  
**No código:** Uso de `fs` (ex.: `readFile`, `writeFile`, `appendFile`, remoção/reescrita de seeds; e atualização de markers). Comentários indicam que `mkdirSync` existe mas pode estar desativado.  
**Não confundir com:** Persistência em CSV (um formato específico para métricas).

---

### Persistência em CSV (RepoDB)
**Definição:** Armazenamento de métricas por repositório em arquivos CSV locais, tipicamente por partição de primeira letra do `owner`.  
**No código:** Arquivo CSV local (ex.: `RepoDBCsvFile`) com linhas do formato `stars,repo`. Regra de deduplicação: não inserir linha duplicada se o repo já existir no arquivo.  
**Não confundir com:** Logs/markers de crawled (CSV guarda dados; markers guardam status).

---

### minStars
**Definição:** Limite mínimo de estrelas para considerar um repositório “elegível” quando o ponto de partida é um usuário.  
**No código:** Constante/variável usada no filtro do Crawler de Usuário (minStars = 10).  
**Não confundir com:** limiares análogos (PREENCHER) — não observado no material.

---

## Termos Técnicos do Domínio

### SeedUsersFile
**Definição:** Arquivo(s) que persistem seeds de **usuários**, particionados por primeira letra do `username`.  
**Contexto de uso:** Ao iniciar/rodar ciclos de crawling para escolher próximos usuários a processar; também ao remover um usuário seed após o processamento.

---

### SeedReposFile
**Definição:** Arquivo(s) que persistem seeds de **repositórios**, particionados por primeira letra do `owner`.  
**Contexto de uso:** Ao escolher próximos repositórios para processar; também ao remover o repo seed após concluir o crawling daquele repo.

---

### CrawledUsersFile
**Definição:** Arquivo(s) marcador(es) de usuários já processados, particionados por primeira letra do `username`.  
**Contexto de uso:** Checagem para não adicionar usuário repetido como seed; atualização após concluir o crawling do usuário.

---

### CrawledReposFile
**Definição:** Arquivo(s) marcador(es) de repositórios já processados, particionados por primeira letra do `owner`.  
**Contexto de uso:** Checagem para não adicionar repo repetido como seed; atualização após concluir o crawling do repositório.

---

### RepoDBCsvFile
**Definição:** Arquivo(s) CSV que persistem dados de repositórios com colunas/estrutura `stars,repo`, particionados por primeira letra do `owner`.  
**Contexto de uso:** Quando o crawler de usuário coleta repositórios elegíveis e registra métricas (stars) no disco; deduplicação por existência prévia do repo no arquivo.

---

### Particionamento por primeira letra
**Definição:** Estratégia de segmentar arquivos (seeds, crawled, CSV) com base na primeira letra do identificador (username ou owner), reduzindo tamanho de arquivos e facilitando operações por lote.  
**Contexto de uso:** Construção de nomes de arquivo e leitura/escrita em disco para cada partição.

---

### jsdom (HTML Parsing)
**Definição:** Biblioteca de parsing/extração de dados de HTML em ambiente Node.js, usada para transformar a resposta HTTP em DOM para localizar links de usuários e repositórios.  
**Contexto de uso:** Após buscar páginas do GitHub (usuários e repositórios) para obter “stars” e “stargazers”.

---

### jsdom + Seletores de Links
**Definição:** Etapa de extração onde o DOM é percorrido para coletar URLs/identificadores (ex.: `owner/repo` e `username`).  
**Contexto de uso:** Quando o crawler descobre novos candidatos a seed.

---

### Ponto de partida (Seed picking)
**Definição:** Mecanismo de seleção de um subconjunto de seeds para processar em cada ciclo (ex.: “pegar alguns seeds” antes do próximo intervalo).  
**Contexto de uso:** Função/rotina associada a `readSomeSeed` (com comportamento descrito como PREENCHER, pois há inconsistências apontadas na análise).  
**Não confundir com:** Crawled Marker (checagem de já processado) — o seed picking decide quais itens processar; o marker decide se item é elegível.

---

### Rate / Intervalo de Execução
**Definição:** Controle de ritmo entre ciclos de crawling para reduzir carga e respeitar limites do GitHub.  
**Contexto de uso:** Variáveis como `crawlInterval = 300` e possíveis `setTimeout`/loop contínuo (o detalhe exato do loop completo está PREENCHER).

---

## Acrônimos e Abreviações

| Sigla | Significado | Contexto |
|---|---|---|
| PREENCHER | (sem informação suficiente no material) | Usar quando o termo/funcionalidade não apareceu com clareza no estado analisado. |

---