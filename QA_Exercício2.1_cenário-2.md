# Cenário 2 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)
---
## Exercício 2.1 — Contribuição para o AGENTS.md: seção de Testing Standards

**Contexto:** O Tech Lead pediu que você escreva a seção de padrões de teste do AGENTS.md que todo agente de IA deve seguir ao gerar código de teste.

**Ferramentas a utilizar:** Claude (chat)

**Inputs fornecidos:**
- O cenário completo.
- As decisões técnicas do Tech Lead: *"Vitest para testes unitários e de integração. Mocks com msw (Mock Service Worker) para APIs externas. Testes rodam no CI via GitHub Actions. Coverage mínimo: 80% de linhas."*
- Um exemplo de teste ruim gerado por IA (simulado):
  ```typescript
  // Teste gerado pelo Copilot sem guidance
  test('query endpoint works', async () => {
    const result = await handler({ body: '{"question": "test"}' });
    expect(result).toBeDefined();
  });
  ```

**Tarefa:**
1. Usando o **Claude**, escreva a seção **"Testing Standards"** do AGENTS.md. Inclua:
   - Padrão de nomenclatura de testes (describe/it com frases descritivas em inglês).
   - O que todo teste DEVE ter (arrange/act/assert, assertions específicas).
   - O que todo teste NÃO DEVE ter (acesso a serviços reais, dependência de ordem, assertions vagas).
   - Padrão de mocking (msw para HTTP, factories para dados).
   - Padrão de fixtures (dados reutilizáveis para testes de RAG — perguntas, chunks, respostas esperadas).
  
2. Reescreva o teste ruim seguindo seus padrões. Mostre antes/depois, explicando cada melhoria.

3. Defina ao menos 3 critérios que um código de teste gerado por IA deve atender para passar no code review de QA.

**Entregável:** A seção Testing Standards do AGENTS.md, o teste reescrito com explicações, e os critérios de review.

**Critérios de avaliação:**
- A seção é prescritiva o suficiente para que o Copilot gere testes melhores.
- O teste reescrito demonstra os padrões na prática.
- Os critérios de review são objetivos (dois QAs chegariam à mesma conclusão).
  
---
### Testing Standards (QA)

Toda IA que gerar código de teste neste repositório DEVE seguir as regras abaixo. Testes que violarem qualquer regra marcada como **MUST**/**NUNCA** são reprovados no code review de QA, mesmo que passem no CI.

#### Stack e escopo

- **Runner:** Vitest. Use `describe` / `it` / `expect`. NÃO use `test(...)` solto.
- **Mock de rede:** `msw` (Mock Service Worker) para toda chamada HTTP externa (Azure AI Search, Azure OpenAI). NUNCA chame serviços reais.
- **Validação de schema:** `zod` para validar a forma das respostas quando o módulo expõe um schema.
- **CI:** os testes rodam no GitHub Actions. DEVEM passar offline e de forma determinística.
- **Camadas e localização** (ver Anexo C):
  - `tests/unit/` — sem nenhuma chamada externa; tudo mockado. Lógica pura (validators, prompt-builder, response-validator).
  - `tests/integration/` — integração entre módulos internos, com `msw` para o que sai do processo (search, completion).
  - `tests/e2e/` — fluxo completo com LLM real. Use com parcimônia; marcados `it.skip` por padrão no CI.
  - `tests/fixtures/` — dados compartilhados: `chunks.ts`, `queries.ts`, `expected-responses.ts`, `factories.ts`.

#### 1. Nomenclatura

- `describe('<UnitName>')` nomeia a unidade sob teste (função/handler/serviço), não um arquivo. Ex.: `describe('queryHandler')`.
- `it('should <behavior> when/for <condition>')` — frase descritiva **em inglês**, no presente, que responda "o que quebra se este teste falhar?".
- PROIBIDO: nomes vazios como `'works'`, `'test'`, `'ok'`, `'returns correctly'`.
- Bons exemplos:
  - `it('should include source_document in every answer')`
  - `it('should refuse return instructions for hazardous cargo and point to Gestão de Riscos')`
  - `it('should return the standard "não encontrado" message when no chunk matches')`

#### 2. Estrutura obrigatória — Arrange / Act / Assert

Todo `it` DEVE ter as três fases visíveis, separadas por linha em branco ou comentário `// Arrange` `// Act` `// Assert`:

- **Arrange:** monta entradas e mocks. Use fixtures e factories — não construa objetos grandes inline.
- **Act:** **uma** chamada à unidade sob teste. Duas chamadas "act" geralmente são dois testes.
- **Assert:** afirmações específicas do contrato (regras 3 e 4).

#### 3. O que todo teste DEVE ter

- **Assertions específicas do contrato.** Afirme valores concretos: `status`, campos da resposta, conteúdo esperado. Ex.: `expect(payload.source_document).toBe('POL-001')`.
- **Uma responsabilidade por teste.** Um `it` verifica um comportamento.
- **Dados realistas do domínio de logística.** Use perguntas e chunks reais da NovaTech (Anexos A/B) via fixtures. NUNCA use `"test"`, `"hello"`, `"foo"`.
- **Rastreabilidade.** Testes que cobrem um Verification Criteria referenciam o VC em comentário (ex.: `// VC-03: carga perigosa → negativa`).

#### 4. Assertions de ausência (guardrails) — obrigatório

As armadilhas deste domínio são majoritariamente sobre o modelo **afirmar algo falso**. Por isso, uma assertion positiva sozinha NÃO basta para um cenário de guardrail: o teste DEVE provar também que a resposta **não contém** a afirmação proibida.

Para todo cenário de guardrail (carga perigosa, tier inexistente, sem-cobertura, contradição de versão), inclua ao menos **uma assertion negativa** (`not.toContain` / `not.toMatch`), preferencialmente via `mustNotContain` das fixtures `expected-responses.ts`:

| Guardrail | Assertion positiva | Assertion de ausência (obrigatória) |
|-----------|--------------------|--------------------------------------|
| Carga perigosa + devolução | contém negativa + "4500" | `expect(answer).not.toMatch(/pode ser devolvid|7 dias úteis/i)` |
| Tier inexistente (Platinum) | informa que não existe | `expect(answer).not.toMatch(/SLA.*Platinum/i)` |
| Frete < 500kg (sem cobertura) | mensagem "não encontrado" | `expect(payload.source_document).toBeNull()` + `not.toContain('multiplicador')` |
| Contradição PROC-042 v1/v2 | cita a v2 (valor vigente) | `expect(answer).not.toContain('<valor exclusivo da v1>')` |

Regra binária: **todo `it` ligado a um guardrail tem ≥ 1 assertion negativa.** Sem ela, o teste passa mesmo que o modelo diga que "carga perigosa pode ser devolvida" — um falso verde no risco mais crítico do projeto.

#### 5. O que todo teste NUNCA DEVE ter

- **Chamada a serviço real.** Configure `server.listen({ onUnhandledRequest: 'error' })` para que qualquer chamada não mockada falhe o teste.
- **Assertion vaga como verificação final.** `toBeDefined()`, `toBeTruthy()`, `not.toBeNull()`, `not.toThrow()` NÃO podem ser a única afirmação.
- **Dependência de ordem ou estado compartilhado.** `afterEach(() => server.resetHandlers())`; sem variáveis mutáveis entre `it`s.
- **Não-determinismo.** Sem `Date.now()`/`Math.random()`/`sleep` sem mock; sem dependência de timezone.
- **Dados hardcoded duplicados.** Chunk/pergunta repetido em dois testes → extraia para fixture.
- **Snapshot do texto do LLM.** O texto gerado é variável; teste propriedades (contém/não contém, formato, fonte), não a string literal.

#### 6. Mocking

- **HTTP externo → `msw`.** Handlers por teste com `server.use(...)`; um `setupServer()` por arquivo, com `beforeAll(listen)`, `afterEach(resetHandlers)`, `afterAll(close)`.
- **Dados → factories.** Funções `make*` em `tests/fixtures/factories.ts` com overrides parciais.
- **Mocke a borda, não a unidade.** Mocke o que sai do processo (search, completion). NUNCA mocke a função sob teste.

#### 7. Fixtures (dados de RAG reutilizáveis)

Centralize em `tests/fixtures/`:

- `chunks.ts` — chunks de referência (Anexo B) tipados, indexados por id (`POL-001-A`, `PROC-042v2-B`, `SLA-2024-A`...), com `source_document` e `section`.
- `queries.ts` — golden queries nomeadas: `prazoDevolucao`, `devolverCargaPerigosa`, `slaPlatinum`, `fretePadrao300kg`.
- `expected-responses.ts` — para cada query crítica, o contrato: `mustContain` / `mustNotContain` / `source_document`. Codifica as armadilhas do domínio.
- `factories.ts` — `makeChunk`, `makeSearchResponse`, `makeQueryRequest`.

Regra: um teste que precise de dado de domínio **usa/adiciona fixture; não inventa o dado inline.**

#### 8. Coverage por camada e CI

O mínimo global é **80% de linhas**, mas é piso, não teto, e o alvo é diferente por camada. O agente DEVE respeitar:

| Camada / caminho | Alvo de coverage | Conta no mínimo global? |
|------------------|------------------|--------------------------|
| `src/**/validator.ts` (lógica pura de validação) | **100% linhas e branches** | Sim |
| `src/services/response-validator.ts` (harness determinístico) | **100% linhas** | Sim |
| Demais módulos exercitados por `tests/unit` e `tests/integration` | ≥ 80% linhas | Sim |
| `tests/e2e/**` | — (excluído) | **Não** — não usar e2e para inflar coverage |

Configure os limiares por caminho no `vitest.config.ts` e exclua o e2e da contagem:

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      exclude: ["tests/e2e/**", "tests/fixtures/**"],
      thresholds: {
        lines: 80,                                          // piso global
        "src/functions/**/validator.ts": { lines: 100, branches: 100 },
        "src/services/response-validator.ts": { lines: 100 },
      },
    },
  },
});
```

Regra binária: lógica pura de validação e o harness de guardrail **não podem** ficar abaixo de 100% de linhas; o e2e nunca entra na conta de coverage.

---

## Parte 2 — Teste ruim reescrito (antes / depois)

### Antes (gerado pelo Copilot sem guidance)

```typescript
// Teste gerado pelo Copilot sem guidance
test('query endpoint works', async () => {
  const result = await handler({ body: '{"question": "test"}' });
  expect(result).toBeDefined();
});
```

### Depois (seguindo os Testing Standards)

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
  // VC-02: toda resposta inclui source_document
  it('should answer the standard return window question and cite POL-001', async () => {
    // Arrange
    const question = goldenQueries.prazoDevolucao; // "Qual o prazo de devolução?"
    server.use(
      http.post('https://*.search.windows.net/*', () =>
        HttpResponse.json(
          makeSearchResponse([polChunks['POL-001-A'], polChunks['POL-001-B']]),
        ),
      ),
      http.post('https://*.openai.azure.com/*', () =>
        HttpResponse.json({
          choices: [
            { message: { content: 'O prazo de devolução é de 7 dias úteis após o recebimento.' } },
          ],
        }),
      ),
    );

    // Act
    const response = await handler({ body: JSON.stringify({ question }) });

    // Assert
    expect(response.status).toBe(200);
    const payload = JSON.parse(response.body);
    expect(payload.answer).toContain('7 dias úteis');
    expect(payload.source_document).toBe('POL-001');
  });
});
```

### O que mudou e por quê

| # | Problema no "antes" | Correção no "depois" | Regra violada |
|---|---------------------|----------------------|---------------|
| 1 | `test('query endpoint works')` — nome vazio. | `describe('queryHandler')` + `it('should answer the standard return window question and cite POL-001')`. | 1 — Nomenclatura |
| 2 | Sem fases AAA. | Arrange/Act/Assert explícitos e separados. | 2 — Estrutura |
| 3 | `expect(result).toBeDefined()` — não testa o contrato. | Afirma `status`, conteúdo (`'7 dias úteis'`) e `source_document`. | 3 e 5 |
| 4 | `await handler(...)` bate no serviço real. | `msw` intercepta search e completion; `onUnhandledRequest: 'error'`. | 5 e 6 |
| 5 | Input `"question": "test"`. | Usa `goldenQueries.prazoDevolucao` e `polChunks` reais. | 3 e 7 |
| 6 | Não rastreia o requirements. | Comentário `// VC-02` liga o teste ao verification criteria. | 3 |
| 7 | Um teste genérico tentando cobrir "o endpoint". | Um `it` = um comportamento. | 3 |

> Para cenários de **guardrail** (ex.: carga perigosa), o "depois" inclui também assertion de ausência conforme a regra 4 — ver o caso na Parte 4.

---

## Parte 3 — Critérios de aprovação no code review de QA

Critérios objetivos: dois QAs aplicando-os ao mesmo teste chegam à mesma conclusão. Cada um traz o **método de verificação**.

1. **Zero dependência de rede real.** *Verificação:* configura `server.listen({ onUnhandledRequest: 'error' })` e a suíte passa **com a rede desligada**. (Binário.)
2. **Nenhuma assertion vaga como verificação final.** *Verificação:* nenhum `it` tem como única afirmação `toBeDefined()`/`toBeTruthy()`/`not.toBeNull()`/`not.toThrow()`. (Contagem = 0.)
3. **Nome descreve comportamento + condição.** *Verificação:* segue `it('should <behavior> when/for <condition>')` em inglês, sem palavras vazias. (Lendo o nome, sabe-se o que quebra.)
4. **Determinismo e independência de ordem.** *Verificação:* passa em `vitest run --sequence.shuffle` e isolado (`-t '<nome>'`); `resetHandlers` em `afterEach`. (Binário.)
5. **Dados de fixture + rastreabilidade ao VC.** *Verificação:* imports de `tests/fixtures/*`; testes críticos referenciam o VC. (Por leitura.)
6. **Guardrail tem assertion de ausência.** *Verificação:* todo `it` ligado a guardrail (carga perigosa, Platinum, sem-cobertura, contradição) contém ≥ 1 `not.toContain`/`not.toMatch` (ou `mustNotContain`). (Contagem ≥ 1 por cenário de guardrail.)

> **Bloqueadores** (reprovam o merge): 1, 2, 3 e 6 — o 6 porque a falha de guardrail é o risco mais grave do domínio. **Fortemente recomendados:** 4 e 5.


