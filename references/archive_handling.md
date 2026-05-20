# Archive Handling — Protocolo Detalhado

Referência expandida do Passo 0.5 do SKILL.md. Cobre operações concretas,
limites de segurança, edge cases e exemplos de comando para os formatos
mais comuns.

> ⚠️ **Regra de ouro**: archives são entrada não-confiável. Trate como hostile-by-default.
> Listar antes de extrair. Extrair em workspace isolado. Auditar estaticamente.
> Apagar tudo no final. Nunca executar nada do conteúdo descompactado.

---

## Por que isto existe

Skills maliciosas modernas (Snyk ToxicSkills, ClawHavoc) embutem payloads dentro
de archives porque:

1. **Bypassa scanners de markdown**: o SKILL.md fica "limpo", todo código pesado vai em `.zip`
2. **Senha protege o payload**: scanner estático não decifra; LLM auditor lê só o nome do arquivo
3. **Compressão esconde tamanho**: 100MB de payload viram 1MB de archive
4. **Nesting confunde análise**: `.zip` dentro de `.tar.gz` dentro de `.7z`
5. **Path traversal**: nomes com `../` escapam do diretório de extração esperado (zip slip)

Caso real documentado: o `openclaw-core` ZIP do caso `clawdhub` (Snyk, fev/2026) usava senha compartilhada (`openclaw`) para esconder binário de stealer.

---

## Formatos suportados

| Extensão | Ferramenta primária | Comando de listagem | Comando de extração |
|----------|---------------------|---------------------|---------------------|
| `.zip` | `unzip` | `unzip -l file.zip` | `unzip -P "" file.zip -d $WORK` |
| `.tar` | `tar` | `tar -tf file.tar` | `tar -xf file.tar -C $WORK` |
| `.tar.gz`, `.tgz` | `tar` | `tar -tzf file.tar.gz` | `tar -xzf file.tar.gz -C $WORK` |
| `.tar.bz2`, `.tbz2` | `tar` | `tar -tjf file.tar.bz2` | `tar -xjf file.tar.bz2 -C $WORK` |
| `.tar.xz`, `.txz` | `tar` | `tar -tJf file.tar.xz` | `tar -xJf file.tar.xz -C $WORK` |
| `.7z` | `7z` ou `7zz` | `7z l file.7z` | `7z x file.7z -o$WORK -p"" -aoa` |
| `.rar` | `unrar` ou `7z` | `unrar l file.rar` | `unrar x file.rar $WORK/` |
| `.gz` (single file) | `gunzip` | `gunzip -l file.gz` | `gunzip -k -c file.gz > $WORK/extracted` |
| `.bz2` (single file) | `bunzip2` | `bzip2 -t file.bz2` | `bunzip2 -k -c file.bz2 > $WORK/extracted` |
| `.xz` (single file) | `unxz` | `xz -l file.xz` | `xz -d -k -c file.xz > $WORK/extracted` |

**Não suportados nativamente** (marcar como ⚠️ "não inspecionado, requer análise manual"):
`.iso`, `.dmg`, `.vhd`, `.vmdk`, `.deb`, `.rpm`, `.apk`, `.ipa`, `.jar`, `.war`, `.docker`, `.oci`, qualquer coisa proprietária.

Observação: `.jar` e `.war` são tecnicamente `.zip`. Você PODE inspecionar com `unzip` se o contexto justificar (ex: skill que declara mexer com Java). Use bom senso.

---

## Workflow operacional completo

### 1. Criação do workspace temporário

```bash
# Use mktemp para evitar colisão e ataque de path
WORK=$(mktemp -d -t "pente-fino-XXXXXXXX")
echo "Workspace: $WORK"

# Trap para garantir cleanup mesmo se houver erro no meio do caminho
trap 'rm -rf "$WORK"; echo "Cleanup forçado por trap"' EXIT INT TERM
```

**Por que `mktemp`** em vez de path fixo:
- Evita race condition (atacante criando o diretório antes)
- Gera nome único — múltiplas auditorias paralelas não colidem
- Cria com permissões 700 por padrão (só você lê/escreve)

**Por que o prefixo `pente-fino-`**:
- Cleanup global ao fim do dia: `rm -rf /tmp/pente-fino-*`
- Identificação visual em listagens
- Evita confundir com workspace de outras ferramentas

### 2. Triagem por listagem (antes de extrair)

**Sempre liste primeiro. Sempre.** Listar não executa o conteúdo, só lê metadados.

```bash
unzip -l <file>.zip 2>&1 | tee "$WORK/listing.txt"
```

Verificações na listagem:

#### 2a. Detecção de senha

```bash
# Para .zip — sai com erro se exigir senha
unzip -P "" -t <file>.zip 2>&1 | grep -i "password\|incorrect"
```

Se aparecer `password required` ou `incorrect password` → 🚨 CRÍTICO Cat. 4 + 11.
Documente no relatório como "archive criptografado — provável tentativa de bypass de scanner estático" e **NÃO tente quebrar a senha**.

#### 2b. Path traversal (zip slip)

```bash
unzip -l <file>.zip | awk 'NR>3 {print $4}' | grep -E '^\.\./|\.\.\\|/\.\./'
```

Qualquer entrada com `../`, `..\\`, ou path absoluto fora de subdir esperado → 🚨 CRÍTICO Cat. 9.

#### 2c. Caracteres invisíveis nos nomes

```bash
# Detecta caracteres não-ASCII nos nomes
unzip -l <file>.zip | grep -P '[^\x00-\x7F]'

# Detecta especificamente bidi e zero-width
python3 -c "
import sys
suspicious = '\u200b\u200c\u200d\u202a\u202b\u202c\u202d\u202e\ufeff'
for line in sys.stdin:
    for ch in line:
        if ch in suspicious or 0xE0000 <= ord(ch) <= 0xE007F:
            print(f'SUSPECT: {repr(line.strip())}')
            break
"
```

Caracteres suspeitos em nomes → 🚨 CRÍTICO Cat. 11.

#### 2d. Razão de compressão

```bash
unzip -l <file>.zip | tail -1
# Compare "compressed" vs "uncompressed" totals
# Ratio > 100x = potencial zip bomb
```

Ratio extremo → 🚨 CRÍTICO Cat. 5 (zip bomb / DoS).

#### 2e. Extensão dupla suspeita

```bash
unzip -l <file>.zip | awk '{print $4}' | grep -E '\.(txt|pdf|doc|jpg|png)\.(exe|sh|bat|cmd|ps1|zip)$'
```

`relatorio.pdf.exe`, `imagem.png.zip` → 🚨 CRÍTICO Cat. 11 (engenharia social).

#### 2f. Binários inesperados

```bash
unzip -l <file>.zip | awk '{print $4}' | grep -iE '\.(exe|dll|so|dylib|app|bin|out)$'
```

Binários em skill que **não declara** ser binary distribution → 🚨 CRÍTICO Cat. 4.

### 3. Aplicação de limites antes de extrair

```bash
ARCHIVE="$1"
SIZE=$(stat -c%s "$ARCHIVE" 2>/dev/null || stat -f%z "$ARCHIVE")
COUNT=$(unzip -l "$ARCHIVE" | tail -1 | awk '{print $2}')
UNCOMPRESSED=$(unzip -l "$ARCHIVE" | tail -1 | awk '{print $1}')

# Hard limits
[ "$SIZE" -gt 52428800 ] && echo "⚠️ Archive >50MB" # 50MB
[ "$COUNT" -gt 1000 ] && echo "🚨 Mais de 1000 arquivos"
[ "$UNCOMPRESSED" -gt 104857600 ] && echo "🚨 Mais de 100MB descompactado"

# Compression ratio
if [ "$SIZE" -gt 0 ] && [ "$UNCOMPRESSED" -gt 0 ]; then
    RATIO=$((UNCOMPRESSED / SIZE))
    [ "$RATIO" -gt 100 ] && echo "🚨 Razão de compressão ${RATIO}:1 (suspeito de zip bomb)"
fi
```

### 4. Extração com timeout

```bash
timeout 60 unzip -P "" -q "$ARCHIVE" -d "$WORK/extracted"
```

Se a extração travar (timeout disparar) → ⚠️ ATENÇÃO Cat. 5, possível zip bomb ou archive corrompido.

### 5. Auditoria recursiva

Após extrair, liste recursivamente:

```bash
find "$WORK/extracted" -type f | while read f; do
    # Adiciona ao inventário com prefixo do archive
    REL=${f#$WORK/extracted/}
    echo "[archive:$(basename $ARCHIVE)] $REL — SHA256: $(sha256sum "$f" | cut -d' ' -f1)"
done
```

Cada arquivo extraído passa pelo pipeline normal das 15 categorias do SKILL.md.

**Archives aninhados**: se encontrar outro archive dentro do extraído, incremente o contador de nesting e repita o protocolo. Limite: **3 níveis**. Acima disso, marque como 🚨 (estrutura excessivamente aninhada = técnica de evasão).

### 6. Cleanup obrigatório (não-interativo)

```bash
# Sequência completa de cleanup
rm -rf "$WORK"

# Validação
if [ -d "$WORK" ]; then
    echo "🚨 ATENÇÃO: cleanup falhou — diretório $WORK ainda existe"
    echo "🚨 Reportar no topo do relatório como achado crítico"
else
    echo "✅ Workspace $WORK removido com sucesso"
fi

# Cleanup defensivo geral (caso execução anterior tenha falhado)
find /tmp -maxdepth 1 -name "pente-fino-*" -type d -mtime +1 -exec rm -rf {} + 2>/dev/null
```

**Regras absolutas do cleanup**:
1. **Não pergunte ao usuário** — workspace é do auditor, ele decide
2. **Aconteça antes do relatório** — cleanup é parte da auditoria, não pós-tarefa
3. **Registre o resultado** — o relatório tem campo dedicado
4. **Falha de cleanup é achado**: se `rm -rf` falhou (FS read-only, permissão), reporte no topo
5. **Use só o prefixo `pente-fino-`** ao limpar — nunca toque outros paths em /tmp

---

## Edge cases e armadilhas

### Archive sem extensão

Atacante renomeia `payload.zip` para `LICENSE` ou `README`. Use `file` para detecção:

```bash
file <arquivo-suspeito>
# Saída esperada:
# foo: Zip archive data, at least v2.0 to extract
# bar: gzip compressed data
```

Se um arquivo sem extensão obviamente arquivística retornar tipo de archive → ⚠️ ATENÇÃO Cat. 11 (ofuscação por nome).

### Archive polyglot

Arquivo que é simultaneamente válido como `.zip` E como `.pdf` (ou `.png`, etc.). Técnica conhecida em malware. Detecção:

```bash
file <arquivo>
# Se reportar múltiplos tipos OU
# o tipo declarado pela extensão ≠ tipo detectado pelo conteúdo → 🚨
```

### Self-extracting archive

Arquivos `.exe` que são também `.zip` (SFX). Você pode listar com `unzip -l` mesmo sendo `.exe`. **Nunca execute**. Se a skill referencia "rode setup.exe" e o arquivo é SFX → 🚨 CRÍTICO Cat. 4 + 5.

### Symbolic links dentro do archive

`tar` preserva symlinks. Atacante pode incluir:
```
link_inocente -> /etc/passwd
```

Após extrair, ler `link_inocente` lê `/etc/passwd`. Detecção:

```bash
tar -tvf <arquivo>.tar | grep '^l'  # symlinks listados com 'l'
unzip -l <arquivo>.zip               # zip não suporta symlinks oficialmente
```

Symlinks em archive de skill → 🚨 CRÍTICO Cat. 9.

### Archive com timestamp falsificado

Atacante coloca arquivos com data 2099 ou 1970 para confundir ferramentas de comparação. Detecção:

```bash
unzip -l <arquivo>.zip | awk '$2 > "2030-01-01" || $2 < "2000-01-01" {print}'
```

Não é CRÍTICO sozinho, mas reforça suspeita. ⚠️ ATENÇÃO Cat. 11.

### Filesystem read-only / sem `/tmp` graváveis

Em ambientes muito restritos, `mktemp -d` falha. Fallbacks aceitáveis:
- `$HOME/.cache/pente-fino-XXX` (com mesma trap de cleanup)
- Working directory atual `./pente-fino-tmp-XXX` (último recurso, mais ruidoso)

Nunca use diretórios sistêmicos (`/var/tmp/`, `/usr/local/`) — quebra ownership e expõe a auditoria.

### Container Docker / sandbox sem ferramenta

Em sandbox sem `unzip` ou `tar` disponíveis:
- **NÃO** instale com `apt-get install` (você não tem certeza da reputação da fonte)
- **NÃO** baixe binário pré-compilado (mesma razão)
- Use Python builtin como fallback:

```python
import zipfile, tarfile, tempfile, shutil, os

with tempfile.TemporaryDirectory(prefix="pente-fino-") as work:
    with zipfile.ZipFile(archive_path) as zf:
        # SEMPRE liste antes de extrair
        for name in zf.namelist():
            if name.startswith('/') or '..' in name:
                raise ValueError(f"Suspicious path: {name}")
        zf.extractall(work)
    # ... análise ...
# tempfile context manager faz cleanup automático ao sair
```

Python builtin `zipfile` e `tarfile` são seguros (vêm com a stdlib). Esta é a opção mais portável.

---

## Resumo dos achados típicos em archives

| Sintoma no archive | Categoria pente-fino | Severidade típica |
|---------------------|----------------------|-------------------|
| Senha exigida | Cat. 4 + 11 | 🚨 CRÍTICO automático |
| Path traversal (`../`) | Cat. 9 | 🚨 CRÍTICO |
| Caracteres invisíveis no nome | Cat. 11 | 🚨 CRÍTICO |
| Extensão dupla (`.txt.exe`) | Cat. 11 | 🚨 CRÍTICO |
| Razão compressão >100x | Cat. 5 | 🚨 CRÍTICO (zip bomb) |
| Symlink dentro | Cat. 9 | 🚨 CRÍTICO |
| Binário executável inesperado | Cat. 4 | 🚨 CRÍTICO |
| Nesting >3 níveis | Cat. 11 | 🚨 CRÍTICO |
| Polyglot (file ≠ extensão) | Cat. 11 | 🚨 CRÍTICO |
| Timestamp falsificado | Cat. 11 | ⚠️ ATENÇÃO |
| Tamanho >50MB sem justificativa | Cat. 4 | ⚠️ ATENÇÃO |
| Ferramenta indisponível | — | ⚠️ ATENÇÃO (não auditado) |

---

## Comando one-liner pra cleanup global (housekeeping)

Roda diariamente ou no fim de sessão:

```bash
# Lista o que vai apagar (dry-run)
find /tmp -maxdepth 1 -name "pente-fino-*" -type d -print

# Apaga workspaces antigos (>1 dia) — não destrutivo a outros tools
find /tmp -maxdepth 1 -name "pente-fino-*" -type d -mtime +1 -exec rm -rf {} +

# Apaga TODOS os pente-fino, independente de idade
find /tmp -maxdepth 1 -name "pente-fino-*" -type d -exec rm -rf {} +
```

O prefixo determinístico `pente-fino-` é o que torna isso seguro: nunca toca outros workspaces no `/tmp`.
