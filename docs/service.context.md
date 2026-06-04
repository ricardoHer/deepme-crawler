# deepme-crawler — Contexto de Negócio

## Propósito
O **deepme-crawler** é um script de coleta de informações públicas do ecossistema GitHub. Seu objetivo de negócio é **descobrir automaticamente novas fontes relevantes (usuários e repositórios)** a partir de relações públicas do GitHub (estrelas e stargazers), ampliando progressivamente a “base” a ser analisada por outras etapas do produto.

Ele também **normaliza e consolida métricas** (especialmente **número de stars por repositório**) em arquivos locais, garantindo que a coleta seja **retomável** e **evite repetição**. Na prática, o serviço fornece insumos contínuos para que o negócio mantenha um catálogo atualizado de repositórios e usuários “candidatos” derivados de sementes iniciais.

## Conceitos-chave
- **Seed:** lista em arquivos locais que indica quais usuários e repositórios devem ser processados no próximo ciclo.
- **Crawler de Usuário:** percorre repositórios mais relevantes (stars) de um usuário e propaga repos elegíveis para virar seeds.
- **Crawler de Repositório:** percorre stargazers de um repositório e propaga usuários elegíveis para virar seeds.
- **Crawled Marker:** registro local (por partição) de itens já processados para evitar recrawl.
- **Persistência em Arquivo (local):** gravação em disco de seeds, marcadores crawled e resultados (CSV) para continuidade operacional.

## Regras de Negócio Críticas
- Ao rastrear um **usuário**, considerar apenas repositórios com **stars > minStars** (minStars = 10).
- Ao rastrear um **usuário**, **ignorar** o repositório cujo **owner** seja o **próprio usuário** (owner = username do seed).
- Ao rastrear um **repositório**, **ignorar** usuários cujo valor seja **igual ao owner** do repositório (repo.split('/')[0]).
- **Só adicionar** um usuário ou repositório como **seed** se ele **ainda não estiver** registrado como **crawled**.
- Após rastrear um **repo**: **remover** esse repo do arquivo de seeds de repos e **registrar** em crawled.
- Após rastrear um **usuário**: **remover** esse usuário do arquivo de seeds de usuários e **registrar** em crawled.
- Ao registrar repositórios no **CSV (RepoDBCsvFile)**, **não adicionar linha duplicada** se o repo já existir no arquivo da partição.

## Integrações Principais
### Publica (eventos que emite)
- **Nenhum evento de domínio identificado**. Este serviço opera por execução contínua/loop local e persistência em disco.

### Consome (eventos que processa)
- **Nenhum evento externo identificado**. O fluxo é acionado localmente a partir de arquivos de seeds e controlado por marcadores crawled.

### Chamadas HTTP
- **github.com** — buscar páginas HTML de usuários e repositórios para extrair links (ex.: abas de **stars** e **stargazers**) e transformar essas ligações em novos seeds elegíveis.

## O que este serviço NÃO faz
- **Não** processa pagamentos, autenticação financeira ou qualquer fluxo que exija credenciais privadas.
- **Não** acessa APIs oficiais autenticadas (OAuth/GitHub API); a coleta descrita depende de **HTML público** via requisições HTTP.
- **Não** garante consistência transacional (não há garantia de “exatamente uma vez”); a consistência é buscada via **marcadores crawled** e controle por arquivos.
- **Não** integra com filas/eventos para coordenação distribuída; o estado e retomada são baseados em **arquivos locais**.

## Decisões de Arquitetura Relevantes
- **Arquitetura monolítica em script (app.js)**: reduz complexidade operacional para coleta contínua, usando um único processo para gerenciar o fluxo.
- **Persistência em arquivos locais (seeds/crawled/CSV)**: escolhida para permitir **retomada simples** e evitar dependência de banco/filas durante o crawling.
- **Particionamento por letra (por username/owner)**: seeds e marcadores são gravados por partição, para facilitar leitura/controle e reduzir custo de varreduras no disco.
- **Conservação de volume via deduplicação**: regras de “não duplicar” (seed apenas se não crawled; CSV sem linhas duplicadas) para manter o crescimento sob controle.

## Owner
- Time: **PREENCHER**
- Contato: **PREENCHER**

---  
> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Revise e enriqueça com contexto tácito antes de mergear.