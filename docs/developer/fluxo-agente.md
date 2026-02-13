# Fluxo detalhado de funcionamento do agente (Agent Zero)

Este documento descreve, ponta a ponta, como o Agent Zero processa uma
mensagem: da entrada via API/UI, passando por contexto, montagem de
prompt, execução de ferramentas/extensões, até a resposta final e
persistência.

------------------------------------------------------------------------

## 1) Bootstrap e inicialização do runtime

O runtime web é iniciado em `run_ui.py`, responsável por:

-   inicializar servidor Flask (HTTP/UI);
-   subir servidor Socket.IO / ASGI para eventos em tempo real;
-   configurar middlewares (autenticação, CSRF, validação de origem);
-   registrar dinamicamente namespaces e handlers WebSocket.

Em paralelo, a configuração do agente é construída por
`initialize.initialize_agent()`, que:

1.  carrega as configurações atuais (settings);
2.  monta `ModelConfig` (chat, modelo utilitário, embedding e browser);
3.  compõe `AgentConfig` (perfil, memória, knowledge, MCP, browser
    headers etc.);
4.  aplica parâmetros de runtime (incluindo execução remota/SSH, quando
    configurada).

------------------------------------------------------------------------

## 2) Entrada da mensagem: API/UI → Contexto

Quando o usuário envia uma mensagem, o endpoint `python/api/message.py`:

1.  normaliza o payload (JSON ou multipart);
2.  salva anexos e gera caminhos internos;
3.  resolve o contexto via `ApiHandler.use_context()`;
4.  dispara extensões `user_message_ui`;
5.  registra a mensagem no log/queue do frontend;
6.  chama `context.communicate(UserMessage(...))`.

------------------------------------------------------------------------

## 3) Gestão de contexto e tarefa assíncrona

`AgentContext` mantém:

-   id, timestamps e metadados da sessão;
-   instância principal do agente (`agent0`);
-   referência da task (`DeferredTask`);
-   estado de pausa/intervenção;
-   stream atual;
-   log incremental para UI.

------------------------------------------------------------------------

## 4) Cadeia de processamento (\_process_chain)

1.  adiciona mensagem ao histórico;
2.  chama `agent.monologue()`;
3.  encaminha resultado ao agente superior (quando aplicável);
4.  executa extensões `process_chain_end`.

------------------------------------------------------------------------

## 5) Ciclo principal (monologue)

Fluxo:

1.  inicializa `LoopData`;
2.  executa `monologue_start`;
3.  inicia loop de iteração;
4.  executa `message_loop_start`;
5.  monta prompt;
6.  chama modelo com streaming;
7.  salva resposta;
8.  processa ferramentas;
9.  encerra se `break_loop=True`;
10. executa `message_loop_end`;
11. executa `monologue_end`.

------------------------------------------------------------------------

## 6) Montagem de Prompt

Composição:

-   System prompt;
-   Histórico formatado;
-   Extras dinâmicos (memória, skills, metadados).

------------------------------------------------------------------------

## 7) Execução de Ferramentas

1.  Parse JSON para identificar tool;
2.  Resolve MCP ou ferramenta local;
3.  Executa hooks before/after;
4.  Finaliza se necessário.

Tool finalizadora padrão: `response`.

------------------------------------------------------------------------

## 8) Extensões

Sistema baseado em:

-   múltiplos paths;
-   override por precedência;
-   ordem lexical (*10*, *20*, *90*).

------------------------------------------------------------------------

## 9) Persistência e Continuidade

-   Compressão assíncrona de histórico;
-   Persistência de chat;
-   Processamento de fila.

------------------------------------------------------------------------

Documento gerado automaticamente em: 2026-02-13T11:19:36.056764 UTC