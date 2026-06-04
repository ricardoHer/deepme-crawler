# deepme-crawler — Guia de Onboarding

## O que você precisa saber primeiro
O `deepme-crawler` é um crawler/script Node.js (monolítico) que varre relações no GitHub para “expandir” um grafo de interesse: **usuários** → **repositórios estrelados** → **usuários que favoritaram/seguem via stargazers** (na prática: stargazers/seguidores dependendo da aba/HTML extraída). O objetivo é coletar métricas (ex.: `stars`) e **persistir tudo em disco** para permitir retomada simples entre execuções.

A execução controla o ciclo usando **arquivos locais** como estado: listas de **seeds** (usuários e repos que ainda não foram processados) e “marcadores” **crawled** (para não repetir). A lógica de scraping/extração usa **HTTP outbound** e renderização/parse via **jsdom**, e a execução contínua (quando aplicável) costuma ser orquestrada fora do app, via **forever**.

> Importante: há pontos com comportamento não totalmente claro no material de contexto (ex.: loop/inicialização não encontrado por completo; algumas referências inconsistentes em funções). Então, além de “rodar”, você precisa validar o fluxo real lendo o `app.js`.

---

## Como rodar localmente

### Pré-requisitos
- **Node.js 18+** (ou versão compatível com o código atual).  
  - Instalação: use o gerenciador do seu SO (nvm é recomendado).
- **npm** (vem com o Node).  
- (Opcional, mas recomendado) **git** para clonar.
- **Dependências do projeto**:
  - `jsdom`
  - `forever`
  - `eslint` (apenas lint)
  
> Para instalar as dependências, primeiro rode `npm ci` (se houver `package-lock.json`) ou `npm install`.

### Passos
```bash
# 1) clone
git clone PREENCHER
cd deepme-crawler

# 2) instalar deps
npm ci   # ou: npm install

# 3) configurar ambiente (ver tabela abaixo)
# 4) rodar uma vez (se houver modo/entrypoint no app.js)
node app.js

# 5) (se o projeto exigir loop contínuo) rodar via forever
# forever start app.js
```

### Configurações necessárias
No contexto fornecido, **não foi possível identificar com segurança** quais variáveis de ambiente existem. Então, preencha com base no seu `app.js`/README (procure por `process.env.*`).

| Variável | Valor local | Para que serve |
|---|---|---|
| PREENCHER | PREENCHER | PREENCHER |

**Ação prática:** abra `app.js` e procure por:
- `process.env.`
- URLs base do GitHub
- tokens/headers (se existir)
- valores como `minStars`, `crawlInterval`, caminhos de diretórios e nomes de arquivos (`seeds.*`, `crawled.*`, `*.csv`)

---

## Arquitetura em 5 minutos
Pense no `deepme-crawler` como um **estado-máquina persistido em arquivo** com 3 responsabilidades acopladas dentro do `app.js`:

1. **Seleção de seed**  
   - Escolhe um próximo item (usuário ou repo) a partir de arquivos do tipo `seeds.*` particionados por letra.
   - Evita item já “crawled” usando arquivos `crawled.*`.

2. **Crawl e expansão**  
   - Para um usuário: busca repos (ex.: “stars”/repositórios relevantes) e transforma alguns em **novos seeds de repos**.
   - Para um repo: busca usuários (ex.: “stargazers”) e transforma alguns em **novos seeds de usuários**.
   - Aplica regras de negócio (ex.: `minStars`, ignorar owner do próprio usuário, etc.).

3. **Persistência**
   - Remove seeds processadas e escreve markers `crawled.*`.
   - Escreve dados de repositórios em CSV (linha `stars,repo`) evitando duplicidade no arquivo.

Isso tudo roda num script único, possivelmente com **intervalos** (`crawlInterval=300` citado) e reexecução contínua (via `forever`). Como é tudo acoplado, qualquer ajuste de domínio (ex.: `minStars`) ou de ritmo pode impactar diretamente o controle de estado.

---

## Onde está cada coisa
> O repo é monolítico (provável que quase tudo esteja em `app.js`). Mesmo assim, use os pontos abaixo para se localizar.

| O que procuro | Onde encontro |
|---|---|
| Lógica principal (loop, scheduling, flow) | `app.js` |
| Funções de leitura/escrita de seeds e crawled | `app.js` (procure por `read*`, `write*`, `fs.readFile`, `fs.writeFile`, `crawled.*`, `seeds.*`) |
| Regras de negócio (`minStars`, filtros owner/self) | `app.js` (procure por `minStars`, `.split('/')`, comparação com owner) |
| Extração via HTML (jsdom) | `app.js` (procure por `jsdom`, `JSDOM`, `document.querySelector`, etc.) |
| Persistência em CSV (evitar duplicados) | `app.js` (procure por `.csv`, `append`, leitura para checar existência) |
| ESLint e scripts de npm | `package.json`, `/.eslintrc*` |
| Execução contínua | `forever` em `package.json` scripts (se existir) ou docs/infra fora do repo |

---

## Fluxos principais para entender primeiro
1. **Crawl de Usuário (seed de usuário → seeds de repos → crawled de usuário)**
   - Começa: um `username` vindo de `seeds.users.*` (particionado por primeira letra).
   - Faz: baixar/parsear a página relevante do usuário e coletar repositórios estrelados/elegíveis.
   - Aplica filtros:
     - estrelas > `minStars` (no contexto: `minStars = 10`)
     - ignorar repo cujo `owner` (primeira parte do repo) seja o próprio usuário
     - só adiciona repo como seed se **não** estiver em `crawled.repos.*`
   - Termina:
     - remover o usuário de `seeds.users.*`
     - marcar em `crawled.users.*`

2. **Crawl de Repositório (seed de repo → seeds de usuários → crawled de repo)**
   - Começa: um `owner/repo` vindo de `seeds.repos.*` (particionado por primeira letra do `owner`).
   - Faz: baixar/parsear a página relevante do repositório para listar usuários ligados (ex.: stargazers).
   - Aplica filtros:
     - ignorar usuários cujo valor seja igual ao `owner` do repo
     - só adiciona usuário como seed se **não** estiver em `crawled.users.*`
   - Termina:
     - remover o repo de `seeds.repos.*`
     - marcar em `crawled.repos.*`
     - registrar `stars,repo` no CSV do owner (evitando duplicidade)

---

## Armadilhas comuns
- **Diretórios/arquivos não existirem**: no contexto, `mkdirSync` parece estar comentado. Se os diretórios (`seeds/`, `crawled/`, `db/` ou similares) não existirem, o script pode quebrar em `fs.readFile`/`fs.writeFile`.  
  **Como lidar:** antes de rodar, crie a estrutura esperada ou habilite criação automática no runtime após confirmar o caminho real no `app.js`.

- **Loop/loop de reinício incompleto ou indefinido**: há menção de inicialização/loop que “não está presente” (ex.: arquivo terminando abruptamente). Isso pode significar que:
  - ou não roda como contínuo localmente,
  - ou o “forever” é necessário,
  - ou existe um bug/trecho truncado no que você recebeu.  
  **Como lidar:** validar entrada/saída do script: ele processa N seeds e termina? ou roda infinitamente com `setTimeout`/intervalo?

- **Função de seleção de seed com referências inconsistentes**: `readSomeSeed` pode referenciar variáveis inexistentes (ex.: `repoSeed`) ou remover/log fora do fluxo correto.  
  **Como lidar:** rode com logs (`console.log`) e coloque breakpoint por leitura do arquivo/variáveis (ou adicione logs antes de cada operação) para confirmar o comportamento real.

- **Rate limit/limites do GitHub**: mesmo com `crawlInterval=300`, scraping sem backoff pode bater limite.  
  **Como lidar:** procure se o código usa headers (`User-Agent`) e token. Se não usar, espere degradação e trate respostas 429/5xx (se já existir tratamento, documente; se não existir, é melhoria necessária).

- **Duplicidade no CSV**: existe regra para “não adicionar linha duplicada se o repo já existir no arquivo”, mas isso depende de como o check é feito (leitura inteira do arquivo? busca por string?).  
  **Como lidar:** valide performance e comportamento com arquivos grandes.

---

## Quem perguntar
- **Dúvidas de negócio:** PREENCHER (provavelmente alguém que definiu `minStars`, regras de filtros e o que é “eligible”)
- **Dúvidas técnicas:** PREENCHER (referente ao responsável pelo `deepme-crawler`, manutenção de `app.js` e infraestrutura de execução com `forever`)

---
> ⚠️ Rascunho gerado automaticamente pelo Alicerce by Malha. Adicione os passos reais de setup antes de mergear.