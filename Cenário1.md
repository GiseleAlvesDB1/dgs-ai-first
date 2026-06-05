# dgs-ai-first

# Cenário 1 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)**
**Blocos cobertos no Cenário 1:** 1) Fundamentos de IA Generativa · 2) Engenharia de Prompt · 3) Engenharia de Contexto · 4) RAG e MCP
**Projeto-base:** Assistente de IA conversacional da NovaTech (logística — Teams + SharePoint + Azure AI Services)
**Prazo do Cenário 1:** 06/06

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

