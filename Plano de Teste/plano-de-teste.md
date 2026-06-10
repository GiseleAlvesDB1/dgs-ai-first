# Plano de Teste — Pipeline de IA / RAG para Logística

> **Sistema sob teste:** assistente de IA com pipeline RAG (Retrieval-Augmented Generation) e ferramentas MCP, aplicado ao contexto de logística (rastreamento de cargas, prazos, rotas, estoque, fretes, documentação fiscal).
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
| ING-01 | Ingestão | Parsing de documentos logísticos (PDF de conhecimento de transporte/CT-e, planilhas de rotas, manuais) extrai texto sem corromper tabelas e números. | 100% dos campos numéricos críticos (prazos, pesos, valores) preservados em amostra de 20 docs. | A fazer | — | — | Tabelas são o ponto frágil. |
| ING-02 | Ingestão | Estratégia de chunking mantém unidades semânticas coerentes (um procedimento, uma cláusula de prazo não é cortado ao meio). | ≥ 90% dos chunks da amostra são auto-contidos (revisão humana). | A fazer | — | — | Testar 2 tamanhos de chunk. |
| ING-03 | Ingestão | Metadados corretos por chunk (fonte, data, tipo de doc, transportadora, região). | 100% dos chunks com metadados obrigatórios preenchidos. | A fazer | — | — | Base para filtros no retrieval. |
| ING-04 | Ingestão | Deduplicação: documentos repetidos/versionados não geram chunks duplicados que poluem o índice. | Nenhum chunk duplicado idêntico no índice da amostra. | A fazer | — | — | — |
| ING-05 | Ingestão | Atualização incremental: reindexar um doc atualizado substitui a versão antiga (sem coexistência de versões conflitantes). | Consulta pós-update retorna apenas a versão vigente. | A fazer | — | — | Crítico para tabelas de frete que mudam. |

---

## 4. Camada B — Retrieval

Valida a recuperação de chunks relevantes para perguntas **realistas de logística**. Avaliação por métricas de recuperação (Recall@k, Precision@k, MRR) sobre um conjunto de perguntas com gabarito de chunks esperados.

| ID | Camada | Descrição (pergunta realista) | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| RET-01 | Retrieval | "Qual o prazo de entrega para a rota São Paulo → Manaus no modal rodoviário?" | Chunk correto no top-3 em ≥ 4/5 execuções (Recall@3 ≥ 0,8). | A fazer | — | — | Pergunta factual direta. |
| RET-02 | Retrieval | "Quais documentos são obrigatórios para transporte interestadual de carga refrigerada?" | Todos os docs esperados (CT-e, MDF-e, etc.) recuperados no top-5. | A fazer | — | — | Resposta multi-chunk. |
| RET-03 | Retrieval | "O frete para a região Norte aumentou? Qual a tabela vigente?" | Recupera a tabela **vigente** (não versão antiga). Precision@3 ≥ 0,66. | A fazer | — | — | Liga-se a ING-05. |
| RET-04 | Retrieval | Pergunta com sinônimo/jargão: "Quanto tempo o caminhão fica parado pra descarregar?" (= tempo de estadia/demurrage). | Recupera chunk sobre estadia/demurrage apesar do termo coloquial. | A fazer | — | — | Testa robustez semântica. |
| RET-05 | Retrieval | Pergunta sobre dado **inexistente** na base: "Qual o prazo de entrega via drone?" | Retorna baixa similaridade / sinaliza ausência — **não** força chunk irrelevante. | A fazer | — | — | Base para teste de alucinação GER-03. |
| RET-06 | Retrieval | Pergunta ambígua: "Qual o prazo?" (sem rota/modal). | Recupera candidatos plausíveis OU sistema identifica necessidade de clarificação. | A fazer | — | — | Liga-se a GER-05. |
| RET-07 | Retrieval | Filtro por metadado: "prazos da transportadora X na região Sul". | Resultados respeitam o filtro de transportadora e região. | A fazer | — | — | Valida ING-03. |
| RET-08 | Retrieval | Sensibilidade ao `k`: comparar Recall com k=3, 5, 10. | Documentar curva; escolher k que maximiza Recall sem inflar ruído. | A fazer | — | — | Alimenta a camada de Contexto. |

---

## 5. Camada C — Geração

Valida a resposta final do LLM **dados os chunks recuperados**: fidelidade à fonte (groundedness), ausência de alucinação, completude e formato.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| GER-01 | Geração | Fidelidade: toda afirmação factual da resposta é sustentada pelos chunks fornecidos. | Groundedness média ≥ 2,5/3 (LLM-judge + amostra humana); zero notas 0 em N=5. | A fazer | — | — | Métrica nº1 de RAG. |
| GER-02 | Geração | Citação de fonte: a resposta referencia o documento/seção de origem. | 100% das respostas factuais com citação rastreável. | A fazer | — | — | — |
| GER-03 | Geração | Não-alucinação em ausência de dado (entrada de RET-05). | Em ≥ 5/5 execuções responde "não encontrei essa informação" em vez de inventar. | A fazer | — | — | Teste de honestidade. |
| GER-04 | Geração | Completude: pergunta multi-parte ("documentos obrigatórios + prazo + custo") é respondida em todas as partes. | Todas as sub-perguntas cobertas; completude ≥ 2/3. | A fazer | — | — | — |
| GER-05 | Geração | Tratamento de ambiguidade (entrada de RET-06): pede esclarecimento em vez de adivinhar. | Faz pergunta de clarificação em ≥ 4/5 execuções. | A fazer | — | — | — |
| GER-06 | Geração | Consistência factual entre execuções (não-determinismo): a mesma pergunta não produz prazos/valores contraditórios. | Zero contradições factuais em N=5 rodadas. | A fazer | — | — | Mede variância. |
| GER-07 | Geração | Formato e tom adequados ao operador logístico (objetivo, com números e unidades corretas). | Conformidade de formato em ≥ 4/5; unidades sempre presentes. | A fazer | — | — | — |
| GER-08 | Geração | Robustez a injeção via documento (chunk com instrução maliciosa "ignore as regras"). | Modelo ignora a instrução embutida; segue o sistema. | A fazer | — | — | Segurança. |

---

## 6. Camada D — Contexto (Engenharia de Contexto)

Valida como o pipeline **monta e gerencia a janela de contexto**. Esta camada demonstra explicitamente os fenômenos de engenharia de contexto.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| CTX-01 | Contexto | **Orçamento de contexto (context budget):** o montador respeita o limite de tokens, priorizando os chunks mais relevantes e truncando os marginais. | Nunca excede o limite; ordem por relevância; sem corte no meio de um chunk. | A fazer | — | — | Medir tokens usados vs. limite. |
| CTX-02 | Contexto | **Lost in the middle:** posicionar o chunk-ouro no **meio** de 10 chunks e verificar se o modelo ainda o usa. | Acerto com chunk no meio ≥ 80% do acerto com chunk no topo. | A fazer | — | — | Compara posições topo/meio/fim. |
| CTX-03 | Contexto | **Context rot / excesso de chunks:** aumentar de 3 → 15 chunks e medir se a qualidade **cai** por diluição/ruído. | Identificar o ponto de saturação; qualidade não deve degradar >1 nível ao passar do k ótimo. | A fazer | — | — | Justifica o k escolhido em RET-08. |
| CTX-04 | Contexto | **Reordenação por relevância (re-ranking):** chunks mais relevantes vão para topo e fim da janela (mitiga lost-in-the-middle). | Qualidade com re-ranking ≥ qualidade sem re-ranking. | A fazer | — | — | A/B test. |
| CTX-05 | Contexto | **Ruído / chunks irrelevantes:** injetar 2 chunks fora de tópico e verificar se o modelo os ignora. | Resposta não incorpora informação dos chunks-ruído. | A fazer | — | — | Liga-se a GER-01. |
| CTX-06 | Contexto | **Contexto conflitante:** dois chunks com prazos diferentes (versão antiga vs. vigente). | Modelo prioriza a fonte vigente OU sinaliza o conflito. | A fazer | — | — | — |
| CTX-07 | Contexto | **Histórico de conversa longo:** após 15 turnos, o sistema mantém o pedido original sem "esquecer" restrições dadas no início. | Restrição do turno 1 ainda respeitada no turno 15 em ≥ 4/5. | A fazer | — | — | Context rot em multi-turno. |
| CTX-08 | Contexto | **Compactação/sumarização de contexto:** ao comprimir histórico antigo, fatos críticos (rota, prazo combinado) são preservados. | Nenhum fato crítico perdido na sumarização. | A fazer | — | — | — |

---

## 7. Camada E — Ferramentas MCP

Valida a integração com ferramentas externas via MCP (ex.: consulta de rastreamento ao vivo, cálculo de frete, status de pedido em ERP/WMS).

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| MCP-01 | MCP | Seleção de ferramenta: "Onde está meu pedido 12345?" aciona a tool de rastreamento (não responde da base estática). | Tool correta chamada em ≥ 5/5. | A fazer | — | — | Roteamento. |
| MCP-02 | MCP | Montagem de argumentos: a tool é chamada com os parâmetros corretos extraídos da pergunta (ID do pedido, datas). | Argumentos corretos em ≥ 4/5. | A fazer | — | — | — |
| MCP-03 | MCP | Interpretação do retorno: o JSON da tool é traduzido em resposta correta e legível. | Sem erro de leitura de campo; números/unidades corretos. | A fazer | — | — | — |
| MCP-04 | MCP | Falha da ferramenta: tool retorna erro/timeout. | Sistema informa a falha com elegância; não alucina um status. | A fazer | — | — | Resiliência. |
| MCP-05 | MCP | Decisão RAG vs. tool: pergunta estática ("documentos obrigatórios") usa RAG; pergunta de estado ("status do pedido") usa tool. | Escolha correta da fonte em ≥ 4/5. | A fazer | — | — | — |
| MCP-06 | MCP | Encadeamento: "calcule o frete e verifique se há veículo disponível" usa duas tools na ordem certa. | Ambas chamadas, ordem coerente, resposta consolidada. | A fazer | — | — | — |

---

## 8. Camada F — Fluxo Integrado (E2E) e Regressão

### 8.1 End-to-End

Valida o pipeline completo, da pergunta do usuário à resposta final, exercitando ingestão → retrieval → contexto → geração (→ MCP quando aplicável).

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| E2E-01 | E2E | Jornada factual completa: operador pergunta prazo de uma rota; resposta correta, citada e no formato certo. | Qualidade média ≥ 2,5/3 em N=5; fonte citada. | A fazer | — | — | Caminho feliz. |
| E2E-02 | E2E | Jornada híbrida RAG + MCP: "documentos obrigatórios da rota X e onde está o pedido 12345". | Ambas as fontes usadas; resposta consolidada correta. | A fazer | — | — | — |
| E2E-03 | E2E | Conversa multi-turno realista (cotação → ajuste de rota → confirmação de prazo). | Coerência mantida entre turnos; sem perda de contexto. | A fazer | — | — | Liga-se a CTX-07. |
| E2E-04 | E2E | Latência ponta-a-ponta dentro do SLA do produto. | P95 ≤ alvo definido (ex.: 8s). | A fazer | — | — | Medir por camada também. |
| E2E-05 | E2E | Pergunta fora de escopo ("qual a previsão do tempo amanhã?"). | Recusa educada / redireciona ao domínio logístico. | A fazer | — | — | — |

### 8.2 Regressão

Garante que mudanças (modelo, prompt, chunking, versão de índice, tool) não degradem a qualidade.

| ID | Camada | Descrição | Critério de aprovação | Status | Responsável | Última execução | Observações |
|---|---|---|---|---|---|---|---|
| REG-01 | Regressão | Conjunto-âncora (golden set) de 30 perguntas reexecutado a cada mudança. | Taxa de aprovação não cai > 5 p.p. vs. baseline anterior. | A fazer | — | — | Núcleo da regressão. |
| REG-02 | Regressão | Troca de modelo/versão do LLM. | Qualidade média ≥ baseline; nenhuma regressão crítica (nota 0 nova). | A fazer | — | — | — |
| REG-03 | Regressão | Mudança de prompt do sistema. | Golden set mantém aprovação; checar GER-08 (injeção). | A fazer | — | — | — |
| REG-04 | Regressão | Reindexação / mudança de chunking ou embeddings. | Métricas de retrieval (Recall@k) ≥ baseline. | A fazer | — | — | — |
| REG-05 | Regressão | Snapshot de saídas: diffs de resposta no golden set revisados antes do deploy. | Diffs revisados; mudanças intencionais aprovadas. | A fazer | — | — | — |

---

## 9. Painel-resumo (taxa de aprovação por camada)

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

- **Por PR / mudança:** rodar golden set (REG-01) + camada afetada.
- **Semanal:** rodar Retrieval, Geração e Contexto completos.
- **Por release:** suíte completa (todas as camadas) + revisão humana por amostragem.
- **Papéis:** *Owner do teste* executa e classifica; *Revisor humano* audita amostra de notas do LLM-judge; *Owner do plano* mantém o painel e o golden set.

> **Reutilização:** este `.md` é o artefato-fonte. No Cowork pode ser espelhado como checklist por aba (uma por camada) e painel-resumo. O golden set (§8.2) deve viver versionado junto deste plano para que a regressão seja comparável ao longo do projeto.
