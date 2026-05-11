# 🪒 Bot de Atendimento — Barbearia Alpha Prime
### Workflow n8n · Versão 8

Bot de atendimento automático via WhatsApp com IA, agendamento completo (criar, atualizar, cancelar), transcrição de áudio, memória de contexto por cliente e integração com Google Calendar + Google Sheets + PostgreSQL.

---

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Pré-requisitos](#pré-requisitos)
- [Credenciais Necessárias](#credenciais-necessárias)
- [Estrutura do Banco de Dados](#estrutura-do-banco-de-dados)
- [Blocos do Fluxo](#blocos-do-fluxo)
- [Configurações Obrigatórias](#configurações-obrigatórias)
- [Como Instalar](#como-instalar)

---

## Visão Geral

O fluxo recebe mensagens do WhatsApp via **Evolution API**, processa o conteúdo com IA (Groq/LLaMA 3.3), gerencia sessões no PostgreSQL e responde ao cliente automaticamente. O bot sabe identificar saudações, tirar dúvidas sobre a barbearia e conduzir um agendamento completo sem intervenção humana.

```
WhatsApp → Evolution API (Webhook) → n8n → Agente IA → WhatsApp
                                        ↕
                              PostgreSQL + Google Sheets + Google Calendar
```

---

## Pré-requisitos

- n8n (self-hosted ou cloud)
- Evolution API configurada e apontando para o webhook do n8n
- PostgreSQL acessível pelo n8n
- Conta Groq com acesso ao modelo `llama-3.3-70b-versatile`
- Conta AssemblyAI (para transcrição de áudio)
- Google Sheets e Google Calendar com OAuth configurado no n8n

---

## Credenciais Necessárias

| Serviço | Onde configurar no n8n |
|---|---|
| PostgreSQL | Credentials → Postgres |
| Groq API | Credentials → Groq |
| AssemblyAI | **⚠️ Atualmente hardcoded nos nós HTTP — mover para Credential** |
| Google Sheets | Credentials → Google Sheets OAuth2 |
| Google Calendar | Credentials → Google Calendar OAuth2 |
| Evolution API | Chave enviada no próprio payload do webhook (campo `apikey`) |

> **⚠️ Atenção:** A chave da AssemblyAI está inserida diretamente nos nós `HTTP Request4`, `HTTP Request5` e `HTTP Request6`. Antes de compartilhar o arquivo JSON, substitua-a por uma Credential do n8n para evitar exposição.

---

## Estrutura do Banco de Dados

O fluxo cria e utiliza as seguintes tabelas no PostgreSQL. Algumas são criadas automaticamente na primeira execução dos nós de setup:

| Tabela | Descrição |
|---|---|
| `public.users` | Cadastro de clientes (external_id = remoteJid do WhatsApp) |
| `public.sessions` | Sessões ativas por usuário (estado + intenção) |
| `public.user_contexts` | Contextos de conversa por usuário |
| `public.temp_context` | Buffer temporário de mensagens por JID (usado/não usado) |
| `public.session_context` | Contexto consolidado da sessão atual |
| `public.agendamentos_calendar` | Agendamentos com `event_id` do Google Calendar |
| `public.agendamentos_rascunho` | Rascunhos de agendamento incompletos |
| `servicos_duracao` | Tabela de serviços com duração em minutos e preço |

---

## Blocos do Fluxo

O workflow é dividido em blocos identificados por Sticky Notes no canvas do n8n. Abaixo a documentação de cada bloco e seus nós.

---

### 🟦 BLOCO 0 — SETUP DO BANCO

> Nós executados manualmente uma única vez para preparar o ambiente no PostgreSQL.

---

#### `criar tabela temp_context`
Cria a tabela `public.temp_context` com as colunas `jid`, `messages` (JSONB), `last_message_at`, `used` e `used_at`. Inclui constraints que garantem consistência entre os campos `used` e `used_at`.

#### `criar_index`
Cria dois índices na tabela `temp_context`: um para consultas de contextos ativos (`used = FALSE`) e outro para limpeza de registros usados.

#### `criar_table` *(setup de agendamentos)*
Cria a tabela `agendamentos_calendar` com as colunas de dados do agendamento, `event_id` do Google Calendar e campos de auditoria.

#### `criar_tabela_De_user`
Cria a tabela `public.users` com `external_id` (JID do WhatsApp), phone, push_name, platform e instance.

#### `criar_sessão2`
Cria a tabela `public.sessions` com `user_id`, `state`, `intent` e timestamps.

#### `criar_tabela_de_contexto1`
Cria a tabela `public.user_contexts` com `user_id`, `context` (TEXT), `is_used` e timestamps de criação e última interação.

#### `servicos_duracao` *(setup de serviços)*
Cria a extensão `unaccent`, a tabela `servicos_duracao` e insere os 4 serviços padrão com duração e preço:
- Corte masculino — 60 min — R$ 45,00
- Barba — 40 min — R$ 35,00
- Corte + Barba — 90 min — R$ 75,00
- Sobrancelha — 20 min — R$ 15,00

> ⚙️ **Para personalizar:** edite os valores diretamente na query SQL deste nó antes de executar.

---

### 🟦 BLOCO 1 — ENTRADA E SEGURANÇA

> Recebe o webhook da Evolution API, filtra mensagens inválidas e aplica proteções básicas.

---

#### `Webhook de segurança`
Ponto de entrada do fluxo. Recebe o payload POST da Evolution API com todos os dados da mensagem WhatsApp (instância, JID, tipo de mensagem, conteúdo, timestamp, apikey e server_url).

#### `Ignorar Grupos`
Verifica se o `remoteJid` termina em `@g.us`. Se sim, descarta a execução — o bot só responde a conversas individuais.

#### `Rate Limit`
Extrai o JID e o timestamp atual (`Date.now()`), preparando os campos `_rl_jid` e `_rl_now` para controle de cadência de mensagens.

#### `Passou Rate Limit?`
Verifica o campo `_rateLimited`. Se verdadeiro, a execução é interrompida. Se falso (ou ausente), o fluxo continua.

> ⚠️ **Bug conhecido:** o campo `_rateLimited` nunca é populado. O rate limit está inativo na versão atual.

---

### 🟦 BLOCO 2 — IDENTIFICAÇÃO DO USUÁRIO

> Busca ou cria o registro do cliente no banco de dados.

---

#### `welcomesent`
Extrai o JID da mensagem e verifica no payload se o campo `_pg_welcome_sent` está marcado como `true`, populando o campo `welcomeAlreadySent` para uso posterior.

#### `welcome`
Detecta se a mensagem atual é uma saudação usando uma lista com mais de 80 variações de cumprimentos em português (oi, eai, salve, bom dia etc.). Popula `isGreeting: true/false` e `intent: welcome/unknown`.

#### `buscar_usuario`
Consulta `public.users` pelo `external_id` (remoteJid) para verificar se o cliente já está cadastrado.

#### `If4`
Verifica se o resultado da busca retornou um `id`. Se sim, o usuário existe e segue para buscar a sessão. Se não, segue para criação.

#### `criar_user_Caso_não_exista`
Insere o cliente em `public.users` com upsert (`ON CONFLICT DO UPDATE`), garantindo que dados como `push_name` sejam sempre atualizados mesmo para clientes existentes.

#### `criar_sessão`
Cria um novo registro em `public.sessions` para o cliente, com `state: 'start'` e a mensagem atual como `intent` inicial.

#### `buscar_sessão`
Busca a sessão mais recente do cliente em `public.sessions` ordenando por `created_at DESC`.

#### `buscar_contexto`
Busca o contexto mais recente não utilizado em `public.user_contexts` para o usuário atual.

#### `criar_tabela_de_contexto`
Insere um novo registro em `public.user_contexts` com o conteúdo da mensagem atual, marcado como `is_used = false`.

#### `Edit Fields1`
Nó Set vazio usado como ponto de passagem para unificar os dados após a criação do contexto.

---

### 🟦 BLOCO 3 — CONTEXTO TEMPORÁRIO

> Gerencia o buffer de mensagens por JID na tabela `temp_context`.

---

#### `select`
Busca o registro ativo (`used = FALSE`) de `temp_context` para o JID atual. Se não existir, retorna um array vazio via `UNION ALL` com fallback.

#### `Edit Fields2`
Prepara os dados do contexto recuperado para ser passado ao próximo nó.

#### `Execute a SQL query1`
Marca o registro de `temp_context` como `used = TRUE` e registra o `used_at`. Isso evita que o mesmo contexto seja lido duas vezes.

#### `mandar contexto`
Nó Set de passagem que consolida os dados do contexto para uso nos agentes IA.

---

### 🟦 BLOCO 4 — NORMALIZAÇÃO DA MENSAGEM

> Identifica o tipo de mensagem recebida e extrai o texto de forma padronizada.

---

#### `Dados`
Nó Set principal que mapeia todos os campos relevantes do payload do webhook: `instancia`, `remotejid`, `fromme`, `pushname`, `conversation`, `messagetype`, `datetime`, `server_url`, `apikey`, `isGreeting` e `welcomeAlreadySent`.

#### `é minha mensagem?`
Verifica se `fromme == "true"`. Se sim, a mensagem foi enviada pelo próprio número do bot e é descartada (sem resposta).

#### `tipo de mensagem`
Switch que roteia a execução com base no `messageType`:
- `conversation` → `msg-conversation`
- `ephemeralMessage` → `msg-ephemeral`
- `extendedtextMessage` → `msg-extended`
- `audioMessage` → `msg-audio`
- `documentMessage` (PDF) → `pdf-message`

#### `msg-conversation`
Extrai o texto de `message.conversation` e o padroniza no campo `message`.

#### `msg-ephemeral`
Extrai o texto de mensagens efêmeras (modo "visualizar uma vez") via `ephemeralMessage.message.extendedTextMessage.text`.

#### `msg-extended`
Extrai o texto de mensagens com formatação rica via `extendedTextMessage.text`.

#### `Merge2`
Merge de 3 entradas (conversation, ephemeral, extended) que consolida o texto normalizado antes de seguir para o processador principal.

#### `Code in JavaScript8`
Processador central de texto. Normaliza a mensagem recebida, detecta o agente correto a acionar (welcome, atendimento ou agendamento) e popula campos como `agent`, `intent`, `textoNormalizado` e `client_message`.

---

### 🟦 BLOCO 4B — PROCESSAMENTO DE ÁUDIO

> Transcreve mensagens de voz usando AssemblyAI.

---

#### `msg-audio`
Extrai o `base64` do áudio da mensagem recebida.

#### `Code in JavaScript1`
Converte o base64 do áudio para binário, preparando o arquivo para upload.

#### `Convert to File`
Converte os dados binários em um arquivo de áudio reconhecido pelo n8n.

#### `HTTP Request4`
Faz upload do arquivo de áudio para o endpoint `https://api.assemblyai.com/v2/upload`, recebendo a `upload_url`.

#### `HTTP Request5`
Envia a `upload_url` para `https://api.assemblyai.com/v2/transcript` para iniciar a transcrição com detecção automática de idioma.

#### `Wait`
Aguarda 10 segundos antes de verificar o resultado da transcrição (polling simples).

#### `HTTP Request6`
Consulta o status da transcrição pelo `id` retornado na etapa anterior. Retorna o texto transcrito quando pronto.

#### `If1`
Verifica se o campo `text` da resposta está vazio. Se sim, a transcrição falhou. Se não, o texto segue para os agentes.

#### `mnsg pro cliente caso a transcrição retorne falha3`
Prepara uma mensagem de erro amigável para o cliente informando que não foi possível entender o áudio.

#### `prosseguir e enviar agente1`
Normaliza o texto transcrito e o encaminha para o fluxo principal como se fosse uma mensagem de texto comum.

---

### 🟦 BLOCO 5 — ROTEAMENTO PRINCIPAL

> Decide qual agente vai responder com base na intenção detectada.

---

#### `If7`
Verifica se `welcomeAlreadySent == false` E `isGreeting == true`. Se ambas as condições forem verdadeiras, aciona o Agente Saudação para enviar a mensagem de boas-vindas.

#### `If8`
Verifica se o agente detectado é `Welcome_agent` E a intenção não é `schedule`. Roteia para o Agente de Saudação ou para o Agente de Atendimento geral.

#### `Merge6`
Merge de 3 entradas que consolida os caminhos após a identificação do usuário (sessão nova, sessão existente e contexto) antes de seguir para o roteamento de intenção.

#### `Merge7`
Merge de passagem que unifica a saída do Agente Saudação e do Agente Atendimento antes de seguir para o processamento da resposta.

---

### 🟦 BLOCO 6 — AGENTES DE ATENDIMENTO GERAL

---

#### `Agente Saudação`
Agente IA especializado em receber o primeiro contato do cliente. Tom descontraído e informal. Responde saudações, apresenta a barbearia brevemente e detecta se o cliente tem alguma intenção além de cumprimentar. Usa memória de sessão por JID (`Simple Memory`).

**Modelo:** `llama-3.3-70b-versatile` via Groq  
**Memória:** `Simple Memory` (janela de 10 mensagens, chave = remoteJid)

#### `Simple Memory`
Buffer de memória de curto prazo ligado ao `Agente Saudação`. Mantém as últimas 10 mensagens da conversa indexadas pelo JID do cliente.

#### `Agente Atendimento`
Agente IA principal para atendimento geral. Responde perguntas sobre horários, serviços, preços e endereço. Detecta intenção de agendamento e instrui o fluxo a mudar de rota. Tom descontraído, aceita gírias e aliases de horário (ex: "meio dia" → 12:00). Usa `Simple Memory3`.

**Modelo:** `llama-3.3-70b-versatile` via Groq (`Groq Chat Model1`)  
**Memória:** `Simple Memory3` (janela de 10 mensagens, chave = remoteJid)

#### `Simple Memory3`
Buffer de memória do `Agente Atendimento`. Mantém contexto das últimas 10 mensagens por JID.

#### `Groq Chat Model1`
LLM conectado ao `Agente Atendimento`. Modelo `llama-3.3-70b-versatile`.

#### `Parse Agent JSON`
Tenta extrair um JSON estruturado da resposta do agente. Se encontrar `acao` e `isScheduleAction`, sinaliza que o cliente quer agendar e o fluxo deve mudar de rota.

---

### 🟦 BLOCO 7 — ROTEAMENTO DE AGENDAMENTOS

---

#### `Resetar Sessão se Novo Tópico`
Verifica se a ação atual é de agendamento (`isScheduleAction`). Se não for, sinaliza que a sessão de agendamento deve ser resetada antes de continuar.

#### `Extrair Dados Iniciais`
Parser inteligente que analisa a mensagem do cliente e tenta detectar previamente: serviço (incluindo gírias como "tapa no visual"), data (hoje, amanhã, dias da semana, "semana que vem", DD/MM), hora (aliases como "meio dia", "1 da tarde") e nome (via pushName do WhatsApp). Esses dados pré-detectados são passados ao agente especializado para reduzir o número de perguntas.

#### `Switch Intenção Agendamento`
Switch que roteia para o agente correto com base no campo `acao`:
- `iniciar_agendamento` → **Agente Criar Agendamento**
- `iniciar_atualizacao` → **Agente Atualizar Agendamento**
- `iniciar_cancelamento` → **Agente Deletar Agendamento**

#### `node evita erro`
Nó Set de segurança que garante que todos os campos obrigatórios (`jid`, `pushName`, `instance`, `acao`, `context_text`, campos `_parcial_*`) estejam presentes antes de entrar no Agente Criar, evitando erros de expressão.

---

### 🟦 BLOCO 8 — CRIAR AGENDAMENTO

---

#### `Agente Criar Agendamento`
Agente IA especializado em coletar os dados necessários para um novo agendamento: nome do cliente, serviço desejado, data e horário. Usa os dados pré-detectados pelo `Extrair Dados Iniciais` para evitar perguntas redundantes. Integrado com a ferramenta `marcar_agendamento` (Google Calendar Tool). Memória isolada por `criar_{jid}`.

**Ferramenta:** `marcar_agendamento` (Google Calendar)  
**Modelo:** Groq LLaMA 3.3 70B

#### `marcar_agendamento`
Tool do Google Calendar conectada ao `Agente Criar Agendamento`. Permite que o agente crie eventos diretamente no calendário com summary (nome + serviço), start e end calculados dinamicamente.

#### `Parse Agente Criar Agendamento` *(mensagem do agente1)*
Extrai o JSON da resposta do agente. Se contiver `acao: "agendar"` com todos os campos preenchidos, marca `concluido: true`. Caso contrário, `concluido: false` e repassa o texto da resposta para o cliente.

#### `Concluído? Criar` *(If)*
Verifica `concluido`. Se `true`, segue para validação dos campos. Se `false`, envia a resposta parcial do agente para o cliente via `Message Splitter`.

#### `Preparar Retry Agente Criar`
Quando o agendamento não foi concluído, prepara os dados de rascunho (campos parciais coletados até o momento) e a mensagem de resposta para envio ao cliente.

#### `Validar Campos Criar` *(Campos OK?)*
Verifica se `nome_cliente`, `data_agendamento` e `hora_agendamento` estão todos preenchidos. Gera uma mensagem listando o que ainda falta caso algum campo esteja vazio.

#### `Campos OK?`
IF que separa os fluxos: campos completos seguem para verificação de disponibilidade; campos incompletos seguem para salvar rascunho e avisar o cliente.

#### `JID Válido? (Rascunho)`
Verifica se o `jid` está presente antes de tentar salvar o rascunho, evitando upserts com chave nula.

#### `Salvar Rascunho no BD`
Upsert na tabela `agendamentos_rascunho` por JID. Usa `COALESCE` para preservar campos já preenchidos em interações anteriores — o cliente não precisa repetir o que já informou.

#### `Retry Campos Faltando`
Prepara a mensagem informando ao cliente quais dados ainda faltam para completar o agendamento.

#### `Msg Campos Faltando`
Monta a mensagem final com os campos faltantes para envio ao cliente.

#### `Verificar Disponibilidade`
Query SQL com `OVERLAPS` que verifica se o horário solicitado conflita com algum agendamento existente na tabela `agendamentos_calendar`. Usa a tabela `servicos_duracao` para calcular a duração dinâmica de cada serviço.

#### `Horário Disponível?`
IF que separa: `total == 0` (horário livre) → cria o agendamento; `total > 0` (horário ocupado) → busca horários alternativos.

#### `Buscar Horários Disponíveis`
Query SQL que retorna todos os slots livres no dia solicitado, considerando a duração do serviço pedido e os agendamentos existentes. Grade padrão: 08:00 às 17:30 com intervalos de 30 min.

#### `Msg Horário Ocupado`
Monta a mensagem informando que o horário está ocupado e listando os slots disponíveis no dia.

#### `Enviar Msg Horário Ocupado`
Envia a mensagem de horário ocupado via Evolution API (`/message/sendText/{instance}`).

#### `Sheets - Criar Agendamento`
Insere o agendamento confirmado na planilha Google Sheets (aba "Agendamentos") com Nome, Telefone, Serviço (campo Email por bug de mapeamento), DataAgendada e status "confirmado".

> ⚙️ **Para personalizar:** altere o ID da planilha no nó e corrija o mapeamento do campo `Email` → `Serviço`.

#### `Msg Confirmação Criar`
Monta a mensagem de confirmação de agendamento para o cliente.

#### `Limpar Sessão Agendamento`
Marca a sessão de agendamento como encerrada, preparando os campos para a limpeza no banco de dados.

#### `Merge Sheets Final`
Merge de passagem que unifica os fluxos de criar, atualizar e cancelar antes de enviar a mensagem final.

---

### 🟦 BLOCO 9 — ATUALIZAR AGENDAMENTO

---

#### `Agente Atualizar Agendamento`
Agente IA especializado em coletar os dados para reagendamento: novo serviço, nova data e novo horário. Busca o agendamento existente do cliente para confirmação antes de atualizar.

#### `Sheets - Atualizar Agendamento1`
Atualiza o registro na planilha Google Sheets com os novos dados do agendamento.

#### `Buscar eventId para Atualizar`
Consulta `agendamentos_calendar` pelo JID para encontrar o `event_id` do Google Calendar a ser atualizado.

#### `Msg Confirmação Atualizar`
Monta a mensagem de confirmação de reagendamento para o cliente.

---

### 🟦 BLOCO 10 — CANCELAR AGENDAMENTO

---

#### `Agente Deletar Agendamento`
Agente IA especializado em conduzir o cancelamento de agendamento. Confirma com o cliente antes de executar.

#### `Agente Confirmar Intenção`
Agente IA que entra em ação quando o cliente sinaliza que não vai poder comparecer (ex: "não vou poder ir hoje", "tive um imprevisto"). Pergunta de forma empática se o cliente quer **reagendar** ou **cancelar**.

#### `Parse Agente Confirmar Intenção`
Extrai o JSON `{"intencao":"reagendar"}` ou `{"intencao":"cancelar"}` da resposta do agente. Se não conseguir parsear, marca `respondeu: false`.

#### `Switch Confirmar Intenção`
Switch que roteia:
- `reagendar` → `Agente Atualizar Agendamento`
- `cancelar` → `Agente Deletar Agendamento`
- sem resposta clara → volta para `Agente Confirmar Intenção` (re-pergunta)

> ⚠️ **Bug conhecido:** o output "sem resposta clara" cria um loop sem saída de escape. Adicione um contador de tentativas para evitar loop infinito.

#### `Sheets - Cancelar Agendamento1`
Remove ou marca como cancelado o registro na planilha Google Sheets.

#### `Buscar eventId para Deletar`
Consulta `agendamentos_calendar` pelo JID para obter o `event_id` do Google Calendar a ser deletado.

#### `Msg Confirmação Cancelar`
Monta a mensagem de confirmação de cancelamento para o cliente.

---

### 🟦 BLOCO 11 — ENVIO DA RESPOSTA

---

#### `Message Splitter`
Divide respostas longas (acima de 1.500 caracteres) em partes menores para envio sequencial no WhatsApp. O corte é inteligente: tenta quebrar em parágrafo duplo → quebra de linha → fim de frase → espaço.

#### `Wait Split`
Aguarda 1 segundo entre cada parte da mensagem dividida, simulando digitação natural.

#### `Code in JavaScript9`
Extrai o campo `message` ou `output` do item atual para garantir que o texto correto seja enviado.

#### `Enviar Mensagem` *(HTTP Request final)*
Envia a resposta ao cliente via Evolution API usando o endpoint `/message/sendText/{instance}`. Usa `server_url` e `apikey` vindos do próprio payload do webhook.

---

### 🟦 BLOCO 12 — CONTEXTO PÓS-RESPOSTA

> Salva as mensagens trocadas no banco para uso futuro como contexto.

---

#### `Execute a SQL query2`
Upsert em `temp_context`: insere ou concatena (`||`) a nova mensagem ao array JSONB existente para o JID, mantendo um histórico contínuo da conversa.

#### `Execute a SQL query3`
Busca o contexto atual de `temp_context` para o JID, usado para montar o resumo de contexto que será injetado na próxima execução.

#### `Code in JavaScript5`
Limita o contexto às últimas 5 mensagens e formata o bloco de texto `context_text` que será inserido no prompt do agente na próxima interação.

#### `onde ta ec`
Upsert em `session_context` com o histórico de mensagens consolidado da sessão atual.

---

## Configurações Obrigatórias

Antes de publicar o workflow para um cliente, ajuste os seguintes pontos:

1. **AssemblyAI Key** — Mova a chave dos nós `HTTP Request4/5/6` para uma Credential do n8n
2. **Google Sheets ID** — Substitua o ID da planilha nos nós `Sheets - Criar/Atualizar/Cancelar Agendamento`
3. **Google Calendar** — Selecione o calendário correto no nó `marcar_agendamento` (hoje aponta para `primary`)
4. **Informações da barbearia** — Edite o `systemMessage` dos agentes `Agente Saudação` e `Agente Atendimento` com endereço, horários e serviços reais do cliente
5. **Tabela `servicos_duracao`** — Execute o nó de setup com os serviços e preços corretos do cliente
6. **Rate Limit** — Implementar a lógica de persistência do campo `_rateLimited` no PostgreSQL para o rate limit funcionar de fato

---

## Como Instalar

1. Importe o arquivo `workflow_v8_overlaps.json` no n8n
2. Configure todas as Credentials listadas acima
3. Ative o workflow
4. Execute manualmente os nós de **BLOCO 0 — SETUP DO BANCO** na ordem:
   - `criar tabela temp_context` → `criar_index`
   - `criar_tabela_De_user` → `criar_sessão2` → `criar_tabela_de_contexto1` → `criar_table`
   - `servicos_duracao`
5. Configure o webhook da Evolution API para apontar para a URL do nó `Webhook de segurança`
6. Teste com uma mensagem real no WhatsApp

---

*Documentação gerada com base na v8 do workflow · Barbearia Alpha Prime*