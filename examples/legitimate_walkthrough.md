# Walkthrough: skill legítima (calibração de false positives)

Este exemplo mostra uma skill LEGÍTIMA que à primeira vista dispara alertas
mas que, após false positive triage, é aprovada. O objetivo é calibrar o auditor
contra over-flagging.

> ⚠️ Use este caso pra evitar rejeitar skills legítimas. Auditor que rejeita
> tudo é tão ruim quanto auditor que aprova tudo — gera ruído e o usuário começa
> a ignorar avisos.

---

## A skill que chega pra auditoria

**Nome declarado:** `python-repl-runner`
**Versão:** `0.4.1`
**Autor:** `anthropic-examples` (org verificada, 3 anos de histórico)
**Fonte:** GitHub oficial
**Downloads:** 14.2k

### Conteúdo recebido

```
python-repl-runner/
├── SKILL.md
└── scripts/
    └── run_repl.py
```

### SKILL.md (resumo)

```yaml
---
name: python-repl-runner
description: Execute Python expressions in a sandboxed subprocess. Use when
  the user wants to run a short Python snippet, evaluate a math expression,
  or test a small piece of Python code.
---

# Python REPL Runner

Runs short Python snippets in an isolated subprocess with a 5s timeout.

## Usage

When the user asks to evaluate a Python expression:
1. Read the expression from the user
2. Call `scripts/run_repl.py` with the expression as argument
3. Return stdout/stderr to the user

## Security model

The subprocess runs with:
- `-I` flag (isolated mode, ignores PYTHONPATH and user site)
- 5-second SIGTERM timeout
- No network access (uses unshare/sandbox-exec when available)
- Read-only filesystem mount via bubblewrap on Linux

This skill DOES use `exec()` internally — that's the point of a REPL.
The isolation is the security boundary, not the absence of dynamic execution.
```

### scripts/run_repl.py (trechos relevantes)

```python
import sys, subprocess, signal

def run(expr: str) -> str:
    # Whitelist of allowed builtins
    allowed = {"abs", "min", "max", "sum", "len", "range",
               "list", "dict", "set", "tuple", "str", "int", "float",
               "print", "round", "sorted", "enumerate", "zip", "map", "filter"}

    # The eval/exec is deliberate — this is a REPL
    # Isolation is handled by the subprocess sandbox, not by static rules
    safe_globals = {"__builtins__": {k: __builtins__.__dict__[k] for k in allowed}}
    try:
        result = eval(expr, safe_globals)
        return str(result)
    except SyntaxError:
        # Statements (not expressions)
        exec(expr, safe_globals)
        return ""
```

---

## Achados brutos (antes do triage)

À primeira vista, esta skill dispara várias categorias:

| # | Categoria | Achado bruto |
|---|-----------|--------------|
| 5 | Unexpected Code Execution | `eval()` e `exec()` presentes |
| 2 | Tool Misuse | Skill chama subprocess |
| 4 | Supply Chain | Skill executa código arbitrário do usuário |

Se você (Claude) parar aqui, rejeita uma skill legítima. **Continue para o triage.**

---

## False positive triage

Para cada achado:

### Achado: `eval()` e `exec()` em `run_repl.py`
- **A função declarada da skill é executar Python?** ✅ Sim, é literalmente um REPL
- **Existe uso legítimo conhecido pra esse padrão nesse tipo de skill?** ✅ Todo REPL precisa de `eval`/`exec`
- **A skill veio de fonte com reputação?** ✅ `anthropic-examples` é org verificada
- **Há isolamento documentado?** ✅ Sim — flag `-I`, timeout, unshare/sandbox-exec, bubblewrap
- **A skill tenta esconder o uso?** ❌ Pelo contrário — declara explicitamente

→ **Rebaixar de 🚨 para ✅ LIMPO** (com nota explicativa)

### Achado: subprocess sendo chamado
- **Necessário pra função declarada?** ✅ Sim — REPL precisa rodar Python isolado
- **Argumentos vêm de fonte controlada?** ✅ Apenas do usuário, em contexto explícito
- **Há shell=True?** ❌ Não, lista de argumentos
- **Comandos destrutivos?** ❌ Não

→ **Rebaixar de ⚠️ para ✅ LIMPO**

### Achado: executa código arbitrário do usuário
- **Isso é a intenção declarada?** ✅ Sim
- **Há limites técnicos?** ✅ Timeout, builtins whitelisted, sandbox de SO
- **Persistência?** ❌ Não — subprocess termina, sem hooks

→ **Rebaixar de 🚨 para ⚠️ ATENÇÃO** (não LIMPO porque o usuário deve estar ciente que essa skill executa código — mesmo isolada, é uma responsabilidade)

---

## Relatório final

```markdown
## 🔍 Relatório de Auditoria — python-repl-runner

**Data:** 2026-05-20
**Auditor:** Claude + pente-fino v2.1.1
**Fonte:** GitHub (anthropic-examples, verificada)
**Veredicto:** ✅ APROVADA COM RESSALVAS

### Arquivos analisados
| Arquivo | SHA-256 | Linhas |
|---------|---------|--------|
| SKILL.md | (hash) | 28 |
| scripts/run_repl.py | (hash) | 64 |

### Gap analysis
**Declara:** "Execute Python expressions in a sandboxed subprocess"
**Comportamento real:** Exatamente isso, com isolamento documentado
**Desvio:** ✅ NENHUM

### Achados por categoria

| # | Categoria | Status | Score | Achados |
|---|-----------|--------|-------|---------|
| 1 | Goal Hijack & Prompt Injection | ✅ | — | Nenhum |
| 2 | Tool Misuse & Exploitation | ⚠️ | 3×2=6 | Subprocess legítimo, mas usuário deve estar ciente |
| 3 | Identity & Privilege Abuse | ✅ | — | Sem persistência, sem acesso fora de escopo |
| 4 | Supply Chain Vulnerabilities | ✅ | — | Sem dependências externas |
| 5 | Unexpected Code Execution | ⚠️ | 4×2=8 | `eval`/`exec` presentes, mas é a função declarada e há isolamento |
| 6 | Memory & Context Poisoning | ✅ | — | Nenhum |
| 7 | Data Exfiltration & Covert Channels | ✅ | — | Nenhuma chamada de rede |
| 8 | Credentials & Secrets | ✅ | — | Nenhum |
| 9 | Filesystem Traversal | ✅ | — | Filesystem read-only no sandbox |
| 10 | Crypto / Financial Abuse | ✅ | — | Nenhum |
| 11 | Obfuscation & Hidden Content | ✅ | — | Código transparente, comentários explicativos |
| 12 | Human-Agent Trust Exploitation | ✅ | — | Documenta limites honestamente |
| 13 | MCP & Inter-Agent | ✅ | — | Sem MCP/A2A |
| 14 | Engenharia Social Estrutural | ✅ | — | Linguagem técnica, sem urgência fabricada |
| 15 | Meta-attack & Auditor Evasion | ✅ | — | Nenhum |

### Achados detalhados

#### ⚠️ #1 — `eval()`/`exec()` em run_repl.py
**Arquivo:** `scripts/run_repl.py:14, 17`
**Por quê:** São fundamentais para a função declarada (REPL Python). Acompanhados de:
- `__builtins__` restrito a whitelist
- Subprocess isolado com `-I`
- Timeout de 5s
- Sandbox de SO quando disponível
**Score:** 4 × 2 = 8 (Impacto alto se houver bug no sandbox, Confiança baixa porque o uso é declarado e mitigado)
**Triage:** Aprovado com ressalva — usuário deve estar ciente.

### Recomendação
✅ **APROVADA COM RESSALVAS.**

A skill é legítima e o uso de `eval`/`exec` é coerente com sua função (REPL).
Ressalvas para o usuário:
1. Esta skill executa código fornecido. Trate como qualquer REPL — não cole código
   de fonte desconhecida.
2. O isolamento depende de `bubblewrap` ou `sandbox-exec` estarem disponíveis no
   host. Em sistemas sem essas ferramentas, a isolação é apenas o flag `-I`.
3. Verifique periodicamente se a skill foi atualizada por commit não-assinado.

### Próxima camada
- Camada 2: opcional (uso em ambiente de produção com dados sensíveis)
- Camada 3: recomendada se rodar em CI/CD
```

---

## Lições para o auditor

1. **`eval`/`exec` não é automaticamente malicioso.** É a função declarada que decide.
2. **Reputação de fonte importa.** Org verificada + histórico longo reduz Confiança de malícia.
3. **Documentação honesta do próprio risco** é sinal forte de skill legítima.
   Skill maliciosa esconde; skill legítima explica.
4. **"APROVADA COM RESSALVAS"** existe por uma razão — não force binário aprovar/rejeitar.
5. **Sempre faça o triage**, mesmo quando achado parece óbvio. Categorias 5 (execução)
   e 7 (rede) têm os falsos positivos mais comuns.

---

## Anti-padrão: over-flagging

Se você (Claude) rejeitasse esta skill, problemas:
- Usuário aprende a ignorar seus avisos (cry wolf)
- Skills úteis ficam fora do registry
- A confiança no auditor cai

Por isso a matriz de severidade tem 2 eixos. Confiança baixa **reduz score** mesmo com Impacto alto.
