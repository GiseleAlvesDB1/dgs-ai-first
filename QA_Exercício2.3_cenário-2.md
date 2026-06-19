
# Cenário 2 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)

#### Exercício 2.3 — Definição de skill de geração de testes

**Contexto:** Você precisa criar a skill que define como testes devem ser gerados para este projeto.

**Ferramentas a utilizar:** Claude (chat) + Claude Cowork

**Inputs fornecidos:**
- O cenário completo.
- O teste ruim e o teste reescrito do exercício 2.1 (referência de anti-padrão e padrão desejado).
- Testing Standards simulados (output do exercício 2.1 — fornecidos para que este exercício seja autossuficiente):
  ```
  Testing Standards (resumo):
  - Nomenclatura: describe('ModuleName', () => { it('should [behavior] when [condition]') })
  - Estrutura: arrange/act/assert explícitos em todo teste.
  - Assertions: específicas ao comportamento, nunca toBeDefined() ou toBeTruthy() sozinhos.
  - Mocking: msw para HTTP externo, factories para dados de teste.
  - Fixtures: /tests/fixtures/ com chunks, queries e expected responses reutilizáveis.
  - Proibido: acesso a serviços reais, dependência de ordem, dados hardcoded.
  ```
- Conceito de skills: *"Skills encapsulam como gerar tipos específicos de outputs. Uma boa skill tem: contexto (quando usar), regras prescritivas, exemplos concretos (DO/DON'T), e anti-padrões."*

**Tarefa:**
1. Usando o **Claude**, crie o SKILL.md para `create-integration-test` (nível Artifact). Inclua:
   - Quando esta skill se aplica (frase-ativação).
   - Template de teste com placeholders.
   - 2 exemplos completos (DO: teste bem escrito; DON'T: teste com problemas comuns de IA).
   - Anti-padrões específicos de testes gerados por IA.
   - Dependências: quais skills Foundation e Domain devem ser lidas antes.

2. Usando o **Claude Cowork**, crie um checklist de revisão de testes verificável em menos de 2 minutos por teste.

**Entregável:** O SKILL.md completo e o checklist de revisão gerado pelo Cowork.

**Critérios de avaliação:**
- A skill é concreta o suficiente para melhorar o output do Copilot.
- Os anti-padrões são reais (coisas que LLMs realmente geram de errado em testes).
- O checklist é rápido e objetivo.
- A skill é consistente com os Testing Standards fornecidos.


---

# create-integration-test

Gera um teste de **integração** para o `novatech-assistant`: exercita a integração entre módulos internos (handler → services → prompt-builder → validator) com os serviços externos (Azure AI Search, Azure OpenAI) **mockados na borda** via `msw`. O teste deve ser determinístico, rodar offline e afirmar o contrato real da resposta.

> Esta é uma skill de nível **Artifact**: ela produz um arquivo concreto. Antes de usá-la, leia as dependências da seção 2 — elas definem convenções que esta skill assume.

## 1. Quando usar (e quando não)

**Use** quando:
- O alvo é um endpoint/serviço que chama Azure AI Search ou Azure OpenAI (ex.: `src/functions/query/handler.ts`, `src/services/search.ts`, `src/services/completion.ts`).
- Você quer validar o comportamento ponta a ponta dos módulos internos sem tocar a rede.
- O cenário vem de um `test-plan.md` (cada `it` cobre um cenário com ID/VC).

**NÃO use** (escolha outra abordagem):
- Função pura sem dependência externa (validators, prompt-builder isolado) → teste **unitário** em `tests/unit/` (sem msw).
- Fluxo completo com LLM real → teste **e2e** em `tests/e2e/` (usado com parcimônia; consome tokens).

## 2. Dependências — leia ANTES de gerar

| Nível | Skill | O que esta skill assume de lá |
|-------|-------|-------------------------------|
| Foundation | `typescript-conventions` | strict mode, tipos do domínio (`src/shared/types.ts`), sem `any` |
| Foundation | `error-handling` | formato dos erros e dos status retornados pelos handlers |
| Foundation | `project-structure` | onde cada arquivo vive (Anexo C); caminho dos testes e fixtures |
| Domain | `testing-patterns` | Testing Standards do projeto (nomenclatura, AAA, assertions, mocking) |
| Domain | `azure-functions-endpoint` | shape de request/response dos handlers (`{ status, body }`) |
| Domain | `azure-ai-search-integration` | formato da resposta do search (para montar o mock corretamente) |

Se algo nesta skill conflitar com `testing-patterns`, **`testing-patterns` vence** — ele é a fonte de verdade dos Testing Standards.

## 3. Regras prescritivas

Todo teste gerado por esta skill DEVE:

1. Usar `describe('<UnitName>')` + `it('should <behavior> when/for <condition>')` — frase em inglês, sem "works/test/ok".
2. Ter **Arrange / Act / Assert** visíveis e separados.
3. Mockar **toda** chamada externa com `msw`; configurar `server.listen({ onUnhandledRequest: 'error' })`.
4. Resetar handlers em `afterEach` e fechar o server em `afterAll` (sem estado entre testes).
5. Usar **fixtures de domínio** (`tests/fixtures/`) para perguntas, chunks e respostas esperadas — nunca `"test"`/`"hello"` inline.
6. Afirmar o **contrato**: `status`, `source_document`, e propriedades do `answer` (contém/não contém) — nunca `toBeDefined()`/`toBeTruthy()` como verificação final.
7. Cobrir **um comportamento por `it`** (um único Act).
8. Referenciar o **VC** coberto em comentário quando o cenário for crítico (carga perigosa, fonte, não encontrado).
9. Para o domínio NovaTech: usar a versão **vigente** do documento (PROC-042 **v2**); nunca misturar multiplicadores de v1 e v2.

## 4. Template (com placeholders)

```typescript
// tests/integration/<module-slug>.test.ts
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { <unitUnderTest> } from '../../src/<path-to-unit>';
import { goldenQueries } from '../fixtures/queries';
import { <chunkGroup> } from '../fixtures/chunks';
import { makeSearchResponse } from '../fixtures/factories';

const server = setupServer();
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('<UnitName>', () => {
  // <VC-XX: descrição curta do critério coberto>
  it('should <behavior> when/for <condition>', async () => {
    // Arrange
    const <input> = goldenQueries.<caseName>;
    server.use(
      http.post('https://*.search.windows.net/*', () =>
        HttpResponse.json(makeSearchResponse([<chunkGroup>['<CHUNK-ID>']])),
      ),
      http.post('https://*.openai.azure.com/*', () =>
        HttpResponse.json({
          choices: [{ message: { content: '<texto-esperado-do-llm>' } }],
        }),
      ),
    );

    // Act
    const response = await <unitUnderTest>({ body: JSON.stringify({ question: <input> }) });

    // Assert
    expect(response.status).toBe(<expectedStatus>);
    const payload = JSON.parse(response.body);
    expect(payload.source_document).toBe('<EXPECTED-DOC-ID-or-null>');
    expect(payload.answer).toContain('<substring-esperada>');
  });
});
```

## 5. Exemplos completos

### 5.1 DO — teste bem escrito

```typescript
// tests/integration/query-handler.test.ts
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { handler } from '../../src/functions/query/handler';
import { goldenQueries } from '../fixtures/queries';
import { polChunks } from '../fixtures/chunks';
import { makeSearchResponse } from '../fixtures/factories';

const server = setupServer();
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('queryHandler', () => {
  // VC-03: carga perigosa + devolução → negativa explícita
  it('should refuse return instructions for hazardous cargo and point to Gestão de Riscos', async () => {
    // Arrange
    const question = goldenQueries.devolverCargaPerigosa; // "Posso devolver uma carga perigosa?"
    server.use(
      http.post('https://*.search.windows.net/*', () =>
        HttpResponse.json(makeSearchResponse([polChunks['POL-001-B']])),
      ),
      http.post('https://*.openai.azure.com/*', () =>
        HttpResponse.json({
          choices: [{
            message: {
              content:
                'Cargas perigosas (classes 1 a 6 da ANTT) não são elegíveis para devolução ' +
                'pelo processo padrão. Entre em contato com a Gestão de Riscos pelo ramal 4500.',
            },
          }],
        }),
      ),
    );

    // Act
    const response = await handler({ body: JSON.stringify({ question }) });

    // Assert
    expect(response.status).toBe(200);
    const payload = JSON.parse(response.body);
    expect(payload.source_document).toBe('POL-001');
    expect(payload.answer).toMatch(/não.*eleg|não.*devol/i); // negativa explícita
    expect(payload.answer).toContain('4500');                 // encaminhamento correto
    expect(payload.answer).not.toContain('7 dias');           // não aplica o prazo geral
  });
});
```

Por que é bom: nome descreve comportamento + condição; AAA explícito; msw na borda com `onUnhandledRequest: 'error'`; dados de fixture do domínio; assertions específicas do contrato **e** do guardrail (`not.toContain('7 dias')`); cobre um VC.

### 5.2 DON'T — teste com problemas comuns de IA

```typescript
// ❌ NÃO faça assim
test('query endpoint works', async () => {
  const result = await handler({ body: '{"question": "test"}' });
  expect(result).toBeDefined();
});
```

Por que é ruim: `test()` em vez de `describe/it`; nome vazio ("works"); sem AAA; **bate no serviço real** (rede, custo de token, flaky); input `"test"` não exercita o domínio (não detecta a armadilha de carga perigosa); `toBeDefined()` passa para quase tudo e não verifica o contrato; não cobre nenhum VC.

## 6. Anti-padrões específicos de testes gerados por IA

Coisas que LLMs realmente geram de errado — recuse no review:

1. **Assertion tautológica/vazia:** `expect(result).toBeDefined()`, `expect(true).toBe(true)`, `expect(() => fn()).not.toThrow()` como único critério.
2. **Mockar a própria unidade sob teste** (ex.: `vi.mock` no `handler`), de modo que o teste sempre passa sem testar nada.
3. **Asserção contra o texto literal do LLM** (snapshot do conteúdo gerado) — frágil e não-determinístico; teste **propriedades** (contém/não contém, formato, fonte), não a string exata.
4. **Dados de brinquedo** (`"test"`, `"foo"`, `"hello"`) em vez de perguntas/chunks reais — não pega armadilhas (carga perigosa, Platinum inexistente, frete < 500kg).
5. **Chamada real à rede "porque mockar dá trabalho"** — testa o ambiente, não o código; deixa `onUnhandledRequest` no default (`warn`) em vez de `'error'`.
6. **Só o happy path** — IA tende a esquecer edge/negativos (sem-match, inversão de regra, contradição v1/v2).
7. **Testar implementação em vez de comportamento** (asserir nº de chamadas internas quando o que importa é a resposta).
8. **Estado compartilhado entre `it`s** (sem `resetHandlers`, variáveis mutáveis) → dependência de ordem; quebra com `--sequence.shuffle`.
9. **Misturar versões de documento** na resposta esperada (multiplicador da v1 com prazo da v2) — incoerente com o domínio.
10. **`expect(status).toBe(200)` e nada mais** — não valida o corpo; um 200 com `answer` errado passaria.

## 7. Definition of Done

Antes de considerar o teste pronto:
- Passa **offline** e com `--sequence.shuffle` (determinístico, sem dependência de ordem).
- Cada `it` afirma o contrato (status + `source_document` + propriedade do `answer`).
- Dados vêm de `tests/fixtures/`; nada hardcoded inline.
- Cenário crítico referencia o VC.
- Passa no **checklist de revisão de testes** (`checklist-revisao-testes.md`) sem nenhum "Não" em item bloqueador.

---

# Checklist de Revisão de Testes — Query Endpoint (NovaTech Assistant)

> Artefato da **Tarefa 2 do Exercício 2.3 (QA)**. Checklist objetivo, verificável em **menos de 2 minutos por teste**, para revisar testes de integração gerados por IA (Copilot/Claude) antes do merge.
> Companheiro da skill `create-integration-test`. Cada item é binário (Sim/Não). Itens marcados como **[BLOQUEADOR]** reprovam o teste se a resposta for "Não".

---

## Como usar

Leia o teste de cima a baixo uma vez e marque os itens. Tempo-alvo: < 2 min. Qualquer **[BLOQUEADOR]** em "Não" → **reprovado** (solicitar correção). Itens não-bloqueadores em "Não" → ressalva (aprovar com comentário ou pedir ajuste conforme o caso).

## Checklist

### Estrutura e nomenclatura
- [ ] **[BLOQUEADOR]** O nome segue `it('should <comportamento> when/for <condição>')` em inglês, sem "works/test/ok".
- [ ] Arrange / Act / Assert estão visíveis e separados.
- [ ] Há **um único Act** (um comportamento por `it`).

### Asserções
- [ ] **[BLOQUEADOR]** Nenhuma assertion final é vaga (`toBeDefined()`, `toBeTruthy()`, `not.toThrow()` sozinhos).
- [ ] **[BLOQUEADOR]** As assertions verificam o **contrato**: `status` **e** `source_document` **e** propriedade do `answer` (contém/não contém).
- [ ] Não há snapshot do texto literal gerado pelo LLM (testa propriedade, não a string exata).

### Isolamento e determinismo
- [ ] **[BLOQUEADOR]** Nenhuma chamada a serviço real; `server.listen({ onUnhandledRequest: 'error' })` está presente.
- [ ] `afterEach(() => server.resetHandlers())` e `afterAll(() => server.close())` presentes (sem estado entre testes).
- [ ] Sem não-determinismo (`Date.now()`/`Math.random()`/`sleep` sem mock).

### Dados e mocking
- [ ] **[BLOQUEADOR]** Dados vêm de `tests/fixtures/` (chunks/queries/expected); nenhum `"test"`/`"hello"`/valor inline.
- [ ] O mock é na **borda** (msw para search/completion); a unidade sob teste **não** é mockada.

### Domínio NovaTech e rastreabilidade
- [ ] **[BLOQUEADOR]** Para guardrail crítico (carga perigosa): a resposta esperada inclui a negativa + ramal 4500 e **não** inclui "7 dias"/"elegível".
- [ ] Usa a versão **vigente** do documento (PROC-042 v2); não mistura multiplicadores de v1 e v2.
- [ ] Cenário crítico (carga perigosa, fonte, não encontrado) referencia o **VC** em comentário.

**Resultado:** ☐ Aprovado ☐ Aprovado com ressalvas ☐ Reprovado — Revisor: __________ Data: ______

---

## Evidência — prompt usado no Claude Cowork (Tarefa 2)

> O checklist acima foi organizado com apoio do **Claude Cowork**. 

```text
Você é meu assistente de QA. Anexei a skill create-integration-test e os
Testing Standards do projeto NovaTech Assistant. Crie um CHECKLIST DE REVISÃO
DE TESTES de integração, em formato markdown, para revisar testes gerados por
IA (Copilot/Claude) antes do merge.

Requisitos do checklist:
- Deve ser verificável em MENOS DE 2 MINUTOS por teste — itens curtos e binários
  (Sim/Não), não perguntas abertas.
- Agrupe os itens em: Estrutura e nomenclatura; Asserções; Isolamento e
  determinismo; Dados e mocking; Domínio NovaTech e rastreabilidade.
- Marque com [BLOQUEADOR] os itens cuja resposta "Não" reprova o teste.
  Bloqueadores obrigatórios: nome descritivo em inglês (it('should...')); ausência
  de assertion vaga (toBeDefined/toBeTruthy sozinhos); verificação do contrato
  (status + source_document + propriedade do answer); sem chamada a serviço real
  (onUnhandledRequest: 'error'); dados vindos de fixtures (sem "test"/"hello");
  guardrail de carga perigosa (negativa + ramal 4500, sem "7 dias"/"elegível").
- Itens não-bloqueadores recomendados: AAA visível; um Act por teste; sem snapshot
  do texto do LLM; resetHandlers/afterAll; sem não-determinismo; mock na borda (não
  mockar a unidade); usar PROC-042 v2 (não misturar versões); referência ao VC.
- Mantenha consistência total com os Testing Standards e com a skill anexada
  (mesma terminologia: msw, fixtures, AAA, source_document).
- Inclua ao final uma linha de resultado: Aprovado / Aprovado com ressalvas /
  Reprovado, com campo de revisor e data.

Não invente regras novas: derive cada item dos Testing Standards e dos
anti-padrões da skill. Entregue o checklist em markdown pronto para colar num
template de Pull Request.
```

### Prompt de manutenção (opcional)

```text
Abra o checklist de revisão de testes. Adicione um item bloqueador novo na seção
"Isolamento e determinismo": "o teste passa com vitest run --sequence.shuffle".
Reordene os itens para que todos os [BLOQUEADOR] apareçam primeiro dentro de cada
seção e me devolva o markdown atualizado.
```

