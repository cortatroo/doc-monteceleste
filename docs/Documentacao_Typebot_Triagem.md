# Documentação técnica e operacional — Typebot Atendimento / Triagem / Homologação

**Documento de referência para manutenção e futura ativação da consulta CRM.**  
**Base documental:** export JSON fornecido do Typebot, versão de exportação `6.1`.  
**Nome no export:** `Atendimento - Triagem - Homologação`.  
**ID interno do fluxo:** `cmtnotfyi00001lqirdv7w312`  
**ID público:** `atendimento-triagem-homologa-o-dv7w312`  
**Criado:** 2026-09-05 01:11:44 UTC; **última alteração no export:** 2026-10-13 14:53:29 UTC.  
**Data desta documentação:** 2026-10-24.  

> **Leitura essencial:** este documento descreve fielmente a lógica *exportada*, e não certifica a configuração atual dos contêineres/servidores ou a execução em produção. A seção de identificação de clientes e CRM não está ligada ao menu inicial **por decisão intencional**: ela foi preparada para quando o NocoDB estiver povoado. O export contém uma chave de gateway embutida, omitida propositalmente desta documentação; trate o JSON original como confidencial e providencie a rotação da chave caso tenha circulado fora do ambiente autorizado.

## 1. Visão Geral do Projeto

### 1.1 Objetivo do servidor e da automação

O propósito é receber o contato, perguntar o motivo do atendimento e transferir a conversa para a equipe certa no **TicketZ**, registrando o contexto. Na versão preparada para o futuro, a área Comercial poderá consultar um cadastro do **NocoDB** por meio de um **gateway de CRM**, verificar o cliente, reconhecer o vendedor responsável e, quando os dados permitirem, aplicar uma etiqueta no TicketZ.

### 1.2 Tecnologias e serviços envolvidos

| Componente | Função | Grau de evidência |
|---|---|---|
| Typebot | Menus, perguntas, decisões, variáveis e webhook HTTP. | Confirmado no export. |
| TicketZ | Recebe comandos textuais especiais para adicionar etiquetas, criar notas, alterar a fila/status e terminar sessão. | Formato dos comandos confirmado; processamento real precisa ser testado no ambiente. |
| Gateway CRM HTTP | Endpoint intermediário para receber documento e devolver informações normalizadas de clientes. | URL, corpo, header e mapeamentos confirmados. |
| NocoDB | Base de dados do CRM indicada como fonte prevista por trás do gateway. | Informado como objetivo; estrutura real das tabelas e configuração interna **não estão no JSON**. |
| Docker, proxy reverso, banco e rede | Possível infraestrutura para hospedar os serviços. | Detalhes de Compose, hosts, volumes e portas **não constam neste arquivo**. |

### 1.3 Arquitetura lógica

```text
Pessoa / canal de atendimento
           |
           v
      Typebot (menu)
           |
           +--> Comprar --> TicketZ / Comercial (fila 4)   [ATIVO]
           +--> Vender  --> TicketZ / fila 3                [ATIVO]
           +--> Financeiro --> Pagar/Receber --> fila 5     [ATIVO]
           +--> RH --> TicketZ / fila 1                     [ATIVO]
           +--> SAC --> TicketZ / fila 2                    [ATIVO]

   Área preparada, NÃO ligada ao menu:
  identificar cliente? --> CPF/CNPJ --> validar localmente
           |
           v
 Typebot --HTTPS + chave--> gateway CRM --consulta--> NocoDB
           |                                 |
           +<-- status, cliente, vendedor, tagId ---------+
           |
           +--> decisões de cadastro --> nota/tag/fila 4 no TicketZ
```

**O que significa “handoff”:** o robô deixa a conversa pronta para uma pessoa, normalmente mudando o setor/fila, marcando o chamado como pendente e finalizando a sessão automática.

**Importante sobre o TicketZ:** os comandos são enviados como texto de formato `#` seguido por um objeto JSON serializado, por exemplo `#{"action":"endSession"}`. Isso pressupõe que a integração do canal interprete essas mensagens como instruções e não as entregue literalmente ao cliente. Validar essa premissa após qualquer atualização.

## 2. Resumo das etapas implementadas e sua ordem lógica

> A ordem abaixo é **funcional**, deduzida das dependências do export. O JSON não contém um histórico de alterações que permita afirmar em que dia cada etapa foi implantada.

1. Criado o fluxo do Typebot e a mensagem de entrada com cinco opções.
2. Configurados direcionamentos simples para Vender, RH e SAC.
3. Criada subdivisão de Financeiro em Pagar e Receber, com etiquetas distintas e mesma fila.
4. Configurado o atalho **Quero Comprar → Comercial**, sem exigência de documento.
5. Construído, em separado, o futuro ramo comercial: confirmação de vínculo com a empresa; entrada de CPF/CNPJ; normalização e validação algorítmica.
6. Adicionados contadores e limites de duas tentativas para documento inválido e cadastro ausente.
7. Preparado o webhook HTTPS para o gateway, com autenticação em cabeçalho e preenchimento de variáveis da resposta.
8. Criadas decisões para cliente encontrado/ausente/duplicado, cadastro ativo/inativo, falha e vendedor mapeado.
9. Preparada a emissão de notas internas, etiquetas condicionais e handoff humano ao Comercial.
10. Ajustadas mensagens de erro do menu e tempo entre balões.

## 3. Estados e rotas de produção versus homologação

| Rota | Acessível a partir do início? | Saída |
|---|---|---|
| Menu → Quero Comprar | **Sim** | Grupo `07 - Handoff Quero Comprar`, fila Comercial 4. |
| Menu → Quero Vender | **Sim** | Fila 3. |
| Menu → Financeiro → Pagar | **Sim** | Tag 3, fila 5. |
| Menu → Financeiro → Receber | **Sim** | Tag 4, fila 5. |
| Menu → Recursos Humanos | **Sim** | Fila 1. |
| Menu → SAC | **Sim** | Fila 2. |
| `07 - Quero Comprar: identificação de cliente` e grupos `08`, `09`, `10`, `11`, `20` | **Não**, no grafo exportado | Lógica já montada para ativação após povoamento/validação do CRM. |

**Como a desconexão foi verificada:** o item `Quero Comprar` no grupo `00 - Menu principal` aponta expressamente para o grupo `07 - Handoff Quero Comprar`, *não* para `07 - Quero Comprar: identificação de cliente`. A existência de dois grupos com prefixo `07` é apenas numeração editorial; seus IDs internos são diferentes.

## 4. Documentação detalhada de cada configuração e grupo

Em todos os itens a seguir, **“onde” significa o editor visual do Typebot** (`fluxo > grupo > bloco`), pois não existe caminho de arquivo do sistema operacional dentro de um export JSON que defina a instalação. O arquivo-fonte local anexado à documentação é o JSON exportado com nome `typebot-export-atendimento-triagem-homologa-o-dv7w312(1).json`; ele não demonstra o diretório no servidor em que o Typebot está instalado.

### 4.1. 00 - Menu principal

**O que faz e por quê:** Ponto de entrada. Mostra a saudação e oferece as cinco opções. Cada botão grava `destino_atendimento` e segue por uma conexão própria. A opção de compra **não** aciona o CRM.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `00 - Menu principal`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «Olá! Para direcionar seu atendimento, escolha uma opção:».
- **Bloco 2 — `choice input`:** Opções: `Quero Comprar` (valor `QUERO_COMPRAR`), `Quero Vender` (valor `QUERO_VENDER`), `Financeiro` (valor `FINANCEIRO`), `Recursos Humanos` (valor `RECURSOS_HUMANOS`), `SAC` (valor `SAC`). Guarda em `destino_atendimento`.

**Conexões de saída neste export:** → `01 - Financeiro: Pagar ou Receber` (saída de opção/condição); → `04 - Handoff Quero Vender` (saída de opção/condição); → `05 - Handoff Recursos Humanos` (saída de opção/condição); → `06 - Handoff SAC` (saída de opção/condição); → `07 - Handoff Quero Comprar` (saída de opção/condição)

### 4.2. 01 - Financeiro: Pagar ou Receber

**O que faz e por quê:** Exibe duas escolhas; registra em `opcao_financeiro` e conduz ao comando respectivo.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `01 - Financeiro: Pagar ou Receber`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «Para direcionar seu atendimento ao setor financeiro, me diga o que você deseja:».
- **Bloco 2 — `choice input`:** Opções: `Quero Pagar` (valor `PAGAR`), `Quero Receber` (valor `RECEBER`). Guarda em `opcao_financeiro`.

**Conexões de saída neste export:** → `03 - Handoff Financeiro: Receber` (saída de opção/condição); → `02 - Handoff Financeiro: Pagar` (saída de opção/condição)

### 4.3. 02 - Handoff Financeiro: Pagar

**O que faz e por quê:** Cria comando de tag 3, encaminha à fila 5 (pending) e termina sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `02 - Handoff Financeiro: Pagar`.

- **Bloco 1 — `Set variable`:** Define `cmd_pagar_tag`. Comando TicketZ `addTag`; etiqueta `3`.
- **Bloco 2 — `Set variable`:** Define `cmd_pagar_handoff`. Comando TicketZ `updateTicket`; fila `5`.
- **Bloco 3 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Estou encaminhando o seu atendimento.».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_pagar_tag}}».
- **Bloco 6 — `text`:** Mensagem/conteúdo: «{{cmd_pagar_handoff}}».
- **Bloco 7 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.4. 03 - Handoff Financeiro: Receber

**O que faz e por quê:** Cria comando de tag 4, encaminha à fila 5 (pending) e termina sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `03 - Handoff Financeiro: Receber`.

- **Bloco 1 — `Set variable`:** Define `cmd_receber_tag`. Comando TicketZ `addTag`; etiqueta `4`.
- **Bloco 2 — `Set variable`:** Define `cmd_receber_handoff`. Comando TicketZ `updateTicket`; fila `5`.
- **Bloco 3 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Estou encaminhando o seu atendimento.».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_receber_tag}}».
- **Bloco 6 — `text`:** Mensagem/conteúdo: «{{cmd_receber_handoff}}».
- **Bloco 7 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.5. 04 - Handoff Quero Vender

**O que faz e por quê:** Move diretamente para fila 3 e encerra a sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `04 - Handoff Quero Vender`.

- **Bloco 1 — `Set variable`:** Define `cmd_vender_handoff`. Comando TicketZ `updateTicket`; fila `3`.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «Seu atendimento será encaminhado ao setor responsável.».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_vender_handoff}}».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.6. 05 - Handoff Recursos Humanos

**O que faz e por quê:** Move para fila 1 e encerra a sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `05 - Handoff Recursos Humanos`.

- **Bloco 1 — `Set variable`:** Define `cmd_rh_handoff`. Comando TicketZ `updateTicket`; fila `1`.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «Seu atendimento será encaminhado ao setor de Recursos Humanos.».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_rh_handoff}}».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.7. 06 - Handoff SAC

**O que faz e por quê:** Move para fila 2 e encerra a sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `06 - Handoff SAC`.

- **Bloco 1 — `Set variable`:** Define `cmd_sac_handoff`. Comando TicketZ `updateTicket`; fila `2`.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «Seu atendimento será encaminhado ao SAC.».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_sac_handoff}}».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.8. 07 - Quero Comprar: identificação de cliente

**O que faz e por quê:** **Ramo isolado intencionalmente.** Pergunta se a pessoa já é cliente. Sim segue para validação; Não vai ao Comercial com nota.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `07 - Quero Comprar: identificação de cliente`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «Que bom ter você por aqui 😊. / Você já é nosso cliente?».
- **Bloco 2 — `choice input`:** Opções: `Sim`, `Não`. Guarda em `cliente_ja_cadastrado`.

**Conexões de saída neste export:** → `08 - Vendas: novo cliente / handoff` (saída de opção/condição); → `09-1 - Inicialização` (saída de opção/condição)

### 4.9. 08 - Vendas: novo cliente / handoff

**O que faz e por quê:** Registra uma nota de que o contato declarou não ser cliente, encaminha à fila 4 e encerra.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `08 - Vendas: novo cliente / handoff`.

- **Bloco 1 — `Set variable`:** Define `cmd_novo_cliente_comercial_nota`. Comando TicketZ `note`.
- **Bloco 2 — `Set variable`:** Define `cmd_novo_cliente_comercial_handoff`. Comando TicketZ `updateTicket`; fila `4`.
- **Bloco 3 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Perfeito. Vou encaminhar seu atendimento para a equipe Comercial.».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_novo_cliente_comercial_nota}}».
- **Bloco 6 — `text`:** Mensagem/conteúdo: «{{cmd_novo_cliente_comercial_handoff}}».
- **Bloco 7 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.10. 09-1 - Inicialização

**O que faz e por quê:** Zera `tentativas_consulta_documento` e `tentativas_documento_invalido` antes da coleta.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `09-1 - Inicialização`.

- **Bloco 1 — `Set variable`:** Define `tentativas_consulta_documento`. Expressão/valor: `0`.
- **Bloco 2 — `Set variable`:** Define `tentativas_documento_invalido`. Expressão/valor: `0`.

**Conexões de saída neste export:** → `09 - Vendas: solicitar CPF/CNPJ` (padrão)

### 4.11. 09 - Vendas: solicitar CPF/CNPJ

**O que faz e por quê:** Exibe pedido do documento e armazena entrada em `doc_informado`.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `09 - Vendas: solicitar CPF/CNPJ`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «Para localizar seu cadastro, informe seu CPF ou CNPJ.».
- **Bloco 2 — `text input`:** Captura entrada em `doc_informado`.

**Conexões de saída neste export:** → `10 - Vendas: validar documento` (padrão)

### 4.12. 10 - Vendas: validar documento

**O que faz e por quê:** Remove pontuação, classifica por tamanho e valida os dígitos de CPF ou CNPJ; inválido segue para tentativa; válido segue ao webhook.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10 - Vendas: validar documento`.

- **Bloco 1 — `Set variable`:** Define `doc_normalizado`. Expressão/valor: `String({{doc_informado}} ?? '').replace(/\D/g, '')`.
- **Bloco 2 — `Set variable`:** Define `doc_tipo`. Lógica JavaScript de 161 caracteres; fonte completa indicada no apêndice de código.
- **Bloco 3 — `Set variable`:** Define `doc_validacao`. Lógica JavaScript de 1533 caracteres; fonte completa indicada no apêndice de código.
- **Bloco 4 — `Condition`:** Condição `doc_validacao` Equal to `INVALIDO`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10A - Documento Inválido` (saída de opção/condição); → `10B - Documento Válido` (padrão)

### 4.13. 10A - Documento Inválido

**O que faz e por quê:** Incrementa contador de documento inválido, decide repetir ou escalar depois de duas ocorrências.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10A - Documento Inválido`.

- **Bloco 1 — `Set variable`:** Define `tentativas_documento_invalido`. Expressão/valor: `Number({{tentativas_documento_invalido}} ?? 0) + 1`.
- **Bloco 2 — `Set variable`:** Define `status_tentativa_documento_invalido`. Expressão/valor: `Number({{tentativas_documento_invalido}} ?? 0) < 2   ? 'REPETIR'   : 'ENCAMINHAR'`.
- **Bloco 3 — `Condition`:** Condição `status_tentativa_documento_invalido` Equal to `REPETIR`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10A-1` (saída de opção/condição); → `10A-2 - Limite de Documento Inválido` (padrão)

### 4.14. 10A-1

**O que faz e por quê:** Mensagem de erro e bloco Return para nova digitação.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10A-1`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «O CPF ou CNPJ informado não é válido.  Confira os números e tente novamente. ».
- **Bloco 2 — `Return`:** Retorna ao ponto de entrada de dados para nova tentativa (confirmar comportamento na versão instalada).

**Conexões de saída neste export:** → `09 - Vendas: solicitar CPF/CNPJ` (padrão)

### 4.15. 10A-2 - Limite de Documento Inválido

**O que faz e por quê:** Marca `DOCUMENTO_INVALIDO_REPETIDO`, envia nota e encaminha ao atendimento humano.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10A-2 - Limite de Documento Inválido`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `DOCUMENTO_INVALIDO_REPETIDO`.
- **Bloco 2 — `Set variable`:** Define `cmd_limite_doc_invalido`. Comando TicketZ `note`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_limite_doc_invalido}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Não foi possível validar o CPF ou CNPJ informado. / Aguarde um momento!».

**Conexões de saída neste export:** → `20 - Vendas: handoff comercial` (padrão)

### 4.16. 10B - Documento Válido

**O que faz e por quê:** Reseta contadores/campos ligados à consulta, executa webhook, decide se aplica tag e avalia status do retorno.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B - Documento Válido`.

- **Bloco 1 — `Set variable`:** Define `tentativas_documento_invalido`. Expressão/valor: `0`.
- **Bloco 2 — `Set variable`:** Define `ticketz_tag_id`. Expressão/valor: `0`.
- **Bloco 3 — `Set variable`:** Define `vendedor_mapeado`. Expressão/valor: `false`.
- **Bloco 4 — `Set variable`:** Define `vendedor_motivo`. Expressão/valor: `''`.
- **Bloco 5 — `Webhook`:** POST HTTPS para `https://crm-api.ncloud.app.br/v1/clientes/documento`, corpo JSON com `doc_normalizado`, header `Content-Type` e header confidencial `X-CRM-Gateway-Key`; mapeia 11 campos de resposta.
- **Bloco 6 — `Set variable`:** Define `decisao_tag_ticketz`. Lógica JavaScript de 252 caracteres; fonte completa indicada no apêndice de código.
- **Bloco 7 — `Condition`:** Condição `consulta_status` Equal to `ENCONTRADO`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10B-0` (padrão); → `11 - Cliente Encontrado por Documento` (saída de opção/condição)

### 4.17. 10B-0

**O que faz e por quê:** Se o status for `NAO_ENCONTRADO`, segue para nova tentativa; demais vão avaliar duplicidade.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-0`.

- **Bloco 1 — `Condition`:** Condição `consulta_status` Equal to `NAO_ENCONTRADO`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10B-1 - Documento Não Localizado` (saída de opção/condição); → `10B-0A` (padrão)

### 4.18. 10B-1 - Documento Não Localizado

**O que faz e por quê:** Incrementa contador de consultas sem cadastro e calcula repetir/encaminhar.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-1 - Documento Não Localizado`.

- **Bloco 1 — `Set variable`:** Define `tentativas_consulta_documento`. Expressão/valor: `Number({{tentativas_consulta_documento}} ?? 0) + 1`.
- **Bloco 2 — `Set variable`:** Define `status_tentativa_documento`. Expressão/valor: `Number({{tentativas_consulta_documento}} ?? 0) < 2   ? 'REPETIR'   : 'ENCAMINHAR'`.

**Conexões de saída neste export:** → `10B-1A` (padrão)

### 4.19. 10B-1A

**O que faz e por quê:** Testa `status_tentativa_documento` e ramifica.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-1A`.

- **Bloco 1 — `Condition`:** Condição `status_tentativa_documento` Equal to `REPETIR`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10B-1A2 - Cliente Não Identificado` (padrão); → `10B-1A1 - Conferir Documento` (saída de opção/condição)

### 4.20. 10B-1A2 - Cliente Não Identificado

**O que faz e por quê:** Marca `NAO_IDENTIFICADO`, cria nota sobre não localização após duas tentativas e transfere para Comercial.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-1A2 - Cliente Não Identificado`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `NAO_IDENTIFICADO`.
- **Bloco 2 — `Set variable`:** Define `cmd_limite_cadastro`. Comando TicketZ `note`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_limite_cadastro}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Não foi possível localizar seu cadastro com os dados informados. / Aguarde um momento!».

**Conexões de saída neste export:** → `20 - Vendas: handoff comercial` (padrão)

### 4.21. 10B-1A1 - Conferir Documento

**O que faz e por quê:** Solicita conferência após primeira consulta sem cadastro.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-1A1 - Conferir Documento`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «Não localizei seu cadastro com o CPF ou CNPJ informado. / Confira atentamente os números e tente novamente.».

**Conexões de saída neste export:** → `Group #11` (padrão)

### 4.22. Group #11

**O que faz e por quê:** Bloco Return, voltando ao pedido do documento.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `Group #11`.

- **Bloco 1 — `Return`:** Retorna ao ponto de entrada de dados para nova tentativa (confirmar comportamento na versão instalada).

**Conexões de saída neste export:** → `09 - Vendas: solicitar CPF/CNPJ` (padrão)

### 4.23. 11 - Cliente Encontrado por Documento

**O que faz e por quê:** Transforma o campo Ativo em `ATIVO`/`INATIVO` e separa as situações.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11 - Cliente Encontrado por Documento`.

- **Bloco 1 — `Set variable`:** Define `cliente_status_cadastro`. Expressão/valor: `String({{cliente_ativo}} ?? '').toLowerCase() === 'true'   ? 'ATIVO'   : 'INATIVO'`.
- **Bloco 2 — `Condition`:** Condição `cliente_status_cadastro` Equal to `ATIVO`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `11B - Cliente Inativo` (padrão); → `11A - Cliente Ativo` (saída de opção/condição)

### 4.24. 11B - Cliente Inativo

**O que faz e por quê:** Marca `IDENTIFICADO_INATIVO`, anota e encaminha à análise humana.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11B - Cliente Inativo`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `IDENTIFICADO_INATIVO`.
- **Bloco 2 — `Set variable`:** Define `cmd_cliente_inativo`. Comando TicketZ `note`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_cliente_inativo}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Olá, {{cliente_nome}}. / Seu cadastro foi localizado. / Aguarde um momento!».

**Conexões de saída neste export:** → `20 - Vendas: handoff comercial` (padrão)

### 4.25. 11A - Cliente Ativo

**O que faz e por quê:** Marca `IDENTIFICADO_ATIVO`, saúda pelo nome e prepara encaminhamento contextualizado.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11A - Cliente Ativo`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `IDENTIFICADO_ATIVO`.
- **Bloco 2 — `text`:** Mensagem/conteúdo: «Olá,{{cliente_nome}} ! / Localizei seu cadastro com sucesso. Vamos continuar seu atendimento.».

**Conexões de saída neste export:** → `11A-1 - Preparar Handoff Humano` (padrão)

### 4.26. 11A-1 - Preparar Handoff Humano

**O que faz e por quê:** Monta comando opcional de tag, nota interna com nome/documento mascarado/local/vendedor/status, fila 4 e encerramento; decisão APLICAR/PULAR.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11A-1 - Preparar Handoff Humano`.

- **Bloco 1 — `Set variable`:** Define `cmd_ticketz_tag`. Lógica JavaScript de 294 caracteres; fonte completa indicada no apêndice de código.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_nota`. Lógica JavaScript de 1955 caracteres; fonte completa indicada no apêndice de código.
- **Bloco 3 — `Set variable`:** Define `cmd_ticketz_pendente`. Comando TicketZ `updateTicket`; fila `4`.
- **Bloco 4 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 5 — `Condition`:** Condição `decisao_tag_ticketz` Equal to `APLICAR`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `11A-1A - Handoff com Tag` (saída de opção/condição); → `11B-1A - Handoff sem Tag` (padrão)

### 4.27. 11A-1A - Handoff com Tag

**O que faz e por quê:** Emite tag, depois nota, atualização para pendente e fim de sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11A-1A - Handoff com Tag`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_tag}}».
- **Bloco 2 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_nota}}».
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_pendente}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.28. 11B-1A - Handoff sem Tag

**O que faz e por quê:** Emite nota, atualização para pendente e fim, sem emitir etiqueta.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `11B-1A - Handoff sem Tag`.

- **Bloco 1 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_nota}}».
- **Bloco 2 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_pendente}}».
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.29. 10B-0A

**O que faz e por quê:** Se status `DUPLICADO` vai à explicação própria; os outros resultados não reconhecidos seguem à falha técnica.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-0A`.

- **Bloco 1 — `Condition`:** Condição `consulta_status` Equal to `DUPLICADO`; caso contrário usa a saída padrão.

**Conexões de saída neste export:** → `10B-0A1 - Cadastro Duplicado` (saída de opção/condição); → `10B-0B1 - Falha Técnica na Consulta` (padrão)

### 4.30. 10B-0A1 - Cadastro Duplicado

**O que faz e por quê:** Marca `DADOS_DUPLICADOS`, anota múltiplos registros e transfere.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-0A1 - Cadastro Duplicado`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `DADOS_DUPLICADOS`.
- **Bloco 2 — `Set variable`:** Define `cmd_cadastro_duplicado`. Comando TicketZ `note`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_cadastro_duplicado}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Localizei mais de um cadastro associado aos dados informados. / Este atendimento precisará seguir para análise humana.».

**Conexões de saída neste export:** → `20 - Vendas: handoff comercial` (padrão)

### 4.31. 10B-0B1 - Falha Técnica na Consulta

**O que faz e por quê:** Marca `ERRO_TECNICO`, anota a falha e transfere para pessoa.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `10B-0B1 - Falha Técnica na Consulta`.

- **Bloco 1 — `Set variable`:** Define `resultado_identificacao`. Expressão/valor: `ERRO_TECNICO`.
- **Bloco 2 — `Set variable`:** Define `cmd_falha_tecnica`. Comando TicketZ `note`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «{{cmd_falha_tecnica}}».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «Não consegui consultar seu cadastro neste momento. / Um momento, você já será atendido.».

**Conexões de saída neste export:** → `20 - Vendas: handoff comercial` (padrão)

### 4.32. 20 - Vendas: handoff comercial

**O que faz e por quê:** Centraliza casos excepcionais na fila 4 (pending), terminando a sessão.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `20 - Vendas: handoff comercial`.

- **Bloco 1 — `Set variable`:** Define `cmd_comercial_handoff`. Comando TicketZ `updateTicket`; fila `4`.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «Estou encaminhando seu atendimento para a equipe Comercial.».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_comercial_handoff}}».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

### 4.33. 07 - Handoff Quero Comprar

**O que faz e por quê:** **Ramo atualmente ligado ao menu.** Envia diretamente para fila Comercial 4 e finaliza o bot.

**Onde:** Typebot → `Atendimento - Triagem - Homologação` → grupo `07 - Handoff Quero Comprar`.

- **Bloco 1 — `Set variable`:** Define `cmd_quero_comprar_handoff`. Comando TicketZ `updateTicket`; fila `4`.
- **Bloco 2 — `Set variable`:** Define `cmd_ticketz_end_session`. Comando TicketZ `endSession`.
- **Bloco 3 — `text`:** Mensagem/conteúdo: «Estou encaminhando o seu atendimento 😊 ».
- **Bloco 4 — `text`:** Mensagem/conteúdo: «{{cmd_quero_comprar_handoff}}».
- **Bloco 5 — `text`:** Mensagem/conteúdo: «{{cmd_ticketz_end_session}}».

**Conexões de saída neste export:** nenhuma; término do ramo.

## 5. Variáveis — dicionário completo (45 registradas)

**Como localizar:** no editor Typebot, abra o fluxo e procure cada variável pelo nome na configuração do bloco. O ID é estável no export e permite identificá-la mesmo após reorganização visual. Variáveis `session=True` são configuradas como variáveis de sessão no export; as quatro entradas do usuário indicadas como `False` não são variáveis de sessão. Não presuma persistência entre contatos nem armazenamento no NocoDB só porque a variável existe.

| Variável | ID interno no export | Sessão? | Função e origem/destino |
|---|---|---|---|
| `cmd_quero_comprar_handoff` | `vwnbvbjmn7ryfdihb559hqxkk` | Sim | Encaminhamento direto da opção Quero Comprar para Comercial. |
| `cmd_falha_tecnica` | `vgn2ay2zjcxbwmghhf1d07398` | Sim | Nota interna indicando falha na consulta ao CRM. |
| `cmd_cadastro_duplicado` | `vaowx15aixytu13iw9lnj1qyh` | Sim | Nota interna indicando mais de um registro para o documento. |
| `cmd_cliente_inativo` | `vlu1oqddz8tfr3klbf0z9sxhp` | Sim | Nota interna sobre cadastro localizado, porém inativo. |
| `cmd_limite_cadastro` | `vv6gld1rm1zv6qhiank5ocsax` | Sim | Nota interna sobre cliente não localizado após duas consultas. |
| `cmd_limite_doc_invalido` | `vu5arn0cb52nnsiedpuwl11r6` | Sim | Nota interna sobre documento inválido após duas tentativas. |
| `cmd_comercial_handoff` | `vdq9sejr6eow2exwq7c237398` | Sim | Encaminhamento genérico para Comercial após exceções no ramo CRM. |
| `cmd_ticketz_tag` | `wykuoaqtwo014wtth6dnbqte` | Sim | Comando condicional para adicionar etiqueta de vendedor. |
| `cmd_ticketz_nota` | `a8fex9v5xtmi5ahaux9ur957` | Sim | Nota interna com dados limitados do cliente identificado. |
| `cmd_ticketz_pendente` | `ywxcl9c7e8354lmxfg7ugw1z` | Sim | Encaminha atendimento identificado para fila Comercial pendente. |
| `cliente_status_cadastro` | `d0qy2o0h5hj21ogtqvhq7qeg` | Sim | Converte cliente_ativo em ATIVO ou INATIVO. |
| `status_tentativa_documento` | `u3nsoxjthy11r7te7iu1ousx` | Sim | REPETIR antes do limite; ENCAMINHAR após duas consultas sem cadastro. |
| `ticketz_tag_id` | `n3ibrmsin8fur0q7oam01ll1` | Sim | Número da etiqueta TicketZ retornado pelo gateway para o vendedor; 0 significa não configurada. |
| `vendedor_mapeado` | `j5yay31f2ukgaqtysewepqro` | Sim | Indica se o gateway conseguiu mapear o vendedor a um relacionamento válido. |
| `vendedor_motivo` | `tqu7m4rklbbn7oaq0kw3h1bg` | Sim | Código de justificativa para não aplicar etiqueta de vendedor. |
| `cliente_nome` | `xcc0r32z5qip57nzzr8d8f2n` | Sim | Nome devolvido pelo gateway do CRM; usado na saudação e nota. |
| `cliente_tipo_pessoa` | `lcew3hxwxeaxq5cug886kf2j` | Sim | TipoPessoa devolvido pelo CRM; reservado para uso/classificação futura. |
| `cliente_id` | `ze4t1mz8k4q56h9wgwm3m7a1` | Sim | Identificador do registro do cliente no CRM; não é o ID da conversa. |
| `cliente_cidade` | `iwlepybc4try606gldm154gu` | Sim | Cidade recebida do CRM, incluída na nota interna. |
| `cliente_uf` | `cwb3euwvc7vs02xljyrhqtm0` | Sim | UF recebida do CRM, incluída na nota interna. |
| `cliente_vendedor` | `r8pbnqdx3flajdii2lqflvc7` | Sim | Nome/identificação do vendedor atribuído no CRM; incluído na nota. |
| `cliente_ativo` | `k89onjne8z0lpbt5buyrxol3` | Sim | Campo Ativo vindo do CRM; define ramo ativo/inativo. |
| `consulta_status` | `avice3sfyok6qtabbhmuhuzp` | Sim | Situação da consulta: ENCONTRADO, NAO_ENCONTRADO, DUPLICADO ou demais casos tratados como erro. |
| `decisao_tag_ticketz` | `q05n5dg9h2m2c27espf0w5yy` | Sim | Define APLICAR se mapeado e tag ID inteiro positivo; caso contrário PULAR. |
| `resultado_identificacao` | `whj4b86eylh7ie15y0abpzvg` | Sim | Classificação final para auditoria da triagem: ativo, inativo, inválido, ausente, duplicado ou erro. |
| `status_tentativa_documento_invalido` | `lqqgylqdd61ezwa77g9xsass` | Sim | REPETIR antes do limite; ENCAMINHAR após duas entradas inválidas. |
| `doc_normalizado` | `gilwocm0rsyp98pn1y6oms93` | Sim | Versão do documento somente com algarismos, usada na validação e na consulta. |
| `doc_tipo` | `kbhqub6h40dei1mhgyyib2pt` | Sim | Classifica pelo comprimento: CPF, CNPJ ou INVALIDO; não equivale à verificação dos dígitos. |
| `doc_validacao` | `f134cv8r92jfp1clf9z1gxfr` | Sim | Resultado da verificação dos dígitos verificadores: VALIDO ou INVALIDO. |
| `doc_informado` | `hvkcau35nu8ngtmponz7ry9n` | Não | Entrada bruta do CPF/CNPJ digitado pela pessoa; pode conter pontuação. |
| `tentativas_consulta_documento` | `qinzv8v2w0u16dbrgz336has` | Sim | Conta documentos válidos não encontrados na consulta; permite uma nova tentativa. |
| `tentativas_documento_invalido` | `nbzw0gmzs6vu1gfow38kcx8d` | Sim | Conta documentos inválidos; após a segunda ocorrência faz handoff. |
| `cmd_novo_cliente_comercial_nota` | `va0k4a7zuu5fb1c7sluhmyq6i` | Sim | Nota interna informando que a pessoa declarou não ser cliente. |
| `cmd_novo_cliente_comercial_handoff` | `vymsm3zd5fk47ghdwt5gkmfrb` | Sim | Encaminha não clientes para Comercial. |
| `cliente_ja_cadastrado` | `vn81vnm7zjtz0fspcun52kr07` | Não | Resposta Sim/Não sobre já ser cliente (ramo preparado). |
| `cmd_receber_tag` | `ttzijqi2j7n4502jsnnzbz6b` | Sim | Marca categoria Receber no TicketZ. |
| `cmd_pagar_tag` | `tafs0byzclkvy9q7w03orvcw` | Sim | Marca categoria Pagar no TicketZ. |
| `cmd_receber_handoff` | `bp6beqewjpcc989xvnese825` | Sim | Envia Financeiro/Receber à fila 5. |
| `cmd_pagar_handoff` | `y6csgzjfydubwigkgc6wymc8` | Sim | Envia Financeiro/Pagar à fila 5. |
| `cmd_vender_handoff` | `ycf2o1ljjvgxnrxv0lgwxhfm` | Sim | Encaminha Quero Vender à fila 3. |
| `cmd_rh_handoff` | `nzh5a3gla2lnifav4l40p4at` | Sim | Encaminha Recursos Humanos à fila 1. |
| `cmd_sac_handoff` | `t3gxho56x01xc0i7twu2hdjf` | Sim | Encaminha SAC à fila 2. |
| `cmd_ticketz_end_session` | `kgapciz4wrd7isoegxzb3s5j` | Sim | Comando TicketZ que encerra a sessão do bot, usado após encaminhar. |
| `opcao_financeiro` | `v2r5dpuy2nhirdo82buaixk2` | Não | Guarda Pagar ou Receber e direciona ao respectivo roteiro financeiro. |
| `destino_atendimento` | `lnws90rvd4pgn3be1ea2arvi` | Não | Registra a opção do menu principal; orienta a seleção de setor. |

### 5.1 Fluxo de dados entre variáveis

1. `doc_informado` → `doc_normalizado` (remove tudo que não é dígito) → `doc_tipo` (11 ou 14 dígitos) → `doc_validacao` (dígitos verificadores).
2. Se documento válido, `doc_normalizado` entra no JSON enviado ao gateway. O gateway devolve `data.status` e possivelmente `data.cliente`.
3. Os dados retornados alimentam `cliente_nome`, `cliente_tipo_pessoa`, `cliente_id`, `cliente_cidade`, `cliente_uf`, `cliente_vendedor`, `cliente_ativo`, `ticketz_tag_id`, `vendedor_mapeado`, `vendedor_motivo`.
4. `consulta_status` decide a próxima etapa; `cliente_ativo` gera `cliente_status_cadastro`; `vendedor_mapeado` + `ticketz_tag_id` geram `decisao_tag_ticketz`.
5. As variáveis prefixadas `cmd_` guardam **instruções a emitir para o TicketZ**, não uma confirmação de que o TicketZ as executou.

### 5.2 Todos os contadores e limiares

| Contador | Inicialização | Incremento | Limite | Consequência |
|---|---|---|---|---|
| `tentativas_documento_invalido` | 0 no grupo `09-1` | A cada CPF/CNPJ inválido (`10A`) | `< 2` repete; a partir de 2 escala | Nota interna, grupo `20`. |
| `tentativas_consulta_documento` | 0 no grupo `09-1` | A cada documento válido mas não encontrado (`10B-1`) | `< 2` repete; a partir de 2 escala | Nota interna, grupo `20`. |

**Atenção:** no grupo `10B - Documento Válido` o contador de documento inválido é zerado, porém o de documentos válidos não localizados não é reinicializado. Ao ativar o ramo, verificar o efeito desejado em novas tentativas. A validação implementada verifica dígitos verificadores do CPF e CNPJ numéricos; não comprova existência, propriedade, situação cadastral pública nem autorização do solicitante. Alterações legais em formatos de CNPJ exigirão revisão do código antes de sua entrada em vigor.

## 6. Gateway de CRM / NocoDB — especificação de integração

### 6.1 Onde fica configurado

No grupo `10B - Documento Válido`, bloco **Webhook**. O export mostra:

- **Método de operação esperado:** POST com corpo JSON customizado.
- **Endpoint:** `https://crm-api.ncloud.app.br/v1/clientes/documento`.
- **Cabeçalhos:** `Content-Type: application/json` e `X-CRM-Gateway-Key: [CHAVE SIGILOSA]`.
- **Payload:** `{ "documento": "{{doc_normalizado}}" }`.
- **Dado de teste incluído no export:** documento numérico de exemplo, **não use em produção** como dado real de cliente.

**O papel do gateway:** intermediar e proteger o NocoDB. O Typebot não acessa diretamente uma tabela do CRM neste export. O gateway recebe o documento, consulta sua própria integração com CRM/NocoDB e devolve uma resposta padronizada. Os endpoints internos, URL do NocoDB, nomes/IDs de tabelas, usuário de banco, banco físico, credenciais e regras do gateway **não constam do JSON**: documentá-los só após examinar o serviço e suas configurações reais.

### 6.2 Contrato de resposta que o Typebot espera

| JSON path na resposta do webhook | Variável Typebot | Uso |
|---|---|---|
| `data.cliente.Nome` | `cliente_nome` | Nome devolvido pelo gateway do CRM; usado na saudação e nota. |
| `data.cliente.TipoPessoa` | `cliente_tipo_pessoa` | TipoPessoa devolvido pelo CRM; reservado para uso/classificação futura. |
| `data.cliente.id` | `cliente_id` | Identificador do registro do cliente no CRM; não é o ID da conversa. |
| `data.cliente.Cidade` | `cliente_cidade` | Cidade recebida do CRM, incluída na nota interna. |
| `data.cliente.UF` | `cliente_uf` | UF recebida do CRM, incluída na nota interna. |
| `data.cliente.Vendedor` | `cliente_vendedor` | Nome/identificação do vendedor atribuído no CRM; incluído na nota. |
| `data.cliente.Ativo` | `cliente_ativo` | Campo Ativo vindo do CRM; define ramo ativo/inativo. |
| `data.status` | `consulta_status` | Situação da consulta: ENCONTRADO, NAO_ENCONTRADO, DUPLICADO ou demais casos tratados como erro. |
| `data.cliente.TicketZTagID` | `ticketz_tag_id` | Número da etiqueta TicketZ retornado pelo gateway para o vendedor; 0 significa não configurada. |
| `data.cliente.VendedorMapeado` | `vendedor_mapeado` | Indica se o gateway conseguiu mapear o vendedor a um relacionamento válido. |
| `data.cliente.VendedorMotivo` | `vendedor_motivo` | Código de justificativa para não aplicar etiqueta de vendedor. |

**Exemplo ilustrativo de formato esperado** (valores fictícios; não é uma resposta capturada do servidor):

```json
{
  "data": {
    "status": "ENCONTRADO",
    "cliente": {
      "id": "id-exemplo",
      "Nome": "Cliente Exemplo",
      "TipoPessoa": "F",
      "Cidade": "Cidade Exemplo",
      "UF": "MG",
      "Vendedor": "Equipe Comercial",
      "Ativo": true,
      "TicketZTagID": 12,
      "VendedorMapeado": true,
      "VendedorMotivo": ""
    }
  }
}
```

Este JSON é um **modelo didático de contrato**; os nomes das chaves foram copiados do mapeamento e os valores são fictícios. O retorno real deve ser verificado no gateway. Para consultas não encontradas, duplicadas ou com erro, o Typebot reconhece `NAO_ENCONTRADO`, `DUPLICADO` e utiliza uma rota residual de falha técnica para qualquer outro status, inclusive ausente/desconhecido.

### 6.3 Regras de classificação e etiqueta

- `ENCONTRADO` + `Ativo=true` → `IDENTIFICADO_ATIVO` → nota e Comercial; pode haver tag.
- `ENCONTRADO` + `Ativo` diferente de `true` → `IDENTIFICADO_INATIVO` → nota e Comercial. **Revisão recomendada:** um valor ausente ou malformado também cai em inativo, não em erro.
- `NAO_ENCONTRADO` → primeira ocorrência solicita repetir; segunda registra nota e encaminha ao Comercial.
- `DUPLICADO` → nota de múltiplos registros e encaminhamento humano.
- Qualquer outro status → tratamento de erro técnico e encaminhamento humano.
- Só aplica etiqueta ao cliente ativo se `VendedorMapeado` convertido para string minúscula for `true` **e** `TicketZTagID` converter para inteiro positivo. Se não, anota/encaminha sem tag.

Códigos de motivo tratados na nota interna: `VENDEDOR_NAO_INFORMADO`, `VENDEDOR_SEM_RELACIONAMENTO`, `VENDEDOR_INATIVO`, `TAG_NAO_CONFIGURADA`. Códigos novos podem aparecer na nota com seu valor literal.

### 6.4 Como atualizar o endpoint e a chave com segurança

1. Exporte um backup do fluxo e guarde em local protegido.
2. Confirme, com quem mantém o gateway, o novo endereço HTTPS e contrato de resposta.
3. No editor, abra `10B - Documento Válido` → **Webhook**.
4. Edite a URL e, se necessário, altere somente o valor do header `X-CRM-Gateway-Key`; **não exponha essa chave em prints, logs, repositórios ou documentação pública**.
5. Garanta que o gateway aceite a chave correspondente e que o certificado TLS esteja válido.
6. Faça testes com massa fictícia/autorizada para encontrado, não encontrado, duplicado, timeout e chave inválida.
7. Confira todos os mapeamentos `data.*`, os comandos TicketZ resultantes e que o usuário não vê as mensagens internas.
8. Só então publique o fluxo e registre a versão/rollback.

**Segurança adicional:** o export contém a chave em texto legível. Recomenda-se substituí-la por segredo gerenciado quando a instalação permitir; evitar compartilhar export bruto com terceiros; revisar permissões e programar rotação conforme política de segurança.

## 7. Integração com TicketZ — catálogo de comandos

Os comandos são construídos por blocos `Set variable` com expressão JavaScript e só são despachados quando blocos `text` exibem as variáveis correspondentes. Não são chamadas HTTP feitas pelo Typebot no JSON; dependem do interpretador de comandos do TicketZ/canal.

### 7.1 Ações reconhecidas no fluxo

| Ação | Exemplo de estrutura | Efeito pretendido |
|---|---|---|
| `updateTicket` | `#{"action":"updateTicket","ticketData":{"queueId":4,"status":"pending","annotation":"..."}}` | Atualiza setor/fila, deixa pendente e adiciona anotação. |
| `addTag` | `#{"action":"addTag","tagId":3}` | Adiciona etiqueta existente à conversa/chamado. |
| `note` | `#{"action":"note","message":{"content":"..."}}` | Registra nota interna (não destinada ao cliente). |
| `endSession` | `#{"action":"endSession"}` | Finaliza participação do bot na sessão. |

### 7.2 Mapeamento das filas e etiquetas

| Caminho | `queueId` | `tagId` | Status | Annotation |
|---|---:|---:|---|---|
| Quero Comprar (direto) | 4 | — | `pending` | `Triagem Atendimento > Quero Comprar` |
| Quero Comprar → não cliente (preparado) | 4 | — | `pending` | `Triagem Atendimento > Quero Comprar > Novo cliente` |
| Quero Comprar → cliente identificado (preparado) | 4 | dinâmico do CRM, se válido | `pending` | `Triagem Atendimento > Quero Comprar > Cliente identificado` |
| Quero Comprar → fallback humano (preparado) | 4 | — | `pending` | `Triagem Atendimento > Quero Comprar > Atendimento humano` |
| Quero Vender | 3 | — | `pending` | `Triagem Atendimento > Quero Vender` |
| Financeiro → Pagar | 5 | 3 | `pending` | `Triagem Atendimento > Financeiro > Pagar` |
| Financeiro → Receber | 5 | 4 | `pending` | `Triagem Atendimento > Financeiro > Receber` |
| Recursos Humanos | 1 | — | `pending` | `Triagem Atendimento > Recursos Humanos` |
| SAC | 2 | — | `pending` | `Triagem Atendimento > SAC` |

Esses IDs são **números de configuração no TicketZ**, não nomes universais. Antes de mudar qualquer `queueId` ou `tagId`, confirme o cadastro correspondente no TicketZ instalado. Alterar números às cegas pode mandar clientes ao departamento errado. As anotações aparecem no payload das atualizações e servem de trilha de origem do atendimento.

### 7.3 Ordem de comandos

- **Financeiro/Pagar e Receber:** mensagem ao cliente → `addTag` → `updateTicket` → `endSession`.
- **Vender, RH, SAC, Comprar direto:** mensagem ao cliente → `updateTicket` → `endSession`.
- **Novo cliente (ramo futuro):** mensagem ao cliente → nota → `updateTicket` → `endSession`.
- **Cliente ativo identificado (ramo futuro):** saudação → [tag, se elegível] → nota → atualização → encerramento.
- **Exceções do CRM:** nota → mensagem de orientação → grupo `20` → atualização → encerramento.

**Não troque o encerramento de lugar** sem teste: se for emitido antes das anotações/filas, a integração pode interromper o processamento posterior.

## 8. Configurações gerais do Typebot

| Configuração | Valor exportado | Onde alterar | Por quê |
|---|---|---|---|
| Mensagem para escolha inválida | `Não entendi sua escolha. Por favor, escolha uma das opções exibidas.` | Typebot → configurações gerais → mensagens do sistema. | Ajuda o usuário a corrigir entrada fora do menu. |
| Intervalo entre balões (`typingEmulation.delayBetweenBubbles`) | `2` | Typebot → configuração de digitação/apresentação. | Melhora legibilidade do diálogo; conferir unidade conforme versão. |
| Tema (`theme`) | `{}` | Editor de tema. | Não há ajustes específicos no export. |
| Domínio personalizado (`customDomain`) | `null` | Publicação/domínio do Typebot. | Nenhum domínio próprio registrado no arquivo. |
| Arquivado / Fechado | `false` / `false` | Estado do fluxo. | Export marcado como disponível, sem comprovar publicação efetiva. |

## 9. Principais diretórios e arquivos

### 9.1 O que é possível afirmar

| Caminho/objeto | Conteúdo | Como utilizar |
|---|---|---|
| `/mnt/data/typebot-export-atendimento-triagem-homologa-o-dv7w312(1).json` (caminho nesta sessão; arquivo original anexo) | Definição completa do bot: blocos, variáveis, conexões, configurações e segredo do webhook. | Fonte de verdade deste documento; guardar criptografado/com acesso restrito. |
| Editor visual Typebot → fluxo `Atendimento - Triagem - Homologação` | Configuração operacional dos grupos. | Alterações pelo editor e posterior publicação. |
| Grupo `10B - Documento Válido` → Webhook | URL, header, corpo e mapeamentos. | Principal ponto de integração CRM. |
| Grupos com `cmd_*` → blocos `Set variable` e `text` | Comandos de encaminhamento ao TicketZ. | Conferir IDs e ordem. |

### 9.2 Caminhos de sistema operacional que **não podem ser determinados pelo export**

Não é possível afirmar o diretório do Docker Compose, caminho de `docker-compose.yml`/`compose.yaml`, `.env`, volumes do NocoDB, código do gateway, arquivos de proxy, bancos e certificados. Esses caminhos dependem da instalação real e **não devem ser inventados**. Para descobri-los no host Linux de cada serviço, consultar a seção 11 (comandos de leitura), anotar os caminhos revelados e complementar o cadastro de infraestrutura. Isto é diferente da *localização lógica* dos blocos, que está documentada aqui.

## 10. Inventário de valores configuráveis

| Valor | Configurado/observado no export | Local | Alteração segura |
|---|---|---|---|
| ID público do bot | `atendimento-triagem-homologa-o-dv7w312` | Raiz do JSON / painel Typebot. | Evitar trocar sem conferir embeds, URLs e gatilhos externos. |
| API do gateway CRM | `https://crm-api.ncloud.app.br/v1/clientes/documento` | `10B` → Webhook → URL. | Atualizar URL e testar HTTPS/contrato. |
| Nome do header de autenticação | `X-CRM-Gateway-Key` | `10B` → Webhook → headers. | Alterar em conjunto com gateway. |
| Valor do segredo | **Presente no JSON, ocultado neste documento** | `10B` → Webhook → header. | Rotacionar no gateway e Typebot, nunca compartilhar. |
| Body field | `documento` = `{{doc_normalizado}}` | `10B` → Webhook → JSON body. | Manter contrato com API. |
| ID Comercial | `queueId: 4` | Grupos `07`, `08`, `11A-1`, `20`. | Substituir em **todos** os grupos após verificar TicketZ. |
| ID Vendas/Quero Vender | `queueId: 3` | Grupo `04`. | Conferir setor no TicketZ. |
| ID Financeiro | `queueId: 5` | Grupos `02`, `03`. | Atualizar ambos. |
| ID Recursos Humanos | `queueId: 1` | Grupo `05`. | Conferir TicketZ. |
| ID SAC | `queueId: 2` | Grupo `06`. | Conferir TicketZ. |
| Tag Pagar / Receber | `3` / `4` | Grupos `02` / `03`. | Conferir etiquetas TicketZ. |
| Tag comercial por vendedor | Campo `data.cliente.TicketZTagID` | `10B` → mapeamento e `11A-1`. | Garantir vínculo no CRM e tag existente. |
| Máximo de documento inválido | `2` | `10A` → cálculo `< 2`. | Ajustar condição, texto da nota e teste. |
| Máximo de consulta sem cadastro | `2` | `10B-1` → cálculo `< 2`. | Ajustar condição, texto da nota e teste. |
| Status de encaminhamento | `pending` | Todos os `updateTicket`. | Validar enum aceito pelo TicketZ. |
| URLs/credenciais internas NocoDB | **Não consta no export.** | Gateway/NocoDB (fora do bot). | Inspecionar configuração do servidor; registrar apenas referências seguras. |
| Portas, volumes, usuários do host | **Não consta no export.** | Ambiente Docker/host. | Levantar no servidor antes de tocar em produção. |

## 11. Manutenção comum — procedimentos práticos

**Onde executar:** no terminal do *servidor Linux que efetivamente hospeda o serviço*, com autorização. Os comandos a seguir são procedimentos gerais; não pressupõem nome de contêiner ou pasta que não foram apresentados no JSON.

### 11.1 Identificar serviços e seus arquivos de instalação

```bash
# Ver containers existentes e seus nomes reais
docker ps --format 'table {{.Names}}	{{.Image}}	{{.Status}}	{{.Ports}}'

# Ver projetos/serviços Compose detectáveis
docker compose ls

# Para um container real, inspecionar origem do compose e volumes
# SUBSTITUIR <CONTAINER_REAL> pelo nome obtido em docker ps
docker inspect '<CONTAINER_REAL>' --format '{{json .Config.Labels}}'
docker inspect '<CONTAINER_REAL>' --format '{{json .Mounts}}'
```

Se houver `com.docker.compose.project.working_dir` ou `com.docker.compose.project.config_files`, os valores podem revelar diretório e arquivo de Compose. **Não cole saídas completas de `docker inspect` em locais públicos:** configurações/ambiente podem conter senhas e tokens.

### 11.2 Logs e estado

```bash
# Exemplo genérico, substitua pelo nome real
docker logs --tail 100 '<CONTAINER_REAL>'
docker stats --no-stream

docker inspect '<CONTAINER_REAL>' --format '{{.State.Status}} {{.State.Health.Status}}'
```

A última linha só funciona diretamente se o contêiner possuir healthcheck; sem ele, consulte apenas `.State.Status`. Logs do Typebot indicam problemas de execução; logs do gateway ajudam a distinguir erro de chave, DNS, TLS, autenticação e consultas; logs do TicketZ esclarecem se as mensagens `#` foram interpretadas.

### 11.3 Reiniciar com cautela

```bash
# Reinício pontual após registrar estado atual
docker restart '<CONTAINER_REAL>'

# Quando houver Compose validado, executar no diretório REAL do projeto:
cd '/CAMINHO/REAL/DO/PROJETO'
docker compose ps
docker compose restart '<NOME_DO_SERVICO_NO_COMPOSE>'
```

Nunca confundir nome do serviço Compose com nome completo do contêiner. Reiniciar pode interromper conversas em andamento; use janela adequada.

### 11.4 Atualizar componentes sem perder dados

1. Obtenha o caminho do Compose real (11.1) e confirme volumes/banco e mecanismo de rollback.
2. Exporte o Typebot no editor antes da mudança e gere cópia protegida do Compose/`.env`.
3. Garanta backup **consistente** do banco e volumes usados por cada serviço, seguindo documentação da versão instalada (não apenas copiando arquivos em uso).
4. Registre tags/digests das imagens existentes (`docker image ls`, `docker inspect`) e changelogs oficiais.
5. Atualize **um serviço por vez**, evitando `latest` sem controle de versão.
6. Com o Compose revisado, use `docker compose pull '<SERVICO>'` e `docker compose up -d --no-deps '<SERVICO>'` quando apropriado à arquitetura, no diretório correto.
7. Rode testes funcionais de todas as filas, etiquetas, CRM e encerramento de sessão.
8. Se houver regressão, use versão anterior das imagens e procedimento de restauração compatível com banco/volumes; não faça downgrade de esquema sem avaliação.

### 11.5 Backup das configurações e recuperação

O backup mínimo do **fluxo** é um export JSON feito no painel Typebot. Isto não substitui o backup dos **dados do servidor**, conversas, dados do NocoDB, configurações de gateway, bancos e volumes.

```bash
# Depois de encontrar o diretório real e AUTORIZADO do Compose:
cd '/CAMINHO/REAL/DO/PROJETO'
mkdir -p '/CAMINHO/SEGURO/DE/BACKUP'
cp -p compose.yaml '/CAMINHO/SEGURO/DE/BACKUP/'  # se o arquivo for compose.yaml
# .env pode conter segredos: realizar cópia SOMENTE criptografada/acesso restrito
```

A restauração do fluxo exige importar o JSON no Typebot **sem sobrescrever a versão boa antes de validar**; confirmar que as referências a filas e etiquetas permanecem corretas. Em recuperação de banco/volumes, restaurar em ambiente de teste primeiro e somente depois em produção. Não exportar arquivos com chaves a serviços públicos de armazenamento sem proteção apropriada.

### 11.6 Checklist de saúde

- [ ] Typebot abre e o menu apresenta exatamente cinco opções.
- [ ] Comprar entra na fila 4 **sem solicitar CPF/CNPJ** no caminho ativo.
- [ ] Vender entra na fila 3, RH na 1, SAC na 2.
- [ ] Financeiro/Pagar cria tag 3 e fila 5; Receber cria tag 4 e fila 5.
- [ ] Cada caminho conclui sessão sem mostrar `#{...}` ou mensagens `{{cmd_*}}` ao cliente.
- [ ] TicketZ registra status `pending`, anotação e fila esperados.
- [ ] O ramo isolado de CRM permanece isolado até aprovação de ativação.
- [ ] Ao testar o ramo isolado, documento inválido, ausente, ativo, inativo, duplicado e falha estão cobertos.
- [ ] Endpoint HTTPS, segredo de gateway e mapeamentos de resposta estão válidos.
- [ ] Logs recentes não mostram erros de autenticação, DNS, TLS, timeout, banco ou mensagem não interpretada.
- [ ] Backups protegidos existem e foi feito ao menos um teste de restauração.

## 12. Guia para iniciantes — alterações comuns

### 12.1 Alterar texto sem mudar a lógica

Abra o grupo desejado no editor, edite **somente o bloco `text` que contém a mensagem amigável**, pré-visualize, publique e teste. Não altere, por engano, blocos de texto cujo único conteúdo seja `{{cmd_*}}`, porque eles são usados para transmitir instruções.

### 12.2 Mudar uma fila do TicketZ

1. No TicketZ, localize o setor novo e seu ID numérico; registre ID atual e novo.
2. No Typebot, encontre todos os grupos relevantes no mapeamento da seção 7.2.
3. Nos blocos `Set variable`, modifique somente o `queueId` da respectiva ação `updateTicket`.
4. Não troque `tagId`, `annotation` ou `status` sem decisão específica.
5. Publique em homologação e confirme em conversa de teste antes da produção.

### 12.3 Trocar etiqueta de Pagar/Receber

No grupo `02` ou `03`, encontre o bloco que monta `action:'addTag'` e substitua o número `tagId`, após validar sua existência no TicketZ. Execute um teste e confirme a etiqueta aplicada.

### 12.4 Alterar quantidade de tentativas

Nos grupos `10A` e `10B-1`, a decisão usa comparação numérica `< 2`; para permitir **três** entradas antes do encaminhamento, alterar para `< 3`, revisar os textos das notas que mencionam duas tentativas, e testar o ciclo inteiro. É importante que os contadores iniciem em zero no grupo `09-1`.

### 12.5 Ativar futuramente o CRM — procedimento de mudança controlada

1. Confirmar que NocoDB contém dados suficientes, com estrutura, chaves, permissões e relacionamentos de vendedor corretamente preenchidos.
2. No gateway, testar busca real com massa autorizada para todos os quatro cenários (`ENCONTRADO`, `NAO_ENCONTRADO`, `DUPLICADO`, erro), sem expor CPF/CNPJ completo em logs.
3. No Typebot, testar o ramo isolado no editor (ou cópia de homologação): cálculo CPF/CNPJ, limite de tentativas, campos mapeados, classificação Ativo e regras de etiqueta.
4. Certificar que a fila 4 e as etiquetas retornadas pelo CRM existem no TicketZ.
5. Fazer backup e duplicar o fluxo antes da alteração de grafo.
6. No grupo `00 - Menu principal`, **mudar exclusivamente a conexão da opção `Quero Comprar`** do atual grupo `07 - Handoff Quero Comprar` para o grupo `07 - Quero Comprar: identificação de cliente`. Não apagar o ramo antigo: mantenha-o como rollback documentado.
7. Testar no canal real, com poucos atendimentos autorizados, monitorando notas, filas, comando de sessão e dados pessoais.
8. Atualizar a documentação, data, responsável e resultados dos testes; só então ampliar a ativação.

## 13. Pontos de atenção e não conformidades a acompanhar

1. **Conexão órfã no export:** a edge `q7am6rymhq84l7m955cylo6k` declara origem em bloco `y675be5y6lpgvmr3go0wc31r`, que **não aparece em nenhum grupo exportado**, e destino no menu `00`. Isso é um resíduo/possível referência quebrada e **não** é a desconexão intencional do CRM. Inspecionar visualmente antes de apagar qualquer coisa.
2. **Dois grupos com o número 07:** um é o atalho ativo, outro a identificação isolada. Não confundir no editor.
3. **Segredo em texto puro:** o JSON original embute o valor do header `X-CRM-Gateway-Key`. Restrinja acesso e considere rotação.
4. **Status da consulta sem código HTTP:** o grafo decide com base em `data.status`; o JSON não configura política explícita de timeout/retry do webhook. Testar falhas de conexão e respostas malformadas.
5. **Padrão `Ativo` incerto:** tudo que não se converte em `'true'` é tratado como cadastro inativo, incluindo nulo; considerar ramo separado de dado faltante.
6. **Ausência de autenticação do solicitante:** CPF/CNPJ matematicamente válido não prova que a pessoa é dona do cadastro. Evitar fornecer dados sensíveis ao usuário sem autenticação específica.
7. **Campos de CRM retornados mas pouco usados:** `cliente_tipo_pessoa` e `cliente_id` são recebidos, mas não são decisivos no grafo exportado; preservá-los pode facilitar expansão futura.
8. **Nota mascarada:** o handoff ativo do ramo CRM registra apenas os 4 últimos dígitos do documento; boa prática de minimização de dados. CPF/CNPJ completo é enviado ao gateway para consulta, então tratar logs e retenção conforme LGPD.
9. **Sem verificação de execução do TicketZ:** gerar variável `cmd_*` não confirma processamento; instrumentar logs/testes de ponta a ponta.
10. **Limites de escopo:** o export não comprova roteamento de proxy, instalação do NocoDB, tabelas, volume de banco, mecanismos de backup ou versão real do TicketZ. Adicionar anexo de infraestrutura após leitura de Compose, `.env` (redigido), configurações do gateway e inventário de contêineres.

## 14. Procedimento de restauração e diagnóstico por sintoma

| Sintoma | Possível causa | Verificação sugerida |
|---|---|---|
| Cliente vê `#{"action":...}` no chat | TicketZ/canal não interpreta comando | Confirmar versão/integrador, formato `#` e logs. |
| Cliente enviado à fila errada | `queueId` alterado ou remapeado no TicketZ | Conferir seção 7.2 e IDs na instância real. |
| Etiqueta não aparece | `tagId` incorreto ou vendedor não mapeado | Consultar retorno `TicketZTagID` e `VendedorMapeado`, existência da etiqueta. |
| CPF válido não encontra cadastro | Dado ausente no CRM, chave errada ou normalização distinta | Comparar payload e resposta de gateway em teste com dado autorizado. |
| Todo cadastro aparece como inativo | `data.cliente.Ativo` vazio/renomeado/valor diferente | Examinar contrato e conversão em grupo `11`. |
| Mesmo CPF retorna duplicado | Mais de uma linha/registro ou relacionamento duplicado | Corrigir unicidade/consulta no gateway/NocoDB, sem escolher registro arbitrário. |
| CRM indisponível | DNS, TLS, token, timeout, gateway ou banco | Testar camadas separadas; não desativar HTTPS para contornar. |
| Cliente fica preso na automação | `endSession` não processado ou bloco não emitido | Logs do TicketZ, ordem dos blocos e texto com variável. |

## 15. Observações finais e limites de confiabilidade

**Confirmado diretamente:** 33 grupos, 39 edges registradas, 45 variáveis, cinco saídas no menu, roteamento para filas/etiquetas, webhook e campos esperados, edição de mensagens e estrutura do ramo CRM. **Fornecido como contexto pelo solicitante:** o isolamento da seção CRM foi intencional para ativação futura com NocoDB povoado. **Não determinado:** sequência cronológica real de deploy, caminhos de instalação, nomes de containers, versões de servidor, portas físicas, usuários/senhas de serviços, tabelas NocoDB, implementação do gateway, alcance de rede e resultado de testes funcionais no ambiente em produção.

Se um técnico futuramente precisar preencher essas lacunas, deverá coletar inventário real **sem compartilhar segredos**. A documentação do Typebot e do gateway precisa evoluir junto com a versão do CRM para que alterações no esquema NocoDB não quebrem os mapeamentos em `data.cliente.*`.

---

## Apêndice A. Inventário de blocos técnicos e expressões especiais

O objetivo deste apêndice é permitir reconstruir a lógica sem depender apenas de imagens do editor. As expressões abaixo foram extraídas do JSON. **Valores de autenticação são sempre substituídos por marcadores.** Expressões são preservadas como exportadas, inclusive detalhes que possam exigir revisão posterior.

### 02 - Handoff Financeiro: Pagar

**Bloco 1, variável `cmd_pagar_tag`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'addTag',
  tagId: 3,
});
```

**Bloco 2, variável `cmd_pagar_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 5,
    status: 'pending',
    annotation: 'Triagem Atendimento > Financeiro > Pagar',
  },
});
```

**Bloco 3, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 03 - Handoff Financeiro: Receber

**Bloco 1, variável `cmd_receber_tag`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'addTag',
  tagId: 4,
});
```

**Bloco 2, variável `cmd_receber_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 5,
    status: 'pending',
    annotation: 'Triagem Atendimento > Financeiro > Receber',
  },
});
```

**Bloco 3, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 04 - Handoff Quero Vender

**Bloco 1, variável `cmd_vender_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 3,
    status: 'pending',
    annotation: 'Triagem Atendimento > Quero Vender',
  },
});
```

**Bloco 2, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 05 - Handoff Recursos Humanos

**Bloco 1, variável `cmd_rh_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 1,
    status: 'pending',
    annotation: 'Triagem Atendimento > Recursos Humanos',
  },
});
```

**Bloco 2, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 06 - Handoff SAC

**Bloco 1, variável `cmd_sac_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 2,
    status: 'pending',
    annotation: 'Triagem Atendimento > SAC',
  },
});
```

**Bloco 2, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 08 - Vendas: novo cliente / handoff

**Bloco 1, variável `cmd_novo_cliente_comercial_nota`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - contato informou que ainda não é cliente.',
  },
});
```

**Bloco 2, variável `cmd_novo_cliente_comercial_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 4,
    status: 'pending',
    annotation: 'Triagem Atendimento > Quero Comprar > Novo cliente',
  },
});
```

**Bloco 3, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 09-1 - Inicialização

**Bloco 1, variável `tentativas_consulta_documento`** (`isCode=true`):

```javascript
0
```

**Bloco 2, variável `tentativas_documento_invalido`** (`isCode=true`):

```javascript
0
```

### 10 - Vendas: validar documento

**Bloco 1, variável `doc_normalizado`** (`isCode=true`):

```javascript
String({{doc_informado}} ?? '').replace(/\D/g, '')
```

**Bloco 2, variável `doc_tipo`** (`isCode=true`):

```javascript
String({{doc_normalizado}} ?? '').length === 11
  ? 'CPF'
  : (
      String({{doc_normalizado}} ?? '').length === 14
        ? 'CNPJ'
        : 'INVALIDO'
    )
```

**Bloco 3, variável `doc_validacao`** (`isCode=true`):

```javascript
const documento = String({{doc_normalizado}} ?? '').replace(/\D/g, '');

// Rejeita sequências como 00000000000, 11111111111 etc.
if (/^(\d)\1+$/.test(documento)) {
  return 'INVALIDO';
}

// =========================
// CPF
// =========================
if (documento.length === 11) {
  const numeros = documento.split('').map(Number);

  let soma1 = 0;
  for (let i = 0; i < 9; i++) {
    soma1 += numeros[i] * (10 - i);
  }

  let digito1 = (soma1 * 10) % 11;
  if (digito1 === 10) digito1 = 0;

  if (digito1 !== numeros[9]) {
    return 'INVALIDO';
  }

  let soma2 = 0;
  for (let i = 0; i < 10; i++) {
    soma2 += numeros[i] * (11 - i);
  }

  let digito2 = (soma2 * 10) % 11;
  if (digito2 === 10) digito2 = 0;

  return digito2 === numeros[10]
    ? 'VALIDO'
    : 'INVALIDO';
}

// =========================
// CNPJ
// =========================
if (documento.length === 14) {
  const numeros = documento.split('').map(Number);

  const pesos1 = [5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2];

  let soma1 = 0;
  for (let i = 0; i < 12; i++) {
    soma1 += numeros[i] * pesos1[i];
  }

  let resto1 = soma1 % 11;
  let digito1 = resto1 < 2 ? 0 : 11 - resto1;

  if (digito1 !== numeros[12]) {
    return 'INVALIDO';
  }

  const pesos2 = [6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2];

  let soma2 = 0;
  for (let i = 0; i < 13; i++) {
    soma2 += numeros[i] * pesos2[i];
  }

  let resto2 = soma2 % 11;
  let digito2 = resto2 < 2 ? 0 : 11 - resto2;

  return digito2 === numeros[13]
    ? 'VALIDO'
    : 'INVALIDO';
}

return 'INVALIDO';
```

### 10A - Documento Inválido

**Bloco 1, variável `tentativas_documento_invalido`** (`isCode=true`):

```javascript
Number({{tentativas_documento_invalido}} ?? 0) + 1
```

**Bloco 2, variável `status_tentativa_documento_invalido`** (`isCode=true`):

```javascript
Number({{tentativas_documento_invalido}} ?? 0) < 2
  ? 'REPETIR'
  : 'ENCAMINHAR'
```

### 10A-2 - Limite de Documento Inválido

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
DOCUMENTO_INVALIDO_REPETIDO
```

**Bloco 2, variável `cmd_limite_doc_invalido`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - CPF/CNPJ inválido após duas tentativas.',
  },
});
```

### 10B - Documento Válido

**Bloco 1, variável `tentativas_documento_invalido`** (`isCode=true`):

```javascript
0
```

**Bloco 2, variável `ticketz_tag_id`** (`isCode=true`):

```javascript
0
```

**Bloco 3, variável `vendedor_mapeado`** (`isCode=true`):

```javascript
false
```

**Bloco 4, variável `vendedor_motivo`** (`isCode=true`):

```javascript
''
```

**Bloco 6, variável `decisao_tag_ticketz`** (`isCode=true`):

```javascript
const vendedorMapeado =
  String({{vendedor_mapeado}} ?? '')
    .trim()
    .toLowerCase() === 'true';

const tagId = Number({{ticketz_tag_id}} ?? 0);

return (
  vendedorMapeado &&
  Number.isInteger(tagId) &&
  tagId > 0
)
  ? 'APLICAR'
  : 'PULAR';
```

### 10B-1 - Documento Não Localizado

**Bloco 1, variável `tentativas_consulta_documento`** (`isCode=true`):

```javascript
Number({{tentativas_consulta_documento}} ?? 0) + 1
```

**Bloco 2, variável `status_tentativa_documento`** (`isCode=true`):

```javascript
Number({{tentativas_consulta_documento}} ?? 0) < 2
  ? 'REPETIR'
  : 'ENCAMINHAR'
```

### 10B-1A2 - Cliente Não Identificado

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
NAO_IDENTIFICADO
```

**Bloco 2, variável `cmd_limite_cadastro`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - CPF/CNPJ informado não localizado no CRM após duas tentativas.',
  },
});
```

### 11 - Cliente Encontrado por Documento

**Bloco 1, variável `cliente_status_cadastro`** (`isCode=true`):

```javascript
String({{cliente_ativo}} ?? '').toLowerCase() === 'true'
  ? 'ATIVO'
  : 'INATIVO'
```

### 11B - Cliente Inativo

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
IDENTIFICADO_INATIVO
```

**Bloco 2, variável `cmd_cliente_inativo`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - cliente localizado no CRM, porém com cadastro inativo.',
  },
});
```

### 11A - Cliente Ativo

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
IDENTIFICADO_ATIVO
```

### 11A-1 - Preparar Handoff Humano

**Bloco 1, variável `cmd_ticketz_tag`** (`isCode=true`):

```javascript
const decisao =
  String({{decisao_tag_ticketz}} ?? '')
    .trim()
    .toUpperCase();

const tagId = Number({{ticketz_tag_id}} ?? 0);

if (
  decisao !== 'APLICAR' ||
  !Number.isInteger(tagId) ||
  tagId <= 0
) {
  return '';
}

return '#' + JSON.stringify({
  action: 'addTag',
  tagId,
});
```

**Bloco 2, variável `cmd_ticketz_nota`** (`isCode=true`):

```javascript
const nome =
  String({{cliente_nome}} ?? '').trim() ||
  'Não informado';

const documento =
  String({{doc_normalizado}} ?? '')
    .replace(/\D/g, '');

const tipoInformado =
  String({{doc_tipo}} ?? '')
    .trim()
    .toUpperCase();

const tipoDocumento =
  ['CPF', 'CNPJ'].includes(tipoInformado)
    ? tipoInformado
    : documento.length === 11
      ? 'CPF'
      : documento.length === 14
        ? 'CNPJ'
        : 'Documento';

const documentoFinal =
  documento.length >= 4
    ? documento.slice(-4)
    : 'não informado';

const cidade =
  String({{cliente_cidade}} ?? '').trim();

const uf =
  String({{cliente_uf}} ?? '').trim();

const localizacao =
  [cidade, uf]
    .filter(Boolean)
    .join('/');

const vendedor =
  String({{cliente_vendedor}} ?? '').trim() ||
  'Não informado';

const situacao =
  String({{cliente_status_cadastro}} ?? '')
    .trim() ||
  'Não informada';
  
const motivoCodigo =
  String({{vendedor_motivo}} ?? '')
    .trim()
    .toUpperCase();

const motivosEtiqueta = {
  VENDEDOR_NAO_INFORMADO:
    'não aplicada - vendedor não informado',
  VENDEDOR_SEM_RELACIONAMENTO:
    'não aplicada - vendedor sem relacionamento no CRM',
  VENDEDOR_INATIVO:
    'não aplicada - vendedor inativo',
  TAG_NAO_CONFIGURADA:
    'não aplicada - ID da etiqueta não configurado',
};

const observacaoEtiqueta =
  motivosEtiqueta[motivoCodigo] ??
  (
    motivoCodigo
      ? `não aplicada - motivo ${motivoCodigo}`
      : null
  );

const linhas = [
  'Nota Interna - cliente identificado',
  `Nome: ${nome}`,
  `Documento: ${tipoDocumento} final ${documentoFinal}`,
  `Cidade/UF: ${localizacao || 'Não informado'}`,
  `Vendedor responsável: ${vendedor}`,
  `Situação cadastral: ${situacao}`,
];

if (observacaoEtiqueta) {
  linhas.push(
    `Etiqueta TicketZ: ${observacaoEtiqueta}`
  );
}

const conteudo = linhas.join('\n');

return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: conteudo,
  },
});
```

**Bloco 3, variável `cmd_ticketz_pendente`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 4,
    status: 'pending',
    annotation: 'Triagem Atendimento > Quero Comprar > Cliente identificado',
  },
});
```

**Bloco 4, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 10B-0A1 - Cadastro Duplicado

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
DADOS_DUPLICADOS
```

**Bloco 2, variável `cmd_cadastro_duplicado`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - foram localizados múltiplos cadastros para o documento informado.',
  },
});
```

### 10B-0B1 - Falha Técnica na Consulta

**Bloco 1, variável `resultado_identificacao`** (`isCode=false`):

```text
ERRO_TECNICO
```

**Bloco 2, variável `cmd_falha_tecnica`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'note',
  message: {
    content: 'Triagem Atendimento - falha técnica na consulta ao CRM; encaminhado para análise humana.',
  },
});
```

### 20 - Vendas: handoff comercial

**Bloco 1, variável `cmd_comercial_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 4,
    status: 'pending',
    annotation: 'Triagem Atendimento > Quero Comprar > Atendimento humano',
  },
});
```

**Bloco 2, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

### 07 - Handoff Quero Comprar

**Bloco 1, variável `cmd_quero_comprar_handoff`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'updateTicket',
  ticketData: {
    queueId: 4,
    status: 'pending',
    annotation: 'Triagem Atendimento > Quero Comprar',
  },
});
```

**Bloco 2, variável `cmd_ticketz_end_session`** (`isCode=true`):

```javascript
return '#' + JSON.stringify({
  action: 'endSession',
});
```

## Apêndice B. Referência bruta do webhook sem credencial

**Bloco:** `10B - Documento Válido` → `Webhook`.

```http
POST https://crm-api.ncloud.app.br/v1/clientes/documento
Content-Type: application/json
X-CRM-Gateway-Key: [REDACTED — OBTER NO GERENCIADOR DE SEGREDOS]

{"documento":"{{doc_normalizado}}"}
```

**Atenção:** o segredo original continua no JSON exportado fornecido pelo usuário. Esta documentação não o reproduz.

## Apêndice C. Glossário rápido

| Termo | Tradução prática |
|---|---|
| Grupo | Um conjunto de etapas dentro do Typebot. |
| Bloco | Pergunta, texto, decisão, variável ou chamada HTTP individual. |
| Variável | Campo temporário que guarda uma escolha, dado ou comando. |
| Webhook | Requisição HTTP de um serviço a outro. |
| Gateway | Camada intermediária que consulta/protege o CRM. |
| CRM | Cadastro e relacionamento com clientes. |
| NocoDB | Plataforma que hospeda/organiza os dados do CRM neste projeto. |
| TicketZ | Sistema que recebe e distribui atendimentos. |
| Fila (`queueId`) | Equipe/setor de destino do atendimento. |
| Etiqueta (`tagId`) | Marcador para classificar o atendimento. |
| `pending` | Estado que o comando solicita para entregar o ticket pendente de atendimento. |
| Nota interna | Informação para a equipe, não para a pessoa atendida. |
| Handoff | Transferência do robô para equipe humana. |
| CPF/CNPJ normalizado | Número sem pontos, barras, espaços ou traços. |
| Edge | Conexão que liga dois grupos/blocos no diagrama. |

**Fim do documento.**
