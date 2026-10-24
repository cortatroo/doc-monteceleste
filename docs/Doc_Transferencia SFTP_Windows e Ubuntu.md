# Documentação Técnica — Transferência de Backups Ubuntu → Windows via SFTP
**Edição revisada e auditada | referência: 24/10/2026**

**Sistema:** tichatbot / TicketZ PRO, Typebot, NocoDB e CRM  
**Escopo:** geração local dos backups, disponibilização via SFTP, espelhamento no Windows, validação e interface com a sincronização em nuvem.  
**Classificação das afirmações:** **V** = verificado por comandos ou logs; **O** = confirmado operacionalmente pelo responsável; **P** = pendente/não inspecionado.

## 1. Objetivo, escopo e delimitação
Este documento substitui a documentação anterior de exportação e sincronização dos backups que ainda apresentava perguntas não respondidas. Descreve o ambiente observado até 24/10/2026, e não uma arquitetura hipotética. Os backups são produzidos no Ubuntu e transferidos por iniciativa do Windows. A cópia posterior para a nuvem, realizada no ambiente Windows, foi confirmada visualmente pelo responsável, mas não submetida a uma validação remota independente de integridade ou retenção (**O/P**).

O Rocket.Chat e seu MongoDB foram excluídos intencionalmente do escopo do Backup DR por decisão do responsável, e esta exclusão deve ser preservada em qualquer interpretação dos resultados. O backup local e a transferência Windows não representam cobertura desses dados.

## 2. Inventário efetivo e fluxo da solução
| Elemento | Configuração observada | Evidência |
|---|---|---|
| Origem | Ubuntu 24.04 LTS, host `tichatbot` | V |
| IP privado Ubuntu | `192.168.200.127` | V |
| Serviço | OpenSSH SFTP em TCP 22 | V |
| Conta de serviço Linux | `cobian-reader`, UID/GID `1001:1001` | V |
| Chroot | `/backup` | V |
| Origem no sistema de arquivos | `/backup/completed/` | V |
| Origem vista no SFTP | `/completed/` | V |
| Windows / scripts | `C:\Backups_MonteCeleste` | V |
| Destino da cópia Windows | `F:\BKPSD\APIMeta` | V |
| Tarefa agendada | `Copia_backup`, todos os dias às 03h | V |
| Cliente | WinSCP; execução da tarefa via `puxar_backup.bat` | V |
| Modo de cópia | `synchronize local -delete` | V/O |
| Verificação de host | fingerprint SSH ED25519 fixa | V |
| Credencial de usuário utilizada | Senha pré-inserida na URL SFTP | V |
| Replicação à nuvem | Nextcloud em execução; upload visto no destino pelo operador | V/O |

**Fluxo:** `backup-dr.timer` no Ubuntu → `/backup/completed/<ID>` → usuário SFTP confinado no `/backup` → WinSCP iniciado pela tarefa Windows `Copia_backup` → `F:\BKPSD\APIMeta\<ID>` → sincronização Nextcloud para nuvem. A última seta foi validada visualmente pelo responsável, e não mediante comparação remota dos hashes.

## 3. Criação e retenção na origem Ubuntu
- **Serviço:** `/etc/systemd/system/backup-dr.service`, tipo `oneshot`, executa `/opt/backup-dr/bin/backup-run.sh` como root, com limite de duas horas (**V**).
- **Timer:** `/etc/systemd/system/backup-dr.timer`, `OnCalendar=*-*-* 05:00:00` UTC, `RandomizedDelaySec=5min`, `Persistent=true` (**V**). Correspondência operacional: aproximadamente **02h a 02h05 no horário de Brasília (UTC−3)**.
- **Retenção:** `KEEP_VALID_BACKUPS="7"` e `KEEP_FAILED_DAYS="30"` (**V**). Trata-se de sete backups válidos, **não necessariamente sete dias**: execuções adicionais podem consumir posições na retenção.
- **Espaço mínimo:** `MIN_FREE_GIB="30"`; fuso configurado para `America/Sao_Paulo` (**V**).
- **Criptografia dos pacotes:** `ENCRYPTION_ENABLED="false"` (**V**). Os pacotes incluem configurações e dados; a confidencialidade em repouso precisa ser tratada na política de segurança.
- **Abrangência:** dumps PostgreSQL customizados TicketZ, Typebot e NocoDB; RDB e AOF Redis TicketZ e NocoDB; inventário S3/Silo; tarballs de configurações TicketZ e CRM; volumes persistentes específicos, incluindo certificados Nginx e conteúdo público/privado do backend (**V**).
- **S3:** inventários da área `minio/typebot` foram gerados; o bucket Typebot encontrava-se vazio na auditoria (**V**). Isso não equivale a um backup de objetos inexistentes.
- **Exclusão por decisão:** Rocket.Chat/MongoDB não fazem parte desta rotina (**O/V**).
- **Recuperação:** procedimento de restauração do Backup DR já havia sido homologado em **VM separada**, conforme documentação histórica; não foi reexecutado no servidor de produção durante esta auditoria.

O backup de referência `/backup/completed/2026-10-23_02h04m28s` continha `BACKUP_OK`, `NOTIFICATION_SENT`, `SHA256SUMS`, manifest e dumps. SHA-256 aprovado na origem. A rotina local completou com `Result=success`, `ExecMainStatus=0`.

## 4. Conta de serviço SFTP — identificação e restrições
A configuração **atual**, em oposição ao antigo exemplo `Match User cobian`, é:

**Arquivo:** `/etc/ssh/sshd_config.d/cobian-sftp.conf` (**V**)

```text
Match User cobian-reader
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /backup
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

**Evidências adicionais do servidor:**
- `id cobian-reader`: `uid=1001(cobian-reader) gid=1001(cobian-reader) groups=1001(cobian-reader)`.
- Home: `/home/cobian-reader`. Shell: `/usr/sbin/nologin`.
- Configuração efetiva `sshd -T -C user=cobian-reader,host=localhost,addr=192.168.200.127`: `forcecommand internal-sftp`, `chrootdirectory /backup`, `passwordauthentication yes`, `pubkeyauthentication yes`, `allowtcpforwarding no`, `x11forwarding no`, `permittty yes`, `permituserrc yes`.
- O servidor **aceita** chaves públicas (`pubkeyauthentication yes`) em geral; isso **não comprova** que esta transferência as utiliza. `permittty yes` não transforma esta conta em um login shell funcional, pois há `ForceCommand internal-sftp` e `/usr/sbin/nologin`.
- `sshd -t` retornou `CONFIGURACAO SSH: VALIDA`; `systemctl is-active ssh`: `active`.
- O arquivo de configuração está com permissões `644 root:root`. O chroot `/backup` e a pasta `/backup/completed` estão com permissões `755 root:root`.

**Não alterar nem reiniciar o SSH apenas para documentar este estado.**

## 5. Controle de acesso aos arquivos exportados
Os diretórios de cada backup são `root:root`, normalmente modo `750`, e os arquivos são restritos. O acesso de leitura SFTP é concedido via ACL nomeada para `cobian-reader`; o usuário não precisa de acesso administrativo ao Ubuntu.

**Falha real diagnosticada em 23/10/2026:** os arquivos marcadores `BACKUP_OK` e `NOTIFICATION_SENT` eram criados **após** a ACL recursiva do diretório staging e recebiam `chmod 0600` sem ACL. A transferência agendada WinSCP falhou em `NOTIFICATION_SENT`, com `Permission denied` e código `1`. O backup de 23/10 não chegou ao Windows na execução das 03h daquele dia.

**Correção aplicada:**
1. ACL de leitura `u:cobian-reader:r--` nos marcadores dos sete backups então existentes.
2. Alteração permanente de `/opt/backup-dr/bin/backup-run.sh`: acrescentado `setfacl -m "u:cobian-reader:r--"` depois de `chmod 0600` para `BACKUP_OK`, `NOTIFICATION_SENT` e `NOTIFICATION_FAILED`.
3. Cópia de segurança do script: `/opt/backup-dr/bin/backup-run.sh.before-sftp-acl-20261023-171811`.
4. `bash -n` aprovado; leitura como `cobian-reader` de `BACKUP_OK`, `NOTIFICATION_SENT`, `SHA256SUMS` aprovada.

**Ressalva:** a criação futura de novos marcadores com ACL corrigida foi verificada estaticamente no script; não se apresentou nesta auditoria o `getfacl` dos marcadores de um backup posterior. A tarefa Windows executada em 24/10 terminou com sucesso.

## 6. Arquivos da automação Windows
Diretório operacional: `C:\Backups_MonteCeleste`.

**`puxar_backup.bat`** — estrutura apresentada e ação agendada confirmada:
```bat
@echo off
"C:\Program Files (x86)\WinSCP\WinSCP.exe" /script="C:\Backups_MonteCeleste\sincronizar_backup.txt" /log="C:\Backups_MonteCeleste\log_transferencia.txt"
```

**`sincronizar_backup.txt`** — representação documental **SEM senha**, baseada na estrutura declarada e nos parâmetros efetivamente testados:
```text
option batch abort
option confirm off
open sftp://cobian-reader:<SENHA_OMITIDA>@192.168.200.127/ -hostkey="ssh-ed25519 255 A6faNYNxO0UltDjt71SZ1AvimSDFqraWglT6DReF5sQ"
synchronize local -delete "F:\BKPSD\APIMeta" "/completed"
exit
```
**Atenção:** `<SENHA_OMITIDA>` é apenas um marcador documental; não execute o trecho como script nem inclua a senha no documento. Na configuração operacional, a senha está incorporada à URL SFTP. `option batch abort` interrompe o lote perante erros; `option confirm off` evita prompts; `-delete` remove no espelho Windows arquivos ausentes na origem.

**Chave de host (identidade do servidor, não login do usuário):**
- SHA-256 obtido no Ubuntu: `SHA256:A6faNYNxO0UltDjt71SZ1AvimSDFqraWglT6DReF5sQ`.
- Formato aceito pelo WinSCP e comprovado por conexão `ExitCode=0`: `ssh-ed25519 255 A6faNYNxO0UltDjt71SZ1AvimSDFqraWglT6DReF5sQ` (sem o prefixo literal `SHA256:` nesse campo).
- A aceitação universal `-hostkey="*"` foi removida em 23/10/2026.
- Cópia de segurança criada: `C:\Backups_MonteCeleste\sincronizar_backup.txt.before-hostkey-20261023-144958`; houve outra cópia antes do ajuste do formato.

## 7. Autenticação: resposta definitiva à dúvida do documento antigo
**Método observado nesta conexão:** **autenticação por senha**. O WinSCP registrou `Autenticando com a senha pré-inserida` e a URL operacional contém usuário e senha. O teste no Windows apontou:
- `Usuario cobian-reader: True`;
- `Chave privada configurada no comando: False`;
- `Chave do servidor verificada: True`;
- `Aceitacao irrestrita desativada: True`.

**Não foi identificada chave privada especificada no comando `open`.** Permanece tecnicamente possível haver configuração externa de agente/sessão, que não foi analisada integralmente; não há evidência de que um mecanismo adicional tenha sido usado na conexão auditada. A indicação `pubkeyauthentication yes` do `sshd` é disponibilidade do mecanismo, e não confirmação de seu uso.

A **chave ED25519** mencionada na opção `-hostkey` é a impressão digital da **chave pública do servidor Ubuntu**, usada para impedir conexões a servidores impostores. Não é uma chave privada de autenticação do usuário `cobian-reader`.

**Ponto de segurança:** a senha incorporada à URL do script exige controle estrito de ACL NTFS e proteção das cópias de segurança do `.txt`, inclusive as versões `before-hostkey*`. Esses controles **não foram auditados** nesta sequência. Evitar compartilhar linha `open`, logs de debug sem filtragem ou cópias do script com a senha.

## 8. Agendamento Windows — parâmetros efetivamente inspecionados
**Tarefa:** `Copia_backup`; ação executa `C:\Backups_MonteCeleste\puxar_backup.bat`; `WorkingDirectory` vazio.

| Campo | Estado confirmado |
|---|---|
| Conta Windows | `Administrador` |
| `Principal.LogonType` | `Interactive` |
| Nível | `Highest` |
| Início do agendamento | `2026-10-14T03:00:00` |
| Habilitado | `True` |
| Frequência | `DaysInterval=1`, diariamente às 03h |
| `StartWhenAvailable` | `False` |
| `DisallowStartIfOnBatteries` | `True` |
| `AllowStartOnDemand` | Resposta vazia no comando utilizado: **não inferir valor** |
| Última execução inspecionada | 24/10/2026, 03h00 |
| Resultado da última execução | `LastTaskResult=0` (sucesso do processo da tarefa) |
| Próxima execução | 25/10/2026, 03h00 |
| Estado no momento da consulta | `Ready` |

**Dependência operacional:** com `Interactive`, a execução automática depende de sessão apropriada da conta `Administrador`; a tarefa não está configurada para recuperação automática de horário perdido via `StartWhenAvailable`. O resultado `0` da tarefa não substitui uma comparação SHA-256 para cada execução.

## 9. Transferência, logs e integridade — evidências históricas
**Falha anterior:** execução automática de 23/10 às 03h terminou `1`, com erro SFTP `Permission denied` no marcador `NOTIFICATION_SENT`. Após correções de ACL e verificação da chave de host, uma execução controlada do WinSCP foi concluída com `ExitCode=0`.

**Backup auditado:** `2026-10-23_02h04m28s`.
- Diretório existente no Windows: `F:\BKPSD\APIMeta\2026-10-23_02h04m28s`.
- **41 arquivos**, total **92.534.858 bytes**.
- Presença confirmada de `BACKUP_OK`, `NOTIFICATION_SENT`, `SHA256SUMS` e dumps TicketZ/Typebot/NocoDB.
- Verificação local Windows de **38 entradas SHA-256** do manifesto: **38 válidas, 0 ausentes e 0 divergentes**. Os outros arquivos no diretório não devem ser apresentados como individualmente checados por hash se não constarem do manifesto.
- Última tarefa diária consultada (24/10, 03h): `LastTaskResult=0`.
- Log da tarefa: `C:\Backups_MonteCeleste\log_transferencia.txt`; log da transferência manual: `C:\Backups_MonteCeleste\log_validacao_backup.txt`.

**Não confundir:** os 38 hashes validam a cópia do backup **de 23/10** no Windows, não todos os futuros backups nem o conteúdo armazenado em nuvem.

## 10. Política `-delete`, risco e decisão expressa
A opção `-delete` foi **mantida intencionalmente** por decisão do responsável para espelhar a retenção da origem Ubuntu. Isso é consistente com um **espelho**, não com um arquivo histórico independente. Se arquivos forem removidos na origem por retenção ou acidentalmente, a sincronização pode removê-los também no Windows.

A nuvem foi confirmada visualmente pelo responsável como funcional (**O**). Não foram auditados versionamento, lixeira, retenção independente ou hashes dos objetos remotos (**P**). Esta ausência de auditoria foi aceita no escopo da conversa; não deve ser convertida em alegação de integridade independente da cópia em nuvem.

## 11. Segurança operacional e melhorias não aplicadas
- **Concluído:** identidade SSH do Ubuntu vinculada a fingerprint ED25519; Chroot SFTP; conta dedicada; porta forward, agent forward, tunnel e X11 desabilitados; ACLs de leitura restritas dos marcadores; testes WinSCP e hashes (**V**).
- **Mantido:** acesso de `cobian-reader` por senha; login de tarefa `Interactive`; `-delete`; estrutura geral WinSCP (decisões do responsável).
- **Não verificado:** ACL NTFS e confidencialidade dos scripts com senha e de cópias `.before-*`; existência de chaves SSH de usuário fora da linha `open`; retenção efetiva/versionamento da nuvem; hash remoto; comportamento após logoff do usuário Windows.
- **Atenção à retenção:** o espelhamento pode replicar exclusões legítimas ou indevidas. Qualquer ampliação da retenção independente deve ser tratada como projeto à parte, sem alterar silenciosamente a automação.

## 12. Diagnóstico e operação
| Sintoma | Verificação |
|---|---|
| Login SFTP falha | Log WinSCP, usuário `cobian-reader`, senha operacional, host key fixa e SSH ativo |
| `Permission denied` em `BACKUP_OK`/`NOTIFICATION_*` | ACL do arquivo e execução dos três `setfacl` no `backup-run.sh` |
| `Host key does not match` | Fingerprint legitimamente obtida no Ubuntu e formato aceito pelo WinSCP; nunca recorrer automaticamente a `*` |
| Pasta `/completed` ausente | `ChrootDirectory /backup` e existência de `/backup/completed` |
| Transferência falha ou não acontece | `Copia_backup`, último resultado, sessão Windows `Administrador`, caminho WinSCP, unidade `F:` |
| Conteúdo não coincide | SHA256SUMS do backup na origem e hashes calculados no Windows |
| Arquivos antigos desaparecem do Windows | Verificar `-delete` e a política `KEEP_VALID_BACKUPS=7` |
| Backup não aparece na nuvem | Consultar operacionalmente o cliente Nextcloud e o destino remoto; verificação detalhada fora deste escopo |

**Consultas seguras:** no Ubuntu, `sudo sshd -t`, `sudo sshd -T -C user=cobian-reader,host=localhost,addr=192.168.200.127`, `sudo getfacl /backup/completed/<ID>/NOTIFICATION_SENT`, `systemctl status backup-dr.timer`; no Windows, `Get-ScheduledTask -TaskName Copia_backup`, `Get-ScheduledTaskInfo -TaskName Copia_backup`. Não publicar a linha completa `open sftp://...` nem executar comandos que alterem o conteúdo dos backups sem plano de mudança.

## 13. Checklist encerrado com as evidências da auditoria
| Controle | Situação |
|---|---|
| Identidade do usuário `cobian-reader` | **V** |
| Match User, Chroot e ForceCommand atuais | **V** |
| SSH ativo e configuração válida | **V** |
| Host key ED25519 exata no WinSCP | **V** |
| Autenticação por senha observada | **V** |
| Chave privada explícita no script | **Não presente (V)** |
| Método de chave externa/agente | **P** |
| `.bat` como ação da tarefa | **V** |
| `synchronize local -delete` | **V/O** |
| Conta `Administrador`, modo Interactive | **V** |
| Agendamento diário às 03h | **V** |
| Último resultado automático 24/10 = 0 | **V** |
| Backup 23/10 recebido | **V** |
| Integridade do backup 23/10 no Windows | **V** |
| Upload da nuvem | **O** |
| Hash/retencão remota na nuvem | **P** |
| ACL NTFS de scripts com senha | **P** |

## 14. Histórico de alterações e rastreabilidade
**23/10/2026:** detecção do erro SFTP `Permission denied`; concessão de ACL nos marcadores existentes; correção permanente do `backup-run.sh`; transferência manual controlada e verificação de integridade da cópia; validação e configuração de host key ED25519.  
**24/10/2026:** execução automática `Copia_backup` com resultado `0`; inspeção de conta Windows (`Administrador`, `Interactive`, `Highest`), agendamento, condições, conta e isolamento SSH efetivos; `sshd -t` aprovado.  
**Edição desta documentação:** reconciliação entre as instruções históricas e as evidências efetivas; nenhuma alteração feita nos servidores durante a redação.

**Fontes:** logs e saídas de comandos enviados pelo responsável durante as etapas 6.7–6.9 e verificação complementar de 24/10/2026; arquivo histórico “Documentação Técnica — Exportação e Sincronização de Backups via SFTP”; documentação de Backup DR anteriormente homologada.

**Nota de atualização:** este documento retrata o estado comprovado no período indicado. Novas alterações em usuários, senhas, agendamento, chave do host ou retenção exigem nova validação e revisão.
