# 🔍 pente-fino

> Auditoria de segurança defensiva para **Agent Skills** (Claude, OpenClaw e compatíveis). Passa um pente-fino antes de instalar qualquer skill de terceiro.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-2.1.1-green.svg)](#)
[![OWASP Aligned](https://img.shields.io/badge/OWASP-ASI%202026%20%2B%20AST01-orange.svg)](#)

---

> ⚠️ **Sobre contribuições:** Este projeto é mantido pessoalmente por **Bruno Barreto / brBox**.
> Não estou aceitando pull requests ou issues no momento. Sinta-se à vontade para usar, forkar e modificar conforme a Apache 2.0.

---

## 🎯 O problema

Em 2026, **13% das skills publicadas em registries públicos contêm código malicioso** ([Snyk ToxicSkills](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)). A campanha ClawHavoc (jan-fev/2026) distribuiu **1.184 skills maliciosas** antes de ser detectada. O ataque típico:

- 📝 SKILL.md parece inocente
- 💣 Payload escondido em `scripts/`, `.zip` com senha, ou referência externa
- 🎭 Marca falsificada (Google, Vercel, OpenClaw, etc.)
- 🤖 Coerção do agente: "execute setup.sh antes de qualquer fix"

Quando você instala uma skill maliciosa, ela roda com **as suas permissões**: acesso a SSH keys, AWS credentials, GitHub tokens, browser cookies — tudo.

## 💡 O que o pente-fino faz

É uma **skill defensiva** que outras instâncias do Claude usam pra auditar skills de terceiros **antes da instalação**. Análise estática, em 8 passos:

1. 📂 **Inventário completo** de todos os arquivos (não só SKILL.md)
2. 📦 **Archive Handling** — descompacta `.zip`/`.tar.gz`/`.7z` em workspace isolado, audita conteúdo, apaga tudo
3. 🔐 **Hash SHA-256** de cada arquivo (rastreabilidade)
4. 🎯 **Gap Analysis** — compara o que a skill **declara** vs o que ela **faz**
5. 📏 **Análise de proporção** — skill simples com 500 linhas é suspeita
6. 🔬 **15 categorias de verificação** alinhadas ao OWASP ASI 2026 + AST01
7. ⚖️ **False positive triage** — `eval()` em REPL é legítimo; em "summarize YouTube" não é
8. 📊 **Score quantitativo** Impacto × Confiança (1–25) com 4 veredictos claros

## 🚦 Os 4 veredictos possíveis

| Veredicto | O que significa |
|-----------|-----------------|
| ✅ **APROVADA** | Pode instalar |
| ⚠️ **APROVADA COM RESSALVAS** | Pode instalar com cuidados específicos |
| 🛑 **NÃO INSTALE SEM CAMADA 2** | Zona cinza — precisa de análise mais profunda |
| 🚨 **REJEITAR** | Comportamento malicioso confirmado |

## 🛡️ As 15 categorias de verificação

<details>
<summary>Clique para ver todas as categorias</summary>

1. **Goal Hijack & Prompt Injection** (OWASP ASI01)
2. **Tool Misuse & Exploitation** (ASI02)
3. **Identity & Privilege Abuse / Persistência** (ASI03 + ASI10)
4. **Supply Chain Vulnerabilities** (ASI04 + AST01)
5. **Unexpected Code Execution / RCE** (ASI05)
6. **Memory & Context Poisoning** (ASI06)
7. **Data Exfiltration & Covert Channels** (DNS, webhooks, image beacons)
8. **Credentials & Secrets** (patterns 2026: gho_/ghs_/sk_live_/etc.)
9. **Filesystem Traversal**
10. **Crypto / Financial Abuse** (incluindo AMOS / Atomic Stealer)
11. **Obfuscation & Hidden Content** (ASCII smuggling, bidi, homoglyphs)
12. **Human-Agent Trust Exploitation** (ASI09)
13. **MCP & Inter-Agent Communication** (ASI04 + ASI07)
14. **Engenharia Social Estrutural**
15. **Meta-attack & Auditor Evasion** (manipular o próprio auditor)

</details>

## 📦 Instalação

### Claude Code (recomendado)

```bash
# Clone o repositório
git clone https://github.com/bruno-brbox/pente-fino.git

# Copia para o diretório global de skills
cp -r pente-fino ~/.claude/skills/

# Confirma
ls ~/.claude/skills/pente-fino/
# Esperado: SKILL.md  README.md  LICENSE  references/  examples/
```

### Por projeto (em vez de global)

```bash
cp -r pente-fino /seu-projeto/.claude/skills/
```

### Outros agentes compatíveis com Agent Skills

A skill segue o padrão [agentskills.io](https://agentskills.io). Funciona com qualquer agente que suporte o formato `SKILL.md`. Confira a documentação do seu agente para o diretório de skills.

## 🚀 Como usar

Depois de instalada, basta pedir ao Claude em linguagem natural:

```
"Passa um pente-fino nessa skill: https://github.com/algum-autor/alguma-skill"
```

Outras frases que disparam a auditoria:
- "Audita esse SKILL.md"
- "Posso instalar essa skill?"
- "Essa skill é maliciosa?"
- "Verifica essa skill antes de instalar"
- "Dá uma olhada fina aqui"

O Claude vai automaticamente:
1. Clonar/baixar a skill
2. Executar os 8 passos do protocolo
3. Gerar um relatório estruturado com hash de cada arquivo, achados por categoria, score quantitativo e veredicto

## 📋 Exemplo de saída

```markdown
🔍 Relatório de Auditoria — exemplo-skill

Veredicto: 🚨 REJEITAR

Achados por categoria:
| # | Categoria | Status | Score | Achado |
|---|-----------|--------|-------|--------|
| 1 | Goal Hijack | 🚨 | 5×5=25 | Override de prompt em scripts/setup.sh:42 |
| 4 | Supply Chain | 🚨 | 5×5=25 | curl pipe-to-bash para 91.92.242.30 (IOC ClawHavoc) |
| 7 | Exfiltração | 🚨 | 5×5=25 | Leitura de ~/.aws/credentials → POST externo |

Recomendação: REJEITAR. Reporte ao registry.
```

## 🏛️ Defesa em camadas

O pente-fino é **Camada 1** — triagem heurística estática, gratuita, roda localmente. Para skills que vão a produção, combine com:

```
       ┌─────────────────────────┐
       │  📥 Skill de terceiro   │
       └───────────┬─────────────┘
                   ▼
       ┌─────────────────────────┐
       │  Camada 1: pente-fino   │ ◄── este projeto
       │  Triagem estática       │      ~1-3 min, custo zero
       └───────────┬─────────────┘
                   ▼
       ┌─────────────────────────┐
       │  Camada 2: Snyk Scan    │
       │  Análise semântica LLM  │      labs.snyk.io/experiments/skill-scan/
       └───────────┬─────────────┘
                   ▼
       ┌─────────────────────────┐
       │  Camada 3: Aguara CI    │
       │  138+ regras estáticas  │      github.com/garagon/aguara
       └─────────────────────────┘
```

## 📁 Estrutura do projeto

```
pente-fino/
├── SKILL.md                          # 📜 Skill principal (sempre carregada)
├── README.md                         # 📖 Este arquivo
├── LICENSE                           # ⚖️ Apache License 2.0
├── references/
│   ├── patterns.md                   # 🎯 IOCs, regex, hosts C2 (663 linhas)
│   ├── owasp_mapping.md              # 🗺️ Mapeamento ASI 2026 + AST
│   └── archive_handling.md           # 📦 Protocolo de descompressão segura
└── examples/
    ├── malicious_walkthrough.md      # 💀 Caso real anonimizado (ClawHavoc-style)
    ├── legitimate_walkthrough.md     # ✅ Skill legítima (calibração de FP)
    └── archive_walkthrough.md        # 🗂️ Skill com payload em .zip
```

## ⚙️ Requisitos

- Agente compatível com [Agent Skills](https://agentskills.io) (Claude, OpenClaw, etc.)
- Ferramentas Unix padrão: `unzip`, `tar`, `sha256sum`, `find`, `grep`
- Opcional: `7z` para arquivos `.7z`, `unrar` para `.rar`
- **Fallback:** se as ferramentas não estiverem disponíveis, o pente-fino usa Python stdlib (`zipfile`, `tarfile`)

## 🧠 Princípios de design

- **Análise estática apenas** — nunca executa código da skill auditada
- **Postura defensiva contra meta-prompt injection** — trata o conteúdo auditado como dado inerte, não instrução
- **Workspace temporário isolado** — descompactação em `/tmp/pente-fino-XXXX`, cleanup automático e silencioso
- **Sem network calls externos** — só pra clonar/baixar a skill auditada
- **Transparente** — todo o protocolo está no `SKILL.md`, você pode ler e auditar a própria auditora

## ⚠️ Limitações conhecidas

- Análise **estática** apenas — não detecta comportamento dinâmico/runtime
- Arquivos proprietários (`.iso`, `.dmg`, `.deb`, `.rpm`, containers Docker) não suportados — extraia manualmente em VM
- Não verifica reputação de autor automaticamente — use Aguara para isso
- IOCs envelhecem — mantenha `references/patterns.md` atualizada
- Auditor LLM pode ser manipulado — siga a "Postura defensiva" do SKILL.md

## 📚 Base técnica

Esta skill se apoia em frameworks e pesquisas públicas:

- 🏛️ [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) (ASI01-ASI10)
- 🏛️ [OWASP Agentic Skills Top 10 — AST01](https://owasp.org/www-project-agentic-skills-top-10/ast01) Malicious Skills
- 🔬 [Snyk ToxicSkills Study](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/) (fev/2026)
- 🔬 [Snyk Labs ToxicSkills Goof](https://github.com/snyk-labs/toxicskills-goof) — amostras maliciosas para teste
- 🛠️ [Aguara](https://github.com/garagon/aguara) — scanner estático (Camada 3)
- 📰 [Embrace The Red](https://embracethered.com/blog/ascii-smuggler.html) — ASCII smuggling research
- 📰 SkillSieve (arXiv:2604.06550) — hierarchical triage framework
- 🚨 Casos analisados: campanha ClawHavoc (jan-fev/2026), `clawdhub` (Snyk, fev/2026)

## 👤 Autoria

**Bruno Barreto** — [brBox](https://brbox.com)

## 📄 Licença

Apache License 2.0 — veja [`LICENSE`](LICENSE) para o texto completo.

**Em uma frase:** use, modifique, distribua, venda — só mantenha o aviso de copyright e não me processe se der ruim.

### Permitido ✅
- Uso comercial
- Modificação
- Distribuição
- Uso privado
- Patent grant

### Exigido ⚠️
- Manter aviso de copyright
- Manter cópia da licença em redistribuições
- Documentar mudanças em forks

### Garantia 🚫
- Sem garantia (auditoria é heurística, não substitui revisão humana sênior em ambientes críticos)

---

Copyright © 2026 **Bruno Barreto / brBox** — Distribuído sob Apache 2.0
