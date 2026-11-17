# Ata â€” notas de reuniÃ£o que nÃ£o saem da sua mÃ¡quina

[![Ata: sua reuniÃ£o vira ata](guia/assets/banner.jpg)](https://inematds.github.io/ata/guia/)

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/ata/guia/**

---

O **Ata** grava a sua reuniÃ£o em **duas faixas**: o que vocÃª ouve (Meet, Zoom, Teams, YouTube) e o que vocÃª fala. Depois transcreve na sua mÃ¡quina, em **portuguÃªs, inglÃªs ou espanhol**, e entrega uma nota em Markdown com cada fala marcada por pessoa. TambÃ©m gera resumo, decisÃµes e aÃ§Ãµes com responsÃ¡vel. NÃ£o entra bot na call, nÃ£o precisa de conta e o Ã¡udio nunca sai do computador.

- **LicenÃ§a:** MIT. Projeto aberto do INEMA.
- **InspiraÃ§Ã£o:** as ideias do [CrunchLog](https://www.skool.com/aiautomationsbyjack), de Christian Landsteiner (duas faixas, "o microfone Ã© vocÃª", gravaÃ§Ã£o como fonte de verdade). O Ata Ã© uma **implementaÃ§Ã£o independente**: nenhuma linha de cÃ³digo do CrunchLog foi usada.
- **Estado:** 0.1.0. Todas as fases estÃ£o implementadas e testadas com motores simulados. Os motores reais (NeMo-Speech.cpp na GPU, onnx na CPU, Ollama) estÃ£o implementados, mas **ainda nÃ£o foram medidos nesta versÃ£o** (veja [Estado e limites](#estado-e-limites)).

## O que ele faz

| Recurso | Como |
|---|---|
| **Duas faixas, sem driver** | Linux: PipeWire (`pw-record`). Windows: WASAPI loopback. macOS: helper Swift com *process tap* (14.2+, cÃ³digo-fonte em `helpers/macos/`) |
| **Microfone = vocÃª** | Tudo o que vem do microfone Ã© "Eu". O *gate* tira do microfone o eco da caixa de som, e a separaÃ§Ã£o de falantes sÃ³ atua do outro lado |
| **pt-BR, EN, ES** | Motor, limpeza de vÃ­cios ("nÃ©", "tipo"; "um", "uh"; "este", "o sea"), nota, resumo, demo e preparaÃ§Ã£o, tudo por idioma |
| **GPU ou CPU** | Motor principal: NeMo-Speech.cpp (Parakeet v3 + Nemotron, CUDA no DGX Spark/GB10, Metal no Mac, CPU nos demais). Reserva em processo: onnx-asr + sherpa-onnx |
| **Resumo local** | Ollama por padrÃ£o (JSON com resumo, tÃ³picos, decisÃµes, aÃ§Ãµes com dono e prazo, perguntas). Opcional: `claude` ou `codex` pela sua assinatura |
| **Busca e perguntas** | Ãndice SQLite (FTS5 + vetores) em todas as reuniÃµes. `ata ask` responde com citaÃ§Ã£o `[reuniÃ£o, mm:ss]` |
| **PreparaÃ§Ã£o** | `ata prep "cliente"` monta a pauta com aÃ§Ãµes em aberto, decisÃµes e perguntas; os itens se marcam sozinhos na reuniÃ£o seguinte |
| **Reconhecer vozes** | Opcional e desligado por padrÃ£o: cadastra a voz de uma pessoa e o "Pessoa 2" vira "Ana" nas prÃ³ximas reuniÃµes |
| **Segundo cÃ©rebro** | `ata connect cerebro --dir ...` escreve a reuniÃ£o em `fontes/` e as decisÃµes em `decisoes/registro.md`. TambÃ©m hÃ¡ Obsidian e Agentic OS |
| **Modo ao vivo** | TranscriÃ§Ã£o local durante a call (Nemotron 3.5 streaming), com a pauta se marcando, sem serviÃ§o pago |
| **Agentes** | Servidor MCP (`ata mcp`), skills e plugin do Claude Code |
| **Dashboard** | `ata dashboard`: gravar, ouvir, transcriÃ§Ã£o clicÃ¡vel, renomear falantes, busca e modo ao vivo (PT/EN/ES) |
| **CompatÃ­vel com o CrunchLog** | LÃª gravaÃ§Ãµes do CrunchLog (`bundle_version: 2`) sem alterar os originais |

## Instalar

```bash
# 1. uv (gerenciador de Python)
curl -LsSf https://astral.sh/uv/install.sh | sh                      # Linux/macOS
# powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"   # Windows

# 2. Ata (com MCP, modo ao vivo e motor CPU)
git clone https://github.com/inematds/ata && cd ata
uv tool install --python 3.12 ".[mcp,live,onnx]"

# 3. Configurar (idioma, pastas, resumo)
ata setup --lang pt-BR --summary ollama

# 4. Motor de transcriÃ§Ã£o (detecta a GPU: cuda13 no GB10, metal no Mac, cpu no resto)
ata engine install --dry-run            # mostra o que vai baixar
ata engine install --allow-unverified   # nesta versÃ£o os hashes ainda nÃ£o estÃ£o fixados
ata engine start

# 5. Conferir
ata doctor
```

## Usar

```bash
ata demo --lang pt-BR                  # reuniÃ£o inventada â†’ nota (com Piper; --no-tts sem voz)
ata toggle                             # comeÃ§a a gravar; de novo para e gera a nota
ata start --title "Kickoff Acme" --speakers 3 --lang pt-BR
ata stop
ata import gravacao.mp4 --lang es      # Ã¡udio/vÃ­deo que vocÃª jÃ¡ tem
ata process <gravaÃ§Ã£o> --speakers 4    # reprocessar
ata speakers <gravaÃ§Ã£o> "Pessoa 2=Ana" "Pessoa 3=Bruno"
ata export <gravaÃ§Ã£o> --format srt     # srt|vtt|txt|json|md|csv
ata search "prazo do beta" --since 30d
ata ask "o que decidimos sobre o lanÃ§amento?"
ata actions --owner Ana
ata prep "acme" --calendar "Qui 14h, Ana e Bruno"
ata voices enroll Ana <gravaÃ§Ã£o> "Pessoa 2"   # opcional (precisa de voices.enabled = true)
ata connect cerebro --dir ~/meu-cerebro
ata live start                         # modo ao vivo durante a gravaÃ§Ã£o
ata dashboard                          # http://127.0.0.1:47530/
ata mcp                                # servidor MCP para Claude Code / Codex
ata privacy <gravaÃ§Ã£o>                 # quais motores e destinos tocaram essa reuniÃ£o
ata bench asr --set <pasta> --engines nemo,onnx --lang pt-BR   # WER e velocidade
```

Atalho de teclado: ligue `ata toggle` a uma tecla (GNOME/KDE, PowerToys/AutoHotkey, Atalhos do macOS ou Stream Deck).

**MCP no Claude Code:** `claude mcp add --scope user ata -- ata mcp`. O plugin com as skills (`meeting-notes`, `action-items`, `meeting-prep`, `ask-meetings`, `meeting-to-wiki`) estÃ¡ em `.claude-plugin/` e `skills/`.

## Como funciona

```
tecla / dashboard / MCP
   â”‚
   â–¼  gravaÃ§Ã£o (PipeWire Â· WASAPI Â· helper macOS) â†’ pasta da reuniÃ£o (bundle ata/1)
far.wav (o que vocÃª ouve) + mic.wav (vocÃª) + meta.json (relÃ³gio, idioma, motores)
   â”‚
   â–¼  ata process (tudo local)
ASR por faixa e idioma â†’ um Ãºnico ponto de alinhamento de tempo â†’ gate tira o eco do mic
â†’ separaÃ§Ã£o de falantes do far (+ mic opcional) â†’ voiceprints (opcional) â†’ turnos
â†’ limpeza por idioma + glossÃ¡rio â†’ resumo JSON (Ollama | claude | codex)
â†’ nota.md (bundle + pasta de notas) â†’ Ã­ndice de busca â†’ segundo cÃ©rebro / Obsidian
```

Contrato completo: [`docs/CONTRATO.md`](docs/CONTRATO.md). Interfaces entre mÃ³dulos: [`docs/INTERFACES.md`](docs/INTERFACES.md).

## Privacidade

| NÃ­vel | O que sai da mÃ¡quina |
|---|---|
| **0 (padrÃ£o)** | Nada. TranscriÃ§Ã£o, separaÃ§Ã£o de falantes, resumo (Ollama), busca e modo ao vivo rodam todos local |
| **1 (assinatura)** | SÃ³ texto da transcriÃ§Ã£o, nunca Ã¡udio, para Anthropic ou OpenAI via `claude -p` ou `codex exec`, com as ferramentas desligadas. Exige `privacy.level = 1` |

Nenhum mÃ³dulo chama API paga. Logs e erros nunca contÃªm texto de reuniÃ£o. As voiceprints ficam sÃ³ na sua mÃ¡quina e podem ser apagadas (`ata voices forget --all`).

## Estado e limites

- **Testes:** suÃ­te com mais de 400 testes, sem rede e sem modelos (motores simulados). A gravaÃ§Ã£o real pelo PipeWire foi testada no Linux.
- **Ainda nÃ£o medido nesta versÃ£o:** desempenho e qualidade dos motores reais (NeMo-Speech.cpp, onnx, Ollama). A API do NeMo-Speech.cpp segue o que estÃ¡ documentado em `src/ata/engines/nemo.py`. Os hashes de verificaÃ§Ã£o dos downloads estÃ£o vazios em `src/ata/data/models.toml`, por isso a instalaÃ§Ã£o exige `--allow-unverified`.
- **macOS:** o helper Swift existe em cÃ³digo-fonte; ainda nÃ£o foi compilado nem testado.
- **Windows:** captura implementada; nÃ£o testada em mÃ¡quina real nesta versÃ£o.

## Desenvolvimento

```bash
uv sync --all-groups
uv run pytest -q                       # suÃ­te padrÃ£o (sem rede, sem modelos)
uv run pytest -q -m linux_audio        # gravaÃ§Ã£o real via PipeWire
```

## CrÃ©ditos

Inspirado nas ideias do CrunchLog (Christian Landsteiner). Modelos: NVIDIA Parakeet / Nemotron (NeMo-Speech.cpp), pyannote e WeSpeaker via sherpa-onnx, Ollama. Pesquisa e proposta de arquitetura: `docs/` e o projeto `crunchlog` do INEMA.

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe