```markdown
# deepme-crawler — Runbook Operacional

## Informações Básicas
| Item | Valor |
|---|---|
| Porta padrão | PREENCHER (o deepme-crawler é script Node.js; se houver endpoint web, documentar aqui) |
| Health check | PREENCHER (`GET /health`) |
| Logs | PREENCHER (ex.: `./logs/*.log` ou stdout do processo forever) |
| Métricas | PREENCHER (dashboard / endpoint) |
| Alertas críticos | PREENCHER (onde configurados — ex.: PagerDuty/Alertmanager/Grafana) |

---

## Health Check e Monitoramento

### Verificar se o serviço está “vivo” (mínimo operacional)
Como o `deepme-crawler` é um crawler monolítico com persistência em arquivos locais (sem indicação clara de endpoint), o “health check” tende a ser observado por sinais indiretos:

1. **Processo está rodando?**
```bash
# Verifique via forever (se esse for o método de execução)
forever list
```

2. **Há progresso no filesystem (atualização de marcadores/CSV/seeds)?**
- Verifique timestamps e crescimento/atualização em:
  - `crawled.*` (marcadores por letra)
  - `seeds.*` (arquivos de seeds por letra)
  - CSV `RepoDBCsvFile` (linhas novas)

Exemplos (ajustar paths reais):
```bash
ls -lah ./crawled*
ls -lah ./seeds*
ls -lah ./db/*.csv
```

3. **Há erros recentes nos logs?**
```bash
# Exemplo: tail do log principal
tail -n 200 PREENCHER_CAMINHO_DO_LOG
```

### Indicadores de problema (o que observar às 3h)
- **Sem novas escritas**: timestamps de `crawled.*` e CSV não mudam há muito tempo (crawler travou ou parou de avançar seeds).
- **Erros de rede/HTTP do GitHub**: timeouts, 403 (rate limit), 5xx.
- **Aumento de reprocessamento**: seeds não diminuem (repo/usuário não está sendo removido como crawled).
- **Inconsistência de seeds**: arquivos de seeds por letra vazios mas marcadores não avançam (picking/remoção falhando).
- **Falhas de I/O**: “ENOENT” (diretório/arquivo não existe) ou “EACCES” (permissão) ao gravar seeds/crawled/CSV.

---

## Procedimentos Comuns

### Reiniciar o serviço
> Use apenas se o crawler estiver travado, sem progresso e você já verificou logs/erros.

1. **Identificar o processo**
```bash
forever list
```

2. **Reiniciar**
```bash
# Ajustar nome/app e o script de start conforme uso real do forever
forever restart PREENCHER_APP_NAME
```

3. **Se não houver app no forever, iniciar novamente**
```bash
# Ajustar comando real de execução:
# forever start -c "node" app.js
forever start app.js
```

4. **Após reiniciar**
- Monitorar:
  - `tail` do log
  - se `seeds.*` começam a diminuir e `crawled.*` a atualizar
  - se há novas linhas no CSV

---

### Verificar logs de erro
```bash
# Verifique logs mais recentes
tail -n 300 PREENCHER_CAMINHO_DO_LOG
```

Se o serviço roda sem arquivo (captura via stdout/stderr do forever), localize o log do forever:
```bash
# Exemplo (ajustar):
# forever logs PREENCHER_APP_NAME
forever logs PREENCHER_APP_NAME
```

Procure por padrões comuns:
- `TypeError` / referência a variável indefinida (há menção a inconsistências como `repoSeed`).
- `ENOENT` (diretório/arquivo ausente)
- `EACCES` (permissões)
- `429` / `403` (rate limit GitHub)
- `ETIMEDOUT` / `ECONNRESET`
- falhas de parsing do HTML/render via jsdom

---

### Forçar re-processamento de mensagem / ciclo (reprocessar seeds)
O deepme-crawler trabalha com:
- **Seeds** (inputs)
- **Marcadores crawled.*** (evitam recrawling)
- Remoção/registro ao processar

> Para “forçar reprocessamento”, você precisa alterar o estado: **marcadores `crawled.*`** e/ou **re-empilhar seeds**.

**Cuidado:** ao editar seeds/crawled, garanta consistência (seed <-> marker). Não invente seeds sem entender a partição por letra.

#### Opção A — Reprocessar um item removendo o marker correspondente
1. Identifique:
- para usuário: `crawled users` por letra (primeira letra do username)
- para repo: `crawled repos` por letra (primeira letra do owner)

2. Remova a linha/entry do arquivo `crawled.*` do item em questão.

Exemplo (paths/nomes a confirmar):
```bash
# Ajustar nomes reais e extensão
grep -n "PREENCHER_ITEM" ./PREENCHER_CAMINHO/crawled.users.a
# editar/remover a linha correspondente
```

#### Opção B — Re-adição do seed
- Adicione manualmente o usuário/repo ao arquivo de seeds da letra correta (mesma regra de particionamento).
- Garanta que o item **não** esteja marcado em `crawled.*`.

> Se houver um bug no picking (`readSomeSeed`) que impede o avanço, reiniciar pode não resolver; nesse caso, corrija a lógica antes de reprocessar em massa.

#### Opção C — Reprocessamento total (não recomendado)
- Apagar/limpar todos os `crawled.*` e reconstituir `seeds.*`.
- Somente faça se tiver janela operacional grande e controle de rate limit.

---

## Troubleshooting

### Problema: o crawler não avança (sem novas linhas no CSV / seeds não mudam)
**Causa provável:**
- loop/batch travado (ex.: exceção não tratada que encerra o ciclo)
- bug no picking/remoção de seeds (há menção a variáveis inconsistentes em `readSomeSeed`)
- falha de I/O (dir/arquivo inexistente) impedindo escrita

**Como diagnosticar:**
1. Verificar processo:
```bash
forever list
```
2. Verificar logs recentes:
```bash
tail -n 300 PREENCHER_CAMINHO_DO_LOG
```
3. Verificar timestamps de marcadores/CSV:
```bash
ls -lah ./crawled* ./db/*.csv ./seeds*
```
4. Checar se o processo ainda faz requests:
- procurar no log por requisições recentes (ex.: URLs de GitHub)
- ou por erros de rede/timeouts

**Como resolver:**
- Se o processo morreu: reiniciar (seção “Reiniciar”).
- Se persistir erro de I/O: criar diretórios/garantir permissões conforme paths esperados e reiniciar.
- Se identificar exceção lógica em `readSomeSeed`: aplicar correção (hotfix) e reiniciar; depois reprocessar seeds removendo markers do subconjunto afetado.

---

### Problema: erros 403/429 do GitHub (rate limit)
**Causa provável:**
- volume alto de requisições sem backoff
- paralelismo/loop sem controle suficiente (crawlInterval e concorrência não documentados)
- ausência de token de API (se aplicável)

**Como diagnosticar:**
- Nos logs, procurar por:
  - status HTTP `403`/`429`
  - mensagens de rate limit
  - falhas recorrentes em páginas HTML

**Como resolver:**
- Pausar/baixar ritmo: reduzir intervalo entre crawls (ajustar `crawlInterval=300` se estiver configurável).
- Implementar backoff exponencial (alteração de código) e reiniciar.
- Garantir uso de credenciais/token (se o script suportar) e documentar no runbook.
- Como mitigação operacional: reiniciar em horário com menor carga e monitorar se a taxa de erro cai.

---

### Problema: `ENOENT` / diretórios/arquivos não encontrados (não cria persistência)
**Causa provável:**
- `mkdirSync` está comentado (conforme análise), então diretórios podem não existir
- primeira execução em ambiente novo sem estrutura

**Como diagnosticar:**
- Logs com `ENOENT: no such file or directory` e path alvo
- confirm ar se `seeds.*`, `crawled.*` e CSV existem

**Como resolver:**
- Criar manualmente os diretórios/arquivos esperados (após confirmar os paths reais no código/ambiente).
- Garanta permissões de escrita para o usuário do processo.
- Reiniciar o serviço.

> Se o nome/estrutura exata não estiver no runbook, extraia do erro (path do log) e documente.

---

### Problema: duplicação no CSV / linhas repetidas
**Causa provável:**
- falha na regra “não adicionar linha duplicada se o repo já existir no arquivo”
- corrida/execução múltipla do script (rodando mais de uma instância)

**Como diagnosticar:**
1. Verificar se há múltiplas instâncias:
```bash
forever list
# verifique se existe mais de uma app do deepme-crawler
```
2. Verificar no CSV se duplicatas são recorrentes para os mesmos `repo`.

**Como resolver:**
- Pare instâncias duplicadas (mantendo apenas uma).
- Se necessário, deduplicar CSV (com cuidado) e reprocessar somente o necessário (por remoção de markers do subconjunto).

---

### Problema: seeds inconsistentes (repo removido mas não marcado como crawled, ou vice-versa)
**Causa provável:**
- fluxo de remoção/registro fora de ordem (mencionado: inconsistências em `readSomeSeed` e referências fora do fluxo)
- crash entre “remove do seed” e “registrar crawled” (janela de escrita)

**Como diagnosticar:**
- Verificar um item específico:
  - está em `seeds.*`?
  - está em `crawled.*`?
  - aparece no CSV?

**Como resolver:**
- Ajustar estado manualmente:
  - Se está em `seeds.*` mas já deveria estar crawled: remova do seed.
  - Se está marcado como crawled mas falta no CSV: reprocessar item ou corrigir escrita (depende da regra real do script).
- Reiniciar após consistência mínima.

---

## Dependências Críticas
| Dependência | O que acontece se cair | Fallback |
|---|---|---|
| GitHub (HTTP outbound) | Seeds não conseguem ser resolvidos (usuários/repos não atualizam); erros 5xx/timeout/HTML inesperado | PREENCHER (ex.: retries com backoff; parar temporariamente; usar cache) |
| Persistência local (filesystem) | Sem leitura/escrita de seeds/crawled/CSV -> loop sem progresso e/ou erros `ENOENT` | PREENCHER (ex.: garantir volume montado, snapshots, checagem de diretórios no start) |
| forever / processo Node.js | Se o processo cair, crawler para totalmente | Reiniciar via `forever restart` e checar causa raiz nos logs |

---

## Contatos de Escalação
- **Primeiro:** PREENCHER (nome/rol: responsável pelo deepme-crawler)
- **Segundo:** PREENCHER (on-call plataforma/SRE responsável por Node/infra)
- **Terceiro:** PREENCHER (time responsável por integrações GitHub/token/rate limit)

---
> ⚠️ Rascunho gerado automaticamente a partir do contexto. Preencha:
> - porta/health check (se existir)
> - paths reais de logs, seeds/crawled e CSV
> - comando real de start/restart do `forever`
> - onde estão dashboards/métricas e alertas
```