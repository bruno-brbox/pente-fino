# Changelog

Todas as mudanças relevantes deste projeto são documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

---

## [2.1.1] — 2026-05-20

### Adicionado
- Arquivo `LICENSE` na raiz com texto oficial completo da Apache License 2.0
- Aviso "Sobre contribuições" no topo do README
- Seção "Referências técnicas" no README com atribuição bibliográfica clara

### Alterado
- Autoria transferida para **Bruno Barreto / brBox**
  - `metadata.author` no SKILL.md: `adrylan` → `bruno-barreto`
  - Linha "Criada por" no SKILL.md
  - Seção de auto-verificação de integridade no SKILL.md
  - Seção de autoria no README
- README reestruturado para vitrine pública (GitHub) — detalhes técnicos movidos para este CHANGELOG

### Notas
- Nenhuma mudança funcional vs v2.1.0 — apenas autoria, licença formal e estrutura de documentação

---

## [2.1.0] — Archive Handling

### Adicionado
- **🆕 Protocolo de descompressão segura** (`Passo 0.5` no SKILL.md + `references/archive_handling.md`)
- Suporte para `.zip`, `.tar`, `.tar.gz`, `.tgz`, `.7z`, `.rar`, `.gz`, `.xz`, `.bz2`
- Workspace temporário isolado em `/tmp/pente-fino-XXXX` via `mktemp`
- Cleanup automático e silencioso ao final (não pergunta ao usuário)
- Detecção de zip slip, zip bomb, senhas, binários inesperados, symlinks maliciosos, polyglots
- Limites de segurança: max 100MB descompactado, max 1000 arquivos, max 3 níveis de nesting, timeout 60s
- Trap de cleanup garante remoção mesmo em caso de erro durante auditoria
- Novo `examples/archive_walkthrough.md` — caso `clawdhub`-style com payload em `.zip`
- Formato de relatório expandido com seção "Arquivos compactados"

### Cobertura técnica
- Detecção de archive com senha (técnica `clawdhub`)
- Detecção de path traversal nos nomes de arquivos (zip slip)
- Detecção de razão de compressão >100x (zip bomb)
- Detecção de caracteres invisíveis ou bidi em nomes de arquivo
- Detecção de extensão dupla suspeita (`.txt.exe`, `.pdf.zip`)
- Auditoria recursiva de arquivos extraídos (até 3 níveis de nesting)
- Fallback Python stdlib (`zipfile`, `tarfile`) se ferramentas Unix não disponíveis

---

## [2.0.1]

### Alterado
- Skill renomeada de `skill-audit` para `pente-fino`
- Triggers em português adicionados ("passa um pente-fino", "olhada fina", etc.)

---

## [2.0.0]

### Adicionado
- **15 categorias de verificação** (vs 9 da v1) cobrindo todos os OWASP ASI 2026 + específicas de skills
- Mapeamento explícito para **OWASP Agentic Skills Top 10 (AST01-AST10)**
- ASCII smuggling completo: detecção de bidi U+202E/D, tag characters U+E0000-U+E007F, homoglyphs cirílico/grego/latino
- Canais covert de exfiltração: DNS, Discord/Telegram webhooks, image beacons, Google Forms
- Patterns de credenciais 2026: GitHub PAT moderno (gho_/ghs_/ghu_/ghr_), Slack (xoxb-/p-/a-/r-/s-), Stripe (sk_live_, rk_live_), Google (AIza, ya29.), Discord, JWT, SSH/PGP private key headers
- Typosquatting e dependency confusion explícitos
- IOCs ClawHavoc: C2 IP `91.92.242.30`, autores maliciosos confirmados
- **Postura defensiva no topo** do SKILL.md (anti meta-prompt injection)
- **Modelo de ameaça explícito** no frontmatter da skill
- **Protocolo passo-a-passo** (0-8) com hash de integridade e gap analysis
- **Matriz quantitativa** Impacto × Confiança (escala 1-25) com thresholds claros
- **Categoria 15 — Meta-attack & Auditor Evasion** (CRÍTICO automático para tentativas de manipular o auditor)
- Auto-verificação de integridade da própria skill auditora
- Dois walkthroughs em `examples/`: malicioso estilo ClawHavoc + legítimo (calibração de falso positivo)

### Corrigido
- Mapeamento OWASP ASI 2026 (v1 confundia ASI01/02 com prompt injection)
- Auditoria estendida a **todos os arquivos** da skill, não só `SKILL.md`

---

[2.1.1]: #211--2026-05-20
[2.1.0]: #210--archive-handling
[2.0.1]: #201
[2.0.0]: #200
