# RAIO-X TÉCNICO E MANUAL DE OPERAÇÃO — TICHATBOT
## TicketZ PRO · Typebot · NocoDB · CRM Gateway/Edge · Silo S3 · Rocket.Chat · Backup DR

**Edição:** 1.0 — consolidada em 23/10/2026  
**Natureza:** documentação técnica de arquitetura e operação, inventário *as-built*, histórico de incidentes e plano de continuidade.  
**Classificação:** USO INTERNO RESTRITO. Não contém senhas, tokens, credenciais SFTP nem dados pessoais de clientes.  
**Referência temporal:** fotografia das inspeções de 22–23/10/2026; dados operacionais mudam com o tempo.

> **Precedência:** os capítulos 1–15 deste documento registram o estado auditado atual e **prevalecem** sobre afirmações históricas divergentes. O **Anexo H** preserva integralmente o manual histórico disponibilizado, para rastreabilidade, e não deve ser interpretado como nova validação de 2026.

## 1. Resumo executivo, método e parecer

O servidor `tichatbot` encontra-se **operacional na evidência auditada**: 16 de 16 containers `running`, nenhum restart, nenhum `OOMKilled`, Docker ativo e habilitado, serviços principais acessíveis via Nginx/HTTPS e a integração Typebot → TicketZ funcionando segundo confirmação operacional do responsável. Os bancos foram auditados em modo somente leitura. O último pacote Backup DR foi gerado corretamente e copiado ao Windows com **38 hashes SHA-256 verificados, sem divergências**. O upload para nuvem foi confirmado visualmente pelo operador, não submetido a conferência remota de hash.

**Classificação de evidências:** `[V]` = saída de comando apresentada e validada; `[O]` = operação confirmada pelo responsável; `[H]` = histórico/documentos anexos, não necessariamente repetido agora; `[P]` = pendente/não ensaiado nesta auditoria; `[R]` = recomendação. Nunca transformar uma inferência em validação.

**Conclusão:** ambiente produtivo funcional e com recuperação já homologada historicamente em **VM de teste separada**. Há riscos residuais conhecidos aceitos/diferidos: backup com `ENCRYPTION_ENABLED=false`, logs Docker atuais ainda sem rotação efetiva, Docker Socket do backend por orientação do desenvolvedor, ausência de limite de memória/PIDs na maioria dos containers, endpoint DNS privado divergente do Rocket.Chat, retenção espelhada `-delete`, ausência intencional do Rocket.Chat/MongoDB no Backup DR. Não declarar “risco zero” ou “alta disponibilidade”: trata-se de **host único**, com dependências de pfSense e cópia externa.

### 1.1 Estado de validação por eixo

| Eixo | Estado | Evidência/limite |
|---|---|---|
| Host, disco, Docker e containers | [V] aprovado | Ubuntu 24.04.5, Docker 29.9.1/Compose v5.6.0; 16 running |
| HTTP/HTTPS e certificados | [V] aprovado com ressalva DNS | mchat, botbuilder, botviewer, minio, nocodb, crm-api, dialog |
| PostgreSQL/Redis | [V] aprovado | Estrutura íntegra, ausência de índices inválidos; AOF+RDB |
| Silo S3 | [V] health OK | bucket `typebot` vazio ao inspecionar, não inferir perda de dados |
| CRM NocoDB → Gateway → Edge | [V] aprovado | Roteamento, validações, códigos, isolamento e saúde |
| Typebot / TicketZ comandos | [V] estrutura; [O] funcionamento | Fluxos publicados e handoff confirmados pelo operador |
| Backup local e SHA256 | [V] aprovado | 23/10, pacote e manifesto |
| Ubuntu → Windows SFTP | [V] aprovado | 41 arquivos transferidos; 38 no manifesto íntegros |
| Windows → nuvem | [O] aprovado | Operador confirmou visualmente; hash remoto não auditado |
| Restore em VM segregada | [H] homologado | Procedimento anterior concluído com sucesso em ambiente isolado |
| Log rotation configurada | [V] arquivo válido; [P] ativação | `daemon.json` alterado, Docker não reiniciado |
| Segurança do host | [V] parcial | pfSense responsável; UFW não será ativado |

## 2. Identidade do host e capacidade

| Campo | Estado as-built |
|---|---|
| Hostname | `tichatbot` |
| OS/kernel | Ubuntu Server 24.04.5 LTS / Linux 6.8 |
| CPU | Ver inventário completo anexado para modelo/threads; não recontado nesta fase |
| RAM/swap | ~15,54 GiB RAM; swap 4 GiB, sem pressão observada |
| Disco | `/dev/sda2`, ext4, 235 GiB, ~22 GiB utilizados, ~203 GiB disponíveis; ~10% uso |
| Inodes | 15 milhões disponíveis logicamente, ~4% utilização |
| IP | 192.168.200.127/23 |
| Gateway/DNS | 192.168.200.1, pfSense/Unbound |
| Fuso host | UTC |
| Fuso operacional do backup | `America/Sao_Paulo` (UTC−3 em 23/10/2026) |
| Docker Engine | 29.9.1, build dfc520d |
| Docker Compose | v5.6.0 |
| Docker storage driver | `overlayfs` |
| Porta host | TCP 22 SSH, 80 HTTP, 443 HTTPS |
| UFW | `inactive` — decisão intencional: firewall de borda no pfSense |

Não alterar UFW para “corrigir segurança”; qualquer mudança de perímetro deve ocorrer pelo responsável pfSense. `iptables INPUT ACCEPT` foi observado no host, sem justificar isoladamente qualquer conclusão sobre regras completas do pfSense.

## 3. Diagrama lógico da solução

```text
                         Clientes / WhatsApp / Operadores
                                      |
                      pfSense (Gateway, Unbound DNS, borda)
                                      |
                            192.168.200.127
                         TCP 80 / TCP 443 (Nginx)
                                      |
           +--------------------------+--------------------------+
           |                          |                          |
        TicketZ                  Typebot                      NocoDB
      Front + Back           Builder + Viewer               App / CRM
           |                          |                          |
      PG16 ticketz              PG16 typebot           PG16 nocodb / Redis
      Redis TicketZ             Redis TicketZ                  |
           |                          |                  CRM Gateway (Node)
           +----------- Silo S3 ------+                          |
                            publicações/anexos?           rede typebot-nocodb
                                                               |
                                                       CRM Edge (Node)
                                                               |
                                                    crm-api.ncloud.app.br
                             Rocket.Chat + MongoDB (stack isolada)

           Ubuntu Backup DR --02h BRT--> /backup/completed (mesmo disco)
                                               |
                                 Windows agendado --03h BRT / WinSCP SFTP
                                               |
                                          F:\BKPSD\APIMeta
                                               |
                                     Cliente Nextcloud / nuvem
                                       [upload confirmado pelo operador]
```

Setas indicam relações arquiteturais e caminhos de dependência; não representam captura de tráfego de cada chamada. PostgreSQL `ticketz` e `typebot` são bancos separados no mesmo cluster TicketZ. NocoDB usa outro cluster PostgreSQL. O armazenamento Silo atende API S3; inventário do bucket `typebot` está vazio no momento observado.

## 4. Inventário Docker — 16 containers

| Container | Serviço | Rede primária | Usuário efetivo/observação | Estado 23/10 |
|---|---|---|---|---|
| `ticketz-docker-acme-typebot-builder-1` | Typebot Builder | nginx-proxy + ticketz | UID 1000 `node` | running, restarts 0 |
| `ticketz-docker-acme-typebot-viewer-1` | Typebot Viewer | nginx-proxy + ticketz | UID 1000 `node` | running, 0 |
| `ticketz-docker-acme-frontend-1` | Nginx frontend | nginx-proxy + ticketz | mestre root, workers UID 101 | running, 0 |
| `ticketz-docker-acme-minio-1` | Silo S3 (`pgsty/silo`) | nginx-proxy + ticketz | UID 0 | running, 0 |
| `ticketz-docker-acme-backend-1` | TicketZ backend PRO | nginx-proxy + ticketz | UID 0, `dockerode` | running, 0 |
| `ticketz-acme-companion` | emissão/renovação ACME | nginx-proxy | UID 0 | running, 0 |
| `ticketz-docker-acme-postgres-1` | PostgreSQL 16 TicketZ/Typebot | ticketz | UID 70 | running, 0 |
| `ticketz-docker-acme-redis-1` | Redis TicketZ | ticketz | UID 999 | running, 0 |
| `ticketz-nginx-proxy` | Reverse proxy | nginx-proxy | UID 0 mestre/UID 101 worker | running, 0 |
| `rocketchat-rocketchat-1` | Rocket.Chat 6.13.1 | rc-internal + nginx-proxy | UID 65533 | running, 0 |
| `rocketchat-mongodb-1` | MongoDB 6.0 | rc-internal | UID 999 | running, 0 |
| `crm-query-gateway` | Gateway CRM Node 22 | nocodb-internal + typebot-nocodb | UID 1000 | healthy, 0 |
| `crm-query-edge` | Edge HTTPS Node 22 | nginx-proxy + typebot-nocodb | UID 10001 | healthy, 0 |
| `nocodb-app` | NocoDB 2026.07.0 | nocodb-internal + nginx-proxy | UID 0 | running, 0 |
| `nocodb-postgres` | PostgreSQL 16 NocoDB | nocodb-internal | UID 70 | healthy, 0 |
| `nocodb-redis` | Redis 7 NocoDB | nocodb-internal | UID 0 | healthy, 0 |

Quatro healthchecks declarados e aprovados (`crm-query-gateway`, `crm-query-edge`, `nocodb-postgres`, `nocodb-redis`); nos outros 12, `Health=nao-configurado` **não significa falha**. Todos com `Privileged=false`, AppArmor `docker-default`, sem dispositivos explícitos, sem `OOMKilled`, IPC e cgroup namespaces privados. CRM Edge/Gateway têm `readOnlyRootfs=true`, `no-new-privileges=true`, `CapDrop=ALL`, limite RAM 128MiB e 100 processos.

### 4.1 Imagens e pinagem

TicketZ backend `ghcr.io/ticketz-oss/ticketz-backend:pro`, front `ghcr.io/ticketz-oss/ticketz-frontend:pro`. Typebot Builder/Viewer usam `baptistearno/typebot-builder:latest` / `...viewer:latest`, **tags mutáveis**. Recomendado registrar digest previamente homologado antes de atualizar; nunca assumir que `latest` instalado hoje reproduzirá amanhã exatamente os mesmos binários. Digest/image IDs coletados:
- Typebot Builder: `9b277e31c84773f49a07ef2d48c90dc83b01cabeaeef6eb1c43005264d194fdd`.
- Typebot Viewer: `1fba8b9a165c196af09c652e44feef01f6948204261d956cf7b545b2acd24425`.
- TicketZ backend PRO: `dc261f1418bdcd84839ee76cc6973b49fc9dc88fa5564cf4f7e438405d08ff2f`.
- TicketZ frontend PRO: `5912cd1b85eaecffb198ccc67581296450c1bfedbf3818b402a6cbd14d5b7e4f`.
- PostgreSQL local: `postgres:16-alpine` ID `deb28ee47118`; Silo `pgsty/silo:latest` `2636fc4155d9`.

Hash da imagem Docker local não substitui digest imutável de registro para reproduzir deploy. Histórico documenta atualização TicketZ v0.99.210-pro → v0.99.213-pro → v0.99.215-pro (22/10/2026); confirmar versão operacional pelo próprio aplicativo em cada nova manutenção.

## 5. Diretórios, stacks Compose e persistência

| Stack | Caminho Compose | Serviços |
|---|---|---|
| TicketZ + Typebot + Nginx + Silo | `/home/tichatbot/ticketz-docker-acme/docker-compose.yaml` e `docker-compose.override.yaml` | front, back, PG, Redis, Typebot, Silo, proxy, ACME |
| NocoDB | `/opt/nocodb/docker-compose.yml` | nocodb-app, nocodb-postgres, nocodb-redis |
| CRM Gateway | `/opt/crm-query-gateway/docker-compose.yml` | crm-query-gateway |
| CRM Edge | `/opt/crm-query-edge/docker-compose.yml` | crm-query-edge |
| Rocket.Chat | `/opt/rocketchat/docker-compose.yml` | Rocket.Chat, MongoDB |

Montagens críticas [V]:
- TicketZ backend: `/var/run/docker.sock:/var/run/docker.sock:ro`; `/root/.docker/config.json:/root/.docker/config.json:ro`; volumes `ticketz-docker-acme_backend_private`, `...backend_public`.
- Frontend: bind `/home/tichatbot/ticketz-docker-acme/confs/frontend-ticketz.d:/etc/nginx/ticketz.d` RW e backend_public RW.
- Nginx proxy: socket em `/tmp/docker.sock:ro`, `ngx_certs` RO, configs ACME/vhost/conf/html/dhparam e bind `confs/nginx-ticketz.conf` RW.
- ACME companion: Docker Socket RO, volumes ACME/certs/vhost/html RW.
- PostgreSQL TicketZ: `ticketz-docker-acme_postgres_data`; Redis TicketZ: `ticketz-docker-acme_redis_data`.
- Silo: `ticketz-docker-acme_minio_data:/data` RW.
- NocoDB: `nocodb_nocodb_data`, `nocodb_postgres_data`, `nocodb_redis_data`.
- Rocket.Chat: `/app/uploads` em volume anônimo com ID `df4dfbf5...`; MongoDB `/data/configdb` volume anônimo ID `b28e7783...`, `/data/db` em `rocketchat_mongodb_data`.
- CRM Gateway `/opt/crm-query-gateway/server.js:/app/server.js:ro`.
- CRM Edge `/opt/crm-query-edge/server.js:/app/server.js:ro` e `/opt/crm-query-edge/secrets/crm_edge_key:/run/secrets/crm_edge_key:ro`.

Não apagar volumes anônimos de Rocket.Chat ou MongoDB; associação é parte da recuperação **fora do escopo do Backup DR**.

## 6. Redes Docker e perímetro

| Rede | Internal | Attachable | Membros / propósito |
|---|---|---|---|
| `ticketz-docker-acme_nginx-proxy` | false | false | proxy, ACME, TicketZ front/back, Typebot ambos, Silo, NocoDB app, CRM Edge, Rocket.Chat |
| `ticketz-docker-acme_ticketz` | false | false | TicketZ front/back, Typebot, Silo, PG TicketZ, Redis TicketZ |
| `nocodb_nocodb-internal` | true | false | NocoDB app/PG/Redis, CRM Gateway |
| `typebot-nocodb` | true | true | CRM Edge e Gateway |
| `rocketchat_rc-internal` | false | false | Rocket.Chat e MongoDB |

Somente **Docker proxy** publica portas do Docker no host: `0.0.0.0:80→80/tcp` e `0.0.0.0:443→443/tcp`. SSH escuta 0.0.0.0:22 e `[::]:22`. Postgres/Redis/MongoDB são `expose` internos, sem `ports` de host. Rede `Internal=false` por si só não implica publicação de serviços na WAN; firewall é pfSense (escopo deliberadamente não alterado).

## 7. Domínios, roteamento HTTPS, DNS

| Domínio | Papel | Resultado auditado |
|---|---|---|
| `mchat.ncloud.app.br` | TicketZ | HTTPS 200 |
| `botbuilder.ncloud.app.br` | Typebot Builder | HTTPS 307 (redirecionamento esperado) |
| `botviewer.ncloud.app.br` | Typebot Viewer | HTTPS 200 |
| `minio.ncloud.app.br` | S3/Silo | raiz 403 (esperado para API sem credencial); health 200 |
| `nocodb.ncloud.app.br` | CRM NocoDB | HTTPS 200 |
| `crm-api.ncloud.app.br` | CRM Edge | `/` 404 esperado; `/health` 200 |
| `dialog.ncloud.app.br` | Rocket.Chat | 200 com `--resolve` ao IP privado; **anomalia Unbound: resposta pública 143.202.0.135** e conexão local recusada nessa rota |

A anomalia `dialog` de DNS interno ficou registrada, sem ajuste por decisão de evitar impacto. Validar DNS no pfSense quando houver demanda de acesso interno ao Rocket.Chat. Nunca substituir certificados ou mexer nas regras pfSense por suposição.

## 8. Banco de dados e Redis — estado auditado

### 8.1 PostgreSQL

- Cluster primário: PostgreSQL 16 Alpine, dois bancos `ticketz` e `typebot`, sem índices inválidos detectados.
- Cluster dedicado NocoDB: PostgreSQL 16 Alpine, banco `nocodb`, sem índices inválidos.
- TicketZ possui função `public.immutable_unaccent(text)` e índice `tags_name_unaccent_lower_index`; inspeção de `proconfig` mostrou array vazio, enquanto procedimento histórico DR documenta cuidado com `search_path`. **Não alterar função/índice em produção sem evidência de defeito.**
- Dumps `pg_dump -Fc` (`custom`, formato v1.15-0) de cada banco constam do Backup DR, além de `globals.sql` e validações de estrutura; restore exige avaliação separada de papéis, extensões, ownership e compatibilidade.
- Ausência de `psql` ou `pg_restore` no *host* Ubuntu não é impeditivo: utilitários existem nos containers PostgreSQL 16 e podem ser usados em ambiente segregado na recuperação.

### 8.2 Redis

- TicketZ e NocoDB Redis 7 com RDB e AOF habilitados; `maxmemory=0`, `maxmemory-policy=noeviction`, sem limites de RAM impostos aos containers.
- Backup completo inclui RDB e **AOF em appendonlydir**, manifestos e arquivos auxiliares.
- TicketZ: `dump.rdb` ~8.298 bytes; AOF incr. ~47.069.303 bytes; base.rdb e manifesto presentes em 23/10.
- NocoDB: `dump.rdb` ~55.368 bytes; AOF incr. ~8.201.299 bytes; base.rdb e manifesto presentes.
- Risco a monitorar: crescimento do AOF e memória/fragmentação; não executar `FLUSHALL`, `BGREWRITEAOF` ou mudanças de políticas sem diagnóstico específico.

### 8.3 Silo (API compatível com S3)

`pgsty/silo` substituiu serviço MinIO anterior preservando endpoint e nomenclatura legado `minio`; bucket `typebot` existe e estava vazio na vistoria. Política de leitura pública restrita a prefixo `typebot/public`; não confundir health 200 e 403 na raiz. O backup inclui inventário do bucket e espelhamento de objetos quando houver; não interpretar inventário vazio como prova de erro, pois 112 resultados Typebot persistidos no PostgreSQL não significam necessariamente arquivos no S3.

## 9. CRM — modelo de dados, APIs e mapeamentos

### 9.1 NocoDB

Base identificada `pjwlscmrg9ksb4p`; entidades:
- `Clientes`: 25 colunas; 1 linha no instante da inspeção (não expor valores).
- `Vendedores`: 12 colunas; 3 registros ativos.
- `_nc_m2m_Clientes_Vendedores`: 4 colunas; 1 associação, sem órfãos.
- Mapeamento vendedor→tag TicketZ [V]: Viviane Stampini (vendedor 1) → tag 1 `Viviane`; Josiane Valerio (2) → tag 2 `Josiane`; Ana Julia Oliveira (3) → tag 5 `Ana Julia`.
- TicketZ Tags adicionais: 3 `Pagar`, 4 `Receber`.
- Queues TicketZ: 1 RH, 2 SAC, 3 Quero Vender, 4 Quero Comprar, 5 Financeiro, 6 Dep Pessoal, 7 Diretoria, 8 Faturamento, 9 Logistica, 10 Produção, 11 Atendimento.

Contagens são **fotografia**, nunca requisito de negócio fixo. Valores de documentos CPF/CNPJ, contatos e mensagens não são reproduzidos.

### 9.2 Gateway privado

Arquivo `/opt/crm-query-gateway/server.js` (349 linhas), SHA256 `03dd796d1cb5bb3f69ace49cd66c984c07d031b7aaca315ad23b86de1190ad82`; usa Node 22 Alpine (ambiente interno Node 22.23.1). Atende consultas CPF de 11 dígitos, CNPJ de 14 e WhatsApp `^55\d{10,11}$`; consome API NocoDB v3 com timeout ~5s, projeção mínima de campos e avaliação de unicidade. Estados `NAO_ENCONTRADO`, `DUPLICADO`, `ENCONTRADO`, eventualmente indicação de falha. Campos selecionados incluem `Nome`, `TipoPessoa`, `Cidade`, `UF`, `Vendedor`, `VendedorRef`, `VendedorNomeLookup`, `TicketZTagIDLookup`, `VendedorAtivoLookup`, `Ativo`. Gateway isolado em rede interna, **sem autenticação própria observada na interface interna**, exigindo preservação rigorosa da segmentação.

### 9.3 Edge exposto via HTTPS

Arquivo `/opt/crm-query-edge/server.js` (217 linhas), SHA256 `e211b1e596f60c42d2d1ca6e159099798465b1c18a0efd5b83ef4177c11b0544`. GET `/health` responde 200; endpoints POST `/v1/clientes/documento` e `/v1/clientes/whatsapp` usam header **`x-crm-gateway-key`** (não `X-API-Key`). Sem chave ou chave inválida: 401. Segurança: `timingSafeEqual`, corpo até 4096 bytes, rate-limit 120 requisições por janela configurada por IP `x-forwarded-for`, timeout upstream 6000 ms, `Cache-Control: no-store`, headers restritivos; falha upstream pode responder 200 com `status=ERRO_TECNICO` — clientes devem interpretar **status no JSON**, não só HTTP. Preservar proteção de borda e limitar confiança em `X-Forwarded-For` ao proxy correto.

### 9.4 Fluxo lógico CRM

```text
Canal WhatsApp -> TicketZ -> Typebot (triagem)
                  |
              CPF/CNPJ ou WhatsApp
                  |
           CRM Edge (chave obrigatória)
                  |
          CRM Gateway (rede privada)
                  |
              NocoDB (vendedor/tag)
                  |
       Typebot interpreta status e mapeamento
                  |
       #{"action":"addTag","tagId":N}
       #{"action":"note",...} / #{"action":"updateTicket",...}
       #{"action":"endSession"}
                  |
             TicketZ: fila/atendente
```

O formato exato de cada payload depende dos blocos publicados e comandos documentados. Nunca testar `endSession`, `updateTicket` ou `addTag` em conversas reais sem janela/contato de homologação.

## 10. Typebot — inventário, versionamento de publicação e protocolo

### 10.1 Inventário do banco Typebot

| Fluxo | Identificador | Situação | Estrutura |
|---|---|---|---|
| `Atendimento - Triagem - Homologação` | `cmtnotfyi00001lqirdv7w312` / público `atendimento-triagem-homologa-o-dv7w312` | Publicado; editor = publicação em hash | 33 grupos, 115 blocos, 45 variáveis, 39 arestas, 1 evento |
| `CRM - Homologação NocoDB` | `cms3wanhu00011lp4gy2m6qgt` / público `crm-homologa-o-noco-db-y2m6qgt` | Publicado; **editor divergente da publicação** | 22 grupos, 53 blocos, 28 variáveis, 24 arestas, 1 evento |
| `Meu typebot` | — | Não publicado | vazio |

- Fluxo principal publicado 13/10/2026 14:53:34; editor 14:53:29, hash de estrutura `31522ff0fb646e78908bbe5600189c7d`.
- Fluxo CRM: publicação histórica 30/07/2026, editor modificado 25/09/2026; hashes do editor `1689e7df050f665a13033d861a28a865`, publicado `e4c0a3004a6b497438933be20c17a808`. **A edição atual não deve ser confundida com o conteúdo em produção**.
- Typebot `Result`: 112 registros na auditoria, sem leitura de respostas pessoais.
- Ambos têm Webhook com 11 mapeamentos de resposta (`data.status`, dados do cliente, vendedor, tag, flags) validados, 22/22 associações consistentes.
- Aresta residual órfã em triagem, ID `q7am6rymhq84l7m955cylo6k`, saída `y675be5y6lpgvmr3go0wc31r` / item `njq8zazwim2w68pt9wy6xbun`, destino grupo `z3dag2usvrlwth8aianpqeif` (`00 - Menu principal`). Sem referência ativa de origem; não afetou funcionamento comprovado. **Não “corrigir” sem uma versão controlada do fluxo.**
- Não há arestas com destino inexistente nos fluxos publicados.

### 10.2 Handoff Typebot ↔ TicketZ

A triagem contempla filas 1–5 para RH, SAC, Quero Vender, Quero Comprar e Financeiro, e tags financeiras 3/4. Comandos usam mensagens com prefixo `#` e JSON serializado. A lógica `decisao_tag_ticketz` aplica tag de vendedor somente quando o vendedor está mapeado e ID é inteiro positivo; caso contrário `PULAR`. Sequências verificadas em blocos publicados: addTag → note → updateTicket → endSession, ou note → updateTicket → endSession sem tag; Financeiro pode incluir tag específica. **Responsável confirmou execução funcional em produção**; não foi disparada nova simulação invasiva.

**Exemplos de sintaxe** (não copiar IDs fictícios para produção):
```text
#{"action":"addTag","tagId":1}
#{"action":"updateTicket","ticketData":{"status":"pending"}}
#{"action":"endSession"}
```

### 10.3 Procedimento de manutenção dos fluxos

1. Registrar o `typebotId`, `publicId`, data do editor/publicação e hash antes de editar.
2. Exportar JSON da versão operacional, preservando segredo e anonimização.
3. Alterar em clone/homologação; conferir mapeamento Webhook e comandos TicketZ.
4. Publicar somente após teste aprovado, com plano de rollback da publicação.
5. Comparar editor e published; atualizar inventário e evidências.

## 11. Backup DR — arquitetura real, cronologia e resultados

### 11.1 Componentes e política

- Scripts em `/opt/backup-dr/bin/`: `backup-check-pre.sh`, `backup-configs.sh`, `backup-execution-id.sh`, `backup-minio.sh`, `backup-postgres.sh`, `backup-redis.sh`, `backup-retention.sh`, `backup-run.sh`, `backup-volumes.sh`.
- Biblioteca `/opt/backup-dr/lib/backup-common.sh`; config `/etc/backup-dr/backup-dr.conf`; unidades `/etc/systemd/system/backup-dr.{service,timer}`.
- Timer: `OnCalendar=*-*-* 05:00:00` UTC, `RandomizedDelaySec=5min`, `Persistent=true`; 02h BRT + até 5 min. `backup-dr.service` oneshot, root, timeout 2h.
- Parâmetros confirmados: `KEEP_VALID_BACKUPS=7`, `KEEP_FAILED_DAYS=30`, `MIN_FREE_GIB=30`, `TIMEZONE=America/Sao_Paulo`, `ENCRYPTION_ENABLED=false`.
- Pastas `/backup/completed`, `/backup/failed`, staging/logs; no mesmo disco `/dev/sda2`. Último pacote `2026-10-23_02h04m28s`, serviço sucesso em 23/10 05:04:28 UTC, ~92.534.858 bytes transferidos (41 arquivos) para Windows.
- `SHA256SUMS` conferido localmente no Ubuntu; no Windows **38 arquivos conferidos, zero ausentes, zero divergentes**. Outros arquivos de controle podem não fazer parte do manifesto; não interpretar 38 como quantidade total.
- `BACKUP_OK`, `NOTIFICATION_SENT`, `manifest.json` presentes; notificação SMTP aceita na rotina auditada.
- Postgres `ticketz.dump` 855.793 B, `typebot.dump` 321.771 B, `nocodb.dump` 394.769 B; dumps custom v1.15-0.
- S3 `minio/typebot/inventory.txt` (38 B), buckets inventory, sem objetos na coleta.
- Pacotes `configs/crm-edge.tar.zst`, `crm-gateway.tar.zst`, `ticketz-project.tar.zst` **contêm segredos/configurações**; proteger acesso e distribuição.
- Pacotes de volumes: backend_public/private, 6 itens nginx cert/acme/config/vhost/html/dhparam e `nocodb_nocodb_data`. Dados de Postgres/Redis protegidos pelos dumps e backups especializados; Rocket.Chat/MongoDB intencionalmente excluídos, inclusive seus volumes.

### 11.2 Cadeia comprovada

```text
Ubuntu 02h BRT Backup DR /backup/completed
   └─ SFTP `cobian-reader`, chroot /backup, leitura ACL limitada
         └─ Windows tarefa `Copia_backup` 03h BRT
              └─ WinSCP synchronize local -delete
                    └─ F:\BKPSD\APIMeta
                          └─ Nextcloud para nuvem [O: upload visto pelo operador]
```

Pasta sincronizada pelo Nextcloud não foi extraída pela consulta ao `nextcloud.cfg`, mas upload foi confirmado visualmente pelo responsável e investigação adicional dispensada por decisão expressa. **Retenção e integridade dos objetos diretamente na nuvem não foram submetidas a teste remoto.** `-delete` mantido conscientemente para refletir sete backups locais; risco de propagação de exclusões ao Windows é aceito, e retenção independente remota não comprovada.

### 11.3 Incidente 23/10 e solução comprovada

**Sintoma:** tarefa Windows `Copia_backup` exit 1 às 03h, WinSCP erro SFTP `Permission denied` ao ler `NOTIFICATION_SENT` de 22/10; backup de 23/10 ausente no Windows até ação corretiva.

**Causa raiz:** `/backup/completed/<backup-id>/` tinha ACL de leitura/travessia `cobian-reader`, mas `backup-run.sh` aplicava ACL recursiva **antes** de criar marcadores `BACKUP_OK`, `NOTIFICATION_SENT`, `NOTIFICATION_FAILED`; cada marcador era criado depois com `chmod 0600` e sem ACL de leitura. Leitura `SHA256SUMS` permitida; leitura de marcadores negada.

**Correção executada e aprovada:**
1. `setfacl -m u:cobian-reader:r--` nos marcadores existentes de sete pacotes sem alterar owner `root:root`.
2. Em `/opt/backup-dr/bin/backup-run.sh`, três linhas `setfacl -m "u:cobian-reader:r--" "${COMPLETED_DIR}/<MARCADOR>"` imediatamente após respectivos `chmod 0600`. Backup do script `/opt/backup-dr/bin/backup-run.sh.before-sftp-acl-20261023-171811`. `bash -n` OK.
3. WinSCP listagem `ls /completed/2026-10-23_02h04m28s/` exit 0; transferência manual não destrutiva `get` de todo o pacote exit 0; 41 arquivos/92.534.858 bytes no Windows, SHA256 38/38 OK.
4. Config WinSCP antes `-hostkey="*"` (**inseguro**); comparada chave pública SSH direta Ubuntu ED25519 `SHA256:A6faNYNxO0UltDjt71SZ1AvimSDFqraWglT6DReF5sQ` com log WinSCP idêntico.
5. Config WinSCP passou a `-hostkey="ssh-ed25519 255 A6faNYNxO0UltDjt71SZ1AvimSDFqraWglT6DReF5sQ"` **sem prefixo `SHA256:`**, pois WinSCP espera essa sintaxe. Verificação final SFTP exit 0, sem aviso de wildcard. Cópias do arquivo antes da mudança: `sincronizar_backup.txt.before-hostkey-20261023-144958` e outra cópia de ajuste de formato com timestamp.
6. Último serviço `backup-dr.service`: `Result=success`, `ExecMainCode=1`, `ExecMainStatus=0`, `ActiveState=inactive` (esperado após `oneshot`). Timer `enabled/active`, próxima execução registrada **24/10/2026 05:00:57 UTC**, correspondendo a **02h00min57s BRT**.

**Limite de evidência:** primeiro backup criado pelo timer *após* mudança de script ainda não ocorreu na fotografia 23/10; deverá ser conferido após 24/10. A tarefa Windows automática ainda apresenta `LastTaskResult=1` da execução **anterior à correção**; não confundir com a transferência manual posterior, que foi bem-sucedida.

### 11.4 Recuperação e DR já homologados

O procedimento de restauração foi **homologado anteriormente em VM de teste separada**, segundo manual DR fornecido; não caracteriza teste novo nesta auditoria. Preservar ordenação, DNS/segredos, funções e migrations registradas no procedimento específico. Para um novo DR:
1. Estabelecer incidente, RTO/RPO permitidos e aprovar janela.
2. Escolher pacote `BACKUP_OK`, validar manifesto `SHA256SUMS` e completude (inclusive AOF, certs, dumps).
3. Preparar VM **isolada**, versões de imagens e Compose auditadas, volumes *limpos somente na VM de teste*.
4. Restaurar PostgreSQL 16 por cluster, `globals.sql` e os bancos necessários; conferir owners e extensões.
5. Restaurar Redis RDB/AOF no caminho esperado para Redis 7 preservando `appendonly.aof.manifest`, respeitando integridade/sem concorrência.
6. Restaurar volumes de upload, Nginx e dados NocoDB; recuperar inventário/objetos S3 se existirem e testar permissões.
7. Aplicar configurações e **credenciais a partir de cofre/backup restrito**, não do manual. Resolver nomes para VM isolada para evitar envio real ao WhatsApp.
8. Validar DB, filas, Typebot publicados, API CRM, frontend, TLS e uma sessão de homologação; registrar evidências e rollback.
9. Só promover caso todos os testes sejam aprovados. **Não executar esses comandos destrutivos em produção.**

Não prometer tempo de restauração: métricas RTO/RPO não foram medidas nesta auditoria, e a cópia em nuvem não foi integralmente testada com SHA256 remoto.

## 12. Segurança operacional — decisões e riscos aceitos

| Item | Estado | Decisão vigente |
|---|---|---|
| Firewall UFW | inactive | **Manter**: pfSense é responsável, por determinação do operador |
| Backend Docker Socket | RO no bind; efetivamente utilizável pela API | **Manter** por orientação do desenvolvedor; imagem possui `dockerode`; risco conhecido |
| `/root/.docker/config.json` backend | RO, JSON `{}`, permissões host 644 root:root | Sem tokens armazenados; manter por ora |
| ACME e Nginx proxy Docker Socket | RO | Necessário para descoberta/certificados conforme Compose; acesso à API continua sensível |
| Segredos CRM | `crm_edge_key` RO em `/run/secrets/` | Não divulgar nem colocar em documentação |
| Containers root | backend, Silo, NocoDB, nginx/acme/redis NocoDB | Registrar; não mudar usuário sem compatibilidade e teste |
| Sem limites RAM/PIDs maioria | 14 containers sem limit explícito | Risco residual aceito; monitorar |
| Logs sem rotação efetiva | `json-file`, `Options={}` em **todos 16** | Config preparada; ativar somente em manutenção |
| Backup criptografia | `ENCRYPTION_ENABLED=false` | Risco residual de confidencialidade em backups locais/cópias |
| Retenção espelhada Windows | `synchronize local -delete` | **Manter** como política intencional |
| Rocket.Chat/MongoDB fora do DR | exclusão intencional | **Manter** e documentar claramente |
| Nextcloud/nuvem | upload visualmente confirmado | **Não auditar mais**, decisão expressa do operador |
| DNS dialog privado | divergência com resposta pública | Registrar, não ajustar sem demanda |
| Typebot aresta órfã e CRM rascunho divergente | evidências estáticas | Não editar fluxos funcionando |

### 12.1 Docker Socket: por que `:ro` não isola a API

Montar `/var/run/docker.sock` `read_only=true` impede alterações ao **arquivo de socket dentro do filesystem**, mas **não impede escrever requisições HTTP na API Docker** por uma conexão Unix ao socket. Um processo root que controla a API pode obter privilégios importantes no host; aceite explícito do risco e dependência de fabricante devem constar na gestão de mudanças.

## 13. Rotação de logs Docker — mudança de 23/10/2026

**Antes:** `/etc/docker/daemon.json` continha um parâmetro `userland-proxy` (valor intencionalmente não reproduzido); `json-file` padrão, `Options={}` em todos 16 containers, sem regra `logrotate` identificada. Log maior MongoDB Rocket.Chat 15,86 MiB, NocoDB Redis 13,79 MiB, proxy 5,51 MiB, TicketZ Backend 5,40 MiB. Disco livre 203 GiB.

**Executado [V]:** backup `/etc/docker/daemon.json.before-log-rotation-20261023-192147`; preservado `userland-proxy`; acrescentados:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```
Este JSON mostra **somente o trecho novo**, não é o arquivo inteiro; o valor real de `userland-proxy` já existente **deve ser preservado**. `dockerd --validate --config-file /etc/docker/daemon.json` retornou `configuration OK`. Docker segue `active`; 16 containers seguem `running`, `Restarts=0`.

**Estado de aplicação:** **PREPARADA, NÃO ATIVADA NO DAEMON EM EXECUÇÃO**. Sem restart do Docker, Engine mantém configuração anteriormente carregada. Mesmo após aplicação ao daemon, containers existentes não passam automaticamente a usar a nova opção: precisam ser **recriados de forma planejada**, por stack, em manutenção. NÃO reiniciar Docker ou recriar serviços apenas por este documento. Para o limite ativo em novos containers: 10 MiB × 5 = aproximadamente 50 MiB por container; 16 containers ≈ 800 MiB se todos adotarem a política, sem considerar outros logs.

### 13.1 Procedimento seguro quando aprovada janela

1. Registrar estado `docker ps`, backup de `daemon.json`, configs Compose e backup DR válido.
2. Garantir acesso administrativo ao host e identificar método compatível de atualização do daemon sem interromper tráfego; mudança pode demandar restart, documentar downtime previsto.
3. Validar `dockerd --validate` e aplicar configuração com plano de retorno.
4. Em janela, recriar **um serviço por vez**, respeitando dependências e persistência; realizar smoke tests (HTTPS, CRUD saudável e roteamento WhatsApp/Typebot) a cada lote.
5. Confirmar `docker inspect <container> --format '{{json .HostConfig.LogConfig.Config}}'` reflete `"max-size":"10m","max-file":"5"` e que Nginx/ACME operam normalmente.
6. Em anomalia, restaurar arquivo backup e plano de rollback do daemon/containers, sem apagar volumes.

## 14. Runbooks de manutenção não destrutiva

### 14.1 Preflight de observação (somente leitura)

```bash
hostnamectl
timedatectl
df -hT / /backup /var/lib/docker
free -h
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker stats --no-stream
systemctl list-timers backup-dr.timer --no-pager
systemctl show backup-dr.service -p Result -p ExecMainStatus
docker system df
```

### 14.2 Conferir backup sem ler segredo

```bash
sudo find /backup/completed -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort -r | head
# Substituir ID por pasta real, jamais usar curingas em restore:
ID="2026-10-23_02h04m28s"
sudo test -f "/backup/completed/$ID/BACKUP_OK" && echo OK
sudo sh -c 'cd "$1" && sha256sum -c SHA256SUMS --status' sh "/backup/completed/$ID" && echo HASHES_OK
sudo namei -l "/backup/completed/$ID/NOTIFICATION_SENT"
sudo getfacl -p "/backup/completed/$ID/NOTIFICATION_SENT"
```

### 14.3 Verificar código e stacks antes de qualquer upgrade

```bash
cd /home/tichatbot/ticketz-docker-acme
docker compose config --quiet
docker compose ps
docker inspect ticketz-docker-acme-backend-1 --format '{{.Image}}'
sudo bash -n /opt/backup-dr/bin/backup-run.sh
sudo dockerd --validate --config-file /etc/docker/daemon.json
```

`docker compose config` normalmente inclui env resolvidas; **não publicar saída completa** nem salvar logs públicos. `docker compose config --quiet` evita impressão de segredos.

### 14.4 Healthchecks HTTP já caracterizados

`https://mchat.ncloud.app.br`, `https://botbuilder.ncloud.app.br`, `https://botviewer.ncloud.app.br`, `https://nocodb.ncloud.app.br` retornaram códigos coerentes; `https://crm-api.ncloud.app.br/health` deve responder 200, enquanto raiz pode responder 404; endpoint S3 raiz 403 pode ser esperado, health 200. Checar DNS pfSense antes de atribuir 502 à aplicação.

### 14.5 Rotina após próximo backup diário

```bash
systemctl list-timers backup-dr.timer --no-pager
systemctl show backup-dr.service -p Result -p ExecMainStatus
ULTIMO=$(sudo find /backup/completed -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort -r | head -1)
echo "Pacote: $ULTIMO"
sudo -u cobian-reader test -r "/backup/completed/$ULTIMO/BACKUP_OK" && echo BACKUP_OK_SFTP_LEITURA_OK
sudo -u cobian-reader test -r "/backup/completed/$ULTIMO/NOTIFICATION_SENT" && echo NOTIFICATION_SFTP_LEITURA_OK
```

Validação de leitura só prova ACL, não integridade. Antes de anunciar sucesso do dia, verificar serviço `success`, SHA256 e `Copia_backup` no Windows exit 0 após 03h BRT.

### 14.6 Gestão de incidentes

- **HTTP 502:** verificar resolução DNS, proxy, container, logs **redigidos**, redes e serviço upstream; não apagar volumes.
- **CRM 401:** verificar header `x-crm-gateway-key` pelo chamador e secret via método aprovado; não enviar segredos por chat.
- **CRM `ERRO_TECNICO` com HTTP 200:** investigar conectividade Gateway/NocoDB, não confundir HTTP com status funcional.
- **Tags incorretas:** conferir vendedor ativo, lookup/tag ID e published JSON antes de editar.
- **Backup SFTP erro 3:** diagnosticar ACL marcadores e execução do timer; jamais `chmod -R 777`.
- **Restauração:** somente sobre ambiente isolado e com procedimentos do DR homologado.

## 15. Matriz de pendências, prioridades e critérios de aceite

| Prioridade | Item | Critério de encerramento |
|---|---|---|
| Alta / próximo ciclo | Novo backup automático após correção ACL | `Result=success`, marcador legível por `cobian-reader`, hash local íntegro, tarefa Windows exit 0 |
| Alta / política de proteção de dados | `ENCRYPTION_ENABLED=false` | Decidir criptografia em repouso e proteção das cópias; sem urgência para alterar sistema vivo, mas risco reconhecido |
| Média / manutenção planejada | Rotação logs em daemon/containers | reinício/recriação aprovados, `LogConfig` com `max-size`, `max-file` |
| Média / processo | Auditoria periódica das versões mutáveis `:latest`/`:pro` | registrar digests, planejar updates e rollback |
| Média / rede | DNS `dialog.ncloud.app.br` | resolver divergência apenas com autorização |
| Média / capacidade | Redis AOF, memória, falta de limites | monitorar tendências; não mudar sem medição |
| Média / continuidade | Cópia nuvem sem hash remoto | Operador validou visualmente e dispensou auditoria adicional; documentar alcance |
| Aceito / intencional | Rocket.Chat/MongoDB excluído do Backup DR | manter escopo documentado |
| Aceito / intencional | Docker Socket backend | manter segundo orientação do fornecedor |
| Aceito / intencional | UFW desligado / pfSense | não alterar |
| Aceito / intencional | WinSCP `-delete` | manter espelhamento da retenção |
| Baixa / controlada | Aresta órfã Typebot | somente revisão em ciclo de versão homologado |

### 15.1 Termo de fechamento de auditoria

Em **23/10/2026**, após comando de auditoria final: 16/16 containers running e restart 0, `/etc/docker/daemon.json` validado com política de rotação preparada, timer Backup DR pronto para próxima execução, cópia local Windows íntegra, upload nuvem confirmado pelo operador. **Nenhuma validação equivale a um compromisso de disponibilidade futura; mudanças exigem procedimento e evidência.**

## 16. Procedência documental e trilha de evidências

Fontes [V]: saídas sucessivas de terminal Ubuntu e PowerShell Windows no processo de auditoria (fases 1 a 7), incluindo `docker inspect`, `docker network inspect`, verificações de bancos, Typebot published JSON, SHA256, WinSCP e `systemctl`. Fontes [H]: manual mestre anterior, relatório DR, resumo de recuperação outubro, Compose fornecido, exports JSON Typebot, documentação CRM Gateway, Typebot e TicketZ. Evidências do usuário [O]: handoff produtivo e upload Nextcloud; decisões explícitas sobre pfSense, Rocket.Chat, Docker Socket, `-delete`, não auditar mais nuvem.

**Controle de mudanças:** este documento consolida o *estado constatado*, não cria nova configuração em servidores. Não há dados sigilosos intencionalmente incorporados, mas nomes internos, IP privado e topologia justificam circulação restrita.

---

# ANEXO H — MANUAL HISTÓRICO (REFERÊNCIA, NÃO ESTADO ATUAL)

O material abaixo é preservado a partir do manual técnico consolidado anterior. **Quando houver conflitos, prevalecem capítulos 1–16, que incorporam a auditoria atual de 23/10/2026.** Em especial, anotações antigas sobre backups falhos, estado anterior de versões e testes não realizados podem ter sido posteriormente resolvidas e homologadas. Os comandos históricos são referências técnicas, não autorização para execução indiscriminada.

# Manual técnico de operação, manutenção e recuperação — TicketZ PRO, Typebot, CRM e Backup DR

**Servidor:** `tichatbot`  
**Data de consolidação:** 23/10/2026  
**Escopo:** documentação consolidada dos arquivos técnicos disponibilizados e das ações efetivamente relatadas nesta conversa, incluindo atualização de `v0.99.210-pro` para `v0.99.213-pro` e, depois, para `v0.99.215-pro`.  
**Classificação:** uso interno — infraestrutura e procedimentos operacionais. **Não distribuir publicamente.**  
**Última evidência de saúde:** verificações Docker/HTTP/HTTPS e comparações de arquivos após `v0.99.215-pro` em 22/10/2026. **Testes funcionais completos e primeiro backup agendado após a atualização não foram comprovados nesta conversa.**

> **Como interpretar o documento:**  
> **Confirmado** = comando/saída fornecidos no diálogo; **documentado anteriormente** = informação dos arquivos técnicos históricos anexados, não revalidada em outubro; **pendente** = precisa de verificação; **recomendação** = instrução proposta, não executada. Nenhum valor secreto real é reproduzido.
>
> **Ponto operacional decisivo:** não restaurar indiscriminadamente arquivos antigos após atualizações. Comparar o estado desejado e corrigir somente a causa demonstrada. Nunca executar `docker compose down -v`, `docker volume prune` ou procedimentos destrutivos como tentativa de corrigir um 502.

---

# 1. Visão geral do projeto

## 1.1 Objetivo do servidor

O host `tichatbot` hospeda o atendimento TicketZ PRO, automação Typebot, CRM NocoDB, gateway de consulta protegida ao CRM, serviços de mensageria e armazenamento, proxy HTTPS e rotinas de backup, retenção e auditoria. O fluxo de negócio descrito na documentação histórica recebe contatos no TicketZ, inicia uma conversa Typebot, valida CPF/CNPJ, consulta o cadastro no NocoDB, identifica o vendedor, aplica tag/nota e devolve o ticket para atendimento humano.

Em outubro de 2026 o objetivo adicional foi **garantir estabilidade após atualizações** e recuperar dois incidentes concretos: `502 Bad Gateway` do Typebot Builder e interrupção da rotina de backup pelo uso de uma imagem Docker inacessível.

## 1.2 Tecnologias e componentes

| Serviço | Papel | Implementação / imagem observada |
|---|---|---|
| Ubuntu Server 24.04 LTS | Sistema operacional | Host `tichatbot`; inventário de outubro relata 24.04.5 LTS |
| Docker Engine + Docker Compose | Execução e configuração de stacks | Serviços em containers |
| TicketZ PRO backend/frontend | Atendimento e painel | `ghcr.io/ticketz-oss/ticketz-backend:pro`, `...frontend:pro` |
| Typebot Builder e Viewer | Edição e execução de fluxos | `baptistearno/typebot-builder:latest`, `baptistearno/typebot-viewer:latest` |
| Nginx Proxy + ACME Companion | HTTPS, roteamento, certificados | `nginxproxy/nginx-proxy`, `nginxproxy/acme-companion` |
| PostgreSQL / Redis da stack TicketZ | Bancos `ticketz` e `typebot`, persistência/cache | `postgres:16-alpine`, `redis:7-alpine` |
| Silo (compatível com MinIO) | Serviço S3 atual | `pgsty/silo`; cliente de buckets `pgsty/mc` |
| NocoDB + PostgreSQL/Redis separados | CRM e persistência isolada | `nocodb/nocodb:2026.07.0` observado em outubro |
| Gateway CRM / Edge HTTPS | Consulta ao CRM sem expor token NocoDB ao Typebot | Node.js 22 Alpine; containers `crm-query-gateway`, `crm-query-edge` |
| Rocket.Chat e MongoDB | Serviços adicionais presentes no host | `rocketchat-rocketchat-1`, `rocketchat-mongodb-1`; não foram alterados nesta conversa |
| `systemd` | Agendamento dos backups e outras rotinas | `backup-dr.service`, `backup-dr.timer` |
| pfSense / Unbound | DNS interno e roteamento da rede | `192.168.200.1` conforme recuperação de 12/10 |

**Nota sobre Typebot `latest`:** a documentação antiga descrevia fixação por versão/digest; o usuário posteriormente informou decisão operacional de **manter `latest`**. Esta é a política atual relatada. O uso de `latest` pode modificar a versão efetiva no próximo `docker compose pull`, portanto registrar o ID/digest antes e depois da atualização é importante.

## 1.3 Arquitetura resumida

```text
Navegador / WhatsApp
   │
   ├─ HTTPS 443 ──> nginx-proxy ──> TicketZ Frontend ──> TicketZ Backend
   │                                      │                    ├── PostgreSQL (ticketz)
   │                                      │                    └── Redis TicketZ
   ├─ HTTPS ──────> nginx-proxy ──> Typebot Builder ── PostgreSQL (typebot)
   ├─ HTTPS ──────> nginx-proxy ──> Typebot Viewer  ── PostgreSQL (typebot)
   │                                     │
   │                                     └───────── Silo/MinIO S3 (atualmente bucket vazio)
   ├─ HTTPS ──────> nginx-proxy ──> NocoDB ── PostgreSQL e Redis exclusivos
   └─ HTTPS ──────> nginx-proxy ──> CRM Edge (:3001)
                                               │ chave X-CRM-Gateway-Key
                                               ▼
                                           CRM Gateway (:3000)
                                               │ token NocoDB Viewer
                                               ▼
                                           API NocoDB

systemd backup-dr.timer ──> backup-dr.service ──> backup-run.sh
        ├─ dumps PostgreSQL TicketZ/Typebot/NocoDB
        ├─ Redis RDB/AOF
        ├─ cliente S3 pgsty/mc ──> bucket Silo
        ├─ arquivos/volumes/configurações
        ├─ SHA256/manifests e retenção
        └─ notificação SMTP
```

Redes descritas na documentação prévia:

- `ticketz-docker-acme_nginx-proxy`: proxy HTTPS e serviços publicados; **é compartilhada com stacks externos**, razão pela qual `docker compose down` indicou `Resource is still in use` durante a atualização.
- `ticketz-docker-acme_ticketz`: rede interna da stack TicketZ, recriada pelo atualizador.
- `nocodb_nocodb-internal`: rede privativa do CRM.
- `typebot-nocodb`: rede de integração Edge/Gateway; nome histórico, sem acesso direto do Builder/Viewer segundo documentação anterior.

**Lição do erro 502:** o container Typebot pode estar ligado a mais de uma rede Docker. A aplicação deve responder na interface acessível ao Nginx; `HOSTNAME=0.0.0.0` permite escutar nas interfaces do container.

---

# 2. Resumo cronológico das etapas

## 2.1 Implantação e personalizações históricas (documentadas nos anexos de julho/agosto)

1. Inventário Ubuntu, Docker, recursos, volumes, portas, DNS e redes.
2. Identificação do risco de `POSTGRES_HOST_AUTH_METHOD=trust` no banco original e decisão de **não compartilhar esse PostgreSQL com o NocoDB**.
3. Implantação do NocoDB em `/opt/nocodb`, com PostgreSQL e Redis próprios, integração ao proxy, DNS e HTTPS.
4. Configuração SMTP, base CRM, tabela de clientes, tabela de vendedores, relacionamentos, lookups e IDs de tags.
5. Criação de conta técnica NocoDB Viewer e token de API somente leitura; testes de escrita com `403`.
6. Desenvolvimento de fluxo Typebot para normalização/validação de CPF/CNPJ, tratamentos de cadastro ativo, inativo, ausente, duplicado e erro técnico.
7. Implantação de Gateway CRM em `/opt/crm-query-gateway` e Edge HTTPS em `/opt/crm-query-edge`, com autenticação própria e sem desativar a proteção SSRF do Typebot.
8. Configuração de handoff ao TicketZ com comandos de tag, nota, status pendente e fim de sessão; homologação histórica de tickets de teste.
9. Implantação de retenção de resultados do Typebot e auditoria automática de documentos duplicados no NocoDB.
10. Snapshots, manifests e documentação dos arquivos e segredos (não incluídos neste manual).
11. Implantação histórica da rotina de backup DR, posteriormente revisada após incidentes de DNS e imagens.

**Precisão histórica:** estas etapas foram documentadas nos arquivos técnicos anteriormente anexados; a conversa de outubro não repetiu a instalação, nem verificou integralmente suas versões/configurações atuais.

## 2.2 Recuperação de rede e Viewer — 12/10/2026

1. Verificado conflito entre DNS Forwarder (`dnsmasq`) e DNS Resolver (`Unbound`) no pfSense.
2. Desativado o Forwarder; reativado o Resolver com entradas internas.
3. Ubuntu e DHCP ADM passaram a usar apenas `192.168.200.1` como DNS; DNS público adicional `208.67.222.222` foi retirado de clientes internos, não necessariamente do próprio pfSense.
4. Viewer Typebot retornava HTTP 502 apesar de container `Up`.
5. Adicionado `HOSTNAME: 0.0.0.0` no serviço `typebot-viewer` em `/home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml`.
6. Recriado **somente Viewer** e validado HTTPS 200.
7. Corrigido envio de notificação SMTP de backup, afetado por DNS; validação de mensagem recebida.
8. Em uma estação Chrome, limpeza de caches/conexões resolveu falha específica do navegador. Outras estações ficaram pendentes.

## 2.3 Recuperação do Builder após v0.99.213-pro — outubro/2026

1. Após atualização de `v0.99.210-pro` para `v0.99.213-pro`, Builder apresentou `502 Bad Gateway nginx/1.31.6`.
2. Logs do container indicavam Prisma conectado ao banco `typebot`, 84 migrations encontradas, nenhuma pendente, Next.js 16.2.9 e aplicação `Ready`.
3. `curl` a partir da rede Docker do Nginx retornou `HTTP 000`/`Connection refused`.
4. Teste dentro do Builder: `127.0.0.1:3000` recusado; `172.18.0.7:3000` respondeu `307`; `172.19.0.9:3000` recusado. Variável `HOSTNAME=ce24feb96ee6`.
5. Identificada forte causa: bind do Next.js em uma interface Docker inadequada à comunicação com o proxy.
6. Criado backup timestamp do Compose Override; adicionada entrada `typebot-builder: environment: HOSTNAME: 0.0.0.0`.
7. `docker compose config --quiet` sem erros; recriado somente Builder com `--no-deps --no-build --force-recreate`.
8. Reteste na rede Nginx passou a responder HTTP 307; interface Builder abriu no navegador.
9. Logs confirmaram `Network: http://0.0.0.0:3000` e `docker compose config` confirmou HOSTNAME correto.
10. Incidente de acesso encerrado. **Não foi provado que a versão do TicketZ, isoladamente, causou o novo bind**; a recriação/atualização foi o contexto observado.

## 2.4 Recuperação do backup DR — 21/10/2026

1. E-mail de 21/10 indicou falha às 02:05 BRT, após cerca de 7 s.
2. Journal comprovou etapas PostgreSQL TicketZ, Typebot e NocoDB concluídas, inclusive validação `pg_restore --list`; Redis TicketZ e NocoDB concluídos.
3. Falha na etapa S3: tentativa de obter `quay.io/minio/mc:latest` retornou **401 Unauthorized**.
4. `grep` encontrou referência em `/opt/backup-dr/bin/backup-minio.sh`, linha 26.
5. `docker image inspect pgsty/mc` comprovou imagem local; testes `mc --version`, `mc mirror --help`, `mc admin info --help`, `mc ls --help` passaram.
6. Feita cópia `backup-minio.sh.bak-<timestamp>`; substituída somente a imagem para `pgsty/mc:latest`; `bash -n` aprovado.
7. `systemctl start backup-dr.service` foi executado manualmente e terminou com `Result=success`, `ExecMainStatus=0`.
8. Backup de 21/10 às 12:27 BRT, ID `2026-10-21_12h27m15s`, arquivado em `/backup/completed/`.
9. SHA256, dumps, Redis, volumes, configs e notificação SMTP aceitos; e-mail efetivamente recebido.
10. Bucket `typebot` apresentou **zero objetos**, confirmado pelo operador como esperado.
11. Aviso não bloqueante: diretório `/backup` estava com permissão `755` (revisão de segurança pendente).
12. `ls /backup/success` não encontrou caminho porque **a pasta correta é `/backup/completed`**.

## 2.5 Preparação e atualização para v0.99.215-pro — 22/10/2026

1. Criado diretório de inventário `/root/diagnostico-pre-update-099215`.
2. Registrados containers, imagens, redes, Compose principal/override, hashes, inspeções Typebot, serviço/timer de backup.
3. Confirmadas configurações `HOSTNAME=0.0.0.0` nos dois componentes e `pgsty/mc:latest` no backup.
4. Referência HTTP/HTTPS registrada: Builder interno 307, Viewer interno 404, TicketZ 200, Builder HTTPS 307, Viewer HTTPS 200, NocoDB 200, Silo 403, CRM Edge raiz 404.
5. Backup automático de **22/10, 02:01:20 BRT** apresentou `BACKUP_OK`, `NOTIFICATION_SENT`, `SHA256SUMS`, 100 MB e etapas concluídas.
6. Baixado, sem executar inicialmente, script `https://update.ticke.tz` em `/root/diagnostico-pre-update-099215/update-ticketz.sh` (aproximadamente 11 KB); sintaxe Bash aprovada.
7. Identificado no script: uso de `git stash`/fetch/checkout/pull, migrações Compose, potencial reinício do Docker condicionado ao `daemon.json`, `docker compose pull`, `down`, `up -d` e `docker system prune -af`.
8. Operador executou **`curl -sSL update.ticke.tz | sudo bash`**. Script informou migração `001` não necessária, `002` e `003` não aplicáveis; baixou imagens, removeu e iniciou containers.
9. Rede `ticketz-docker-acme_nginx-proxy` não foi removida por estar em uso (compartilhamento conhecido); nova rede `ticketz` foi criada.
10. Após atualização: Builder/Viewer `Up`, ambos `HOSTNAME=0.0.0.0`; conectividade interna e HTTP/HTTPS manteve códigos anteriores.
11. Comparação `diff -u` dos dois arquivos Compose com inventário pré-update deu **nenhuma diferença**.
12. Script de backup manteve `pgsty/mc:latest`; timer `backup-dr.timer` continuou agendado.
13. Filtro nos logs recentes de backend, Builder, Viewer, postgres e Silo não encontrou os padrões `error|fatal|panic|exception|refused|timeout|failed` na amostra.
14. **Ainda pendentes neste histórico:** testes funcionais de login/edição/publicação, consulta CRM, handoff real, e resultado do **primeiro backup automático posterior à atualização**. Estado `Up` e HTTP não substituem estes testes.

---

# 3. Configuração detalhada por componente

## 3.1 Stack principal do TicketZ

**Caminho:** `/home/tichatbot/ticketz-docker-acme`  
**Arquivos críticos:** `docker-compose.yaml`, `docker-compose.override.yaml`, arquivos `.env-*` e `confs/`.

**O que faz:** orquestra frontend/backend TicketZ, Nginx/ACME, PostgreSQL, Redis, Silo e Typebot.

**Por quê:** separa serviços e redes; possibilita atualização, monitoramento, cópia de configuração e recriação individual.

**Operação segura:**

```bash
cd /home/tichatbot/ticketz-docker-acme
docker compose config --quiet
docker compose ps -a
docker compose logs --tail 100 frontend backend
```

**Alterar valor:** primeiro copiar o arquivo que será editado com `cp -p arquivo arquivo.bak-$(date +%Y%m%d-%H%M%S)`; editar com `sudo nano`; verificar `docker compose config --quiet`; identificar exatamente quais serviços precisam ser recriados. Jamais usar `down -v` para aplicar variável.

**Compose Override efetivamente homologado:**

```yaml
services:
  backend:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ~/.docker/config.json:/root/.docker/config.json:ro
  typebot-builder:
    environment:
      HOSTNAME: 0.0.0.0
  typebot-viewer:
    environment:
      HOSTNAME: 0.0.0.0
  minio:
    image: pgsty/silo
  createbuckets:
    image: pgsty/mc
```

`backend` monta o socket Docker e configuração do cliente em modo leitura, conforme migração oficial. **Isso representa acesso sensível ao Docker e exige revisão de segurança**, mesmo com montagem `:ro` no socket. Os serviços MinIO e createbuckets têm esses nomes históricos no Compose embora utilizem Silo e cliente PGSTY.

### 3.1.1 Variáveis `.env` relevantes

Valores inferidos dos exemplos de configuração fornecidos; **verificar no servidor antes de alteração**:

| Arquivo em `/home/tichatbot/ticketz-docker-acme/` | Papel / exemplos |
|---|---|
| `.env-backend` | `BACKEND_HOST`, `BACKEND_PATH=/backend`, `FRONTEND_HOST`, `DB_HOST=postgres`, `DB_PORT=5432`, `DB_USER=ticketz`, `DB_NAME=ticketz`, `REDIS_URI=redis://redis:6379`, `TZ` |
| `.env-frontend` | `VIRTUAL_HOST`, `VIRTUAL_PORT=80`, `LETSENCRYPT_HOST`, `SETUP_SYSTEM` |
| `.env-integrations` | `TYPEBOT_BUILDER_HOST`, `TYPEBOT_VIEWER_HOST`, `MINIO_HOST`, parâmetros SMTP |
| `.env-typebot-common` | `DATABASE_URL`, `NEXTAUTH_URL`, `NEXT_PUBLIC_VIEWER_URL`, `S3_*`, `ENCRYPTION_SECRET`, `DISABLE_SIGNUP` |
| `.env-typebot-builder` | `AUTH_TRUST_HOST`, `VIRTUAL_HOST`, `VIRTUAL_PORT=3000` |
| `.env-typebot-viewer` | `VIRTUAL_HOST`, `VIRTUAL_PORT=3000` |
| `.env-minio` | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, rotas e vhost |
| `.env-secrets` | Segredos do Typebot/MinIO; **não exibir valores** |

**Atenção técnica:** o Compose pode interpolar variáveis a partir de `.env` da shell/projeto, mas valores presentes em `env_file` não são automaticamente intercambiáveis em todos os contextos de interpolação. Não imprimir `docker compose config` completo em tickets públicos, pois pode expandir segredos.

## 3.2 Proxy reverso Nginx e HTTPS

- Container: `ticketz-nginx-proxy`.
- Portas publicadas no host: `80/tcp` e `443/tcp`.
- Companion: `ticketz-acme-companion`.
- Arquivos de configuração: `/home/tichatbot/ticketz-docker-acme/confs/nginx-ticketz.conf`, `/home/tichatbot/ticketz-docker-acme/confs/frontend-ticketz.d/`.
- Volumes Compose: certificados, vhosts, configuração e arquivos ACME (`ngx_*`).
- `VIRTUAL_HOST`/`VIRTUAL_PORT` direcionam nomes para containers e portas internas.

**Testes:**

```bash
docker exec ticketz-nginx-proxy nginx -t
docker logs --tail 100 ticketz-nginx-proxy
curl -sSI https://botbuilder.ncloud.app.br/
```

`HTTP 502` geralmente significa upstream inacessível/sem resposta válida, não necessariamente problema de certificado ou DNS. Antes de reiniciar proxy, testar diretamente a aplicação **pela rede do proxy**.

## 3.3 Typebot Builder / Viewer — configuração de rede

**Configuração decisiva:** `/home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml`, em ambos os serviços:

```yaml
environment:
  HOSTNAME: 0.0.0.0
```

**Por quê:** instruir o Next.js a escutar nas interfaces de rede do container para que o Nginx consiga conectá-lo. `HOSTNAME` observado antes da correção Builder era o hostname Docker, e apenas uma rede respondia na porta 3000.

**Comprovação efetiva:**

```bash
cd /home/tichatbot/ticketz-docker-acme
docker compose config --format json |
  jq -r '.services["typebot-builder"].environment.HOSTNAME,
         .services["typebot-viewer"].environment.HOSTNAME'
```

Resultado homologado: duas linhas `0.0.0.0`.

**Teste pela rede do Nginx:**

```bash
docker run --rm --network ticketz-docker-acme_nginx-proxy \
  --entrypoint sh curlimages/curl:8.12.1 -c '
for svc in typebot-builder typebot-viewer; do
  echo "$svc"
  curl -sS -o /dev/null -w "HTTP %{http_code} / IP %{remote_ip}\n" \
    --max-time 15 "http://ticketz-docker-acme-${svc}-1:3000/"
done'
```

**Valores referenciais pós-update 215:** Builder `307`; Viewer na raiz interna `404`; Viewer HTTPS `200`. O `404` interno é comportamento observado na rota raiz, **não teste de execução de fluxo**.

**Correção se a variável faltar ou a porta ficar inacessível:** antes de alterar, coletar `docker compose config`, `docker logs`, `docker inspect` e teste HTTP. Fazer backup do Override, restaurar bloco HOSTNAME se ausente, validar Compose, recriar **somente** o serviço afetado:

```bash
cd /home/tichatbot/ticketz-docker-acme
docker compose config --quiet
docker compose up -d --no-deps --no-build --force-recreate typebot-builder
# alternativamente: typebot-viewer, somente se esse for o serviço afetado
```

Após recriar, conferir logs (`Next.js ... Network: http://0.0.0.0:3000` no Builder), teste da rede Nginx e acesso navegador.

**Nunca alterar a variável de ambos sem constatar necessidade.** E não atualizar para tag/digest fixa sem decisão formal: política relatada atual é `latest`.

## 3.4 PostgreSQL e Redis

- Stack principal: `ticketz-docker-acme-postgres-1` e `ticketz-docker-acme-redis-1`.
- PostgreSQL: `postgres:16-alpine`, porta interna 5432; bancos `ticketz` e `typebot`.
- Redis: `redis:7-alpine`, porta interna 6379; persistência em AOF e RDB conforme evidências de backup.
- NocoDB possui `nocodb-postgres` e `nocodb-redis` **independentes**.

**Cuidados:** não trocar usuário, banco, volume ou estratégia de autenticação com o serviço em uso. O Compose original apresentado utilizava `POSTGRES_HOST_AUTH_METHOD=trust`; isso constitui risco a revisar com migração segura e testes, não mudá-lo diretamente no meio desta atualização.

**Diagnóstico de leitura:**

```bash
docker ps --filter name=postgres
docker logs --tail 80 ticketz-docker-acme-postgres-1
docker logs --tail 80 nocodb-postgres
docker logs --tail 80 ticketz-docker-acme-redis-1
```

**Backups:** usar rotina DR existente; não considerar cópia de volume PostgreSQL em operação equivalente a dump consistente.

## 3.5 Silo (MinIO) e cliente S3

**Stack:** serviço Compose chamado `minio`, container `ticketz-docker-acme-minio-1`, imagem `pgsty/silo`. `createbuckets` usa `pgsty/mc` e normalmente encerra em `Exited (0)`.

- API S3 interna: `9000`; console interna: `9001`.
- Host HTTPS: `minio.ncloud.app.br`.
- Bucket `typebot`, verificado vazio em 21/10.
- No teste HTTPS da rota raiz retornou `403`, igual antes da atualização: indica resposta de endpoint restrito; não comprova listagem de objetos.

**Motivo da escolha da imagem atual:** as imagens MinIO antigas foram substituídas no Compose por alternativas PGSTY. Não substituir sem validar compatibilidade S3, ACL e scripts.

**Decisão futura:** operador pretende trocar armazenamento de imagens por links externos e remover Silo, **mas não agora**. Não remover Silo nem variáveis `S3_*` antes de verificar se o Typebot as usa para arquivos, mídia, imports ou outras funções.

## 3.6 NocoDB e CRM (implantação histórica)

**Diretório:** `/opt/nocodb`; configuração histórica `/opt/nocodb/.env` e `/opt/nocodb/docker-compose.yml`.

- Containers observados: `nocodb-app`, `nocodb-postgres`, `nocodb-redis`.
- Host público: `https://nocodb.ncloud.app.br`.
- Porta aplicação: `8080` **interna**, sem publicação pública direta recomendada.
- Base CRM histórica: `CRM`, ID `pjwlscmrg9ksb4p`.
- Tabela Clientes histórica: ID `m3t1rfxn4cvy1es`.
- Tabela Vendedores e campos `VendedorRef`, `TicketZTagID`, `Ativo`, lookups de nome/tag/status.
- Auditoria de documentos usa `QtdOcorrenciasDocumento`, `StatusDuplicidadeDocumento`.

**Propósito:** manter cadastro organizado e isolado da base do TicketZ. A auditoria detecta duplicidade, mas **não impede** cadastro duplicado por constraint.

**Procedimento de alteração:** registrar IDs atuais pela UI/API, copiar `/opt/nocodb/.env` com permissões restritas, editar somente parâmetro necessário, validar `docker compose config --quiet`, recriar somente app quando a mudança exigir, testar consulta via UI e Gateway. Troca de chaves JWT/criptografia/senhas do banco requer plano especial de rotação e recuperação.

## 3.7 Gateway e Edge CRM (implantação histórica)

**Gateway:**
- Diretório `/opt/crm-query-gateway`;
- arquivos `/opt/crm-query-gateway/server.js`, `.env`, `docker-compose.yml`;
- container `crm-query-gateway`, porta `3000` apenas interna;
- usa token NocoDB de conta Viewer, armazenamento protegido em `.env`;
- consulta documento/WhatsApp, minimiza resultados, reconhece encontrados, não encontrados, duplicados, erros técnicos e vendedor ativo/inativo.

**Edge:**
- Diretório `/opt/crm-query-edge`;
- arquivos `/opt/crm-query-edge/server.js`, `docker-compose.yml`, `secrets/crm_edge_key`;
- container `crm-query-edge`, porta interna `3001`;
- domínio `https://crm-api.ncloud.app.br`;
- exige header `X-CRM-Gateway-Key`, com chave separada do token NocoDB;
- proteção de limite de requisições, body, timeout e resposta de erro.

**Por quê:** Typebot utiliza endpoint HTTPS sem precisar acessar rede privada nem receber token da API NocoDB.

**Testes:**

```bash
docker ps --filter name=crm-query
docker logs --tail 80 crm-query-gateway
docker logs --tail 80 crm-query-edge
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  https://crm-api.ncloud.app.br/health
```

O `404` observado para `/` no Edge após a atualização **não é** teste suficiente da API de consulta. Não publicar header com chave real em logs, shell history ou conversa.

**Alterar URL/tabela/token:** backup restrito de `/opt/crm-query-gateway/.env`; gerar credencial Viewer apropriada; mudar só valores necessários; recriar Gateway com Compose desse diretório; testar consulta autorizada. **Alterar chave Edge exige atualização coordenada do arquivo de segredo E do header configurado no fluxo Typebot** — caso contrário a automação retornará 401.

## 3.8 Fluxos Typebot e handoff para TicketZ (histórico)

A documentação prévia e o export JSON fornecido descrevem fluxos com menu de triagem, validação de documento, consulta CRM, tags e passagem para humano.

**Integração TicketZ:** a fila com driver Typebot usa URL base **do Viewer** e `Public ID` do fluxo publicado. O TicketZ fornece variáveis de contexto, conforme documentação anexada, incluindo `number`, `pushName`, `firstMessage`, `ticketId`, `backendURL`, `token`, `metadata`. O token contextual não deve ser exibido publicamente.

**Triggers especiais enviados por bolha de texto** começam com `#` seguido de JSON, por exemplo:

```text
#{"action":"endSession"}
#{"action":"updateTicket","ticketData":{"status":"pending"}}
#{"action":"addTag","tagId":99}
```

`99` é **exemplo**, não ID confirmado do ambiente atual. `queueId` e `userId` devem ser obtidos no TicketZ antes de alterar qualquer handoff. Algumas integrações anteriores optaram por deixar ticket na mesma fila `Vendas` em `pending`, evitando disparar novamente o bot.

**Como alterar fluxo com segurança:** duplicar/exportar o fluxo, usar ambiente/contato de teste, conferir variáveis/IDs e cabeçalhos de autenticação, publicar versão desejada, testar no Viewer e via TicketZ. Não usar tickets de clientes reais como laboratório. O export JSON anexado é um artefato de homologação e não garante ser a versão atualmente publicada.

## 3.9 Retenção Typebot e auditoria NocoDB (histórico)

**Retenção Typebot:** `/opt/typebot-results-retention`, com serviço/timer systemd associados, script Python, configuração e token protegido (`secrets/api_token`). Rotina remove resultados antigos que possam conter dados pessoais, conforme documentação anterior (aproximadamente 24 horas). **Confirmar nomes exatos de unidades** com:

```bash
systemctl list-timers --all | grep -Ei 'typebot|retention'
find /opt/typebot-results-retention -maxdepth 2 -type f -printf '%p\n'
```

**Auditoria de duplicidade:** `/opt/nocodb-duplicate-audit`, `config.env`, `secrets/api_token`, timer de 30 em 30 minutos, usuário Linux `nocodb-audit`. Campos principais: `QtdOcorrenciasDocumento` e `StatusDuplicidadeDocumento`. Histórico registra `DRY_RUN=false`, `PAGE_SIZE=100`, `PATCH_BATCH_SIZE=100`. Não iniciar auditoria manual sem verificar concorrência e necessidade de escrita.

**Configuração/alteração:** backup de `config.env` e scripts com restrição de permissão, testar inicialmente `DRY_RUN` conforme mecanismo da ferramenta, validar consulta paginada e depois habilitar escrita. Credencial Editor da auditoria deve ser distinta da credencial Viewer do Gateway.

## 3.10 Rede corporativa, DNS e certificados

Histórico de 12/10:

- pfSense `192.168.200.1` fornece DNS às estações ADM;
- desativado `dnsmasq` concorrente, mantido `Unbound`;
- servidor Ubuntu e DHCP ADM foram ajustados para não alternar indevidamente para DNS público;
- a resolução DNS interna de `mchat.ncloud.app.br` apontava à rede local e acesso externo depende de NAT/HTTPS;
- problema de cache Chrome em uma estação não era erro de servidor.

**Alterar DNS futuro:** documentar registros no pfSense, preparar janela, testar `resolvectl status`, `getent hosts nome`, consultas DNS da rede interna e HTTPS. Não desativar regras NAT ou alterar Netplan remotamente sem plano de reversão.

---

# 4. Backup DR — documentação operacional detalhada

## 4.1 Componentes

| Caminho / unidade | Função |
|---|---|
| `/etc/systemd/system/backup-dr.service` | serviço oneshot `User=root`, `Group=root`, `WorkingDirectory=/opt/backup-dr`, `ExecStart=/opt/backup-dr/bin/backup-run.sh`, timeout `2h`, `UMask=0077` |
| `backup-dr.timer` | agendamento diário automático |
| `/opt/backup-dr/bin/backup-run.sh` | orquestrador das etapas |
| `/opt/backup-dr/bin/backup-minio.sh` | backup S3 com cliente `mc` |
| `/opt/backup-dr/lib/backup-common.sh` | funções comuns de log e execução |
| `/etc/backup-dr/backup-dr.conf` | configuração do backup; pode conter dados sensíveis |
| `/backup/.staging/` | cópias em progresso |
| `/backup/completed/` | backups concluídos |
| `/backup/failed/` | tentativas interrompidas |
| `/backup/retention/` | componentes de retenção |
| `BACKUP_OK` | marcador de execução concluída |
| `NOTIFICATION_SENT` | registro de notificação |
| `SHA256SUMS`, `manifest/` | metadados e verificações de integridade |

**Valores da execução automática de referência:** `2026-10-22_02h01m20s` com ~100 MB, backup PostgreSQL, Redis, MinIO, volumes e configurações, marcador `BACKUP_OK`, SMTP aceito e SHA256SUMS.

## 4.2 Incidente comprovado e correção permanente

Trecho crítico original:

```bash
local mc_image="quay.io/minio/mc:latest"
```

Falhou com `401 Unauthorized` ao tentar baixar a imagem. Corrigido, em `/opt/backup-dr/bin/backup-minio.sh`:

```bash
local mc_image="pgsty/mc:latest"
```

**Como mudar no futuro:** primeiro verificar imagem disponível e comandos (`mc admin info`, `mc ls`, `mc mirror`), criar `cp -p` do script, editar somente a variável, executar `sudo bash -n /opt/backup-dr/bin/backup-minio.sh`, conferir `grep -n mc_image` e então supervisionar backup de teste. Se a imagem for atualizada, testar API S3; ter o comando `mc` não garante compatibilidade de autenticação.

## 4.3 Execução manual e validação

Antes de executar, conferir se serviço já está ativo e espaço livre:

```bash
sudo systemctl is-active backup-dr.service
sudo systemctl list-timers --all backup-dr.timer
df -h /backup
```

Se não houver execução em andamento, iniciar sob supervisão:

```bash
sudo systemctl start backup-dr.service
sudo systemctl show backup-dr.service -p Result -p ExecMainStatus
sudo journalctl -u backup-dr.service --since '30 minutes ago' --no-pager
```

Critério mínimo: `Result=success`, `ExecMainStatus=0`, diretório novo em `/backup/completed/`, `BACKUP_OK`, checksums e notificação. Recebimento na caixa postal é verificação adicional ao aceite SMTP.

**Verificar integridade do último backup em diretório conhecido:**

```bash
cd /backup/completed/2026-10-22_02h01m20s
sudo sha256sum -c SHA256SUMS
```

O comando verifica arquivos contidos na lista; **não comprova uma restauração funcional**. Faça restore em ambiente isolado regularmente.

## 4.4 Política de retenção e permissões

Logs anteriores indicam política de **manter os sete backups mais recentes**; a operação chegou a remover backup expirado. **Confirmar política atual** em `/etc/backup-dr/backup-dr.conf` e nos scripts antes de mudar.

Aviso persistente: `/backup` com permissão `755`, enquanto a rotina recomenda `700`. **Pendência de segurança**, não causa da falha de 21/10. Antes de mudar: avaliar ACLs de `/backup/completed`, eventual usuário de cópia externa e acesso legítimo; ajustar de forma coordenada, não cegamente.

Não há evidência nesta conversa de que exista cópia externa completa/restauração testada; isso deve entrar na política DR.

---

# 5. Principais diretórios e arquivos

| Diretório / arquivo completo | Conteúdo, responsabilidade e cautelas |
|---|---|
| `/home/tichatbot/ticketz-docker-acme/` | projeto principal Docker Compose |
| `/home/tichatbot/ticketz-docker-acme/docker-compose.yaml` | definição principal dos serviços, volumes e redes |
| `/home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml` | personalizações persistentes Builder/Viewer/PGSTY/backend |
| `/home/tichatbot/ticketz-docker-acme/.env-backend` | conexão e URLs do backend |
| `/home/tichatbot/ticketz-docker-acme/.env-frontend` | frontend, vhost, certificados |
| `/home/tichatbot/ticketz-docker-acme/.env-integrations` | domínios integração/SMTP, potencialmente sensível |
| `/home/tichatbot/ticketz-docker-acme/.env-typebot-common` | banco Typebot, URLs e S3 |
| `/home/tichatbot/ticketz-docker-acme/.env-typebot-builder` | host/porta e autenticação Builder |
| `/home/tichatbot/ticketz-docker-acme/.env-typebot-viewer` | host/porta Viewer |
| `/home/tichatbot/ticketz-docker-acme/.env-minio` | credenciais/virtual host Silo |
| `/home/tichatbot/ticketz-docker-acme/.env-secrets` | segredos criptográficos |
| `/home/tichatbot/ticketz-docker-acme/confs/` | configurações adicionais Nginx e frontend |
| `/opt/nocodb/` | stack NocoDB, `.env`, `docker-compose.yml` |
| `/opt/crm-query-gateway/` | API intermediária CRM, `.env`, `server.js`, Compose |
| `/opt/crm-query-edge/` | borda HTTP protegida, `server.js`, Compose, `secrets/` |
| `/opt/typebot-results-retention/` | rotina de retenção de dados Typebot |
| `/opt/nocodb-duplicate-audit/` | auditoria de CPF/CNPJ duplicado |
| `/opt/backup-dr/bin/backup-run.sh` | orquestração backup DR |
| `/opt/backup-dr/bin/backup-minio.sh` | cliente `pgsty/mc:latest`, backup bucket |
| `/opt/backup-dr/lib/backup-common.sh` | funções auxiliares |
| `/etc/backup-dr/backup-dr.conf` | parâmetros do backup |
| `/etc/systemd/system/backup-dr.service` | execução backup |
| `/etc/systemd/system/backup-dr.timer` | agenda |
| `/backup/completed/`, `/backup/failed/`, `/backup/.staging/` | resultados, falhas e temporários |
| `/root/diagnostico-pre-update-099215/` | inventário anterior a atualização e cópias dos dois Compose |
| `/root/diagnostico-pre-update-099215/update-ticketz.sh` | script oficial baixado em 22/10 para inspeção |
| `/etc/docker/daemon.json` | configuração global Docker; atualizador pode verificar `userland-proxy` |
| `/etc/netplan/` | Netplan de rede; nome do arquivo específico deve ser identificado no host |

**Restrição:** o inventário em `/root` e arquivos `.env` podem expor informações de infraestrutura e segredos; manter controle de acesso.

---

# 6. Variáveis, domínios, portas, IDs e credenciais

## 6.1 Domínios e URLs

| Identificador | Valor de referência | Onde alterar |
|---|---|---|
| TicketZ frontend | `https://mchat.ncloud.app.br` | `.env-frontend` e DNS |
| TicketZ backend | `https://mchat.ncloud.app.br/backend` | `.env-backend`, DNS, proxy |
| Typebot Builder | `https://botbuilder.ncloud.app.br` | `.env-integrations`, `.env-typebot-builder` |
| Typebot Viewer | `https://botviewer.ncloud.app.br` | `.env-integrations`, `.env-typebot-viewer` |
| Silo/MinIO | `https://minio.ncloud.app.br` | `.env-integrations`, `.env-minio` |
| NocoDB | `https://nocodb.ncloud.app.br` | `/opt/nocodb/.env`, DNS |
| Edge CRM | `https://crm-api.ncloud.app.br` | `/opt/crm-query-edge/docker-compose.yml`, DNS |

## 6.2 Portas e bancos

| Recurso | Porta/valor | Exposição |
|---|---|---|
| Nginx HTTP/HTTPS | `80/443` | host |
| SSH | `22` | host (controlar com firewall) |
| Builder / Viewer | `3000` | interna |
| Backend TicketZ | `3000` | interna, roteada via frontend/proxy conforme config |
| Gateway CRM | `3000` | interna |
| Edge CRM | `3001` | interna, via Nginx |
| NocoDB | `8080` | interna |
| PostgreSQL | `5432` | interna |
| Redis | `6379` | interna |
| Silo API / console | `9000/9001` | internas |
| Banco TicketZ | `ticketz` | no PostgreSQL principal |
| Banco Typebot | `typebot` | no PostgreSQL principal |
| DB NocoDB | `nocodb` (histórico) | PostgreSQL próprio |

## 6.3 IDs, relacionamentos e opções

| Item | Valor registrado | Origem / alteração |
|---|---|---|
| NocoDB Base ID | `pjwlscmrg9ksb4p` | `/opt/crm-query-gateway/.env` e auditoria (histórico) |
| NocoDB Clientes Table ID | `m3t1rfxn4cvy1es` | gateway/auditoria (histórico) |
| Campo vendedor referência | `VendedorRef` | interface CRM NocoDB |
| Campo tag TicketZ | `TicketZTagID` | tabela `Vendedores` |
| Campos duplicidade | `QtdOcorrenciasDocumento`, `StatusDuplicidadeDocumento` | auditoria |
| `HOSTNAME` Builder/Viewer | `0.0.0.0` | Compose Override, **validado em outubro** |
| Cliente backup S3 | `pgsty/mc:latest` | `/opt/backup-dr/bin/backup-minio.sh`, validado |
| Imagens Typebot | `baptistearno/typebot-*:latest` | Compose, decisão operacional atual |
| Retenção backup | 7 últimos (observado) | confirmar config DR |
| Bucket S3 | `typebot` | Silo; vazio em 21/10 |
| `TYPEBOT_ENCRYPTION_SECRET` | **REDACTED** | `.env-secrets`; não rotacionar sem planejamento |

## 6.4 Segredos — nunca incluir valores no manual

| Segredo / usuário | Local de armazenamento | Alteração segura |
|---|---|---|
| SMTP Gmail / senha de aplicativo | `.env-integrations` e configuração NocoDB | gerar nova senha, substituir com arquivo restrito, testar SMTP |
| Token NocoDB Viewer | `/opt/crm-query-gateway/.env` | regenerar conta Viewer, alterar Gateway e testar consultas |
| Token NocoDB Editor | `/opt/nocodb-duplicate-audit/secrets/api_token` | atualizar auditor e testar `DRY_RUN` |
| Chave CRM Edge | `/opt/crm-query-edge/secrets/crm_edge_key` | atualizar Edge e header Typebot de modo coordenado |
| Token Typebot retenção | `/opt/typebot-results-retention/secrets/api_token` | gerar novo token e validar job |
| Credenciais MinIO/Silo | `.env-secrets`/`.env-minio` | rotação coordenada com Typebot e rotina backup |
| Segredos banco Redis/Postgres NocoDB | `/opt/nocodb/.env` | rotação com janela e validação |
| Segredo de criptografia Typebot | `.env-secrets` e `.env-typebot-common` | risco de inviabilizar dados criptografados se trocado arbitrariamente |
| Docker auth | `/root/.docker/config.json` ou montagem de configuração backend | restringir acesso, não compartilhar saída |

**Forma segura de alteração:** backup com permissão restrita; modificar arquivo fonte (não apenas `docker exec`), validar serviço e dependências, recriar mínimo necessário, testar operação fim a fim; evitar `cat` de arquivo de segredo na conversa. Não copiar credenciais de produção para documentação.

---

# 7. Procedimentos de manutenção comuns

## 7.1 Diagnóstico inicial (somente leitura)

```bash
hostnamectl
free -h
df -h
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker network ls
cd /home/tichatbot/ticketz-docker-acme
docker compose ps -a
docker compose config --quiet
```

## 7.2 Logs e serviços individuais

```bash
docker logs --tail 100 ticketz-docker-acme-typebot-builder-1
docker logs --tail 100 ticketz-docker-acme-typebot-viewer-1
docker logs --tail 100 ticketz-nginx-proxy
docker compose logs --since 15m --tail 100 backend frontend postgres minio
sudo journalctl -u backup-dr.service -n 100 --no-pager
```

Logs podem conter tokens, URLs internas, documentos ou credenciais. Sanitizar antes de compartilhar.

## 7.3 Reiniciar / recriar sem interromper toda a stack

```bash
cd /home/tichatbot/ticketz-docker-acme
docker compose restart typebot-builder
# se a variável Compose mudou e precisa recriar:
docker compose up -d --no-deps --no-build --force-recreate typebot-builder
```

Para Viewer, trocar `typebot-builder` por `typebot-viewer`. `restart` reinicia com configuração antiga do container; **mudança de variáveis no Compose exige recriação**.

Para outras stacks, entrar em seu diretório e confirmar nome exato do serviço com `docker compose ps` antes do `restart`.

## 7.4 Diagnóstico de HTTP 502 Typebot

1. Verificar container `Up` e logs.
2. Confirmar `HOSTNAME` efetivo no Compose e no container (`docker exec ... printenv HOSTNAME`).
3. Testar `127.0.0.1` e IPs Docker a partir do container se necessário.
4. Testar a aplicação a partir de `ticketz-docker-acme_nginx-proxy`.
5. Conferir Nginx/upstream só então.
6. Se comprovado bind errado, backup do Compose Override, configurar `0.0.0.0`, validar e recriar somente container afetado.
7. Repetir teste interno/HTTPS e abrir interface real. Não associar qualquer 502 automaticamente à variável HOSTNAME.

## 7.5 Atualizar TicketZ com segurança

**Procedimento oficial utilizado pelo operador:**

```bash
curl -sSL update.ticke.tz | sudo bash
```

**Risco:** executa imediatamente conteúdo remoto com privilégios root. O script inspecionado em 22/10 executava operações Git, migrações Compose, pull/down/up e prune; poderia reiniciar Docker sob certas condições. Em novas atualizações, **rebaixar/reinspecionar** porque o endpoint pode mudar.

**Prática recomendada:**

1. Janela de manutenção e comunicação aos operadores.
2. Testes funcionais e inventário pré-update.
3. Confirmar backup DR concluído e, idealmente, cópia externa/restauração testada.
4. Registrar `docker image inspect` dos serviços e IDs/digests atuais.
5. Baixar script remoto com HTTPS em arquivo protegido, conferir sintaxe e revisar efeitos colaterais.
6. Executar procedimento aprovado.
7. Comparar configuração e imagens antes/depois, sem reverter atualizações legítimas.
8. Validar Nginx, Typebot, Silo, CRM, TicketZ e backup.
9. Registrar versão efetiva na interface/log oficial. O texto do atualizador `Concluído` **não demonstra por si só a versão instalada**.

**Atenção ao `docker compose down`:** a atualização indisponibiliza temporariamente a stack. `docker system prune -af` pode eliminar imagens antigas necessárias a um rollback rápido, embora volumes nomeados não sejam removidos por esse comando sem flag de volumes.

## 7.6 Comparação após atualização

Usar pré-inventário criado em:

`/root/diagnostico-pre-update-099215/`

```bash
sudo diff -u \
 /root/diagnostico-pre-update-099215/docker-compose.yaml \
 /home/tichatbot/ticketz-docker-acme/docker-compose.yaml

sudo diff -u \
 /root/diagnostico-pre-update-099215/docker-compose.override.yaml \
 /home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml

sudo grep -n 'mc_image=' /opt/backup-dr/bin/backup-minio.sh
sudo systemctl list-timers --all backup-dr.timer
```

Em 22/10 os dois `diff -u` não retornaram diferenças. Comparar também IDs das imagens, que **podem mudar sem que Compose mude**.

## 7.7 Backup de configurações

Exemplo não destrutivo, local e restrito:

```bash
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
sudo install -d -m 700 "/root/config-snapshot-$STAMP"
sudo cp -p \
  /home/tichatbot/ticketz-docker-acme/docker-compose.yaml \
  /home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml \
  "/root/config-snapshot-$STAMP/"
sudo cp -p /opt/backup-dr/bin/backup-minio.sh \
  "/root/config-snapshot-$STAMP/"
```

O backup DR completo já contempla configurações e dados, mas é necessário verificar escopo real e testar restauração em ambiente separado. A cópia acima não inclui `.env` sensível nem substitui o DR.

## 7.8 Restaurar em caso de problema

**Princípio:** restaurar o menor componente possível após identificar a causa.

- **Configuração do Builder/Viewer:** comparar Override com backup; recolocar apenas a variável perdida; recriar somente afetado.
- **Script de backup:** comparar com `.bak-<timestamp>`, restaurar linha correta se houver regressão, `bash -n`, fazer teste completo.
- **Banco/volumes:** restaurar **apenas após plano de DR**, snapshot seguro e teste em ambiente isolado; não sobrescrever produção com dumps só porque um HTTP retornou 502.
- **Rollback de imagem:** se operação/compatibilidade exigir, usar ID ou digest registrado previamente, plano de migrações e validação. Imagem antiga pode ter sido removida pelo prune; não assumir rollback automático disponível.
- **Restaurar configurações com `cp`:** somente com backup datado, aprovação e preservação de alterações legítimas.

## 7.9 Checklist de saúde pós-manutenção

- [ ] Host estável, memória/disco disponíveis.
- [ ] Contêineres persistentes `Up`, sem `Restarting`.
- [ ] `createbuckets` `Exited (0)` quando tarefa concluída.
- [ ] Builder e Viewer com `HOSTNAME=0.0.0.0`.
- [ ] Testes HTTP Docker pelos nomes internos via rede Nginx.
- [ ] TicketZ HTTPS 200 e login real.
- [ ] Builder HTTPS 307 e login/abertura de fluxo.
- [ ] Viewer HTTPS 200 e execução de fluxo publicado.
- [ ] NocoDB HTTPS 200 e consulta real de registro.
- [ ] Gateway/Edge `healthy` e consulta CRM autenticada de teste.
- [ ] Silo API e bucket avaliados (403 raiz pode ser esperado).
- [ ] Fluxo de atendimento WhatsApp/Typebot com tag, nota e handoff de teste.
- [ ] Script de backup aponta para `pgsty/mc:latest`.
- [ ] Timer de backup ativo; próxima execução prevista.
- [ ] Backup posterior à alteração com `BACKUP_OK` e checksums; teste de restauração em plano separado.
- [ ] Logs sem erro relevante, incluindo aplicações externas da stack.
- [ ] Documentado o resultado, hora, versão/imagem e evidências.

---

# 8. Histórico de referências HTTP/HTTPS para comparação

Valores observados **antes e depois** de `v0.99.215-pro` em 22/10/2026:

| Requisição | Antes | Depois | Interpretação |
|---|---:|---:|---|
| `http://ticketz-docker-acme-typebot-builder-1:3000/` pela rede Nginx | 307 | 307 | serviço acessível |
| `http://ticketz-docker-acme-typebot-viewer-1:3000/` pela rede Nginx | 404 | 404 | resposta HTTP interna da rota raiz |
| `https://mchat.ncloud.app.br/` | 200 | 200 | página TicketZ disponível |
| `https://botbuilder.ncloud.app.br/` | 307 | 307 | redirecionamento Builder |
| `https://botviewer.ncloud.app.br/` | 200 | 200 | Viewer disponível |
| `https://nocodb.ncloud.app.br/` | 200 | 200 | NocoDB disponível |
| `https://minio.ncloud.app.br/` | 403 | 403 | endpoint raiz restrito |
| `https://crm-api.ncloud.app.br/` | 404 | 404 | rota raiz sem endpoint público esperado |

**Não confundir mesmo código HTTP com mesma funcionalidade:** uma alteração pode preservar página inicial e quebrar autenticação, consultas ou handoff. Exigir testes de negócio.

---

# 9. Orientações para iniciantes

## 9.1 Conceitos essenciais

- **Container:** processo isolado com uma aplicação. `Up` significa que está ativo; não garante saúde HTTP.
- **Imagem Docker:** pacote da aplicação. `latest` é uma etiqueta móvel, não uma versão imutável.
- **Docker Compose:** arquivo YAML que declara serviços, redes, volumes e variáveis.
- **Override:** arquivo que complementa/substitui configurações do Compose sem editar necessariamente o principal.
- **Volume Docker:** armazenamento persistente; remover volume pode destruir dados.
- **Proxy reverso:** recebe requisição HTTPS e encaminha para serviço interno.
- **HTTP 502:** proxy não recebeu resposta adequada do serviço interno.
- **HTTP 307:** redirecionamento; pode ser normal.
- **HTTP 403:** acesso negado; pode ser normal em endpoint restrito.
- **HTTP 404:** caminho inexistente; pode ser normal na raiz de uma API.
- **Backup lógico PostgreSQL:** dump de banco, mais adequado para restauração consistente do que cópia direta de arquivos ativos.
- **systemd timer:** agenda serviço sem depender de cron tradicional.

## 9.2 O que NÃO fazer sem plano e backup

- Nunca executar `docker compose down -v` ou `docker volume prune` para “limpar” ambiente.
- Não desativar autenticação nem SSRF para contornar falha da integração.
- Não sobrescrever Compose novo por antigo sem analisar diferenças.
- Não modificar chaves criptográficas, tokens ou senhas em múltiplos serviços sem rotação coordenada.
- Não colocar credenciais em docs, tickets, Git ou capturas de tela.
- Não supor que `docker compose restart` carrega um `.env` alterado; recriação costuma ser necessária.
- Não executar atualizador remoto desconhecido sem inspeção.
- Não assumir que `HTTP 200` significa fluxo WhatsApp funcional.
- Não remover MinIO/Silo apenas porque seu bucket está vazio; verificar dependências internas primeiro.

## 9.3 Boas práticas recomendadas

- Trabalho faseado: **diagnosticar → registrar → copiar → modificar mínimo → validar → documentar**.
- Backups de arquivos datados e arquivos secretos com permissão restrita.
- Change log contendo horário, operador, versão, hash, comandos, resultados e motivo.
- Testar em homologação quando possível.
- Manter acesso SSH alternativo antes de alterar DNS, Netplan, Docker daemon ou firewall.
- Evitar testes destrutivos em dados reais.
- Definir teste de restauração periódico e cópia off-site.
- Definir alertas para `backup-dr.service` em erro ou falta de `BACKUP_OK`.

---

# 10. Pendências e pontos de atenção futuros

| Prioridade | Pendência | Motivo |
|---|---|---|
| Alta | Homologar fluxo ponta a ponta pós-`v0.99.215-pro` | HTTP e `Up` não testam CRM, WhatsApp, publicação e handoff |
| Alta | Verificar primeiro backup automático **pós** atualização | último backup inspecionado de 22/10 ocorreu antes do update |
| Alta | Testar restauração real em ambiente isolado | checksums e dumps não garantem recuperação fim a fim |
| Média | Revisar permissão `/backup` (755) e ACLs | segurança dos dados |
| Média | Inventariar versões/digests efetivos pós-update | `latest` pode mudar sem mudança do Compose |
| Média | Auditar permissões sobre Docker socket do backend | alto impacto potencial |
| Média | Revisar autenticação `trust` no PostgreSQL original | risco histórico documentado, exigirá migração planejada |
| Média | Conferir saúde e histórico dos timers de retenção/auditoria | não revalidados nesta conversa |
| Média | Elaborar diagnóstico pós-update automatizado | comparar estado desejado e aplicar correção pontual supervisionada |
| Média | Revisar DNS de outras estações ADM, visitantes e VPNs | validação parcial em 12/10 |
| Futura | Avaliar retirada do Silo | dependências Typebot/backup e novo servidor de mídia |
| Futura | Documentar procedimento de rollback testado | prune do update pode apagar imagens antigas |
| Futura | Auditoria da atualização de licença/versão efetiva TicketZ | saída do script não contém confirmação independente da versão exibida |

---

# 11. Protocolo sugerido para uma futura atualização

**Antes:**

1. Identificar versão atual pela interface e imagem.
2. Validar backup recente, SHA256 e (idealmente) restauração.
3. Capturar `docker ps -a`, redes, imagem IDs/digests, hashes dos dois Compose e dos scripts críticos.
4. Registrar valores `HOSTNAME` Builder/Viewer, `mc_image`, timers e endpoints.
5. Baixar e revisar script remoto atualizado.
6. Preparar janela, responsáveis e plano de reversão.

**Durante:**

1. Executar instalador aprovado, com console/log disponível.
2. Observar migrações, imagens, `down/up`, falhas e containers.
3. Não aplicar modificações paralelas enquanto a stack estiver recriando.

**Depois:**

1. Checar containers e env/Compose; comparar hashes e diffs.
2. Comparar **IDs de imagem**, não apenas etiquetas.
3. Testar HTTP interno pela rede proxy e HTTPS público.
4. Testar login TicketZ/Builder, execução Viewer, CRM NocoDB, Gateway/Edge e handoff controlado.
5. Conferir backup DR e acompanhar primeiro backup automático pós-update.
6. Se algo falhar, coletar evidências antes de corrigir; somente remediar causa conhecida.
7. Registrar conclusão somente após homologação funcional.

---

# 12. Registro final de estado e fontes

## 12.1 Situação confirmada ao final desta conversa

- Instalador oficial para `v0.99.215-pro` executado e informou sucesso.
- Containers principais da stack iniciaram; `createbuckets` encerrou `0`.
- Contêineres adicionais NocoDB, Gateway/Edge, Rocket.Chat e MongoDB permaneceram `Up`/`healthy` quando aplicável.
- Builder/Viewer preservaram `HOSTNAME=0.0.0.0`.
- Códigos HTTP internos e HTTPS comparados com o pré-update foram preservados.
- `docker-compose.yaml` e `docker-compose.override.yaml` não apresentaram diferenças em relação ao inventário de 22/10.
- `backup-minio.sh` permaneceu usando `pgsty/mc:latest`.
- `backup-dr.timer` estava programado para 23/10/2026 05:03:25 UTC conforme saída enviada.
- Logs dos serviços amostrados não continham os padrões de erro pesquisados.
- **Não há evidência enviada da homologação funcional completa nem do backup automático de 23/10.** Não declarar esses itens aprovados sem resultado.

## 12.2 Materiais de referência utilizados

- Histórico de comandos, saídas e decisões desta conversa (12/10 a 22/10/2026).
- `docker-compose.yaml` e `Arquivos env.txt` fornecidos.
- `DOCUMENTACAO_TECNICA_TYPEBOT_NOCODB_TICKETZ.md`.
- `CONFIGURACAO TYPEBOT_TICKETZ.md`.
- `Resumo_Orientativo_Recuperacao_TicketZ_PRO_2026-10-12(1).md`.
- `inventario-tichatbot-2026-10-18_024118.txt`.
- PDFs fornecidos sobre Typebot e triggers TicketZ PRO.
- JSON de exportação de fluxo Typebot anexado (artefato, não necessariamente publicado atualmente).

As instruções dos arquivos históricos podem representar condições de julho/agosto, anteriores às alterações de outubro. Em divergências, **preferir a configuração real observada e validada em produção**, registrando mudanças de forma controlada.

---

**Fim do manual.** Este documento é um inventário técnico-operacional e registro de incidentes, não um atestado de restauração testada, conformidade de segurança ou homologação funcional completa.

---


# APÊNDICE OPERACIONAL A — Inventário aprofundado do CRM, fluxos e customizações

> **Origem e validade:** os dados deste apêndice derivam da documentação de implantação de 04/08/2026 e do export Typebot fornecido em 23/10/2026. Caminhos e valores devem ser entendidos como **última configuração documentada**, não como leitura ao vivo do servidor. Em caso de divergência, prevalece a inspeção da configuração efetiva e o inventário de 22/10/2026. IDs e nomes de filas não devem ser alterados a partir da memória ou de um export isolado.

## A.1 CRM NocoDB: estrutura e finalidades

O CRM foi instalado separadamente da stack TicketZ, em `/opt/nocodb`, para manter seu próprio PostgreSQL e Redis. Essa separação reduz o impacto das mudanças no CRM sobre o banco TicketZ (cuja configuração original inclui `POSTGRES_HOST_AUTH_METHOD=trust`). O NocoDB oferece telas de edição e API para consultar clientes e vendedores; o Gateway limita o que o Typebot pode acessar.

| Objeto | Valor histórico documentado | Uso e onde alterar |
|---|---|---|
| Base CRM | `pjwlscmrg9ksb4p` | Identifica a base na API; atualizar referências em `/opt/crm-query-gateway/.env` e `/opt/nocodb-duplicate-audit/config.env` após recriação de base |
| Tabela Clientes | `m3t1rfxn4cvy1es` | Identifica a tabela; atualizar as mesmas configurações em eventual migração |
| Tabela Vendedores | `Vendedores` | Interface NocoDB; relaciona vendedor e tag TicketZ |
| API interna | `http://nocodb-app:8080` | `NOCODB_URL` no Gateway; acessível somente nas redes autorizadas |
| API externa NocoDB | `https://nocodb.ncloud.app.br` | Proxy, DNS público/interno e opções de `/opt/nocodb/.env` |
| Containers | `nocodb-app`, `nocodb-postgres`, `nocodb-redis` | Stack `/opt/nocodb` |
| Imagem NocoDB observada | `nocodb/nocodb:2026.07.0` | Inventário outubro; atualização precisa revisão de migrações |

### Tabela `Clientes`

Os campos relacionados no documento de 04/08/2026 incluem `Id`, `Nome`, `TipoPessoa`, `Documento`, `TelefoneWhatsApp`, `Email`, `Cidade`, `UF`, `CEP`, `Endereco`, `Numero`, `Complemento`, `Bairro`, `Ativo`, `Observacoes` e o campo legado `Vendedor`.

O relacionamento principal é `VendedorRef` (link para outro registro da tabela `Vendedores`), com `VendedorNomeLookup`, `TicketZTagIDLookup` e `VendedorAtivoLookup`. `FormatoDocumento` e `FormatoWhatsApp` são campos de fórmula de inspeção; **não substituem a validação matemática** de CPF/CNPJ feita no fluxo. Para auditoria de duplicidade são usados `QtdOcorrenciasDocumento` (inteiro) e `StatusDuplicidadeDocumento` (`UNICO`, `DUPLICADO`, `VAZIO`). CPF/CNPJ, telefone e CEP devem ser tratados como **texto**, não números, para preservar zeros iniciais.

**Como mudar um vendedor sem editar código:** acesse `CRM > Clientes`, localize o cliente por ID ou documento, abra `VendedorRef` e escolha o registro da tabela `Vendedores`. Em `CRM > Vendedores`, revise `Nome`, `Ativo` e `TicketZTagID`. Verifique que o ID da tag existe no TicketZ. Depois faça atendimento de teste e confira a tag, anotação e responsável. O campo legado `Vendedor` era mantido como fallback; **não exclua antes de confirmar a inexistência de dependências**.

**Limite da auditoria:** detectar e sinalizar duplicidade não equivale a impedir sua criação. A prevenção exige validação de inserção e/ou restrição apropriada no banco, com estudo de registros existentes e sem alterar diretamente o esquema gerenciado pelo NocoDB.

## A.2 Gateway CRM e Edge HTTPS

O Gateway em `/opt/crm-query-gateway` consulta a API do NocoDB com credencial Viewer, mas não deve publicar a credencial. O Edge em `/opt/crm-query-edge` fornece o ponto HTTPS protegido por uma chave independente (`X-CRM-Gateway-Key`) para o Typebot. Assim, o Typebot não recebe o token permanente do CRM e não é necessário desabilitar a proteção contra SSRF.

| Arquivo | Função |
|---|---|
| `/opt/crm-query-gateway/server.js` | Regras de consulta, normalização, tratamento de duplicidades e mapeamento de vendedor |
| `/opt/crm-query-gateway/docker-compose.yml` | Imagem, redes, mounts, saúde e execução do Gateway |
| `/opt/crm-query-gateway/.env` | `PORT=3000`, `NOCODB_URL`, `NOCODB_BASE_ID`, `NOCODB_TABLE_ID`, `NOCODB_TOKEN` |
| `/opt/crm-query-edge/server.js` | Autenticação, encaminhamento, rate limit, timeout e respostas do Edge |
| `/opt/crm-query-edge/docker-compose.yml` | Porta interna 3001, serviços e redes |
| `/opt/crm-query-edge/secrets/crm_edge_key` | Chave de autenticação para Typebot (não reproduzir) |

O Gateway possui `GET /health`, `POST /v1/clientes/documento` e `POST /v1/clientes/whatsapp`; o endpoint de WhatsApp consulta `TelefoneWhatsApp`. A requisição por documento usa um corpo JSON como `{"documento":"somente_digitos"}`. Entre os resultados tratados estão `ENCONTRADO`, `DUPLICADO` e `ERRO_TECNICO`. Para vendedor podem surgir razões como `VENDEDOR_NAO_INFORMADO`, `VENDEDOR_SEM_RELACIONAMENTO` e `VENDEDOR_INATIVO`. **Não interprete um erro de API como cliente inexistente.**

O Edge opera historicamente como UID/GID `10001:10001`, com segredo em arquivo protegido e acesso apenas via proxy. Segundo a documentação histórica, foram implementados comparação da chave em tempo constante, limite de requisições, limite de tamanho de payload, timeout, `Cache-Control: no-store`, filesystem somente leitura e capacidades reduzidas. A integridade dessa configuração após alterações de imagem deve ser verificada.

**Procedimento para alterar um parâmetro do Gateway:**

```bash
sudo mkdir -p /root/snapshots-manutencao
sudo tar -C /opt -czf "/root/snapshots-manutencao/crm-gateway-$(date -u +%Y%m%d-%H%M%S).tar.gz" crm-query-gateway
sudo nano /opt/crm-query-gateway/.env
sudo chmod 600 /opt/crm-query-gateway/.env
cd /opt/crm-query-gateway
sudo docker compose config --quiet
sudo docker compose up -d --no-deps --force-recreate
sudo docker logs --tail 80 crm-query-gateway
```

O backup acima contém segredo; mantenha o diretório somente para root, evite copiar para mídia insegura e remova snapshots antigos conforme política. Para trocar o token, crie/valide primeiro um token Viewer novo no NocoDB; troque em `.env`; recrie apenas o Gateway; faça testes de leitura e confirme que escrita é negada; só então revogue o token anterior. Para trocar a chave Edge, prepare a atualização **coordenada** do arquivo `/opt/crm-query-edge/secrets/crm_edge_key` e do cabeçalho HTTP configurado no fluxo Typebot; valide autenticação nova e negativa sem expor segredos nos logs. **Não execute `cat` desses arquivos ao coletar evidências.**

## A.3 Retenção dos resultados Typebot

O trabalho de retenção reduz o período de armazenamento de respostas que podem conter CPF/CNPJ. O documento histórico define `/opt/typebot-results-retention/config.env`, `/opt/typebot-results-retention/retention.py`, `/opt/typebot-results-retention/secrets/api_token`, `/etc/systemd/system/typebot-results-retention.service` e `.timer`.

| Chave | Valor histórico | Interpretação |
|---|---|---|
| `TYPEBOT_API_BASE` | `https://botbuilder.ncloud.app.br/api/v1` | API dos resultados |
| `TYPEBOT_ID` | Valor não reproduzido | Identificador do bot a limpar; verificar antes de alterar |
| `RETENTION_HOURS` | `24` | Janela de retenção em horas |
| `DRY_RUN` | `false` | Executa remoção; usar simulação ao validar |
| Timer | `OnCalendar=hourly`, `RandomizedDelaySec=10min` | Rodada aproximadamente horária |

```bash
sudo systemctl status typebot-results-retention.timer --no-pager
sudo systemctl list-timers typebot-results-retention.timer --all --no-pager
sudo journalctl -u typebot-results-retention.service --since "24 hours ago" --no-pager
sudo nano /opt/typebot-results-retention/config.env
# Após revisão da nova configuração, executar em simulação quando compatível:
sudo systemctl start typebot-results-retention.service
```

**Cuidado:** o último comando aciona o serviço e pode excluir dados se `DRY_RUN=false`; nunca use como teste inicial sem confirmar o modo de simulação, backup e prazo. O estado `inactive (dead)` de um serviço `oneshot` entre execuções é normal.

## A.4 Auditoria de documentos duplicados NocoDB

Código: `/opt/nocodb-duplicate-audit/audit.py`. Configuração: `/opt/nocodb-duplicate-audit/config.env`. Execução exclusiva: `/opt/nocodb-duplicate-audit/run-locked.sh`. Segredo: `/opt/nocodb-duplicate-audit/secrets/api_token`. Unidades: `/etc/systemd/system/nocodb-duplicate-audit.service` e `/etc/systemd/system/nocodb-duplicate-audit.timer`.

A rotina consulta registros paginados, normaliza documentos para apenas dígitos, conta ocorrências e atualiza os campos de auditoria somente quando necessário, utilizando API NocoDB. O usuário técnico Linux `nocodb-audit` foi criado sem login interativo; a conta API precisa poder editar especificamente os dois campos de auditoria. O timer histórico executa a cada 30 minutos.

| Configuração | Valor histórico |
|---|---|
| `NOCODB_API_BASE` | `https://nocodb.ncloud.app.br/api/v3` |
| `NOCODB_BASE_ID` | `pjwlscmrg9ksb4p` |
| `NOCODB_TABLE_ID` | `m3t1rfxn4cvy1es` |
| `AUDIT_FIELD_COUNT` | `QtdOcorrenciasDocumento` |
| `AUDIT_FIELD_STATUS` | `StatusDuplicidadeDocumento` |
| `PAGE_SIZE` | `100` |
| `PATCH_BATCH_SIZE` | `100` |
| `DRY_RUN` | `false` |

```bash
sudo systemctl is-active nocodb-duplicate-audit.timer
sudo systemctl list-timers nocodb-duplicate-audit.timer --all --no-pager
sudo journalctl -u nocodb-duplicate-audit.service --since "24 hours ago" --no-pager
sudo nano /opt/nocodb-duplicate-audit/config.env
sudo systemctl cat nocodb-duplicate-audit.service nocodb-duplicate-audit.timer
```

Em caso de mudança de periodicidade, edite **somente** `/etc/systemd/system/nocodb-duplicate-audit.timer`, revise `OnCalendar`, depois `sudo systemctl daemon-reload` e `sudo systemctl restart nocodb-duplicate-audit.timer`. Verifique `list-timers`. Antes de uma primeira execução manual, configure `DRY_RUN=true` e confirme comportamento nos logs (sem CPF/CNPJ completos).

## A.5 Integração TicketZ e protocolo de comandos

Na documentação operacional TicketZ, o driver Typebot da fila precisa da URL base do Viewer e do **Public ID publicado** do fluxo. As variáveis de contexto incluem `number`, `pushName`, `firstMessage`, `ticketId`, `backendURL`, `token` e `metadata`. O token contextual é diferente das credenciais permanentes do CRM.

Os comandos são enviados pelo Typebot em bolhas de texto iniciadas por `#` seguido imediatamente de JSON válido; por exemplo:

```text
#{"action":"addTag","tagId":1}
#{"action":"updateTicket","ticketData":{"status":"pending","annotation":"Triagem finalizada"}}
#{"action":"endSession"}
```

Os campos `queueId` e `userId` designam entidades do TicketZ, não suas posições na interface. Antes de usar `queueId` ou `tagId`, confirmar o mapeamento **no TicketZ atual**. Uma sequência documentada para o handoff comercial anterior era: adicionar tag quando válida; incluir nota mínima (CPF/CNPJ mascarado); definir `pending`; encerrar sessão. O comando `endSession` evita o bot continuar respondendo após a transferência. O modelo inicial mantinha o cliente na fila `Vendas` (ID 4).

IDs de referência **histórica de agosto**: Company ID `2`; filas RH `1`, SAC `2`, Compras `3`, Vendas `4`; integração Typebot `1`; tags de vendedores Carlos `1`, Mariana `2`. No **export de triagem de outubro** aparecem também fila Financeiro `5` e tags Financeiro Pagar `3`/Receber `4`; portanto o conjunto mudou ou foi ampliado. Não converter esses IDs em documentação do estado de produção sem consultar a interface ou o banco.

## A.6 Novo export Typebot: triagem multiárea

**Arquivo de origem:** `typebot-export-atendimento-triagem-homologa-o-dv7w312.json` (anexo; não há evidência de que esteja salvo em algum diretório do servidor). Nome do fluxo: `Atendimento - Triagem - Homologação`; identificador interno do export `cmtnotfyi00001lqirdv7w312`; formato do export `version: 6.1`. São **33 grupos**. Isto comprova a configuração **contida no arquivo exportado**, mas não sua publicação, Public ID em uso ou vínculo ativo a alguma fila.

| Percurso no export | Comportamento identificado no JSON | IDs literais que exigem conferência |
|---|---|---|
| Menu principal | Comprar, Vender, Financeiro, Recursos Humanos, SAC | Valores de escolha textuais |
| Financeiro / Pagar | Adiciona tag, atualiza ticket para `pending` com anotação e encerra sessão | `tagId:3`, `queueId:5` |
| Financeiro / Receber | Handoff equivalente com anotação distinta | `tagId:4`, `queueId:5` |
| Quero vender | Roteia para Compras | `queueId:3` |
| Recursos Humanos | Roteia para RH | `queueId:1` |
| SAC | Roteia para SAC | `queueId:2` |
| Quero comprar / Vendas | Identificação de cliente, CPF/CNPJ, consulta CRM e handoff humano | `queueId:4` |
| Vendas: fluxo CRM | Inicialização, validação de documento, registro ausente/inativo/duplicado, erro técnico | Credencial do Edge deve permanecer protegida |

**Para alterar o fluxo com segurança:** exporte a versão publicada como backup; importe uma **cópia de homologação**, nunca edite produção diretamente; mude os IDs somente após confrontar a tabela de filas/tags do TicketZ; revise as referências `{{variavel}}` e os blocos Set variable que geram `'#' + JSON.stringify(...)`; execute testes por rota (inclusive `endSession`); publique a nova versão; confirme qual Public ID a fila realmente executa. A importação de um JSON não atualiza automaticamente a configuração do TicketZ.

## A.7 Serviço externo e infraestrutura

O inventário registra ainda `rocketchat-rocketchat-1` e `rocketchat-mongodb-1`; são serviços presentes e podem compartilhar recursos de CPU, memória ou disco, mas **não são parte comprovada do fluxo principal TicketZ/Typebot/CRM**. Não os desinstale durante limpeza de containers.

O pfSense é o resolvedor DNS da rede ADM (`192.168.200.1`). O servidor Ubuntu historicamente usa Netplan; a **localização exata do arquivo Netplan ativo deve ser identificada** com `sudo ls -la /etc/netplan/` e `sudo netplan get` antes de editar. O Unbound deve manter os registros privados. Alterações de IP público, DNS interno ou certificados requerem teste de acesso de **dentro e fora** da rede.

# APÊNDICE OPERACIONAL B — Matriz de mudanças e recuperação para iniciantes

| Objetivo | Lugar correto | Antes de editar | Depois de editar | Risco |
|---|---|---|---|---|
| Domínio TicketZ | `/home/tichatbot/ticketz-docker-acme/.env-backend` e `.env-frontend`; proxy e DNS | Salvar Compose e env; conferir domínio de backend | Validar HTTPS, login, API, websocket | Alto |
| Domínio Typebot | `.env-integrations`, `.env-typebot-common`, `.env-typebot-builder`, `.env-typebot-viewer` | Conferir NextAuth, cookies, certificados | Recriar só serviços afetados; testar Builder e Viewer | Alto |
| Ligação Nginx ⇄ Typebot | `/home/tichatbot/ticketz-docker-acme/docker-compose.override.yaml` | `docker compose config --quiet`; cópia | Conferir `HOSTNAME=0.0.0.0` e HTTPS 200/307 apropriado | Médio |
| Caminho das mensagens TicketZ | Fluxo no Builder, fila no TicketZ | Export de backup, tabela de IDs | Testar todas as rotas de handoff | Alto |
| Vendedor/tag | NocoDB `Vendedores`, `Clientes.VendedorRef`, TicketZ tags | Confirmar identificadores | Atendimento de teste e auditoria | Médio |
| Token API CRM | `/opt/crm-query-gateway/.env` | Novo token com permissão mínima | Recriar Gateway, testar acesso positivo/negativo | Alto |
| Chave Edge | `/opt/crm-query-edge/secrets/crm_edge_key`, header Typebot | Planejar troca coordenada | Testar 401 sem chave e acesso com chave | Alto |
| SMTP Typebot | `.env-integrations` | Conta, DNS e app password protegida | Testar envio sem logar segredo | Médio |
| Frequência backup | `/etc/systemd/system/backup-dr.timer` | Último backup válido e janela | `daemon-reload`, `list-timers`, verificar próxima execução | Alto |
| Política backup | `/etc/backup-dr/backup-dr.conf` | Espaço, retenção, offsite | Executar backup e validar manifest | Alto |
| Imagem cliente S3 | `/opt/backup-dr/bin/backup-minio.sh` | Nome/tag disponíveis localmente | `bash -n`, backup de teste, inventário | Médio |
| Imagem TicketZ | Método de atualização e Compose do fornecedor | Backup de banco, rollback e imagens prévias | Saúde técnica e funcional | Alto |
| Modelo de dados NocoDB | Interface CRM; `/opt/nocodb` | Backup PostgreSQL testado | Testar API/Gateway e auditoria | Muito alto |
| Password DB/redis | Arquivos `.env` e configurações de servidor/cliente | Plano de rotação coordenada | Testes internos e serviço após atualização | Muito alto |
| Chave criptográfica Typebot/NocoDB | `.env-secrets`, `.env` | Backup, estudo de impacto e plano de recuperação | Validar decrypt de dados existentes | Crítico |

## B.1 Checagem inicial não destrutiva (usar antes de qualquer manutenção)

```bash
hostnamectl
date -Is
df -hT /
free -h
sudo docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
sudo docker system df
sudo systemctl --no-pager --failed
sudo systemctl list-timers --all --no-pager | grep -E 'backup-dr|nocodb-duplicate-audit|typebot-results-retention'
sudo ls -ld /backup /backup/completed /opt/backup-dr /home/tichatbot/ticketz-docker-acme
cd /home/tichatbot/ticketz-docker-acme && sudo docker compose config --quiet
```

**Resultado esperado:** sem erros de configuração, containers críticos `Up`, filesystem sem saturação, timers esperados visíveis, diretórios presentes. Ausência de um timer não deve ser "corrigida" recriando unidades por suposição; investigue `systemctl cat` e o inventário primeiro.

## B.2 Backup e restauração: distinção essencial

O marcador `BACKUP_OK` representa o término dos controles implementados pelo script; **não equivale a recuperação integral comprovada**. O conjunto mínimo de evidências inclui ID da execução, arquivos gerados dentro da pasta correspondente, tamanho, manifest, checksums, `pg_restore --list` nos dumps custom, validação dos Redis, inventário S3, cópia externa e teste isolado de restore. Um dump íntegro pode continuar insuficiente se não houver roles, secrets, Docker volumes, certificados e arquivos de configuração corretos.

Procedimento **não destrutivo** para conferir último backup:

```bash
sudo find /backup/completed -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort | tail -10
sudo find /backup/completed -type f -name BACKUP_OK -printf '%h\n' | sort | tail -5
sudo systemctl status backup-dr.timer --no-pager
sudo journalctl -u backup-dr.service --since '48 hours ago' --no-pager
```

Em incidente, **não sobrescreva um banco de produção** com arquivo de backup por tentativa. Restauração segura requer: identificar o ponto de recuperação, conferir integridade e cópia externa, preparar ambiente isolado, recuperar volumes/segredos/roles, restaurar bancos, testar login, leitura CRM e ciclo TicketZ ⇄ Typebot, documentar tempo de retorno e só então planejar corte. O relatório de setembro descrevia defeito estrutural grave nos artefatos de backup; a evidência de outubro mostra reparos e conclusão, mas **não substitui teste integral de DR**.

## B.3 Atualizações: evitar a repetição dos incidentes de outubro

1. Registrar estado inicial (`docker ps`, imagens e digests, redes, Compose efetivo com **segredos mascarados**, status dos timers).
2. Garantir backup recente e método de retorno anterior à atualização.
3. Avaliar mudanças entre versões, especialmente imagem `latest`, schema Prisma, rotas Nginx e imagens S3.
4. Atualizar no procedimento suportado pelo fornecedor, sem remover volumes e sem disparar Watchtower desnecessariamente.
5. Confirmar que o override com `HOSTNAME=0.0.0.0` permanece no Builder e Viewer.
6. Verificar saúde técnica (`docker ps`, logs, TCP/HTTP pelo proxy e navegador) **e funcional** (WhatsApp, CRM, tags, filas, pending, endSession, notificação de backup).
7. Preservar evidence de backup, datas, ID dos containers e um registro de qualquer alteração.
8. Se ocorrer falha, isolar rede, DNS, aplicação, banco e proxy antes de reverter. Rollback de imagem **não** garante compatibilidade com migrações do banco.

## B.4 Coleta de evidências para suporte sem expor dados pessoais

Envie ao responsável técnico: horário com timezone, serviço e URL afetados, resultado do comando, status HTTP, nomes dos containers, versão/digest, trecho do log com credenciais removidas e identificador do backup. **Nunca compartilhe** arquivo `.env` integral, cookies, tokens bearer, chave Edge, dumps de produção, senhas, CPF/CNPJ reais ou `docker inspect` sem higienização: essas saídas podem conter segredos.

## B.5 Checklist de aceite de produção

- [ ] `docker ps` sem reinicializações inesperadas nos serviços críticos.
- [ ] `docker compose config --quiet` passa em todas as stacks.
- [ ] Certificados HTTPS válidos e resolução DNS interna/externa correta.
- [ ] TicketZ login, recepção e envio WhatsApp operando.
- [ ] Builder edita e publica; Viewer executa o **Public ID usado na fila**.
- [ ] Todas as rotas do fluxo de triagem efetivamente publicado testadas.
- [ ] Endpoint CRM Edge valida chave (200 autorizado, 401 não autorizado).
- [ ] Consulta de CPF/CNPJ cobre ativo, inativo, não encontrado, duplicado e erro técnico.
- [ ] Tags, notas, fila, `pending` e `endSession` confirmados em tickets reais de homologação.
- [ ] Auditoria de duplicados executa; cronologia e `DRY_RUN` revisados.
- [ ] Retenção Typebot executa sem remover resultados fora da política.
- [ ] Backup agendado mais recente passa checksums, manifest e verificações de artefatos.
- [ ] Cópia externa protegida e teste de restauração isolada documentados.
- [ ] Segredos com permissões mínimas; PostgreSQL `trust` e `/backup` modo 755 revisados.
- [ ] Evidência de validação de estações internas, VPNs e DNS está registrada.

# APÊNDICE C — Fontes, divergências e critérios de atualização documental

| Fonte | Data/ênfase | Como foi usada |
|---|---|---|
| `Manual_Tecnico_TicketZ_PRO_Typebot_NocoDB_2026-10-23(1).md` | Consolidado 23/10 | Documento-base: estado e incidentes mais recentes |
| `inventario-tichatbot-2026-10-22_050454(2).txt` | Coleta 22/10 | Sistema operacional, containers, redes e evidências contemporâneas |
| `Documento Gemini(1).txt` | DR e scripts | Configuração não secreta, unidades e runbook DR como referência |
| `relatorio-tecnico-backup-dr-tichatbot(1).md` | Setembro | Causa histórica e cuidados no DR; conclusões de setembro não substituem recuperação de outubro |
| `Resumo_Orientativo_Recuperacao_TicketZ_PRO_2026-10-12(2).md` | 12/10 | Recuperação de pfSense/DNS, Viewer e notificações |
| `CONFIGURACAO TYPEBOT_TICKETZ.md` | 04/08 | Customizações CRM, gateway, handoff, auditoria, retenção e IDs históricos |
| `DOCUMENTACAO_TECNICA_TYPEBOT_NOCODB_TICKETZ.md` | 28/07 | Primeira implantação e modelo de integração |
| `docker-compose.yaml` | Anexo de stack | Exemplo de estrutura; contém `minio/minio`, mas o runtime de outubro usa `pgsty/silo`; **não aplicar cegamente** |
| `Arquivos env.txt` | Anexo de variáveis | Nomes e relações entre envs; valores podem ser históricos e conter segredos |
| `Triggers e Comandos _ Ticketz PRO.pdf` | Guia fornecedor | Sintaxe e semântica de triggers |
| `Integração com Typebot _ Ticketz PRO.pdf` | Guia fornecedor | Viewer, Public ID e variáveis padrão |
| `typebot-export-atendimento-triagem-homologa-o-dv7w312.json` | Export | Estrutura de triagem e IDs codificados; publicação não confirmada |
| `Dados Servidor tichatbot0.txt` | Inventário técnico anterior | Referência histórica para inspeção, não padrão de estado atual |

**Discrepância relevante:** `docker-compose.yaml` anexo define `minio/minio` e `minio/mc` em serviços `minio`/`createbuckets`, ao passo que o inventário recente identifica `pgsty/silo` como backend S3 e a correção do backup usa `pgsty/mc`. A implementação efetiva deve ser determinada pelo Compose resolvido *no host*, incluindo `docker-compose.override.yaml`, e por `docker inspect` com mascaramento de segredos. **Jamais reverter para o YAML anexo sozinho**.

**Outra discrepância relevante:** a configuração de agosto usava uma sequência de handoff para Vendas sem troca de fila; o JSON de triagem entregue agora contém comandos de troca explícita para múltiplas filas. O JSON documenta o design exportado, não comprova que o novo design foi publicado ou que está em uso pelos contatos.

**Atualize o manual somente após nova evidência:** toda mudança deve registrar data, motivo, arquivo completo, valor anterior (exceto segredo), valor novo (exceto segredo), comando realizado, verificação, possível rollback e responsável.
