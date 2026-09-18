Dublagem Automática — Contexto Central do Projeto
Cole este documento no início de qualquer conversa dedicada a um módulo específico deste projeto. Ele dá o contexto mínimo pra trabalhar naquele módulo sem quebrar o resto do sistema.

O que é o projeto
Sistema de dublagem automática (voz → texto → tradução → voz), rodando em servidor próprio, com interface web para configurar parâmetros, subir arquivos e gerenciar jobs. Também suporta texto → voz direto (reaproveitando o módulo de TTS). Estruturado desde o início pensando em distribuição futura.

Prioridade: primeiro funcionar, depois embelezar e adicionar funções.

Pipeline (visão geral)
Áudio → ASR (WhisperX) → Tradução (TowerInstruct) → Validação (Qwen2.5) → TTS (F5-TTS/XTTS) → Resync → Mixagem → Saída
Texto → TTS (F5-TTS/XTTS) → Saída
Etapa	Ferramenta	Responsabilidade
ASR	WhisperX	Transcrição + timestamps + speakers
Tradução	TowerInstruct	Traduz o texto original
Validação	Qwen2.5	Corrige erros, ajusta tempo/limpeza da tradução
TTS	F5-TTS / XTTS	Gera áudio da fala traduzida
Resync	script próprio (time-stretch)	Ajusta duração gerada pra bater com o timestamp original
Mixagem	script próprio	Junta voz traduzida com instrumental/fundo separado

Regras de arquitetura (não quebrar entre módulos)
Cada módulo tem seu próprio venv isolado. Nenhum módulo importa código de outro diretamente.
Comunicação só via JSON de segmentos, salvo em jobs/<job_id>/segments/. O orquestrador chama cada módulo como subprocesso, nunca via import.
Contrato de dados — cada módulo só adiciona campos ao mesmo objeto de segmento:
{
  "id": 12,
  "speaker": "SPEAKER_00",
  "start": 34.21,
  "end": 37.85,
  "text_original": "...",
  "text_translated": "...",
  "text_validated": "...",
  "audio_path": "audio_segments/012.wav",
  "duration_target": 3.64,
  "duration_generated": null
}
Se um módulo precisa de um campo novo, adiciona ao schema acima — não cria um formato paralelo.

Estrutura de diretórios (resumo)
papagaio/
├── orchestrator/        # FastAPI + web UI — nunca importa código de módulo direto
├── modules/
│   ├── asr_whisperx/
│   ├── translate_tower/
│   ├── validate_qwen/
│   ├── tts_engine/
│   ├── resync/
│   └── audio_mixing/
├── jobs/<job_id>/        # input, segments, audio_segments, output — por execução
├── config/
├── docs/
└── scripts/
Cada módulo tem: venv/, run.py (recebe segments.json, devolve segments.json atualizado), requirements.txt.

Infraestrutura
Servidor: Ubuntu 24.04, GPU A4000, 64GB RAM, 12 núcleos E5-2680.
Client: Linux Mint, pasta espelhada com o servidor via Syncthing.
O que espelha: código, configs, docs. O que NÃO espelha (fica só no servidor): venvs, checkpoints de modelo, jobs/*/audio_segments, jobs/*/separated.
Escopo atual: só servidor + client. Suporte a outros sistemas/dispositivos fica para uma fase futura.

Fases do projeto
#	Fase	Módulo/entrega
0	Infraestrutura	Servidor provisionado + pasta espelhada funcionando
1	ASR	modules/asr_whisperx/ funcionando isolado via CLI
2	Tradução	modules/translate_tower/ funcionando isolado
3	Validação	modules/validate_qwen/ funcionando isolado
4	TTS	modules/tts_engine/ funcionando isolado
5	Resync	modules/resync/ funcionando isolado
6	Mixagem	modules/audio_mixing/ funcionando isolado
7	Integração web	Orquestrador chamando tudo em sequência pela interface
8	Polimento	UI, gerenciamento de jobs, fila assíncrona

Fase atual: 0 — infraestrutura (servidor + pasta espelhada).

Como usar este documento
Ao abrir uma conversa nova pra trabalhar num módulo específico, cole este documento inteiro e diga qual fase/módulo é o foco daquela conversa. Isso garante que qualquer trabalho feito isoladamente já nasce respeitando o contrato de dados e a estrutura de pastas do projeto como um todo.

Documento de arquitetura completo (com diagrama e detalhes de cada fase): ver artifact "Arquitetura — Sistema de Dublagem Automática".

teste