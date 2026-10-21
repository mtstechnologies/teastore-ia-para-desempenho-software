# AGENT SPEC — Setup e Execução do Experimento (TeaStore · k6 × Locust)

> **Como usar:** aponte seu agente (Claude Code / Codex / Gemini) para este arquivo.
> Execute **uma fase por vez**. Ao final de cada fase, **pare** e confirme os *Critérios
> de aceite* antes de avançar. Não avance com uma fase quebrada.

## Contexto (leia primeiro, é curto)

- **Objetivo:** comparar k6 e Locust gerando carga sobre o **TeaStore** (já clonado neste
  repositório), medindo latência, throughput, falhas e CPU/RAM, de forma **reprodutível**.
- **Já existe um kit pronto** (não reescreva): `scripts/k6/script.js`,
  `scripts/locust/locustfile.py`, `run_experiment.sh`, `sample_host.py`,
  `analise/analyze.py`. Documentos de referência: `00_escopo.md` e
  `PROTOCOLO_EXPERIMENTO.md`. Seu trabalho é **integrar e operar** esse kit, não substituí-lo.
- **Desenho:** 2 ferramentas × 3 cargas (100/500/1000 VUs) × 10 repetições = **60 execuções**,
  15 min cada (5 rampa + 10 platô), ordem aleatória.

## Guardrails (invioláveis)

1. **Nunca invente dados nem números.** Todo resultado vem de arquivo bruto salvo em `results/raw/`.
2. **Mantenha a equivalência k6 ↔ Locust:** mesmo think time (1–2 s), mesmo timeout (60 s),
   mesmo critério de falha (status 200–399) e as mesmas 5 operações. Se mexer num script, replique no outro.
3. **Não commite segredos** (senhas, IPs privados, tokens): use variáveis de ambiente / `.env` (no `.gitignore`).
4. **Não altere `00_escopo.md` nem `PROTOCOLO_EXPERIMENTO.md`** sem me avisar — eles são a fonte da verdade.
5. Preserve os brutos: **nunca sobrescreva** um arquivo de `results/raw/`.
6. Faça mudanças pequenas e verificáveis; ao final de cada fase, resuma o que fez e o que salvou.

## Estrutura de repositório alvo

```
.
├── teastore/                  # o clone do TeaStore (já existe)
├── scripts/{k6,locust}/       # do kit
├── run_experiment.sh          # do kit
├── sample_host.py             # do kit
├── analise/analyze.py         # do kit
├── results/
│   ├── raw/                   # brutos por execução (não versionar se forem grandes)
│   └── diario_execucao.csv
├── docs/
│   ├── 00_escopo.md
│   ├── 01_ambiente.md         # você preenche (Fase 1)
│   ├── 02_parametros.md       # você preenche (Fase 2)
│   └── pilot_equivalencia.md  # você preenche (Fase 3)
├── .env.example               # você cria (Fase 2)
└── AGENT_SPEC.md
```

---

## Fase 1 — Subir o TeaStore localmente

**Tarefas**
1. Localize no clone o `docker-compose` do TeaStore (não crie um novo; use o oficial do projeto).
2. Suba os serviços via Docker Compose e aguarde ficarem saudáveis.
3. Registre em `docs/01_ambiente.md`: versões (`docker --version`, `docker compose version`,
   `k6 version`, `locust --version`, `python3 --version`), commit do TeaStore, e hardware/SO da máquina.

**Critérios de aceite**
- A UI responde (HTTP 200) na porta do webui.
- `docs/01_ambiente.md` preenchido.

## Fase 2 — Confirmar os parâmetros do TeaStore

Os valores reais JÁ foram levantados do código e estão em `docs/02_parametros.md` e embutidos
no kit. Seu papel aqui é **validar**, não descobrir do zero:
- Base: `/tools.descartes.teastore.webui` em `:8080`; compose `examples/docker/docker-compose_default.yaml`.
- Rotas/params: `/login`; `/loginAction` (`username`,`password`; logout via `logout`);
  `/` (home); `/category?category=3&page=1`; `/product?id=<id>`.
- Credencial: `user1` / `password`. Categoria válida: `3`.
- **Produto:** confirme um `id` real abrindo `/category?category=3&page=1` (o `id=7` default é palpite).
- **Semeadura do banco:** `GET /dataBaseAction?confirm=true&categories=5&products=100&users=100&orders=5`,
  UMA vez (workload é read-only). NÃO regenerar entre execuções.

**Tarefas**
1. Preencha `docs/02_parametros.md` com todos os valores acima e **como você os confirmou**.
2. Crie `.env.example` (e um `.env` local, no `.gitignore`) com: `HOST`, `BASE_PATH`,
   `USERNAME`, `PASSWORD`, `CATEGORY_ID`, `PRODUCT_ID`, `RESET_URL`.
3. Atualize os `TODO` de `scripts/k6/script.js`, `scripts/locust/locustfile.py` e o topo de
   `run_experiment.sh` para lerem esses valores.

**Critérios de aceite**
- Um `curl` manual a cada um dos 6 endpoints retorna status esperado (200–399).
- `docs/02_parametros.md` e `.env` preenchidos; nenhum segredo commitado.

## Fase 3 — Piloto de equivalência (obrigatório)

**Tarefas**
1. Rode ~1 min de **cada** ferramenta em carga baixa (ex.: 10 VUs).
2. Extraia, para as duas, a contagem de requisições por operação e a distribuição de status.
3. Registre lado a lado em `docs/pilot_equivalencia.md` e confirme que batem
   (mesmas operações, proporções semelhantes, mesmos status).

**Critérios de aceite**
- As duas ferramentas exercitam as mesmas 6 requisições, com status esperados, sem 4xx/5xx inesperado.
- Divergência? **Pare e conserte os scripts** (mantendo equivalência) antes de seguir.

## Fase 4 — Preparar o ambiente remoto (2 máquinas)

Alvo: **Máquina A** = SUT (TeaStore); **Máquina B** = gerador (k6+Locust+kit).

**Tarefas**
1. Máquina A: subir o TeaStore (mesmo compose) e expor a porta para a Máquina B; anotar IP/porta.
2. Máquina B: instalar k6, Locust, Python (pandas/numpy/scipy/matplotlib) e copiar o kit.
3. Ajustar o `.env` da Máquina B: `HOST` = IP da Máquina A.
4. Validar conectividade A↔B (um `curl` da B para o webui da A) e registrar a rede em `docs/01_ambiente.md`.

**Critérios de aceite**
- Da Máquina B, um `curl` ao TeaStore da Máquina A retorna 200.
- k6, Locust e Python prontos na Máquina B.

## Fase 5 — Executar a matriz (60 execuções)

**Tarefas**
1. Na **Máquina A**, inicie a captura contínua de recursos do SUT:
   `python3 sample_host.py results/raw/stats_sut_SESSAO.csv`
2. Na **Máquina B**, rode `./run_experiment.sh` (ele já: **semeia o banco uma vez**, amostra
   CPU/RAM do gerador, roda 15 min por execução em ordem aleatória, e salva tudo com a nomenclatura do protocolo).
   Antes de disparar, confirme na `/status` que a semeadura terminou.
3. Acompanhe o `results/diario_execucao.csv`; se uma execução falhar, **não a descarte
   silenciosamente** — registre o motivo e re-execute uma repetição extra, anotando.

**Critérios de aceite**
- 60 execuções concluídas, cada uma com seu bruto de carga **e** seu `stats_gen_*.csv`.
- `diario_execucao.csv` com 60 linhas coerentes.

## Fase 6 — Análise e verificação

**Tarefas**
1. Rode `python3 analise/analyze.py`.
2. Confira `analise/out/`: `dados_consolidados.csv` (60 linhas), `estatistica.csv`,
   `dispersao.csv`, `figuras/`.
3. Faça uma **checagem de sanidade**: os números batem com os relatórios brutos de algumas
   execuções amostradas? (se não baterem, o problema é no parser — conserte, não maquie o dado.)
4. Crie `docs/RASTREABILIDADE.md` ligando cada tabela/figura ao(s) arquivo(s) de origem.

**Critérios de aceite**
- `dados_consolidados.csv` com 60 linhas e sem NaN inexplicado.
- Sanidade confere em ≥3 execuções amostradas.

---

## Definition of Done (global)
- TeaStore no ar (local e remoto), parâmetros confirmados, piloto de equivalência OK,
  60 execuções salvas com diário, e `analise/out/` gerado e verificado.
- **Entregável final:** compactar `results/raw/` + `results/diario_execucao.csv` +
  `analise/out/` e me enviar — daí eu preencho o artigo (template SBC) com os números reais.

## O que NÃO fazer
- Não rodar as 60 execuções antes do piloto passar.
- Não misturar `localhost` (SUT e gerador na mesma máquina) na fase remota — o objetivo é separá-los.
- Não "ajustar" números para ficarem bonitos. Se algo destoar, investigue a origem.
- Não expandir o escopo (JMeter, spike/soak, cloud) — está fora, é trabalho futuro.
