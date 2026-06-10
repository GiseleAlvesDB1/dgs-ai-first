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

## 10. Cadência e papéis

- **Por deploy:** suíte de fumaça (E2E-02, E2E-04, E2E-05, E2E-06) antes de liberar aos atendentes.
- **Por PR / mudança:** rodar golden set (REG-01) + camada afetada pelo gatilho (ver §8.2).
- **Semanal:** rodar Retrieval (RET-*), Geração (GEN-*) e Contexto (CTX-*) completos.
- **Por release:** suíte completa (todas as 7 camadas) + revisão humana por amostragem dos casos de geração.
- **Papéis:** *Owner do teste* (QA) executa e classifica; *Revisor humano* audita amostra de notas do LLM-judge; *Owner do plano* mantém o painel e o golden set versionado.

> **Reutilização:** este `.md` é o artefato-fonte. No Cowork pode ser espelhado como checklist por aba (uma por camada) e painel-resumo. O golden set (§8.2) deve viver versionado junto deste plano para que a regressão seja comparável ao longo do projeto.
