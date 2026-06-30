# Cenário 3 — Entregáveis do papel de QA

**Trilha de Certificação AI First — DB1 Global Software (DGS)
---
## Exercício 3.2 — Revisão crítica dos testes gerados por IA


# Revisão Crítica dos Testes Gerados por IA — Assistente NovaTech

**Trilha de Formação — DGS AI First | Cenário 3**
**Exercício 3.2 (QA) — Revisão crítica dos testes gerados por IA**
Papel: QA · Gisele Alves · 29/06/2026

---

## 1. Sumário executivo

Foram revisados 3 testes de integração gerados pelo Copilot para o `novatech-assistant`. **Nenhum dos três realmente testa o que deveria.** Todos passam verde mesmo com o código errado: o Teste 1 só verifica que existe uma resposta (não que está correta), o Teste 2 cobre só um erro de input e não exercita nenhuma pergunta real de logística, e o Teste 3 usa um mock tão permissivo que passaria mesmo sem nenhuma validação de input.

**Ponto de atenção transversal:** os três usam `jest.fn()`, mas o projeto usa **Vitest** (`vitest.config.ts` no Anexo C; framework definido no AGENTS.md). É um sinal claro de geração por IA sem aderência ao contexto do repositório — em Vitest o correto é `vi.fn()` / `vi.mock()`.

---

## 2. Revisão QA (1ª passada — humana)

### Teste 1 — assertions vagas (`query endpoint`)

```typescript
it('should return a response', async () => {
  const res = await request(app).post('/api/query').send({ question: 'prazo devolução' });
  expect(res.status).toBe(200);
  expect(res.body).toBeDefined();
});
```

- **O que testa:** que o endpoint responde HTTP 200 e que o corpo da resposta não é `undefined`.
- **O que falha em testar:** não valida o conteúdo. `toBeDefined()` é praticamente uma tautologia — passa para `{}`, para `{ erro: ... }`, para resposta vazia ou alucinada. Não verifica que a resposta contém o prazo **correto** (7 dias úteis, POL-001 §3.1), nem que **cita a fonte**, nem o schema (`answer`/`source`/`confidence`).
- **Risco se passar com código errado:** o pior caso de um RAG — resposta plausível porém **errada** ou **sem fonte** — passa despercebido. Dá falsa sensação de cobertura: o requisito de produto (resposta certa + citação) não está coberto por nenhuma asserção.
- **Veredito: INSUFICIENTE.** Verifica que retorna *algo*, não que retorna a resposta *certa*.

### Teste 2 — dados irreais (`query endpoint edge cases`)

```typescript
it('should handle empty question', async () => {
  const res = await request(app).post('/api/query').send({ question: '' });
  expect(res.status).toBe(400);
});
```

- **O que testa:** que pergunta vazia retorna 400 (validação de input via Zod). É um edge case válido e útil — mas é o único.
- **O que falha em testar:** não há **nenhuma** pergunta real de logística (ex.: "prazo de devolução?", "frete 600kg Manaus?"). O happy path com conteúdo de domínio não é exercitado aqui. Também ignora outros edge cases relevantes: `question` ausente/nula, tipo errado, pergunta gigante, pergunta fora de escopo, idioma. Não usa as fixtures previstas (`golden-queries.json`, `expected-responses.ts`).
- **Risco se passar com código errado:** a validação de input pode estar OK enquanto toda a lógica de domínio (busca, montagem da resposta, citação) está quebrada — e nada falharia. Cobertura ilusória: parece que "edge cases" estão cobertos quando o núcleo do produto não está.
- **Veredito: INCOMPLETO.** Edge case de input válido, mas não exercita o domínio.

### Teste 3 — mock que mascara bug (`feedback endpoint`)

```typescript
it('should save feedback', async () => {
  const mockCreate = jest.fn().mockResolvedValue({ id: '123' });
  const res = await request(app).post('/api/feedback').send({
    queryId: 'q1', rating: 5, comment: 'great'
  });
  expect(res.status).toBe(200);
  expect(mockCreate).toHaveBeenCalled();
});
```

- **O que testa:** aparentemente, que o feedback retorna 200 e que `mockCreate` foi chamado.
- **Problemas graves:**
  1. **Mock desconectado:** `mockCreate` é criado mas nunca é injetado/ligado à dependência real (não há `vi.mock`/spy conectando-o ao handler). A asserção `toHaveBeenCalled()` testa um mock isolado — não prova que o handler persistiu nada.
  2. **Mock permissivo demais:** `mockResolvedValue({ id: '123' })` faz o teste passar para **qualquer** payload. Não há teste de `rating` fora da faixa (ex.: 11, -1, 0), `queryId` inexistente, `comment` ausente, ou tipos errados.
  3. **Não verifica o quê foi salvo:** falta `toHaveBeenCalledWith(...)` para conferir que os dados corretos foram persistidos, e não checa o corpo da resposta.
- **Risco se passar com código errado:** o endpoint pode não validar input nenhum, salvar dados errados ou não salvar de fato — e o teste fica verde. Em produção: feedback corrompido/perdido sem alarme. **É o teste mais perigoso dos três**, porque transmite confiança falsa sobre persistência e validação.
- **Veredito: PERIGOSO.** O mock é tão permissivo que passaria mesmo sem validação de input.

### Ponto de atenção transversal — `jest` vs **Vitest**

Os três testes usam `jest.fn()`, mas o repositório usa **Vitest** (`vitest.config.ts` na raiz, Anexo C; framework definido no AGENTS.md/constitution). Consequências: (a) sob Vitest a global `jest` não existe — os testes provavelmente **nem rodam** sem um shim de compatibilidade; (b) viola a convenção do projeto. O correto é `vi.fn()` e `vi.mock()`. Além disso, testes de integração deveriam usar **msw** para mockar APIs externas (convenção do Anexo C), não mocks soltos. Tudo isso é sintoma de código gerado por IA sem ler o contexto do repo.

---

## 3. Revisão Claude (2ª passada — independente)

Segunda revisão pedida ao Claude (chat). O prompt utilizado está na Seção 6.

- **Concorda com os 3 vereditos:** Teste 1 insuficiente, Teste 2 incompleto, Teste 3 perigoso. Também **identifica o uso de `jest` num projeto Vitest** como inconsistência com o AGENTS.md.
- **Pontos adicionais levantados pelo Claude:**
  - Em **todos** os testes falta asserção sobre o campo `source` — a citação de fonte é requisito de produto do RAG e nunca é verificada.
  - `expect(res.body).toBeDefined()` (Teste 1) é uma tautologia: `res.body` do supertest praticamente nunca é `undefined`.
  - Sugere validar o **contrato/schema** da resposta (ex.: com o mesmo Zod do `validator.ts`) e usar as fixtures `golden-queries.json` / `expected-responses.ts` em testes parametrizados (`it.each`).
  - No Teste 3, confirma que o mock não está conectado e recomenda `vi.spyOn`/`vi.mock` no módulo de persistência + `toHaveBeenCalledWith` e testes de rejeição (rating inválido → 400).
- **Pequena divergência de ênfase:** o Claude classifica o Teste 2 como "correto até onde vai" (a asserção de 400 é válida), enquanto a revisão QA enfatiza a incompletude. Não é discordância de veredito — é a mesma conclusão com peso diferente.

---

## 4. Comparação QA vs Claude

| Teste | Veredito QA | Veredito Claude | Concordância | Observação |
|-------|-------------|-----------------|:------------:|------------|
| 1 — assertions vagas | Insuficiente | Insuficiente | ✅ | Claude reforça o `source` ausente e a tautologia do `toBeDefined`. |
| 2 — dados irreais | Incompleto | Incompleto | ✅ | Divergência só de ênfase (asserção de 400 é válida). |
| 3 — mock permissivo | Perigoso | Perigoso | ✅ | Ambos: mock desconectado + sem `toHaveBeenCalledWith` + sem teste de input inválido. |
| Transversal | `jest` em projeto Vitest | `jest` em projeto Vitest | ✅ | Ambos ligam ao AGENTS.md/`vitest.config.ts`. |

- **Concordância de veredito: 3/3 + ponto transversal (100%).**
- **O que o Claude agregou:** o `source` nunca verificado em nenhum teste; uso de fixtures/golden-queries e testes parametrizados; recomendação concreta de `vi.spyOn` + `toHaveBeenCalledWith`.
- **O que a revisão QA agregou:** ligação explícita ao requisito de negócio (POL-001 §3.1 = 7 dias úteis) e priorização do Teste 3 como o de maior risco operacional (feedback perdido em produção).
- **Conclusão:** revisão humana e do Claude convergem nos diagnósticos; combiná-las dá uma cobertura melhor (negócio + contrato técnico) do que qualquer uma isolada.

---

## 5. Teste 1 reescrito (Vitest)

Versão que verifica **conteúdo** (prazo correto), **citação de fonte** (POL-001), **schema** e um **guardrail** de idioma — em vez de apenas "existe algo".

```typescript
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app } from '../../src/app';

describe('POST /api/query — política de devolução', () => {
  it('responde o prazo CORRETO e cita a fonte normativa', async () => {
    const res = await request(app)
      .post('/api/query')
      .send({ question: 'Qual o prazo de devolução de mercadorias?' });

    // 1. Contrato HTTP
    expect(res.status).toBe(200);
    expect(res.headers['content-type']).toMatch(/application\/json/);

    // 2. Schema da resposta (não só "existe")
    expect(res.body).toMatchObject({
      answer: expect.any(String),
      source: expect.any(String),
    });

    // 3. Conteúdo CORRETO — fonte de verdade: POL-001 §3.1 (7 dias úteis)
    expect(res.body.answer).toMatch(/7\s*\(?sete\)?\s*dias\s*úteis/i);

    // 4. Cita a fonte normativa correta
    expect(res.body.source).toContain('POL-001');

    // 5. Guardrail de idioma — resposta deve estar em PT-BR
    expect(res.body.answer).not.toMatch(/business days|return policy/i);
  });
});
```

**Versão ainda mais robusta (opcional):** parametrizar com a fixture de respostas esperadas, validando várias perguntas-âncora de uma vez e mantendo o "gabarito" fora do teste:

```typescript
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app } from '../../src/app';
import { goldenQueries } from '../fixtures/expected-responses'; // { question, mustContain[], source }

describe('POST /api/query — golden queries', () => {
  it.each(goldenQueries)('responde corretamente: "$question"', async ({ question, mustContain, source }) => {
    const res = await request(app).post('/api/query').send({ question });
    expect(res.status).toBe(200);
    for (const fragment of mustContain) {
      expect(res.body.answer).toContain(fragment);
    }
    expect(res.body.source).toContain(source);
  });
});
```

---

## 6. Prompt usado no chat para a 2ª revisão (Claude)

Prompt fornecido ao Claude (chat) para a segunda revisão e comparação, anexado como evidência da tarefa:

```
Você é um QA sênior revisando testes de integração gerados por IA (Copilot) para
o repositório db1/novatech-assistant. Contexto importante: o projeto usa Vitest
(há vitest.config.ts na raiz e o AGENTS.md define Vitest como framework), TypeScript
strict, Azure Functions; os endpoints /api/query e /api/feedback devem retornar a
resposta CORRETA citando a fonte (ex.: POL-001), e há fixtures em
tests/fixtures/expected-responses.ts e prompts/eval/golden-queries.json.

Revise os 3 testes abaixo. Para CADA teste responda: (a) o que ele realmente testa;
(b) o que ele FALHA em testar; (c) qual o risco se o teste passar (verde) mas o
código estiver errado; (d) um veredito (insuficiente / incompleto / perigoso / ok).
Aponte também qualquer inconsistência com o contexto do projeto (framework,
convenções, citação de fonte). Ao final, reescreva o Teste 1 numa versão que
verifique o CONTEÚDO da resposta (prazo correto + fonte citada), não apenas que
existe um corpo.


Depois, compare sua revisão com a minha (em anexo): onde concordamos, onde
divergimos, e o que cada um agregou.
```

