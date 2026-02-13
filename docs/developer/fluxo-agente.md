# Fluxo detalhado de funcionamento do agente (Agent Zero)

> Versão consolidada em português do Brasil para facilitar merge e revisão na PR.

Este documento descreve, de ponta a ponta, como o Agent Zero processa uma mensagem: desde a entrada via API/UI, passando por contexto, montagem de prompt e execução de ferramentas/extensões, até a resposta final e a persistência.

## 1) Bootstrap e inicialização do runtime

O runtime web é iniciado em `run_ui.py`, que configura:

- servidor Flask para HTTP/UI;
- servidor Socket.IO/ASGI para eventos em tempo real;
- middlewares de autenticação, CSRF e validação de origem;
- descoberta dinâmica de namespaces WebSocket e handlers.

Em paralelo, a configuração do agente é construída por `initialize.initialize_agent()`, que:

1. carrega as configurações atuais;
2. monta `ModelConfig` para chat, modelo utilitário, embedding e browser;
3. compõe `AgentConfig` (perfil, memória, knowledge, MCP, cabeçalhos do browser etc.);
4. aplica parâmetros de runtime (incluindo execução remota/SSH, quando configurada).

## 2) Entrada da mensagem: API/UI → contexto

Quando o usuário envia uma mensagem, o endpoint `python/api/message.py`:

1. normaliza o payload (JSON ou multipart);
2. salva anexos enviados e gera caminhos internos;
3. resolve o contexto com `ApiHandler.use_context()` (reutiliza contexto existente ou cria um novo com `initialize_agent()`);
4. dispara extensões de `user_message_ui` para pré-processar texto/anexos;
5. registra a mensagem no log/fila do frontend;
6. chama `context.communicate(UserMessage(...))`.

## 3) Gerenciamento de contexto e tarefa assíncrona

`AgentContext` é o “container” da conversa. Ele mantém:

- `id`, timestamps e metadados da sessão;
- `agent0` (instância principal do agente);
- referência de tarefa em execução (`DeferredTask`);
- estado de pausa/intervenção e stream atual;
- log incremental para a UI.

Ao receber uma mensagem, `communicate()` faz:

- se já existe tarefa rodando: transforma a nova entrada em **intervenção** (broadcast para agente atual/superiores);
- se não existe tarefa: inicia `_process_chain()` em thread assíncrona via `DeferredTask`.

## 4) Cadeia de processamento (`_process_chain`)

A cadeia de processamento encapsula o ciclo de alto nível:

1. adiciona a mensagem do usuário ao histórico (`hist_add_user_message`);
2. chama `agent.monologue()` para executar o loop de raciocínio/ação;
3. se houver agente superior na hierarquia, encaminha o resultado como retorno de ferramenta (`call_subordinate`) para esse superior;
4. ao final, executa extensões de `process_chain_end` (ex.: processamento de fila).

Isso permite suporte nativo ao modelo hierárquico multiagente, em que um agente subordinado retorna saída para seu superior até chegar ao topo.

## 5) Ciclo principal do agente (`monologue`)

`Agent.monologue()` executa em loop contínuo, com tratamento de intervenção e retentativas:

1. inicializa `LoopData` da iteração;
2. executa extensões `monologue_start`;
3. entra no loop interno de mensagens, incrementando `iteration`;
4. executa extensões `message_loop_start`;
5. monta o prompt com `prepare_prompt()`;
6. executa extensões `before_main_llm_call`;
7. chama o modelo (`call_chat_model`) com callbacks de streaming;
8. salva a resposta no histórico;
9. tenta extrair chamada de ferramenta (`process_tools`);
10. se a ferramenta retornar `break_loop=True` (como `response`), encerra e devolve a resposta final;
11. executa extensões `message_loop_end`;
12. ao sair, executa `monologue_end`.

Erros reparáveis viram mensagens de warning para o próprio LLM tentar corrigir. Erros críticos passam por retentativa controlada e, se persistirem, encerram a execução.

## 6) Montagem de prompt e contexto dinâmico

`prepare_prompt()` combina três blocos:

1. **System prompt** (`get_system_prompt`) vindo de extensões `system_prompt`;
2. **Histórico** convertido para o formato do modelo;
3. **Extras** temporários/persistentes (memórias, skills carregadas, metadados).

A ordem dos hooks permite composição incremental. Exemplo típico:

- `system_prompt/_10_system_prompt.py` injeta prompt base, ferramentas, MCP, skills disponíveis, segredos e contexto de projeto;
- `system_prompt/_20_behaviour_prompt.py` injeta regras dinâmicas de comportamento (`behaviour.md`);
- `message_loop_prompts_after/*` injeta memória recuperada, skills carregadas, data/hora, informações do agente e outros extras.

O prompt final também é salvo como “janela de contexto atual”, com contagem aproximada de tokens.

## 7) Execução de ferramentas (tools)

Após a resposta do LLM, `process_tools()`:

1. parseia JSON “sujo” para identificar `tool_name` e `tool_args`;
2. tenta resolver ferramenta MCP;
3. se não houver MCP, faz fallback para ferramenta local via `get_tool()` (carregamento dinâmico por perfil/agente);
4. executa hooks:
   - `tool.before_execution()`
   - extensões `tool_execute_before`
   - `tool.execute()`
   - extensões `tool_execute_after`
   - `tool.after_execution()`
5. se a resposta da ferramenta sinaliza `break_loop`, retorna a mensagem final.

A ferramenta `response` (`python/tools/response.py`) é a finalizadora padrão: devolve texto ao usuário e encerra o loop.

## 8) Extensões: “espinha dorsal” de customização

O mecanismo `call_extensions()`:

- localiza extensões em múltiplos paths (default + overrides por perfil/projeto/usuário);
- aplica deduplicação por nome de arquivo (o primeiro encontrado vence, habilitando override);
- executa extensões em ordem lexical do arquivo (prefixos `_10_`, `_20_` etc. controlam a sequência).

Com isso, quase todo o comportamento do agente é plugável sem alterar o núcleo.

## 9) Persistência, compressão e continuidade

Ao final das iterações:

- `message_loop_end/_10_organize_history.py` dispara compressão assíncrona do histórico;
- `message_loop_end/_90_save_chat.py` persiste chat temporário (exceto contexto BACKGROUND);
- `process_chain_end/_50_process_queue.py` despacha a próxima mensagem enfileirada, permitindo “fila de prompts”.

Esse desenho sustenta conversas longas com compressão incremental de contexto e continuidade entre mensagens.

## 10) Resumo do fluxo em sequência

1. UI/API recebe mensagem.
2. Handler resolve/cria contexto.
3. Mensagem entra no histórico.
4. `monologue` inicia iteração.
5. Prompt é montado (system + history + extras).
6. LLM responde em stream.
7. Agente executa a ferramenta solicitada.
8. Ferramentas/extensões atualizam logs, estado e memória.
9. Ferramenta `response` encerra loop com resposta final.
10. Histórico é comprimido/persistido e a fila é processada.

## 11) Pontos de extensão mais importantes para evoluir o fluxo

- `python/extensions/system_prompt/*`: altera instruções base e contexto sistêmico.
- `python/extensions/message_loop_prompts_after/*`: injeta memória/skills/projeto por iteração.
- `python/extensions/tool_execute_before|after/*`: intercepta argumentos/retornos de tools.
- `agents/<perfil>/prompts`, `agents/<perfil>/tools`, `agents/<perfil>/extensions`: customização por perfil sem fork do core.

---

Se você quiser, o próximo passo pode ser um **diagrama de sequência (mermaid)** com esse fluxo (UI → API → AgentContext → Monologue → Tool → Response), incluindo os pontos de extensão por etapa.
