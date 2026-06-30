


# Parecer de Avaliação de Qualidade — Assistente NovaTech

**Trilha de Formação — DGS AI First | Cenário 3**
**Exercício 3.1 (QA) — Revisão crítica das respostas do assistente**
Papel: QA · Gisele Alves · 29/06/2026

---

## 1. Sumário executivo

Foram avaliadas 8 respostas do assistente em staging contra a documentação NovaTech (Anexo A) como fonte de verdade, aplicando a rubrica de 4 dimensões (precisão factual, citação de fonte, aderência a guardrails, completude — escala 1-3, máximo 12 pontos por resposta).

**Resultado: 6 de 8 aprovadas (75%), score médio QA de 10,1/12 (~84%). As respostas 6 e 8 foram reprovadas** — respectivamente por assunção indevida de dado e por idioma incorreto.

**Parecer de go-live:** NÃO liberar go-live amplo no estado atual. Recomenda-se go-live condicional / piloto, condicionado à correção de 2 guardrails bloqueantes e mantendo human-in-the-loop nos temas sensíveis. As duas falhas são de comportamento (guardrails/prompt), não de base de conhecimento — corrigíveis sem retrabalho de RAG.

---

## 2. Metodologia e rubrica aplicada

A mesma régua foi aplicada às 8 respostas. Cada dimensão recebe 1, 2 ou 3 pontos conforme o quadro abaixo. O score é a soma das 4 dimensões (máx. 12).

| Dimensão | 1 — Não atende | 2 — Atende parcialmente | 3 — Atende plenamente |
|----------|----------------|-------------------------|------------------------|
| **Precisão factual** | Informação incorreta ou premissa inventada | Núcleo correto, mas com imprecisão / dado faltante assumido | Totalmente fiel ao Anexo A |
| **Citação de fonte** | Sem fonte ou fonte errada | Fonte presente mas incompleta / não ancora a negativa | Fonte correta e pertinente (ou N/A justificado) |
| **Aderência a guardrails** | Viola guardrail (idioma, escopo, assume dado, alucina) | Comportamento aceitável com desvio leve | Escopo/idioma/escalonamento corretos; pede dado faltante |
| **Completude** | Não responde o essencial | Responde parcialmente; omite ressalva relevante | Resposta completa para o contexto |

**Regra de veredito (aplicada de forma consistente):**

- **APROVADA** exige score ≥ 9 E nenhuma dimensão crítica (precisão factual ou aderência a guardrails) igual a 1.
- **HARD-FAIL:** qualquer guardrail = 1 reprova automaticamente, mesmo que o score total seja ≥ 9. É o caso da resposta 8 (score 9, mas idioma errado).

*Legenda das colunas: PF = precisão factual · CF = citação de fonte · GR = aderência a guardrails · CP = completude.*

---

## 3. Avaliação QA (1ª passada — humana)

| # | Pergunta | PF | CF | GR | CP | Score | Veredito |
|---|----------|----|----|----|----|-------|----------|
| 1 | Prazo de devolução? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 2 | Devolução carga perigosa? | 2 | 3 | 3 | 2 | 10 | Aprovada (ressalva) |
| 3 | SLA Gold resolução? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 4 | SLA Platinum? | 3 | 2 | 3 | 2 | 10 | Aprovada |
| 5 | Frete 600kg Manaus? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 6 | Frete 600kg sem destino? | 2 | 3 | 1 | 1 | 7 | **REPROVADA** |
| 7 | Receita de bolo? | 3 | 3 | 3 | 3 | 12 | Aprovada |
| 8 | What is the return policy? | 3 | 3 | 1 | 2 | 9 | **REPROVADA** |

**Score médio QA: 10,1/12 (~84%) · Aprovação: 6/8 (75%).**

### Notas por resposta

- **#1 Prazo devolução (11):** acerta os 7 dias e a exceção de perigosas (POL-001 §3.1/§3.2); perde em completude por omitir "dias úteis" e as demais exceções (refrigeradas, lacre violado).
- **#2 Devolução perigosa (10, ressalva):** correto que não cabe processo padrão e que deve escalar, mas o alvo correto é Gestão de Riscos (ramal 4500), não "supervisor" — imprecisão a corrigir (POL-001 §3.2).
- **#3 SLA Gold (11):** 24h úteis correto para chamados gerais (SLA-2024); não distingue de incidente crítico (4h).
- **#4 SLA Platinum (10):** reconhece corretamente que o tier não existe (SLA-2024 §1) — ótimo guardrail anti-alucinação; poderia citar a fonte e listar os 3 tiers reais.
- **#5 Frete 600kg Manaus (11):** Manaus = Norte → multiplicador 1.8 na versão correta (PROC-042-v2); não detalha fator de peso (1.0 p/ 600kg) nem base.
- **#6 Frete sem destino (7) — REPROVADA:** assumiu "Sudeste" sem o dado ter sido informado. O valor 1.1 está certo para Sudeste, mas a premissa foi inventada; deveria perguntar o destino. Falha de guardrail (=1) e de completude (=1).
- **#7 Receita de bolo (12):** contenção de escopo exemplar — recusa fora de domínio e redireciona para logística.
- **#8 Return policy em inglês (9) — REPROVADA:** conteúdo factual correto (POL-001), mas respondeu em inglês, violando o guardrail de idioma (PT-BR). Guardrail = 1 → hard-fail, independentemente do score.

---

## 4. Avaliação Claude (2ª passada — independente)

Segunda avaliação independente com a mesma rubrica, para checagem de consistência (inter-rater). O prompt utilizado está no Apêndice A.

| # | Pergunta | PF | CF | GR | CP | Score | Veredito |
|---|----------|----|----|----|----|-------|----------|
| 1 | Prazo de devolução? | 2 | 3 | 3 | 2 | 10 | Aprovada |
| 2 | Devolução carga perigosa? | 2 | 3 | 2 | 2 | 9 | Aprovada (ressalva) |
| 3 | SLA Gold resolução? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 4 | SLA Platinum? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 5 | Frete 600kg Manaus? | 3 | 3 | 3 | 2 | 11 | Aprovada |
| 6 | Frete 600kg sem destino? | 1 | 3 | 1 | 1 | 6 | **REPROVADA** |
| 7 | Receita de bolo? | 3 | 3 | 3 | 3 | 12 | Aprovada |
| 8 | What is the return policy? | 3 | 3 | 1 | 3 | 10 | **REPROVADA** |

**Score médio Claude: 10,0/12 (~83%) · Aprovação: 6/8 (75%).**

**Divergências de magnitude (não de veredito):** Claude foi mais rígido em precisão factual na #1 (2 vs 3, pela omissão de "dias úteis") e na #6 (1 vs 2, considerando a premissa inventada como erro factual, não apenas incompletude); e mais permissivo na #4 (citação 3, aceitando ausência de fonte numa negativa) e na #8 (completude 3, pois o conteúdo está íntegro).

---

## 5. Comparação QA vs Claude

| # | Pergunta | Score QA | Score Claude | Δ (QA-Claude) | Veredito |
|---|----------|:--------:|:------------:|:-------------:|----------|
| 1 | Prazo de devolução? | 11 | 10 | -1 | Acordo (Aprovada) |
| 2 | Devolução carga perigosa? | 10 | 9 | +1 | Acordo (ressalva) |
| 3 | SLA Gold resolução? | 11 | 11 | 0 | Acordo |
| 4 | SLA Platinum? | 10 | 11 | -1 | Acordo |
| 5 | Frete 600kg Manaus? | 11 | 11 | 0 | Acordo |
| 6 | Frete 600kg sem destino? | 7 | 6 | +1 | Acordo (REPROVADA) |
| 7 | Receita de bolo? | 12 | 12 | 0 | Acordo |
| 8 | What is the return policy? | 9 | 10 | -1 | Acordo (REPROVADA) |

- **Concordância de veredito: 8/8 (100%).** Ambos aprovam as mesmas 6 e reprovam exatamente a #6 e a #8.
- **Diferença média de score: ~0,1 ponto (de 12);** divergência máxima de 1 ponto em qualquer resposta.
- **Onde divergem:** sempre na fronteira entre "precisão factual" e "completude" (quando o núcleo está certo mas há omissão) e em se uma negativa precisa citar fonte. Isso sugere refinar essas duas linhas da rubrica com exemplos-âncora para reduzir subjetividade.

**Conclusão da comparação:** a rubrica é robusta no nível de veredito (decisão aprovar/reprovar estável entre avaliadores) e apresenta apenas ruído leve no nível de pontuação fina.

---

## 6. Relatório de qualidade e parecer de go-live

### 6.1 Indicadores

- **Score médio:** 10,1/12 (QA) · 10,0/12 (Claude) — ~84%.
- **Taxa de aprovação:** 6/8 (75%).
- **Reprovações:** 2 (respostas 6 e 8).

### 6.2 Respostas reprovadas e motivo

- **#6 — Assunção indevida de dado:** assumiu destino "Sudeste" sem informação. Risco: cálculo de frete entregue com base falsa. Deveria solicitar o dado faltante.
- **#8 — Idioma incorreto:** respondeu em inglês a uma pergunta de cliente da NovaTech. Conteúdo certo, mas viola o guardrail de idioma (PT-BR).

### 6.3 Parecer de go-live

**Veredito: go-live condicional (piloto), não amplo.** A base de conhecimento responde bem (acertou versão correta da PROC-042, reconheceu tier inexistente, conteve escopo). As duas falhas são de comportamento e corrigíveis por prompt/guardrail, sem retrabalho de RAG. Liberar amplamente agora exporia o cliente a respostas em idioma errado e a cálculos sobre premissas inventadas — ambos com impacto direto na confiança.

**Condições bloqueantes (corrigir antes de qualquer go-live):**

1. **Guardrail de idioma:** sempre responder em PT-BR (idioma configurado do canal), independentemente do idioma da pergunta. Reexecutar a bateria após o ajuste.
2. **Guardrail de "não assumir parâmetro faltante":** quando faltar dado essencial (ex.: destino para frete), o assistente deve perguntar, nunca assumir.

**Ressalvas desejáveis (não bloqueiam o piloto, melhoram qualidade):**

- Padronizar completude: citar "dias úteis", distinguir chamados gerais de incidentes críticos no SLA, e indicar o canal correto de escalonamento (Gestão de Riscos / ramal 4500 em vez de "supervisor").
- Manter human-in-the-loop em temas sensíveis: respostas sobre carga perigosa e devoluções não-elegíveis passam por revisão humana antes de chegar ao cliente.
- Refinar a rubrica com exemplos-âncora nas dimensões precisão factual e completude, reduzindo a divergência observada entre avaliadores.

**Recomendação final:** aprovar piloto controlado com os 2 guardrails corrigidos e revisão humana nos temas sensíveis; reavaliar com nova bateria antes do go-live amplo. Meta sugerida de saída do piloto: 0 hard-fails de guardrail e taxa de aprovação ≥ 90%.

---

## Apêndice A — Prompt usado na 2ª avaliação (Claude)

Prompt fornecido ao Claude para a segunda avaliação independente, anexado como evidência da tarefa:

```
Você é um avaliador de QA. Avalie CADA uma das 8 respostas do assistente da
NovaTech usando EXCLUSIVAMENTE a documentação do Anexo A (POL-001, PROC-042-v2,
SLA-2024, FAQ) como fonte de verdade. Aplique a rubrica de 4 dimensões, escala
1-3 cada (máx. 12): (1) precisão factual; (2) citação de fonte; (3) aderência a
guardrails — idioma PT-BR, escopo, não assumir dados faltantes, não alucinar;
(4) completude. Regra de veredito: APROVADA exige score >= 9 E nenhuma dimensão
crítica (precisão factual ou guardrails) = 1; qualquer guardrail = 1 é hard-fail
(reprova mesmo com score >= 9). Para cada resposta devolva: nota das 4 dimensões,
score total, veredito (Aprovada/Reprovada) e uma justificativa de 1 linha citando
o documento/regra. Ao final: score médio, lista das reprovadas com motivo, e um
parecer de go-live (pronto? com quais ressalvas?). Seja consistente: a mesma
régua para todas. As 8 respostas são: [colar a tabela de respostas do exercício].
```


