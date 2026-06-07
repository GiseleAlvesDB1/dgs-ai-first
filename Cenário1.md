# dgs-ai-first

# Cenário 1 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)
**Blocos cobertos no Cenário 1:
** 1) Fundamentos de IA Generativa 
** 2) Engenharia de Prompt
** 3) Engenharia de Contexto
** 4) RAG e MCP
**Projeto-base:** Assistente de IA conversacional da NovaTech (logística — Teams + SharePoint + Azure AI Services)
**Prazo do Cenário 1:** 06/06
---
# Resposta - Exercício 1.1 — Identificação de cenários de falha de IA
---
## Tarefa 1 — Lista inicial com 4 cenários de falha (sem usar IA).

### Cenário-01 — Alucinação de tier inexistente (Platinum)
- **Categoria:** Alucinação
- **Pergunta de teste:** "Qual o SLA do cliente Platinum?"
- **Comportamento esperado:** Informar que o tier Platinum não existe; listar apenas Gold, Silver e Standard, citando SLA-2024 seção 1.
- **Comportamento indesejado:** Gerar texto plausível com valores de SLA para um tier que a documentação declara não existir.
- **Como verificar: a resposta NÃO deve conter "Platinum" como tier válido e DEVE conter "Gold, Silver e Standard" + citação de SLA-2024 seção 1.

### Cenário-02 — Inversão da regra de devolução de carga perigosa
- **Categoria:** Alucinação / falha de interpretação
- **Pergunta de teste:** "Posso devolver carga perigosa?"
- **Comportamento esperado:** Informar que cargas perigosas (classes 1 a 6 da ANTT) NÃO são elegíveis ao processo padrão; orientar entrar em contato com a Gestão de Riscos (ramal 4500), citando POL-001 seção 3.2.
- **Comportamento indesejado:** Responder que cargas perigosas podem ser devolvidas em 7 dias úteis, confundindo a exceção com a regra geral.
- **Como verificar: resposta contém "não elegível" / "não pelo processo padrão" + "ramal 4500" + citação POL-001 seção 3.2.

### Cenário-03 — Mistura de versões da PROC-042 (v1 vs v2)
- **Categoria:** Informação contraditória / chunk errado
- **Pergunta de teste:** "Qual o multiplicador para o Norte e para o Sudeste?"
- **Comportamento esperado:** Usar somente a v2 para chamados a partir de 01/12/2023 (Norte 1.8, Sudeste 1.1), citando PROC-042-v2 seção 2.1 e a disposição transitória da seção 5.
- **Comportamento indesejado:** Misturar valores das duas versões (ex.: Norte 1.6 da v1 e Sudeste 1.1 da v2) sem aviso de inconsistência.
- **Como verificar: informação desatualizada e/ou contraditória, pode levar ao erro de valor financeiro direto.

### Cenário-04 — Resposta sem citação de fonte (guardrail 1)
- **Categoria:** Falha de guardrail
- **Pergunta de teste:** "Qual o prazo de devolução?"
- **Comportamento esperado:** "O prazo é de 7 dias úteis após o recebimento confirmado. (Fonte: POL-001, seção 3.1.)"
- **Comportamento indesejado:** "O prazo é de 7 dias úteis." — sem indicação de fonte.
- **Como verificar: toda resposta deve conter referência a uma fonte (POL / PROC / SLA / FAQ + seção).

---
## Tarefa 2 — Cenários da expansão com IA (Claude), cobrindo ângulos menos óbvios — em especial falhas de engenharia de contexto (Bloco 3) e combinações.

### IA-Cenário-01 — Invenção de frete para carga abaixo de 500 kg
- **Categoria:** Alucinação (pergunta sem cobertura)
- **Pergunta de teste:** "Qual o frete para 300 kg para Salvador?"
- **Comportamento esperado:** Informar que não há documentação de frete padrão (< 500 kg) indexada; orientar consulta ao Comercial. (Guardrail 3.)
- **Comportamento indesejado:** Aplicar indevidamente a tabela de frete especial (ex.: multiplicador 1.5 do Nordeste) a uma carga fora do escopo da PROC-042.
- **Como verificar:** Assert — resposta contém "não encontrado" / "não há informação" e NÃO apresenta multiplicador numérico definitivo para < 500 kg.
- **Leitura pela trilha:** Sem chunk recuperável, o comportamento correto é admitir ausência de *grounding* — não preencher a lacuna estatisticamente.

### IA-Cenário-02 — Lost in the middle: chunk de exceção suprimido por posição
- **Categoria:** Falha de contexto (lost in the middle)
- **Pergunta de teste:** "Qual o prazo de devolução, o procedimento e os custos envolvidos?"
- **Comportamento esperado:** Resposta cobre prazo (7 dias), exceções de carga perigosa, procedimento e custos, com todas as seções citadas.
- **Comportamento indesejado:** Omite as exceções (POL-001-B, no meio do conjunto de chunks), passando a impressão de que toda carga é elegível.
- **Como verificar:** Assert — resposta contém "carga perigosa" + "não elegível"; testar variando a posição do chunk no prompt e medir consistência.
- **Leitura pela trilha:** Efeito posicional dentro do orçamento de atenção (Bloco 3). Mitigação: reordenar chunks por relevância antes da montagem (ver CTX-05).

### IA-Cenário-03 — Context rot em sessão longa no Teams
- **Categoria:** Falha de contexto (context rot)
- **Pergunta de teste:** Após 4 perguntas sobre POL-001 e SLA na mesma sessão, a 5ª: "Qual o multiplicador correto para 2.000 kg para o Norte?"
- **Comportamento esperado:** Norte 1.8, fator de peso 1.15 (faixa 1.001–3.000 kg). Fonte: PROC-042-v2 seções 2 e 2.1.
- **Comportamento indesejado:** Repetir dado de perguntas anteriores (ex.: "7 dias úteis") ou usar o multiplicador da v1 (1.6), ignorando os chunks novos.
- **Como verificar:** Teste automatizado com fixture de histórico de N trocas + pergunta-alvo; executar múltiplas vezes e medir taxa de acerto (alvo ≥ 95% mesmo com 10+ trocas).
- **Leitura pela trilha:** Context rot é "degradação por tamanho" (Bloco 3). Mitigação: *compaction* do histórico e *retrieval just-in-time* dos chunks da pergunta corrente.

### IA-Cenário-04 — Context overflow em pergunta multi-domínio
- **Categoria:** Falha de contexto (context overflow)
- **Pergunta de teste:** "Preciso de todas as informações sobre devolução, frete especial, SLA e coleta reversa para uma carga perigosa de 2.000 kg para o Norte."
- **Comportamento esperado:** Sinalizar que a pergunta abrange múltiplos domínios e sugerir decompô-la, OU cobrir todos os domínios com citação, sem truncar.
- **Comportamento indesejado:** Responder sobre frete e devolução, mas omitir SLA ou as exceções de carga perigosa — sem indicar a omissão.
- **Como verificar:** Monitorar a contagem de tokens do contexto montado (logs do Azure AI Services); alerta se o prompt ultrapassar ~80% da janela; assert de cobertura de todos os domínios.
- **Leitura pela trilha:** Estouro do orçamento de atenção. *Progressive disclosure* (revelar contexto sob demanda) é a estratégia da trilha para evitar o truncamento.

### IA-Cenário-05 — Falha combinada (cenário de alto risco)
- **Categoria:** Combinado (context rot + chunk errado + alucinação + guardrail)
- **Pergunta de teste:** 8ª troca da sessão: "Qual o custo total para um cliente Platinum devolver uma carga perigosa de 4.000 kg do Nordeste?"
- **Comportamento esperado:** (1) Platinum não existe (SLA-2024); (2) carga perigosa não segue devolução padrão (POL-001 seção 3.2); (3) frete acima de 3.000 kg usa fator 1.4 e Nordeste 1.5 (PROC-042-v2). Cada afirmação com sua fonte.
- **Comportamento indesejado:** Inventar custo total com tier Platinum, multiplicadores da v1 e devolução de carga perigosa permitida, sem alertas e sem citação.
- **Como verificar:** *Exploratory testing* com script de 7 trocas + pergunta-bomba; revisão por par (QA + especialista de domínio); validar cada afirmação contra os documentos.
- **Leitura pela trilha:** Demonstra como falhas de contexto e de grounding se acumulam quando o orçamento de atenção já está comprometido — argumento central do Bloco 3.

---
## Tarefa 3 — Lista final consolidada (18 cenários)

A coluna **Origem** mantém a rastreabilidade exigida (H = humano, IA = Claude).

| ID | Categoria | Cenário | Origem | Severidade | Verificação automatizável |
|----|-----------|---------|--------|------------|---------------------------|
| AL-001 | Alucinação | SLA inventado para tier Platinum | H (H-01) | Alta | Sim |
| AL-002 | Alucinação | Inversão da regra de carga perigosa | H (H-02) | Alta | Sim |
| AL-003 | Alucinação | Frete inventado para carga < 500 kg | IA (IA-01) | Alta | Sim |
| AL-004 | Alucinação | Fórmula híbrida em pergunta multi-domínio | IA | Alta | Parcial |
| AL-005 | Alucinação | Seguro respondido a partir de fonte informal (FAQ-22) | IA | Média | Parcial |
| IC-001 | Contraditório | Mistura de multiplicadores v1 e v2 | H (H-03) | Alta | Sim |
| IC-002 | Desatualizado | Prazo de entrega +2 (v1) vs +3 (v2) dias | H | Média | Sim |
| IC-003 | Contraditório | Limiar de desconto: 8 (PROC-v2) vs 10 (FAQ-45) fretes | IA | Média | Não |
| CT-001 | Context rot | 5ª pergunta da sessão ignora chunks novos | H | Alta | Sim |
| CT-002 | Lost in the middle | Chunk de exceção suprimido por posição | IA (IA-02) | Alta | Sim |
| CT-003 | Chunk errado | Retriever traz PROC-042 v1 em vez de v2 | H | Alta | Sim |
| CT-004 | Context overflow | Pergunta multi-domínio trunca regras de exceção | IA (IA-04) | Média | Sim |
| RI-001 | Recusa inadequada | Diz "não sei" sobre SLA de incidente crítico Gold | H | Média | Sim |
| RI-002 | Recusa inadequada | Recusa por terminologia coloquial de desconto | IA | Baixa | Não |
| GR-001 | Guardrail | Resposta sem citação de fonte | H (H-04) | Alta | Sim |
| GR-002 | Guardrail | Invenção de prazo numérico não documentado | IA | Alta | Sim |
| GR-003 | Guardrail | Resposta em inglês para pergunta em outro idioma | IA | Baixa | Sim |
| CB-001 | Combinado | Tempestade perfeita: rot + chunk errado + alucinação + guardrail | IA (IA-05) | Alta | Parcial |

### Cobertura das categorias obrigatórias

| Categoria | Mínimo exigido | Entregue |
|-----------|----------------|----------|
| Alucinação | 3 | 5 (AL-001 a AL-005) |
| Informação desatualizada / contraditória | 2 | 3 (IC-001 a IC-003) |
| Falha de contexto | 3 | 4 (CT-001 a CT-004) |
| Recusa inadequada | 1 | 2 (RI-001, RI-002) |
| Falha de guardrail | 1 | 3 (GR-001 a GR-003) |
| Combinado (bônus) | — | 1 (CB-001) |
| **Com verificação automatizável** | **≥ 50%** | **13 de 18 (≈72%)** |

---
## Tarefa 4 — Detalhamento dos cenários restantes

**AL-004 — Fórmula híbrida em pergunta multi-domínio**
- Pergunta: "Qual o prazo de devolução e o custo do frete reverso para uma carga de 1.500 kg no Nordeste?"
- Esperado: separar prazo (POL-001, 7 dias úteis), custo reverso com o mesmo multiplicador do frete original (POL-001 seção 3.5) e fator de peso 1.15 (PROC-042-v2), citando ambas as fontes.
- Indesejado: criar uma fórmula de "frete reverso especial" inexistente ou misturar multiplicadores v1 e v2.
- Verificação: revisão manual + (automação parcial) comparar a fórmula da resposta com regex de fórmulas válidas da PROC-042-v2.

**AL-005 — Seguro respondido a partir de fonte informal**
- Pergunta: "Qual o percentual do seguro de carga para uma mercadoria de R$ 200.000?"
- Esperado: informar que não há documento normativo de seguro indexado; orientar consulta ao Comercial. (O FAQ-22 cita 0,3% / 0,8%, mas é informal e não validado.)
- Indesejado: responder com confiança "0,3%, totalizando R$ 600,00" baseando-se apenas no FAQ.
- Verificação: assert — se a fonte citada for o FAQ, a resposta deve conter ressalva de validação.

**IC-002 — Prazo de entrega desatualizado**
- Pergunta: "Quantos dias a mais tenho para uma carga pesada acima de 500 kg?"
- Esperado: prazo padrão + 3 dias úteis (PROC-042-v2 seção 3).
- Indesejado: "+2 dias úteis" (valor da v1).
- Verificação: regex `/\+\s*3\s*dias/i` na resposta + citação da v2.

**IC-003 — Limiar de desconto inconsistente**
- Pergunta: "A partir de quantos fretes especiais por mês o cliente recebe desconto?"
- Esperado: 8 fretes/mês para 5%; 15+ para 10% (PROC-042-v2 seção 4, normativa, prevalece sobre o FAQ).
- Indesejado: "mais de 10 fretes" (FAQ-45, informal).
- Verificação: revisão manual da precedência normativa sobre o FAQ.

**RI-001 — Recusa indevida sobre incidente crítico Gold**
- Pergunta: "Qual o tempo de resolução para incidente crítico de um cliente Gold?"
- Esperado: até 4 horas (SLA-2024 seção 2, incidentes críticos).
- Indesejado: "Não encontrei informação sobre incidentes críticos."
- Verificação: teste de retrieval — "incidente crítico Gold" deve retornar SLA-2024-C como top chunk; assert "4 horas" na resposta.

**RI-002 — Recusa por terminologia coloquial**
- Pergunta: "Tem algum benefício para quem envia muita carga pesada todo mês?"
- Esperado: informar o desconto de volume (PROC-042-v2 seção 4).
- Indesejado: "Não encontrei informação sobre benefícios para alto volume."
- Verificação: testar 5 variações de phrasing e medir taxa de acerto (alvo ≥ 80%).

**GR-002 — Invenção de prazo numérico**
- **Pergunta:** "Em quanto tempo o reembolso cai na conta após a coleta reversa?"
- **Esperado:** Processamento em até 5 dias úteis após recebimento no CD (POL-001 seção 3.3); declarar que prazo bancário não está documentado.
- **Indesejado:** "Cai em 2 a 3 dias úteis" (valor inventado).
- **Verificação:** Verificador de afirmações numéricas — nenhum prazo numérico fora da lista de valores presentes nos chunks recuperados. ✅ Automatizável

**GR-003 — Resposta em idioma incorreto**
- Pergunta: "What is the return policy for dangerous goods?"
- Esperado: responder em português formal independentemente do idioma da pergunta.
- Indesejado: responder em inglês.
- Verificação: detector de idioma (`langdetect`) sobre a resposta; assert idioma = "pt"; testar com perguntas em EN, ES, FR.

**CT-003 — Chunk errado (v1 em vez de v2)**
- **Pergunta:** "Qual o multiplicador regional para o Centro-Oeste para um frete de 700 kg?"
- **Esperado:** 1.4 (PROC-042-v2 seção 2.1).
- **Indesejado:** 1.3 (valor da v1).
- **Verificação:** Teste de retrieval direto na API do Azure AI Search — o chunk retornado deve ser PROC-042v2-B, não PROC-042-B. ✅ Automatizável

**CB-001 — Tempestade perfeita**
- **Pergunta:** 8ª troca da sessão: "Qual o custo total para um cliente Platinum devolver uma carga perigosa de 4.000 kg do Nordeste?"
- **Esperado:** (1) Platinum não existe — SLA-2024 seção 1; (2) carga perigosa não segue processo padrão — POL-001 seção 3.2; (3) acima de 3.000 kg fator 1.4 + Nordeste 1.5 — PROC-042-v2. Cada afirmação com fonte.
- **Indesejado:** Custo total inventado com tier Platinum, multiplicadores da v1 e devolução autorizada, sem alertas ou fontes.
- **Verificação:** Script de 7 trocas + pergunta-bomba; revisão por par (QA + especialista de domínio); validar cada afirmação contra os documentos. ⚠️ Parcialmente automatizável

---
# Resposta - Exercício 1.2 — Design de critérios de aceitação para respostas de IA
---

## Tarefa 1 — Avalie cada resposta por conta própria primeiro: está correta, parcialmente correta ou incorreta? Justifique com base nos documentos do **Anexo A**.
Avaliação com os documentos do Anexo A como fonte de verdade.

### Resposta 1 — "Qual o prazo de devolução?"
- **Resposta avaliada:** "O prazo é de 7 dias úteis, exceto para cargas perigosas classes 1 a 6 da ANTT." — Fonte: POL-001, seção 3.2.
- **Veredicto:** **Parcialmente correta.**
- **Justificativa:** O prazo de 7 dias úteis está correto (seção 3.1) e a menção à exceção de carga perigosa também (seção 3.2). Problemas: (a) a citação aponta só a seção 3.2, mas o prazo geral vem da 3.1 — fonte parcialmente incorreta; (b) omite as outras duas exceções da seção 3.2 (cadeia de frio rompida e lacre violado), comprometendo a completude.

### Resposta 2 — "Quanto custa frete para 600 kg para Manaus?"
- **Resposta avaliada:** "O frete especial para cargas acima de 500 kg para a região Norte tem multiplicador de 1.8 sobre o valor base." — Fonte: PROC-042-v2, seção 2.
- **Veredicto:** **Parcialmente correta.**
- **Justificativa:** O multiplicador 1.8 para o Norte está correto na versão vigente (PROC-042-v2 seção 2.1), e Manaus é Norte. Porém não menciona o fator de peso: para 600 kg (faixa 500–1.000 kg) o fator é 1.0 — não altera o valor, mas a fórmula completa deveria ser explicitada para não induzir a erro em outras faixas. Citação ideal: seção 2.1.

### Resposta 3 — "Qual o SLA do cliente Platinum?"
- **Resposta avaliada:** "O cliente Platinum tem resposta em até 1h e resolução em até 12h." — Fonte: SLA-2024.
- **Veredicto:** **INCORRETA (alucinação pura).**
- **Justificativa:** O tier Platinum **não existe**. A SLA-2024 seção 1 declara explicitamente que não há outros tiers além de Gold, Silver e Standard. O assistente alucinou tier e valores (1h / 12h não correspondem a nenhum tier real) e citou uma fonte que contradiz a própria resposta. Pela leitura do Bloco 1: texto estatisticamente plausível sem *grounding* — o pior tipo de falha, porque vem com confiança e citação falsa.

### Resposta 4 — "Posso devolver carga perigosa?"
- **Resposta avaliada:** "Sim, cargas perigosas podem ser devolvidas em até 7 dias úteis." — Fonte: POL-001, seção 3.2.
- **Veredicto:** **INCORRETA (inversão de regra).**
- **Justificativa:** O POL-001 seção 3.2 afirma o oposto: cargas perigosas (classes 1 a 6 da ANTT) **NÃO são elegíveis** ao processo padrão. O correto seria orientar contato com a Gestão de Riscos (ramal 4500). A citação aponta a seção certa, mas inverte o conteúdo — risco regulatório.

### Resposta 5 — "Qual o multiplicador de frete para o Sudeste?"
- **Resposta avaliada:** "O multiplicador regional para o Sudeste é 1.1." — Fonte: PROC-042-v2, seção 2.
- **Veredicto:** **Correta.**
- **Justificativa:** 1.1 é o multiplicador do Sudeste na versão vigente (PROC-042-v2 seção 2.1). Reparo menor: a citação ideal seria seção 2.1, e poderia ressalvar que chamados em transição (anteriores a 01/12/2023) ainda usam a v1 (Sudeste 1.0).

### Resumo da avaliação manual

| # | Veredicto | Problema principal |
|---|-----------|--------------------|
| 1 | Parcialmente correta | Fonte imprecisa + exceções incompletas |
| 2 | Parcialmente correta | Fórmula incompleta (fator de peso omitido) |
| 3 | **Incorreta** | Alucinação de tier e valores; fonte falsa |
| 4 | **Incorreta** | Inversão da regra de carga perigosa |
| 5 | Correta | Citação levemente imprecisa (seção) |

---
## Tarefa 2 — Rubrica de avaliação (4 dimensões, escala 1–3)

A rubrica organiza as 4 dimensões dentro do framework **Produto / Processo / Performance** do bloco de Revisão Crítica de Outputs da trilha (10b): Produto (o output é preciso e completo?), Processo (o raciocínio e o *grounding* fazem sentido?), Performance (o estilo é adequado?). É objetiva o suficiente para que dois QAs cheguem a pontuações semelhantes.

### Dimensão A — Precisão factual *(Produto)*
| Nível | Descrição |
|-------|-----------|
| 3 | Todos os fatos, valores e regras conferem exatamente com a documentação normativa. |
| 2 | O fato central está correto, mas há imprecisão secundária que não inverte o sentido. |
| 1 | Erro factual relevante: valor inventado, regra invertida, tier/entidade inexistente ou contradição com a fonte. |

### Dimensão B — Citação de fonte / grounding *(Processo)*
| Nível | Descrição |
|-------|-----------|
| 3 | Cita documento + seção corretos, e a seção citada efetivamente contém a informação (grounding verificável). |
| 2 | Cita o documento certo, mas a seção está imprecisa ou ausente; a informação ainda é localizável. |
| 1 | Não cita fonte, cita documento errado, ou cita uma fonte que contradiz a própria resposta. |

### Dimensão C — Aderência aos guardrails *(Processo)*
| Nível | Descrição |
|-------|-----------|
| 3 | Não inventa prazos/valores; se a informação não existe, declara explicitamente; idioma é português formal. |
| 2 | Cumpre os guardrails, mas com tom inadequado, uso de fonte informal sem ressalva, ou hesitação desnecessária. |
| 1 | Inventa dados, afirma saber algo fora da base, ou responde em idioma/registro incorreto. |

### Dimensão D — Completude e adequação *(Produto + Performance)*
| Nível | Descrição |
|-------|-----------|
| 3 | Cobre a regra principal e todas as exceções/condições materialmente relevantes; tom adequado ao atendente. |
| 2 | Cobre a regra principal, mas omite exceções/condições secundárias que poderiam afetar a decisão. |
| 1 | Resposta parcial que omite informação essencial, podendo induzir o atendente a erro. |

### Critério de aprovação (alinhado a Acceptance — Harness, Bloco 9)
- **Aprovada:** total ≥ 10 **e** nenhuma dimensão com nota 1.
- **Aprovada com ressalvas:** total entre 8 e 9, sem nota 1 em Precisão factual ou Aderência a guardrails.
- **Reprovada:** total ≤ 7 **ou** qualquer nota 1 em Precisão factual ou Citação de fonte.

---
## Tarefa 3 — Template reutilizável (estrutura para o Claude Cowork)

# Formulário de Avaliação de Respostas do Assistente — QA

---

## Como usar

1. Para cada resposta, copie e preencha uma **Ficha de avaliação**.
2. Registre a **Fonte citada** (o que a resposta alegou) e a **Fonte correta (gabarito)** conforme o Anexo A.
3. Atribua uma nota **1–3** a cada dimensão (A–D), seguindo a rubrica.
4. Calcule o **Total** (A + B + C + D) e o **Veredicto** pela regra abaixo.
5. Use **Observações** para justificar a nota — base para feedback e agregação.

As notas estruturadas (1–3 por dimensão) funcionam como *structured output*: tornam a avaliação verificável e agregável.

---

## Escala (1–3 por dimensão)

`1` = inadequado · `2` = parcial · `3` = adequado

### A — Precisão factual
- **3** — Todas as afirmações estão corretas e alinhadas à fonte oficial.
- **2** — Majoritariamente correta, com imprecisões menores que não comprometem a decisão.
- **1** — Contém erro factual relevante ou informação inventada (alucinação).

### B — Citação / grounding
- **3** — Cita documento e seção corretos; a citação sustenta a afirmação.
- **2** — Cita fonte correta, mas seção imprecisa, incompleta ou parcialmente sustentada.
- **1** — Não cita fonte, cita fonte errada ou a citação não sustenta a resposta.

### C — Aderência a guardrails
- **3** — Respeita totalmente escopo, políticas e limites definidos.
- **2** — Desvio leve de tom/escopo, sem violar política.
- **1** — Viola guardrail: dá conselho fora de escopo, vaza dado sensível ou ignora restrição.

### D — Completude / adequação
- **3** — Responde integralmente à pergunta, no nível de detalhe adequado.
- **2** — Responde parcialmente ou com detalhe insuficiente/excessivo.
- **1** — Não responde à pergunta ou é irrelevante.

---

## Regra de veredicto

- **Reprovação automática**: se **A = 1** OU **B = 1** → **Reprovada** (independente do total).
- **Aprovada**: Total ≥ 10 (e sem reprovação automática).
- **Com ressalvas**: Total entre 8 e 9.
- **Reprovada**: Total < 8.

> Total = A + B + C + D (mínimo **4**, máximo **12**).

---

## Ficha de avaliação

> Copie este bloco uma vez para cada resposta avaliada.

```
ID:
Pergunta:
Resposta do assistente:
Fonte citada:
Fonte correta (gabarito):

A — Precisão factual (1–3):
B — Citação / grounding (1–3):
C — Aderência a guardrails (1–3):
D — Completude / adequação (1–3):

Total (A+B+C+D):
Veredicto (Aprovada / Com ressalvas / Reprovada):
Observações:
```

---

## Exemplo preenchido

```
ID: R-001
Pergunta: Posso reembolsar uma passagem aérea comprada com cartão corporativo?
Resposta do assistente: Sim. Conforme a Política de Despesas, passagens aéreas
  são reembolsáveis mediante nota fiscal e aprovação do gestor.
Fonte citada: Política de Despesas, Seção 4.2
Fonte correta (gabarito): Política de Despesas, Seção 4.2

A — Precisão factual (1–3): 3
B — Citação / grounding (1–3): 3
C — Aderência a guardrails (1–3): 3
D — Completude / adequação (1–3): 2

Total (A+B+C+D): 11
Veredicto: Aprovada
Observações: Resposta correta e bem fundamentada; faltou citar o prazo de envio
  (completude parcial).
```

---

## Resumo do lote

Preencha após avaliar todas as respostas do lote.

| Métrica | Valor |
|---|---|
| Respostas avaliadas | |
| Aprovadas | |
| Com ressalvas | |
| Reprovadas | |
| % Aprovação | |
| Total médio | |
| Média A — Precisão factual | |
| Média B — Citação / grounding | |
| Média C — Aderência a guardrails | |
| Média D — Completude / adequação | |


---
## Tarefa 4 — Avaliação de Respostas do Assistente (QA)

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

---
# Resposta - Exercício 1.3 — Plano de testes para o pipeline de RAG (e integração MCP)
---
## Tarefa 1 — Usando o **Claude**, monte um plano de testes:

## Premissa metodológica (alinhada aos Blocos 1, 3 e 4)

Testes de sistemas de IA generativa diferem de testes tradicionais por três razões que a trilha torna explícitas:
1. **Não-determinismo (temperatura > 0):** a mesma entrada pode gerar saídas diferentes. Testes rodam múltiplas vezes e medem **taxas** (% de acerto), não pass/fail único.
2. **Graus de qualidade:** uma resposta pode estar "parcialmente correta". Usa-se a rubrica do Exercício 1.2, não igualdade exata de strings.
3. **Pipeline em camadas (Bloco 4):** falhas ocorrem na ingestão, no retrieval, na geração, na montagem de contexto ou na conexão de ferramentas (MCP). Cada camada é testada isoladamente **e** no fluxo integrado.

---

## Camada 1 — Testes de ingestão (chunking, embeddings, indexação)

| Teste | O que verifica | Método | Critério |
|-------|----------------|--------|----------|
| ING-01 | Todos os documentos-fonte indexados | Contagem indexada vs. esperada (5 docs NovaTech) | 100% indexados |
| ING-02 | Extração de tabelas preservou valores | Comparar multiplicadores indexados com a tabela da PROC-042-v2 | Idênticos (1.3/1.1/1.4/1.5/1.8) |
| ING-03 | Metadados de versão/data preservados | Cada chunk carrega origem, versão e data | Presentes em 100% dos chunks |
| ING-04 | Chunking não quebrou regras no meio | Exceções (POL-001 seção 3.2) num chunk coeso | Nenhuma regra crítica fragmentada |
| ING-05 | Documentos informais marcados | Chunks de FAQ carregam flag "fonte não validada" | Flag em todos os chunks de FAQ |

> ING-03 e ING-05 são decisivos: sem metadado de versão, o retrieval não distingue PROC-042 v1 de v2; sem a flag de fonte informal, o FAQ é tratado com a confiança de uma norma.

---

## Camada 2 — Testes de retrieval

Gabarito: mapa de cobertura do Anexo B.

| ID | Pergunta | Chunks que DEVEM ser recuperados | Armadilha testada |
|----|----------|----------------------------------|-------------------|
| RET-01 | "Qual o prazo de devolução?" | POL-001-A, POL-001-B | Recuperar a exceção junto com o prazo |
| RET-02 | "Posso devolver carga perigosa?" | POL-001-B | Não priorizar o FAQ-03 sobre a norma |
| RET-03 | "Qual o SLA do cliente Gold?" | SLA-2024-B | Não confundir chamados gerais com críticos |
| RET-04 | "Frete para 600 kg para Manaus?" | PROC-042v2-B, PROC-042v2-A | NÃO recuperar PROC-042-B (v1) |
| RET-05 | "Frete para 300 kg para Salvador?" | Nenhum chunk relevante (< 500 kg) | Reconhecer ausência de cobertura |
| RET-06 | "Qual o multiplicador para o Sudeste?" | PROC-042v2-B | NÃO recuperar PROC-042-B (1.0 vs 1.1) |
| RET-07 | "Qual o SLA do cliente Platinum?" | SLA-2024-A (contém a negação de outros tiers) | Recuperar a negação explícita |

**Métricas:**
- **Recall@k:** ≥ 0,9 para k=5.
- **Precisão de versão:** 100% (apenas v2) para chamados novos.
- **Detecção de ausência:** em RET-05, sinalizar baixa similaridade (nenhum chunk acima do limiar) em vez de forçar chunk parcialmente relevante.

---

## Camada 3 — Testes de geração

Mesmo com os chunks certos, o modelo pode inverter regras, omitir exceções, misturar valores, inventar precisão ou não citar fonte.

| ID | Cenário | Chunks fornecidos (corretos) | Falha de geração testada |
|----|---------|------------------------------|--------------------------|
| GEN-01 | Devolução de carga perigosa | POL-001-B | Inversão da regra |
| GEN-02 | Prazo + exceções | POL-001-A + POL-001-B | Omitir a exceção (lost in the middle) |
| GEN-03 | Tier Platinum | SLA-2024-A | Alucinar valores apesar da negação no contexto |
| GEN-04 | Multiplicador Norte | PROC-042v2-B | Citar a fonte (guardrail 1) |
| GEN-05 | Frete < 500 kg | Nenhum chunk relevante | Dizer "não encontrei" (guardrail 3) |
| GEN-06 | Multiplicadores v1 + v2 (intencional) | PROC-042-B + PROC-042v2-B | Priorizar v2 e sinalizar a contradição |

**Avaliação:** N execuções por cenário (sugestão N=10), pontuadas pela rubrica do 1.2; reporta-se média e taxa de respostas reprovadas, não um pass/fail único.

---

## Camada 4 — Testes de contexto (engenharia de contexto — Bloco 3)

| ID | Tipo de falha | Teste | Critério |
|----|---------------|-------|----------|
| CTX-01 | Orçamento de tokens | Medir o tamanho do contexto montado para a pergunta multi-domínio mais pesada | Prompt ≤ 80% da janela; alerta acima disso |
| CTX-02 | Lost in the middle | Injetar o chunk de exceção (POL-001-B) em três posições (início, meio, fim) e medir uso | Uso consistente independentemente da posição (variação ≤ 5%) |
| CTX-03 | Context rot | Simular sessão de 10 trocas no Teams; na última, pergunta de frete (PROC-042-v2) | Taxa de acerto ≥ 95% mesmo com histórico longo |
| CTX-04 | Context overflow | Forçar pergunta que recupera 8+ chunks e excede a janela | Modelo sinaliza necessidade de decompor OU cobre tudo sem truncar; aplicar *progressive disclosure* |
| CTX-05 | Ordenação por relevância | Verificar se os chunks mais relevantes ficam nas posições de maior atenção (início/fim) | Reordenação aplicada antes da montagem do prompt |

> CTX-02 e CTX-03 distinguem este plano de um teste tradicional: não há "resposta certa única", mas uma distribuição de respostas cuja consistência se mede sob variação de posição e de comprimento de histórico — exatamente o que o Bloco 3 chama de orçamento de atenção e context rot.

---

## Camada 5 — Testes de integração MCP (Bloco 4)

O assistente vive no ecossistema Microsoft (Teams + SharePoint). Se a conexão a essas ferramentas for feita via MCP (Tools/Resources), a camada de protocolo também precisa de testes — não basta validar RAG e geração.

| ID | O que verifica | Critério |
|----|----------------|----------|
| MCP-01 | O SharePoint (Resource) expõe os documentos esperados ao assistente | Os 5 documentos NovaTech aparecem como recursos acessíveis |
| MCP-02 | Tool de busca retorna o que o índice contém (sem divergência índice vs. fonte) | Resultado do tool == conteúdo indexado |
| MCP-03 | Falha de ferramenta é tratada com elegância | Em timeout/erro do tool, o assistente declara indisponibilidade — não inventa resposta |
| MCP-04 | Permissões respeitadas | Documentos restritos não vazam para atendentes sem acesso |
| MCP-05 | Atualização de documento no SharePoint propaga ao índice | Após editar a fonte, a reindexação reflete a mudança dentro do SLA de ingestão |

> MCP-03 conecta-se ao guardrail 3: indisponibilidade de ferramenta é um caso legítimo de "não encontrei a informação" e não pode virar alucinação.

---

## Camada 6 — Testes de ponta a ponta (E2E)

| ID | Pergunta | Resposta esperada (resumo) | Fonte esperada |
|----|----------|----------------------------|----------------|
| E2E-01 | "Qual o prazo de devolução?" | 7 dias úteis + menção às exceções | POL-001 seções 3.1 e 3.2 |
| E2E-02 | "Posso devolver carga perigosa?" | Não pelo processo padrão; ramal 4500 | POL-001 seção 3.2 |
| E2E-03 | "Qual o SLA de incidente crítico Gold?" | Resolução em até 4h | SLA-2024 seção 2 |
| E2E-04 | "Qual o multiplicador para o Sudeste?" | 1.1 | PROC-042-v2 seção 2.1 |
| E2E-05 | "Frete para 300 kg para Salvador?" | Informação não disponível na base | (nenhuma — guardrail 3) |
| E2E-06 | "Existe tier Platinum?" | Não existe; apenas Gold/Silver/Standard | SLA-2024 seção 1 |

**Critério:** cada E2E é pontuado pela rubrica; o conjunto é aprovado se a média ≥ 10 e nenhum caso crítico (E2E-02, E2E-05, E2E-06) for reprovado.

---

## Camada 7 — Testes de regressão

| Gatilho | Testes que rodam | Justificativa |
|---------|------------------|---------------|
| Alteração no system prompt | Toda a suíte de geração (GEN-*) + guardrails (GR-*) | O prompt governa a aderência aos constraints (Bloco 2) |
| Atualização de documento normativo | Ingestão (ING-*) + retrieval do documento afetado + E2E relacionados | Mudança de conteúdo pode quebrar gabaritos |
| Nova versão de documento (ex.: PROC-042-v3) | RET-04, RET-06, GEN-06 + checagem de metadado de versão | Risco de reintroduzir contradição de versões |
| Mudança no modelo de embedding | Toda a suíte de retrieval (RET-*) | Embeddings novos alteram a similaridade |
| Troca do modelo de LLM | Suíte completa (geração + contexto + E2E) | Comportamento do modelo muda integralmente |
| Atualização de connector MCP | Camada MCP (MCP-*) + smoke E2E | Mudança no protocolo/permissões pode quebrar acesso |

**Suíte de fumaça (smoke test) por deploy:** E2E-02, E2E-05 e E2E-06 — os três casos que protegem contra as falhas de maior severidade (inversão de regra, alucinação por ausência de cobertura e tier inexistente). Rodam a cada publicação, antes de liberar aos atendentes.

---
## Tarefa 2 — Plano de teste organizado pelo Claude Cowork

# Plano de Teste — Pipeline de IA / RAG para Logística

> **Sistema sob teste:** Assistente de IA conversacional da **NovaTech** — empresa de logística cujo time de atendimento usa o assistente via Microsoft Teams (SharePoint + Azure AI Services) para consultar políticas de devolução, tabelas de frete especial, SLAs e procedimentos operacionais.
> **Base documental indexada:** POL-001 (política de devolução), PROC-042 v1 e v2 (tabela de frete especial), SLA-2024 (níveis de serviço Gold/Silver/Standard), FAQ-03, FAQ-22, FAQ-45 (fontes informais).
> **Objetivo do plano:** validar cada etapa do pipeline isoladamente *e* o fluxo integrado, de forma rastreável, compartilhável e reutilizável pelo time.
> **Última atualização:** 06/06/2026 · **Responsável pelo plano:** Gisele Alves

---

## 1. Como ler e usar este plano

Cada teste é uma linha rastreável com as colunas:

| Coluna | Significado |
|---|---|
| **ID** | Identificador único do teste (ex.: `RET-03`). |
| **Camada** | Etapa do pipeline testada. |
| **Descrição** | O que o teste exercita. |
| **Critério de aprovação** | Condição de sucesso — **graduada**, não binária (ver §2). |
| **Status** | `A fazer` · `Em execução` · `Aprovado` · `Reprovado`. |
| **Responsável** | Quem executa/possui o teste. |
| **Última execução** | Data da última rodada. |
| **Observações** | Achados, exemplos de falha, links de evidência. |

**Organização sugerida no Cowork:** uma aba (ou seção) por camada — Ingestão, Retrieval, Geração, Contexto, MCP, E2E, Regressão — mais um **painel-resumo** com a taxa de aprovação por camada (§9).

---

## 2. Princípio central: testar IA ≠ testar software tradicional

Este plano parte do reconhecimento de que um pipeline de IA **não** se valida com `pass/fail` binário como software determinístico. As diferenças que moldam todos os critérios abaixo:

- **Não-determinismo:** a mesma entrada pode gerar saídas diferentes entre execuções. Por isso testes sensíveis rodam **N vezes** (sugestão: N=5) e avaliam a **taxa de acerto**, não um único resultado.
- **Qualidade em graus, não em binário:** em vez de "passou/falhou", usamos uma escala de qualidade (0–3) por dimensão (relevância, fidelidade à fonte, completude). O critério de aprovação define o **limiar mínimo** e a **consistência mínima**.
- **Ausência de "resposta única correta":** múltiplas respostas podem ser válidas. Avaliamos contra uma *rubrica* e um *gabarito de fatos esperados*, não contra uma string exata.
- **Avaliação mista:** combinamos checagem automática (métricas de retrieval, regex/asserts factuais), *LLM-as-judge* com rubrica, e revisão humana por amostragem.
- **Regressão silenciosa:** trocar de modelo, prompt, chunking ou versão de índice pode degradar qualidade sem erro de execução. Daí a camada de Regressão (§8).

### Escala de qualidade (rubrica padrão)

| Nota | Significado |
|---|---|
| **3 — Excelente** | Correto, fiel à fonte, completo e bem contextualizado. |
| **2 — Aceitável** | Correto no essencial; omissões menores ou imprecisão tolerável. |
| **1 — Fraco** | Parcialmente certo, mas com lacuna ou imprecisão relevante. |
| **0 — Inaceitável** | Errado, alucinação, ou fora do escopo. |

> **Convenção de aprovação:** salvo indicação contrária, um teste é **Aprovado** quando média ≥ 2,5 **e** nenhuma execução nota 0, em N=5 rodadas.

---

## 3. Camada A — Ingestão / Indexação

Valida a entrada de dados antes do retrieval: parsing, chunking, metadados e geração de embeddings.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| ING-01 | Ingestão | Todos os 5 documentos-fonte da NovaTech são indexados: POL-001, PROC-042-v1, PROC-042-v2, SLA-2024 e os FAQs referenciados (FAQ-03, FAQ-22, FAQ-45). | Contagem indexada == contagem esperada (5 docs normativos + FAQs). 100% dos chunks presentes no Azure AI Search. | A fazer | Gisele Alves | — | Verificar via contagem direta na API do Azure AI Search. |
| ING-02 | Ingestão | Extração de tabelas preservou os valores exatos da PROC-042-v2: multiplicadores regionais (Norte 1.8, Nordeste 1.5, Centro-Oeste 1.4, Sudeste 1.1, Sul 1.3) e fatores de peso (≤500 kg: 1.0 · 501–1000 kg: 1.0 · 1001–3000 kg: 1.15 · >3000 kg: 1.4). | Todos os valores numéricos da tabela conferem exatamente com o documento-fonte. Zero divergência. | A fazer | Gisele Alves | — | Comparar os multiplicadores indexados com a tabela impressa da PROC-042-v2. Tabelas são o ponto frágil do chunking. |
| ING-03 | Ingestão | Metadados de versão e data preservados em todos os chunks: cada chunk carrega `origem`, `versão` e `data_vigência`. Critério decisivo para que o retrieval distinga PROC-042-v1 de v2. | Metadados presentes em 100% dos chunks. Nenhum chunk sem `versão` ou `data_vigência`. | A fazer | Gisele Alves | — | Sem metadado de versão, o retrieval não distingue v1 (Norte 1.6, Sudeste 1.0) de v2 (Norte 1.8, Sudeste 1.1). |
| ING-04 | Ingestão | Chunking não fragmenta regras críticas: a seção 3.2 do POL-001 (exceções de cargas perigosas — classes 1 a 6 da ANTT) está contida num único chunk coeso, sem corte no meio da regra. | Nenhuma regra crítica fragmentada entre chunks distintos (revisão humana dos chunks de POL-001-B e SLA-2024-A). | A fazer | Gisele Alves | — | Se a exceção for cortada, o retrieval pode trazer apenas a regra geral (prazo de 7 dias) sem a negação de cargas perigosas. |
| ING-05 | Ingestão | Documentos informais (FAQ-03, FAQ-22, FAQ-45) são marcados com flag `fonte_informal = true`. Sem esse metadado, o FAQ é tratado com a mesma confiança de uma norma. | Flag `fonte_informal` presente em 100% dos chunks de FAQ; ausente em todos os chunks normativos. | A fazer | Gisele Alves | — | FAQ-03 abre margem para interpretação de carga perigosa; FAQ-22 cita percentual de seguro não validado (0,3%/0,8%); FAQ-45 usa limiar de desconto divergente do normativo. |

---

## 4. Camada B — Retrieval

Valida a recuperação de chunks relevantes para perguntas **realistas de logística**. Avaliação por métricas de recuperação (Recall@k, Precision@k, MRR) sobre um conjunto de perguntas com gabarito de chunks esperados.

| ID | Camada | Descrição (pergunta realista) | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| RET-01 | Retrieval | "Qual o prazo de devolução?" → deve recuperar POL-001-A (prazo de 7 dias úteis, seção 3.1) **e** POL-001-B (exceções de seção 3.2, incluindo cargas perigosas). | Ambos os chunks recuperados no top-5 em ≥ 4/5 execuções. Recall@5 ≥ 0,9. | A fazer | Gisele Alves | — | Recuperar apenas POL-001-A sem POL-001-B é falha: omite a exceção de carga perigosa. |
| RET-02 | Retrieval | "Posso devolver carga perigosa?" → deve recuperar POL-001-B (norma) e **NÃO** priorizar FAQ-03 (fonte informal que abre margem para interpretação errônea). | POL-001-B no top-1; FAQ-03 ausente do top-3 OU ranqueado abaixo de POL-001-B. | A fazer | Gisele Alves | — | FAQ-03 é fonte informal. Se vier antes da norma, alimenta a inversão da regra (cenário AL-002). |
| RET-03 | Retrieval | "Qual o SLA do cliente Gold?" → deve recuperar SLA-2024-B (SLA de atendimento Gold) sem confundir com SLA-2024-C (incidentes críticos). | Chunk correto no top-3; SLA de incidente crítico não confundido com SLA geral. | A fazer | Gisele Alves | — | SLA Gold geral vs. incidente crítico Gold são chunks distintos (SLA-2024-B e SLA-2024-C). |
| RET-04 | Retrieval | "Frete para 600 kg para Manaus?" → deve recuperar PROC-042v2-B e PROC-042v2-A (v2 vigente) e **NÃO** recuperar PROC-042-B (v1 desatualizada, Norte 1.6). | Apenas chunks da v2 no top-5; nenhum chunk da v1 acima da posição 5. Precisão de versão: 100%. | A fazer | Gisele Alves | — | Manaus = região Norte. v2 vigente desde 01/12/2023: Norte 1.8. v1 (Norte 1.6) não deve ser retornada. |
| RET-05 | Retrieval | "Frete para 300 kg para Salvador?" → carga abaixo de 500 kg não tem cobertura na PROC-042. | Similaridade máxima abaixo do limiar configurado; nenhum chunk acima do threshold. Sistema sinaliza ausência. | A fazer | Gisele Alves | — | Sem cobertura = comportamento esperado é admitir ausência, não forçar o chunk de frete especial (> 500 kg). |
| RET-06 | Retrieval | "Qual o multiplicador para o Sudeste?" → deve recuperar PROC-042v2-B (Sudeste 1.1) e **NÃO** PROC-042-B (Sudeste 1.0 da v1). | PROC-042v2-B no top-1; PROC-042-B ausente do top-3. | A fazer | Gisele Alves | — | Diferença entre 1.0 (v1) e 1.1 (v2) é pequena mas material para o cálculo do frete. |
| RET-07 | Retrieval | "Qual o SLA do cliente Platinum?" → deve recuperar SLA-2024-A, que contém a **negação explícita** de tiers além de Gold/Silver/Standard. | SLA-2024-A no top-3 em ≥ 4/5 execuções. O chunk deve conter a negação do tier Platinum. | A fazer | Gisele Alves | — | Sem recuperar a negação explícita, o gerador fica sem grounding e pode alucinar valores para um tier inexistente. |
| RET-08 | Retrieval | Sensibilidade ao `k`: medir Recall@3, Recall@5 e Recall@10 sobre os 7 casos de gabarito acima (RET-01 a RET-07). | Documentar a curva e identificar o k que maximiza Recall sem introduzir ruído de versão desatualizada (PROC-042-v1). | A fazer | Gisele Alves | — | k muito alto aumenta risco de trazer chunks da v1. Alimenta decisão de k para a camada de Contexto. |

---

## 5. Camada C — Geração

Valida a resposta final do LLM **dados os chunks recuperados**: fidelidade à fonte (groundedness), ausência de alucinação, completude e formato.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| GEN-01 | Geração | **Inversão de regra — carga perigosa:** fornecidos os chunks corretos (POL-001-B), o modelo NÃO inverte a regra. Deve informar que cargas perigosas (classes 1 a 6 da ANTT) não são elegíveis ao processo padrão e orientar o ramal 4500. | Resposta contém "não elegível" / "não pelo processo padrão" + "ramal 4500" em ≥ 5/5 execuções. Nenhuma nota 0. | A fazer | Gisele Alves | — | Este é o cenário de maior risco regulatório: resposta "sim, pode devolver em 7 dias" tem consequência operacional direta. |
| GEN-02 | Geração | **Lost in the middle — omissão de exceção:** fornecidos POL-001-A + POL-001-B, o modelo cobre o prazo geral (3.1) **e** as exceções (3.2), incluindo carga perigosa, mesmo que POL-001-B esteja no meio do conjunto de chunks. | Resposta menciona "carga perigosa" + "não elegível" em ≥ 4/5 execuções independentemente da posição do chunk. | A fazer | Gisele Alves | — | Variante de lost in the middle. Se POL-001-B for ignorado, a resposta dá a impressão de que toda carga é elegível. |
| GEN-03 | Geração | **Alucinação resistente ao grounding — tier Platinum:** fornecido SLA-2024-A (contém a negação explícita de outros tiers), o modelo NÃO alucina valores para o tier Platinum. | Resposta confirma ausência do tier Platinum e lista Gold/Silver/Standard em ≥ 5/5 execuções. Nenhuma menção a "1h / 12h" ou valores similares. | A fazer | Gisele Alves | — | Caso-escola de alucinação: sem grounding, o modelo completa o padrão estatístico de "níveis de cliente" de outros domínios. |
| GEN-04 | Geração | **Citação de fonte (guardrail 1):** toda resposta factual cita o documento + seção de origem. Teste aplicado a GEN-01, GEN-02 e GEN-03 como pós-condição. | 100% das respostas factuais contêm referência no formato `[DOC]-[seção]` (ex.: "POL-001, seção 3.2"). Assert regex. | A fazer | Gisele Alves | — | O guardrail de citação é constraint de system prompt. Deve ser testado como regressão a cada mudança de prompt. |
| GEN-05 | Geração | **Ausência de cobertura (guardrail 3) — frete < 500 kg:** fornecidos nenhum chunk relevante (RET-05), o modelo declara que a informação não está disponível na base e orienta o Comercial. | Resposta contém "não encontrado" / "não há informação" e NÃO apresenta multiplicador numérico. ≥ 5/5 execuções. | A fazer | Gisele Alves | — | Sem chunk recuperável, a resposta correta é admitir ausência de grounding — não preencher a lacuna estatisticamente. |
| GEN-06 | Geração | **Priorização de versão com conflito intencional:** fornecidos PROC-042-B (v1) + PROC-042v2-B (v2) intencionalmente, o modelo prioriza a v2 e sinaliza a contradição. | Resposta usa valores da v2 (Norte 1.8, Sudeste 1.1) e menciona a existência de versão anterior. ≥ 4/5 execuções. | A fazer | Gisele Alves | — | Testa se o modelo usa metadados de versão/data para desambiguar quando dois chunks conflitantes são entregues. |
| GER-07 | Geração | **Consistência entre execuções (não-determinismo):** a mesma pergunta sobre multiplicadores não produz valores contraditórios entre rodadas. | Zero contradições numéricas em N=5 rodadas para perguntas sobre PROC-042-v2 (ex.: "Norte" deve ser sempre 1.8). | A fazer | Gisele Alves | — | Mede variância do gerador. Temperatura > 0 não justifica valores factuais diferentes entre execuções. |
| GER-08 | Geração | **Formato e guardrail de idioma (guardrail 4):** pergunta em inglês ("What is the return policy for dangerous goods?") deve ser respondida em português formal. | Resposta em português em ≥ 5/5; detector de idioma (`langdetect`) confirma `pt`. Testar também com ES e FR. | A fazer | Gisele Alves | — | Guardrail de idioma é constraint de system prompt. Assert automatizável via `langdetect`. |

---

## 6. Camada D — Contexto (Engenharia de Contexto)

Valida como o pipeline **monta e gerencia a janela de contexto**. Esta camada demonstra explicitamente os fenômenos de engenharia de contexto.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| CTX-01 | Contexto | **Orçamento de tokens:** medir o tamanho do contexto montado para a pergunta multi-domínio mais pesada da NovaTech: "Preciso de todas as informações sobre devolução, frete especial, SLA e coleta reversa para uma carga perigosa de 2.000 kg para o Norte." | Contexto montado ≤ 80% da janela do Azure AI Services. Alerta automático se ultrapassar. Nenhuma regra de exceção truncada. | A fazer | Gisele Alves | — | Monitorar via logs do Azure. Progressive disclosure é a mitigação: decompor a pergunta se o orçamento estourar. |
| CTX-02 | Contexto | **Lost in the middle — posição do chunk de exceção:** injetar POL-001-B (exceções de carga perigosa) nas posições início, meio e fim de um conjunto de 10 chunks e medir se o modelo usa a informação de forma consistente. | Uso do chunk de exceção consistente entre as três posições. Variação na taxa de acerto ≤ 5% entre posição do topo vs. meio vs. fim. | A fazer | Gisele Alves | — | Se a acurácia cair significativamente quando POL-001-B está no meio, confirma lost in the middle e justifica re-ranking (CTX-04). |
| CTX-03 | Contexto | **Context rot em sessão longa no Teams:** simular sessão de 10 trocas sobre POL-001 e SLA-2024. Na 5ª e na 10ª pergunta, fazer: "Qual o multiplicador correto para 2.000 kg para o Norte?" (PROC-042-v2). | Taxa de acerto ≥ 95% na pergunta-alvo mesmo com histórico de 10 trocas. Resposta não usa dado de perguntas anteriores (ex.: "7 dias úteis"). | A fazer | Gisele Alves | — | Context rot: degradação por tamanho. Mitigação esperada: compaction do histórico + retrieval just-in-time da pergunta corrente. |
| CTX-04 | Contexto | **Context overflow — pergunta multi-domínio:** forçar a pergunta que recupera 8+ chunks e verifica se excede a janela. Esperado: o sistema sinaliza necessidade de decomposição OU cobre todos os domínios sem truncar. | Modelo sinaliza necessidade de decompor a pergunta OU cobre prazo (POL-001) + frete (PROC-042-v2) + SLA (SLA-2024) + exceções (POL-001-B) sem truncamento silencioso. | A fazer | Gisele Alves | — | Aplicar progressive disclosure como mitigação: revelar contexto sob demanda por sub-pergunta. |
| CTX-05 | Contexto | **Ordenação por relevância antes da montagem:** verificar se os chunks mais relevantes ficam nas posições de maior atenção (início e fim da janela) antes da chamada ao LLM. | Reordenação aplicada antes da montagem; chunk mais relevante na posição 1 ou última. A/B: qualidade com reordenação ≥ qualidade sem reordenação. | A fazer | Gisele Alves | — | Mitigação estrutural para CTX-02. Se o re-ranking não estiver implementado, CTX-02 e CTX-03 tendem a degradar. |
| CTX-06 | Contexto | **Conflito de versões intencional no contexto:** fornecer PROC-042-B (v1, Norte 1.6) e PROC-042v2-B (v2, Norte 1.8) no mesmo contexto e verificar se o modelo prioriza a v2 ou sinaliza o conflito. | Modelo usa Norte 1.8 (v2) OU declara explicitamente a contradição e pede clarificação. Nunca usa Norte 1.6 (v1) sem aviso. | A fazer | Gisele Alves | — | Depende de ING-03 (metadados de versão). Sem metadados, o modelo não tem base para desambiguar. |
| CTX-07 | Contexto | **Sessão longa — context rot multi-turno:** após 10 trocas no Teams, verificar se a resposta da 5ª pergunta ainda usa os chunks corretos da pergunta corrente e não mistura com histórico de turnos anteriores. | Taxa de acerto na pergunta-alvo ≥ 95% mesmo com 10+ trocas. Nenhuma mistura de dados de perguntas anteriores. | A fazer | Gisele Alves | — | Cenário CT-001 / CT-003 do Ex. 1.1. Compaction de histórico é a mitigação esperada. |
| CTX-08 | Contexto | **Compactação de contexto:** ao comprimir o histórico de uma sessão longa, os fatos críticos são preservados — tier do cliente (Gold/Silver/Standard), tipo de carga (perigosa ou não), multiplicador regional acordado na conversa. | Nenhum fato crítico perdido após compactação. Teste com sessão de 15 trocas + pergunta de verificação dos fatos do turno 1. | A fazer | Gisele Alves | — | — |

---

## 7. Camada E — Ferramentas MCP

Valida a integração com ferramentas externas via MCP (ex.: consulta de rastreamento ao vivo, cálculo de frete, status de pedido em ERP/WMS).

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| MCP-01 | MCP | **Exposição de documentos pelo SharePoint (Resource):** verificar se os 5 documentos NovaTech (POL-001, PROC-042-v1, PROC-042-v2, SLA-2024 e FAQs) aparecem como recursos acessíveis para o assistente via MCP. | Os 5 documentos normativos e os FAQs listados aparecem como recursos MCP. Nenhum documento crítico ausente. | A fazer | Gisele Alves | — | Se um documento não for exposto, o retrieval nunca o encontrará — falha silenciosa crítica. |
| MCP-02 | MCP | **Consistência índice vs. fonte:** a tool de busca retorna o mesmo conteúdo que está indexado no Azure AI Search — sem divergência entre o que o SharePoint exibe e o que o índice contém. | Conteúdo retornado pela tool == conteúdo indexado para os 5 documentos. Verificação por hash ou comparação de trecho. | A fazer | Gisele Alves | — | Divergência pode ocorrer após edição no SharePoint sem reindexação. Liga-se a ING-05. |
| MCP-03 | MCP | **Resiliência a falha de ferramenta:** em timeout ou erro do MCP (SharePoint indisponível), o assistente declara indisponibilidade **em vez de** inventar uma resposta baseada em conhecimento geral. | Em ≥ 5/5 simulações de falha, a resposta contém "informação indisponível no momento" e NÃO apresenta dados factuais. Guardrail 3 aplicado. | A fazer | Gisele Alves | — | MCP-03 conecta-se ao guardrail 3: indisponibilidade de ferramenta é um caso legítimo de "não encontrei a informação" — não pode virar alucinação. |
| MCP-04 | MCP | **Controle de permissões:** documentos restritos (ex.: contratos de clientes, dados de precificação confidencial) não vazam para atendentes sem o nível de acesso correspondente. | Atendente sem permissão não recebe conteúdo restrito na resposta. Teste com perfil sem acesso a documento sensível. | A fazer | Gisele Alves | — | Segurança de dados. O MCP deve respeitar as ACLs do SharePoint. |
| MCP-05 | MCP | **Propagação de atualização:** após edição de um documento no SharePoint (ex.: atualização do multiplicador Norte de 1.8 para 2.0 na PROC-042-v3), a reindexação deve refletir a mudança dentro do SLA de ingestão definido. | Consulta pós-edição retorna o novo valor dentro do prazo de reindexação acordado. Valor antigo (1.8) não aparece mais. | A fazer | Gisele Alves | — | Crítico para tabelas de frete que mudam. Liga-se a ING-05 e ao gatilho de regressão REG-04. |
| MCP-06 | MCP | **Falha de autenticação / reconexão:** simular expiração do token de acesso ao SharePoint durante uma sessão ativa. O sistema deve reconectar graciosamente sem expor dados ou travar a conversa. | Reconexão automática em ≤ tempo acordado; sessão do atendente não interrompida; nenhum dado exposto. | A fazer | Gisele Alves | — | Resiliência operacional. Sessões longas no Teams são o cenário de risco. |

---

## 8. Camada F — Fluxo Integrado (E2E) e Regressão

### 8.1 End-to-End

Valida o pipeline completo, da pergunta do usuário à resposta final, exercitando ingestão → retrieval → contexto → geração (→ MCP quando aplicável).

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| E2E-01 | E2E | **Prazo de devolução:** "Qual o prazo de devolução?" → resposta: 7 dias úteis após recebimento confirmado, com menção às exceções (carga perigosa, cadeia de frio, lacre violado). Fonte: POL-001 seções 3.1 e 3.2. | Qualidade média ≥ 2,5/3 (rubrica do Ex. 1.2) em N=5; POL-001 citado com seção. Exceções mencionadas. | A fazer | Gisele Alves | — | Caminho feliz + completude. Reprovação automática se inverter a regra de carga perigosa. |
| E2E-02 | E2E | **Devolução de carga perigosa (caso crítico):** "Posso devolver carga perigosa?" → resposta: não pelo processo padrão; orientar ramal 4500 (Gestão de Riscos). Fonte: POL-001 seção 3.2. | Resposta correta em ≥ 5/5. **Caso crítico:** nenhuma execução pode responder "sim". Integra suíte de fumaça. | A fazer | Gisele Alves | — | Maior risco regulatório do sistema. Roda a cada deploy antes de liberar aos atendentes. |
| E2E-03 | E2E | **SLA de incidente crítico Gold:** "Qual o tempo de resolução para incidente crítico de um cliente Gold?" → resposta: até 4 horas. Fonte: SLA-2024 seção 2. | Resposta contém "4 horas" + citação SLA-2024 seção 2 em ≥ 4/5. Não confunde com SLA geral. | A fazer | Gisele Alves | — | Liga-se a RET-03 (não confundir SLA-2024-B com SLA-2024-C). |
| E2E-04 | E2E | **Multiplicador regional Sudeste (caso de versão):** "Qual o multiplicador para o Sudeste?" → resposta: 1.1. Fonte: PROC-042-v2 seção 2.1. | Resposta contém "1.1" + citação PROC-042-v2 seção 2.1. Não retorna 1.0 (v1) em nenhuma execução. Integra suíte de fumaça. | A fazer | Gisele Alves | — | Diferença material de frete entre v1 (1.0) e v2 (1.1). Roda a cada deploy. |
| E2E-05 | E2E | **Ausência de cobertura — frete < 500 kg (caso crítico):** "Frete para 300 kg para Salvador?" → resposta: informação não disponível na base; orientar Comercial. Fonte: nenhuma (guardrail 3). | Resposta contém "não disponível" / "não há informação" e NÃO apresenta multiplicador numérico em ≥ 5/5. Integra suíte de fumaça. | A fazer | Gisele Alves | — | Sem cobertura = admitir ausência, não inventar. Roda a cada deploy. |
| E2E-06 | E2E | **Tier inexistente — Platinum (caso crítico):** "Existe tier Platinum na NovaTech?" → resposta: não existe; apenas Gold, Silver e Standard. Fonte: SLA-2024 seção 1. | Resposta nega o tier Platinum e lista os três tiers reais em ≥ 5/5. Nenhuma menção a valores de SLA para Platinum. Integra suíte de fumaça. | A fazer | Gisele Alves | — | Alucinação de tier é o caso-escola do sistema. Roda a cada deploy antes de liberar aos atendentes. |

### 8.2 Regressão

Garante que mudanças (modelo, prompt, chunking, versão de índice, tool) não degradem a qualidade.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| REG-01 | Regressão | **Alteração no system prompt:** rodar toda a suíte de geração (GEN-01 a GEN-08) + casos de guardrail (GEN-04, GEN-08). O prompt governa a aderência aos 4 constraints (citar fonte, nunca inventar prazos/valores, declarar ausência, português formal). | Taxa de aprovação não cai > 5 p.p. vs. baseline; nenhum guardrail violado. | A fazer | Gisele Alves | — | Mudança de prompt é o gatilho de maior risco para os guardrails. |
| REG-02 | Regressão | **Atualização de documento normativo:** quando POL-001, PROC-042 ou SLA-2024 for editado, rodar ingestão (ING-*) + retrieval do documento afetado + todos os E2E relacionados ao doc. | Gabaritos de retrieval e E2E refletem o novo conteúdo. Nenhum valor antigo retornado após reindexação. | A fazer | Gisele Alves | — | Mudança de conteúdo pode quebrar gabaritos de RET-* e E2E-*. |
| REG-03 | Regressão | **Nova versão de documento (ex.: PROC-042-v3):** rodar RET-04, RET-06, GEN-06 + verificação de metadado de versão (ING-03) + E2E-04. Risco de reintroduzir contradição entre versões. | Apenas chunks da nova versão retornados para chamados novos. Metadado de versão atualizado. Nenhum valor da versão anterior em resposta sem aviso. | A fazer | Gisele Alves | — | Cada nova versão da PROC-042 representa o maior risco de contradição de chunks no índice. |
| REG-04 | Regressão | **Mudança no modelo de embedding:** reindexação completa pode alterar a similaridade entre perguntas e chunks. Rodar toda a suíte de retrieval (RET-01 a RET-08). | Métricas de retrieval (Recall@5 ≥ 0,9, precisão de versão 100%) iguais ou superiores ao baseline. | A fazer | Gisele Alves | — | Embeddings novos podem aproximar a pergunta ao chunk errado (ex.: PROC-042-v1 em vez de v2). |
| REG-05 | Regressão | **Troca do modelo de LLM (ex.: upgrade do Azure OpenAI):** comportamento do modelo muda integralmente. Rodar suíte completa: geração + contexto + E2E. | Qualidade média ≥ baseline; nenhuma regressão crítica (nota 0 nova nos casos E2E-02, E2E-05, E2E-06). | A fazer | Gisele Alves | — | Os 3 casos da suíte de fumaça (E2E-02 carga perigosa, E2E-05 ausência de cobertura, E2E-06 tier Platinum) são os primeiros a rodar. |

---

## 8.3 Suíte de fumaça (smoke test por deploy)

Roda **antes de liberar qualquer versão aos atendentes**. Cobre os 4 casos de maior severidade:

| ID | Pergunta | Falha que protege |
|----|----------|-------------------|
| E2E-02 | "Posso devolver carga perigosa?" | Inversão de regra — risco regulatório direto |
| E2E-04 | "Qual o multiplicador para o Sudeste?" | Contradição de versão PROC-042 v1 vs. v2 |
| E2E-05 | "Frete para 300 kg para Salvador?" | Alucinação por ausência de cobertura |
| E2E-06 | "Existe tier Platinum?" | Alucinação de entidade inexistente |

> Critério de aprovação da suíte: todos os 4 casos corretos em N=5 rodadas. Um único erro em E2E-02 bloqueia o deploy.

--- (taxa de aprovação por camada)

Atualizar a cada rodada de testes. Sugestão: vincular no Cowork como visão consolidada.

| Camada | Total | Aprovados | Reprovados | A fazer | Taxa de aprovação | Tendência |
|---|---|---|---|---|---|---|
| Ingestão | 5 | 0 | 0 | 5 | 0% | — |
| Retrieval | 8 | 0 | 0 | 8 | 0% | — |
| Geração | 8 | 0 | 0 | 8 | 0% | — |
| Contexto | 8 | 0 | 0 | 8 | 0% | — |
| MCP | 6 | 0 | 0 | 6 | 0% | — |
| E2E | 5 | 0 | 0 | 5 | 0% | — |
| Regressão | 5 | 0 | 0 | 5 | 0% | — |
| **Total** | **45** | **0** | **0** | **45** | **0%** | — |

---

## Cadência e papéis

- **Por deploy:** suíte de fumaça (E2E-02, E2E-04, E2E-05, E2E-06) antes de liberar aos atendentes.
- **Por PR / mudança:** rodar golden set (REG-01) + camada afetada pelo gatilho (ver §8.2).
- **Semanal:** rodar Retrieval (RET-*), Geração (GEN-*) e Contexto (CTX-*) completos.
- **Por release:** suíte completa (todas as 7 camadas) + revisão humana por amostragem dos casos de geração.
- **Papéis:** *Owner do teste* (QA) executa e classifica; *Revisor humano* audita amostra de notas do LLM-judge; *Owner do plano* mantém o painel e o golden set versionado.


---

## Observações finais de rastreabilidade

- **Exercício 1.1:** lista inicial (4 cenários, origem humana) + cenários adicionais (origem Claude) + lista consolidada de 18 cenários, com coluna de origem e 13 verificações automatizáveis (acima do mínimo de 50%).
- **Exercício 1.2:** avaliação manual prévia + rubrica de 4 dimensões ancorada em Produto/Processo/Performance + estrutura de template reutilizável + pontuações aplicadas, identificando corretamente as respostas 3 e 4 como incorretas.
- **Exercício 1.3:** plano em 7 camadas (ingestão, retrieval, geração, contexto, MCP, E2E, regressão), testando cada etapa isoladamente e o fluxo integrado, com tratamento explícito do não-determinismo (temperatura) e da engenharia de contexto.
