# Metodo: Anatomia do Agente Claude Code + Tmux

## Visao Geral
Como funciona, peca por peca, um agente de IA autonomo rodando Claude Code dentro de tmux numa VPS, com bot Telegram externo, memoria em arquivos markdown e subagentes especializados.

## Stack
- VPS Linux Ubuntu 22.04 (recomendado: Hostinger, cupom AVALANCHE)
- tmux: sessao persistente sempre viva
- Node.js 20+ e npm (pra rodar o Claude CLI)
- Python 3.10+ e pip (pra rodar o bot do Telegram)
- ffmpeg (pra processar audio in/out)
- Conta Anthropic (modelo: Claude Opus 4.7)
- Conta Telegram (bot via @BotFather)
- (Opcional) PostgreSQL 14+ com pgvector (pra memoria vetorial)

## As 5 partes do agente
1. **ALMA**: modelo Claude Opus 4.7 (mora na nuvem da Anthropic)
2. **CORPO**: Claude Code (programa Node.js que roda no terminal)
3. **CASA**: VPS Linux + tmux (sessao persistente)
4. **MEMORIA**: pastas memory/ e knowledge/ com arquivos markdown
5. **CANAL**: bot Python externo conectado ao Telegram

Mais opcional: subagentes em .claude/agents/*.md, cada um com personalidade propria.

## Estrutura de pastas
```
~/agente/
  CLAUDE.md                    # personalidade do agente principal
  memory/
    decisions.md               # decisoes permanentes do dono
    projects.md                # projetos em andamento
    pending.md                 # aguardando algo
    people.md                  # contatos importantes
    lessons.md                 # licoes aprendidas
    daily/YYYY-MM-DD.md        # notas do dia
  knowledge/
    tools/                     # manuais de ferramentas
    agents/                    # manuais dos subagentes
  .claude/agents/
    paulo-dev.md               # subagente: dev full-stack
    juliana-ops.md             # subagente: design e processos
    jonathan-copy.md           # subagente: copy e pesquisa
```

## Pipeline ponta a ponta de uma mensagem
1. Chefe digita no Telegram
2. Servidores Telegram seguram a mensagem
3. Bot Python faz polling a cada 1-2s e baixa
4. Bot grava `inbox/<msg_id>.json` (audit log)
5. Bot roda `tmux send-keys -t naia "..." Enter` (teclado fantasma)
6. Claude Code recebe como se fosse digitado
7. Claude Code empacota: system prompt + tools + historico + msg nova
8. Claude Code faz POST `https://api.anthropic.com/v1/messages`
9. Anthropic roteia pro Opus 4.7 num data center
10. Modelo decide chamar tools (Bash, Read, Edit, Agent, etc.)
11. Claude Code executa tools localmente, devolve outputs pro modelo
12. Modelo gera resposta final em texto
13. Naia grava `outbox/<msg_id>.json`
14. Bot Python detecta novo arquivo em outbox/
15. Bot faz POST na API do Telegram (sendMessage ou sendVoice)
16. Telegram entrega no celular do Chefe
17. Bot move arquivo pra sent/

Tempo medio: 2-8 segundos pra mensagens simples; minutos pra tarefas com subagente.

## Os 11 passos pra montar
1. **Contrate VPS Hostinger**: 4 vCPU, 8GB RAM, Ubuntu 22.04. Cupom AVALANCHE.
2. **SSH no servidor**: `ssh root@IP`. `apt update && apt upgrade -y`.
3. **Instale dependencias**: `apt install -y nodejs npm tmux git python3 python3-pip ffmpeg`
4. **Instale Claude Code**: `npm install -g @anthropic-ai/claude-code`
5. **Crie estrutura**: `mkdir -p ~/agente/{memory/daily,knowledge,.claude/agents}`
6. **Escreva CLAUDE.md inicial**: identidade + hierarquia + tom + 3-5 regras
7. **Crie 1-3 subagentes**: ao menos um dev.md e um copy.md em .claude/agents/
8. **Suba sessao tmux**: `cd ~/agente && tmux new -s naia` -> `claude` -> login OAuth -> Ctrl+B D
9. **Crie bot no Telegram**: @BotFather -> /newbot -> copie TOKEN. @userinfobot pra chat_id.
10. **Suba daemon Python do bot**: codigo de exemplo no repo desse metodo. Configure TOKEN/CHAT_ID/TMUX_SESSION.
11. **Configure systemd**: arquivo /etc/systemd/system/agente-bot.service com Restart=always. `systemctl enable --now agente-bot`.

## Componentes detalhados

### CLAUDE.md (system prompt)
Arquivo markdown puro na raiz do agente. Carregado em toda chamada como instrucao de sistema. Define:
- Identidade (nome, papel)
- Hierarquia (quem manda, quem reporta)
- Tom de voz (anti-patterns, idioma, estilo)
- Regras operacionais (o que faz sozinho, o que pede confirmacao)
- Conhecimento essencial (atalhos pra arquivos, credenciais)
- Memoria (quais arquivos ler no boot)

Recomendacao: comece com 50 linhas. A cada erro, adicione regra. Em 2 semanas, voce tem um arquivo cirurgico.

### Memoria em arquivos
Sem banco. Sem ORM. Sem schema. Markdown puro. Vantagens:
- Auditavel (abre no editor e le)
- Editavel a mao
- Versionavel via git
- Portatil (copia pasta entre servidores)

O agente le no boot e sob demanda (tool Read).

### Tools
Funcoes que o Claude Code expoe pro modelo. Padrao:
- Bash, Read, Write, Edit
- WebFetch, WebSearch
- Agent (criar subagente em sessao isolada)

Tools podem ser limitadas por subagente. Exemplo: SDR so tem Read e Write em memory/ (sem Bash).

### Subagentes
Outros arquivos .md em `.claude/agents/`. Cada um e um agente isolado com:
- YAML frontmatter (name, description, tools, model)
- System prompt em markdown

Naia delega via tool Agent. Cada subagente abre sub-sessao isolada com seu proprio system prompt. Permite paralelismo (3 Paulos rodando em escopos diferentes).

### Bot Telegram externo
Daemon Python independente do Claude Code. Vantagens:
- Desacoplamento: bot e Claude sao processos separados
- Persistencia: fila no filesystem (inbox/, outbox/, sent/)
- Auditavel: cada mensagem fica em JSON pra sempre
- Multi-canal: mesma arquitetura serve pra WhatsApp, Discord, email

Roda como systemd service com Restart=always.

## Modelos disponiveis
| Modelo | Qualidade | Janela | Quando usar |
|---|---|---|---|
| Claude Opus 4.7 | Maxima | 1M tokens | Decisoes criticas, codigo complexo |
| Claude Sonnet 4.5 | Alta | 200K | Tarefas medias, conversa, copy |
| Claude Haiku 3.5 | Boa | 200K | Triagem, classificacao, respostas curtas |

Naia roda Opus 4.7. SDRs rodam Sonnet 4.5.

## O que pode dar errado e como recuperar
| Falha | Causa | Recuperacao |
|---|---|---|
| Internet caiu | VPS sem acesso a Anthropic | tmux mantem sessao, retoma quando volta |
| Anthropic 503 | API fora | Claude Code retry automatico |
| Bot Python travou | Daemon parou | systemd restart em 5s |
| tmux session morreu | Comando matou claude | Bot detecta, recria sessao |
| Token Telegram expirou | Bot nao polla mais | Renovar com BotFather |

## Custos aproximados
- VPS: R$50-300/mes (Hostinger)
- Anthropic: US$50-300/mes pra uso pessoal intenso
- Telegram bot: gratis
- ElevenLabs (audio out): US$5-99/mes opcional
- Total: R$400-2000/mes pra agente completo 24/7

## Regras de ouro
- Se importa, escreve em arquivo. Memoria fora de markdown nao existe pra proxima sessao.
- Tarefa que demora mais de 30s = delega pra subagente.
- Naia orquestra, subagentes executam, Chefe valida.
- CLAUDE.md cresce com erros. Cada erro vira regra nova.
- Bash livre = poder destrutivo. Sempre limita por subagente.

## Ativacao
Inicie pelo capitulo 1 do site (Visao geral) e siga ate o capitulo 13 (Como montar o seu).

URL do site: https://anatomia.denderson.com
Repositorio: https://github.com/denderson2013-bot/anatomia-agente-claude-tmux
