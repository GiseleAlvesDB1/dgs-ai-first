# dgs-ai-first

# Cenário 2 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)

#### Exercício 2.1 — Contribuição para o AGENTS.md: seção de Testing Standards

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
---
### Testing Standards (QA)

Toda IA que gerar código de teste neste repositório DEVE seguir as regras abaixo. Testes que violarem qualquer regra marcada como **MUST**/**NUNCA** são reprovados no code review de QA, mesmo que passem no CI.

#### Stack e escopo

- **Runner:** Vitest. Use `describe` / `it` / `expect`. NÃO use `test(...)` solto.
- **Mock de rede:** `msw` (Mock Service Worker) para toda chamada HTTP externa (Azure AI Search, Azure OpenAI). NUNCA chame serviços reais.
- **Validação de schema:** `zod` para validar a forma das respostas quando o módulo sob teste expõe um schema.
- **CI:** os testes rodam no GitHub Actions. DEVEM passar offline e de forma determinística.
- **Coverage:** mínimo de **80% de linhas**. Coverage NÃO é objetivo em si — não escreva testes só para tocar linhas; cubra comportamento.
- **Camadas e localização** (ver Anexo C):
  - `tests/unit/` — sem nenhuma chamada externa; tudo mockado. Lógica pura (validators, prompt-builder, response-validator).
  - `tests/integration/` — integração entre módulos internos, com `msw` para o que sai do processo (search, completion).
  - `tests/e2e/` — fluxo completo. Use com parcimônia (consome tokens); marque com `it.skip` por padrão no CI a menos que explicitamente habilitado.
  - `tests/fixtures/` — dados compartilhados: `chunks.ts`, `queries.ts`, `expected-responses.ts`, `factories.ts`.

#### 1. Nomenclatura

- `describe('<UnitName>')` nomeia a unidade sob teste (a função/handler/serviço), não um arquivo. Ex.: `describe('queryHandler')`, `describe('responseValidator')`.
- `it('should <behavior> when/for <condition>')` — frase descritiva **em inglês**, no presente, que responda "o que quebra se este teste falhar?".
- PROIBIDO: nomes vazios como `'works'`, `'test'`, `'ok'`, `'returns correctly'`. Se o nome não descreve um comportamento observável, reescreva.
- Bons exemplos:
  - `it('should include source_document in every answer')`
  - `it('should refuse return instructions for hazardous cargo and point to Gestão de Riscos')`
  - `it('should return the standard "não encontrado" message when no chunk matches')`

#### 2. Estrutura obrigatória — Arrange / Act / Assert

Todo `it` DEVE ter as três fases visíveis, separadas por linha em branco ou comentário `// Arrange` `// Act` `// Assert`:

- **Arrange:** monta entradas e mocks. Use fixtures e factories — não construa objetos grandes inline.
- **Act:** **uma** chamada à unidade sob teste. Se precisar de duas chamadas "act", provavelmente são dois testes.
- **Assert:** afirmações específicas do contrato (ver regra 3).

#### 3. O que todo teste DEVE ter

- **Assertions específicas do contrato.** Afirme valores concretos: `status`, campos da resposta, conteúdo esperado. Ex.: `expect(payload.source_document).toBe('POL-001')`.
- **Uma responsabilidade por teste.** Um `it` verifica um comportamento. Vários comportamentos = vários `it`.
- **Dados realistas do domínio de logística.** Use perguntas e chunks reais da NovaTech (Anexos A/B) via fixtures. NUNCA use `"test"`, `"hello"`, `"foo"`.
- **Rastreabilidade.** Testes que cobrem um Verification Criteria do requirements referenciam o VC no `describe` ou em comentário (ex.: `// VC-02: toda resposta cita fonte`).

#### 4. O que todo teste NUNCA DEVE ter

- **Chamada a serviço real.** Nada de rede para Azure AI Search/OpenAI ou qualquer endpoint externo. Configure `server.listen({ onUnhandledRequest: 'error' })` para que qualquer chamada não mockada faça o teste falhar.
- **Assertion vaga como verificação final.** `toBeDefined()`, `toBeTruthy()`, `not.toBeNull()` NÃO podem ser a única afirmação de um teste. São aceitáveis apenas como guarda antes de uma assertion específica.
- **Dependência de ordem ou de estado compartilhado.** Cada teste passa isolado e em ordem aleatória. Resete handlers (`afterEach(() => server.resetHandlers())`) e não compartilhe variáveis mutáveis entre `it`s.
- **Não-determinismo.** Sem `Date.now()`/`Math.random()`/relógio real sem mock; sem `sleep` arbitrário; sem depender de timezone.
- **Dados hardcoded duplicados.** Se o mesmo chunk/pergunta aparece em dois testes, extraia para fixture.
- **Snapshots de respostas do LLM.** O texto gerado é variável; teste propriedades (cita fonte, contém/não contém termo, formato), não o texto literal.

#### 5. Mocking

- **HTTP externo → `msw`.** Defina handlers por teste com `server.use(...)`. Um `setupServer()` por arquivo, com `beforeAll(listen)`, `afterEach(resetHandlers)`, `afterAll(close)`.
- **Dados → factories.** Use funções `make*` em `tests/fixtures/factories.ts` que retornam objetos válidos com overrides parciais. NÃO instancie objetos de domínio grandes à mão em cada teste.
- **Mocke a borda, não a unidade.** Mocke o que sai do processo (search, completion). NUNCA mocke a função que você está testando — isso é o anti-padrão "teste que sempre passa".

#### 6. Fixtures (dados de RAG reutilizáveis)

Centralize em `tests/fixtures/`:

- `chunks.ts` — chunks de referência (Anexo B) tipados, indexados por id (`POL-001-A`, `PROC-042v2-B`, `SLA-2024-A`...). Incluem `source_document` e `section`.
- `queries.ts` — perguntas-âmbar (golden queries) nomeadas: `prazoDevolucao`, `devolverCargaPerigosa`, `slaPlatinum`, `fretePadrao300kg`.
- `expected-responses.ts` — para cada query crítica, o contrato esperado: `mustContain` / `mustNotContain` / `source_document`. Codifica as armadilhas do domínio (carga perigosa não é devolvível; Platinum não existe; frete < 500kg não tem cobertura).
- `factories.ts` — `makeChunk`, `makeSearchResponse`, `makeQueryRequest`.

Regra: **um teste novo que precise de um dado de domínio adiciona/usa fixture; não inventa o dado inline.**

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
| 1 | `test('query endpoint works')` — nome vazio, não descreve comportamento. | `describe('queryHandler')` + `it('should answer the standard return window question and cite POL-001')`. | 1 — Nomenclatura |
| 2 | Sem fases AAA; uma linha faz tudo. | Arrange/Act/Assert explícitos e separados. | 2 — Estrutura |
| 3 | `expect(result).toBeDefined()` — passa para quase qualquer coisa; não testa o contrato. | Afirma `status`, conteúdo (`'7 dias úteis'`) e `source_document`. | 3 e 4 — assertion específica vs vaga |
| 4 | `await handler(...)` bate no serviço real (Azure AI Search + OpenAI): rede, custo de token, não-determinismo. | `msw` intercepta search e completion; `onUnhandledRequest: 'error'` garante que nada vaza para a rede. | 4 e 5 — sem serviço real, mock na borda |
| 5 | Input `"question": "test"` — não exercita comportamento de domínio. | Usa `goldenQueries.prazoDevolucao` e `polChunks` reais da NovaTech. | 3 e 6 — dados realistas / fixtures |
| 6 | Não rastreia nada do requirements. | Comentário `// VC-02` liga o teste ao verification criteria. | 3 — rastreabilidade |
| 7 | Um teste genérico tentando cobrir "o endpoint". | Um `it` = um comportamento; outros comportamentos (carga perigosa, sem-cobertura, Platinum) viram `it`s próprios. | 3 — uma responsabilidade |

> **Observação:** o teste original tentava ser "o teste do endpoint". No padrão correto ele se desdobra em vários `it`s focados — p.ex. um para a recusa de devolução de carga perigosa (VC-03), um para a mensagem de "não encontrado" em frete < 500kg (VC-04), um para Platinum inexistente. O exemplo acima é o happy path; os edge cases entram como `it`s adicionais no mesmo `describe`.

---

## Parte 3 — Critérios de aprovação no code review de QA

Critérios objetivos: dois QAs aplicando-os ao mesmo teste chegam à mesma conclusão (aprova/reprova). Cada um traz o **método de verificação**.

1. **Zero dependência de rede real.**
   *Verificação:* o arquivo configura `server.listen({ onUnhandledRequest: 'error' })` e a suíte passa **com a rede desligada**. Se algum teste falhar por chamada não mockada, reprova. (Binário: passou offline = sim/não.)

2. **Nenhuma assertion vaga como verificação final.**
   *Verificação:* nenhum `it` tem como única afirmação `toBeDefined()`, `toBeTruthy()`, `toBeFalsy()` ou `not.toBeNull()`. Toda `it` afirma pelo menos um valor concreto (status, campo, conteúdo). (Checável por leitura/grep: contagem de `it`s sem assertion específica = 0.)

3. **Nome descreve comportamento + condição.**
   *Verificação:* todo nome segue `it('should <behavior> when/for <condition>')` em inglês e não contém palavras vazias (`works`, `test`, `ok`, `correctly` isolado). Teste prático: lendo só o nome, o revisor consegue dizer o que quebra se ele falhar. (Reprova se a resposta for "não dá pra saber".)

4. **Determinismo e independência de ordem.**
   *Verificação:* a suíte passa em ordem embaralhada (`vitest run --sequence.shuffle`) e cada teste passa isolado (`vitest run -t '<nome>'`). Sem estado compartilhado entre `it`s; `resetHandlers` em `afterEach`. (Binário: shuffle continua verde = sim/não.)

5. **Dados de teste vêm de fixtures do domínio e há rastreabilidade ao VC.**
   *Verificação:* perguntas/chunks/respostas esperadas vêm de `tests/fixtures/` (não literais inline tipo `"test"`), e todo teste que cobre um verification criteria referencia o VC. (Checável por leitura: imports de `fixtures/*` presentes; comentário/`describe` com o VC nos testes críticos.)

> Os critérios 1–3 são **bloqueadores** (reprovam o merge). Os critérios 4–5 são **fortemente recomendados**; reprovam quando o teste cobre comportamento crítico de IA (recusa de carga perigosa, citação de fonte, sem-cobertura).
