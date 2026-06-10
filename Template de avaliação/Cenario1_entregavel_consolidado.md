# Cenário 1 — Avaliação de Respostas do Assistente (QA)

> **Entregável consolidado.** Reúne os quatro artefatos do exercício: a avaliação manual (feita antes da rubrica), a rubrica gerada com o Claude, o template reutilizável do Cowork e as pontuações aplicadas às 5 respostas.
>
> *Nota sobre o conteúdo:* o texto de Pergunta/Resposta/Fontes das 5 respostas foi reconstruído de forma coerente com os comentários e critérios fornecidos (domínio SLA-2024 / POL-001). As **notas, totais e veredictos** são exatamente os definidos na aplicação da rubrica; ajuste o fraseado das perguntas se divergir do material original.

---

## 1. Metodologia

A avaliação seguiu quatro etapas, do julgamento subjetivo ao processo objetivo e reutilizável:

1. **Avaliação manual (pré-rubrica)** — leitura qualitativa das 5 respostas, com veredicto "no olho", sem escala estruturada.
2. **Rubrica gerada com o Claude** — formalização dos critérios em quatro dimensões (A–D), escala 1–3 e regra de veredicto determinística.
3. **Template do Cowork** — planilha (`.xlsx`) e formulário (`.md`) genéricos, com fórmulas e agregação, aplicáveis a qualquer lote.
4. **Pontuações aplicadas** — a rubrica aplicada às 5 respostas, com notas, total, veredicto e observações.

O objetivo da progressão é mostrar que o veredicto intuitivo da etapa 1 se confirma na etapa 4, mas agora de forma **verificável e reproduzível** entre avaliadores.

---

## 2. Avaliação manual (pré-rubrica)

Leitura inicial de cada resposta, antes de existir qualquer escala. O veredicto é uma impressão geral do avaliador.

| # | Leitura qualitativa | Veredicto intuitivo |
|---|---------------------|---------------------|
| 1 | Prazo parece certo; a citação remete a uma seção um pouco diferente e faltam ressalvas. | Aprovada |
| 2 | Cálculo bate; explicação um pouco rasa e fonte genérica. | Aprovada |
| 3 | Cita um "tier Platinum" que não reconheço na tabela; valores parecem inventados. | Reprovada |
| 4 | Diz que carga perigosa pode ser devolvida — soa errado, contraria a política. | Reprovada |
| 5 | Correta; só falta um detalhe de seção. | Aprovada |

**Limitação observada.** O julgamento manual é rápido, mas subjetivo: não há nota por dimensão, a fronteira entre "aprovar" e "aprovar com ressalvas" fica a critério de cada avaliador, e dois QAs podem divergir nos casos-limite (#1 e #2). Isso motivou a criação da rubrica.

---

## 3. Rubrica gerada com o Claude

As notas estruturadas (1–3 por dimensão) funcionam como *structured output*: tornam a avaliação verificável e agregável.

### Escala (1–3 por dimensão) — `1` inadequado · `2` parcial · `3` adequado

**A — Precisão factual**

- **3** — Todas as afirmações corretas e alinhadas à fonte oficial.
- **2** — Majoritariamente correta, com imprecisões menores que não comprometem a decisão.
- **1** — Erro factual relevante ou informação inventada (alucinação).

**B — Citação / grounding**

- **3** — Cita documento e seção corretos; a citação sustenta a afirmação.
- **2** — Cita a fonte correta, mas seção imprecisa, incompleta ou parcialmente sustentada.
- **1** — Não cita fonte, cita fonte errada ou a citação não sustenta a resposta.

**C — Aderência a guardrails**

- **3** — Respeita totalmente escopo, políticas e limites definidos.
- **2** — Desvio leve de tom/escopo, sem violar política.
- **1** — Viola guardrail: conselho fora de escopo, vaza dado sensível ou ignora restrição.

**D — Completude / adequação**

- **3** — Responde integralmente à pergunta, no nível de detalhe adequado.
- **2** — Responde parcialmente ou com detalhe insuficiente/excessivo.
- **1** — Não responde à pergunta ou é irrelevante.

### Regra de veredicto

- **Reprovação automática:** se **A = 1** OU **B = 1** → **Reprovada** (independente do total).
- **Aprovada:** Total ≥ 10 (e sem reprovação automática).
- **Com ressalvas:** Total entre 8 e 9.
- **Reprovada:** Total < 8.
- Total = A + B + C + D (mínimo **4**, máximo **12**).

### Âncoras de calibração (casos-limite)

Para alinhar dois avaliadores nos casos difíceis:

- Tier/categoria que não existe na fonte (ex.: "Platinum" na SLA-2024) → **A = 1**.
- Exceção explícita da política ignorada (ex.: POL-001 sobre cargas perigosas) → **A = 1** e **C = 1**.
- Fonte correta, mas seção imprecisa → **B = 2** (não dispara reprovação).

---

## 4. Template do Cowork (reutilizável)

Dois formatos do mesmo instrumento, genéricos e aplicáveis a qualquer lote — nada neles é específico das 5 respostas:

- **`template_avaliacao.xlsx`** — três abas: **Avaliação** (uma linha por resposta, validação 1–3, Total e Veredicto automáticos), **Rubrica & Instruções** e **Resumo** (agregação do lote). As fórmulas estão em inglês (`SUM`/`IF`/`COUNTIF`) e o Excel pt-BR as exibe como `=SOMA`/`=SE`.
- **`template_avaliacao.md`** — formulário equivalente: instruções, escala, regra de veredicto e um bloco de ficha para copiar a cada resposta.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| ID | Texto | Identificador da resposta |
| Pergunta | Texto | Pergunta do atendente |
| Resposta do assistente | Texto | Saída gerada |
| Fonte citada | Texto | Documento + seção informados pela resposta |
| Fonte correta (gabarito) | Texto | Documento + seção segundo o Anexo A |
| A / B / C / D | 1–3 | Conforme rubrica |
| Total | Fórmula | `=SOMA(F:I)` |
| Veredicto | Fórmula | Aprovada / Com ressalvas / Reprovada |
| Observações | Texto | Justificativa do avaliador |

A aplicação às 5 respostas (abaixo) está gravada em **`Cenario1_aplicacao_5_respostas.xlsx`**, uma cópia preenchida do template — o template original permanece em branco para reuso.

---

## 5. Pontuações aplicadas (rubrica → 5 respostas)

| # | ID | A | B | C | D | Total | Veredicto |
|---|----|:-:|:-:|:-:|:-:|:-----:|-----------|
| 1 | R-001 | 3 | 2 | 3 | 2 | 10 | Aprovada |
| 2 | R-002 | 3 | 2 | 3 | 2 | 10 | Aprovada |
| 3 | R-003 | 1 | 1 | 1 | 1 | 4 | **Reprovada** |
| 4 | R-004 | 1 | 2 | 1 | 2 | 6 | **Reprovada** |
| 5 | R-005 | 3 | 2 | 3 | 2 | 10 | Aprovada |

### Detalhamento

**R-001 — Aprovada (10)**
Pergunta: *Qual é o prazo de entrega do frete expresso para a região Sudeste?*
Resposta: *O prazo do frete expresso para o Sudeste é de 2 dias úteis, conforme a SLA-2024.*
Fonte citada: SLA-2024, Seção 3.2 · Gabarito: SLA-2024, Seção 3.1
Observação: prazo correto, mas a citação aponta a Seção 3.2 quando o prazo está na 3.1 (B=2); omite duas das três exceções de prazo (D=2).

**R-002 — Aprovada (10)**
Pergunta: *Como calcular o adicional de uma carga acima de 500 kg?*
Resposta: *Multiplique a tarifa base pelo fator 1,5 para cargas acima de 500 kg.*
Fonte citada: SLA-2024, Tabela de Tarifas · Gabarito: SLA-2024, Seção 4.3 (fator de peso)
Observação: multiplicador (1,5) correto, mas a citação aponta a tabela genérica em vez da Seção 4.3 (B=2); a fórmula não explicita o fator de peso (D=2).

**R-003 — Reprovada (4) — reprovação automática (A=1 e B=1)**
Pergunta: *Quais são os valores de SLA do tier Platinum?*
Resposta: *O tier Platinum garante entrega em 4 horas e disponibilidade de 99,99%.*
Fonte citada: SLA-2024, Tier Platinum · Gabarito: SLA-2024, Seção 2 — tiers existentes: Bronze, Prata, Ouro (não há Platinum)
Observação: tier "Platinum" e valores de SLA **inventados** — não existem na SLA-2024 (A=1). Fonte citada inexistente e que contradiz a tabela (B=1). Viola o guardrail de não inventar (C=1). Resposta inútil para o atendente (D=1).

**R-004 — Reprovada (6) — reprovação automática (A=1)**
Pergunta: *Posso autorizar a devolução de uma carga classificada como perigosa?*
Resposta: *Sim, cargas perigosas podem ser devolvidas mediante registro no sistema.*
Fonte citada: POL-001, Seção 5 (Devoluções) · Gabarito: POL-001, Seção 5.2 — exceção: cargas perigosas NÃO podem ser devolvidas
Observação: regra **invertida** — o POL-001 proíbe expressamente a devolução de cargas perigosas (A=1). Orientar a devolução viola guardrail de política/segurança (C=1). Citação na seção certa de devoluções (B=2), mas ignora a exceção (D=2).

**R-005 — Aprovada (10)**
Pergunta: *O cliente pode reagendar a coleta sem custo adicional?*
Resposta: *Sim, o reagendamento de coleta é gratuito desde que solicitado com 24h de antecedência.*
Fonte citada: POL-001, Seção 6 · Gabarito: POL-001, Seção 6.1
Observação: resposta correta; pequena imprecisão na citação (Seção 6 em vez de 6.1) (B=2) e ausência da nota sobre a regra de transição do novo POL-001 (D=2).

### Resumo do lote

| Métrica | Valor |
|---|---|
| Respostas avaliadas | 5 |
| Aprovadas | 3 |
| Com ressalvas | 0 |
| Reprovadas | 2 |
| % Aprovação | 60% |
| Total médio | 8,0 |
| Média A — Precisão factual | 2,2 |
| Média B — Citação / grounding | 1,8 |
| Média C — Aderência a guardrails | 2,2 |
| Média D — Completude / adequação | 1,8 |

---

## 6. Validação dos critérios

**Resposta 3 identificada como incorreta.** A=1 (tier "Platinum" e valores de SLA inventados — alucinação) e B=1 (fonte que contradiz a resposta) → reprovação automática. A rubrica define A=1 como "informação inventada (alucinação)", capturando o caso diretamente.

**Resposta 4 identificada como incorreta.** A=1 (regra invertida: o POL-001 tem exceção explícita proibindo a devolução de cargas perigosas) e C=1 (violação de guardrail) → reprovação automática.

**Objetividade entre QAs.** A escala 1–3 tem critério descrito por dimensão (não é "bom/ruim" subjetivo) e a regra de veredicto é determinística: para o mesmo conjunto de notas o resultado é sempre igual (os 5 casos batem 100%). Os dois gatilhos de maior risco — alucinação (A=1) e grounding falho (B=1) — forçam reprovação independentemente do total, reduzindo a divergência justamente nos casos críticos. As âncoras de calibração (Seção 3) alinham os casos-limite.

**Reutilizável, não one-off.** Colunas/campos genéricos, rubrica e regra fixas, validação 1–3 e Resumo que agrega qualquer quantidade de linhas. As 5 respostas são apenas exemplo de aplicação; o template em branco serve a qualquer lote.

---

## Anexo A — Gabarito de fontes

Referência para preencher a coluna "Fonte correta (gabarito)" ao avaliar.

| Documento | Seção | Conteúdo |
|-----------|-------|----------|
| SLA-2024 | 2 | Tiers de serviço existentes: Bronze, Prata, Ouro (não há Platinum). |
| SLA-2024 | 3.1 | Prazos de entrega por região (inclui frete expresso). |
| SLA-2024 | 3.1 (exceções) | Três exceções de prazo: feriados, áreas remotas e força maior. |
| SLA-2024 | 4.3 | Fator de peso para cargas acima de 500 kg (multiplicador 1,5). |
| POL-001 | 5.2 | Exceção: cargas perigosas NÃO podem ser devolvidas. |
| POL-001 | 6.1 | Reagendamento de coleta gratuito com 24h de antecedência. |
