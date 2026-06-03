---
name: criar-nova-aula-Delta
description: "Cria e publica aulas em tools.deltaacademy.com.br. Usar quando professor pedir para criar aula, módulo ou aula avulsa no repositório deltaacademy-ai/tools."
---

# criar-nova-aula-Delta

Skill para professores da Delta Academy gerarem e publicarem aulas em `tools.deltaacademy.com.br`.

**Repositório GitHub:** `deltaacademy-ai/tools`
**URL pública (domínio customizado):** `tools.deltaacademy.com.br/`

---

## PASSO 0 — Carregar o Design System (OBRIGATÓRIO)

Ler na íntegra e na ordem antes de qualquer outra ação:

1. `design-system/DESIGN_SYSTEM.md`
2. `design-system/palettes.css`
3. `design-system/templates/index.reference.html`
4. `design-system/templates/tutorial.reference.html`

Todos os caminhos são relativos à raiz do projeto. Se qualquer arquivo não existir, parar imediatamente e avisar o usuário.

Após carregar, confirmar em uma linha. Exemplo:
> ✅ Design System Delta Academy carregado (paleta default: dusk).

---

## PASSO 1 — Identificar o professor

### Primeira vez usando a skill

Perguntar em uma única mensagem:

1. **Username GitHub** — ex: `ana` (será a pasta raiz no repositório)
2. **Senha do painel** — senha pessoal que protege a página de controle `dominio.com/ana` — só o professor sabe, nunca é repassada a alunos

Salvar essas informações e seguir para o Passo 2.

### Já usou antes

Perguntar apenas o username e verificar se a pasta existe no repositório:

```bash
gh api repos/deltaacademy-ai/tools/contents/{usuario} --silent && echo "existe" || echo "não encontrado"
```

- **Existe:** seguir para o Passo 2.
- **Não encontrado:** avisar o professor — *"Não encontrei sua pasta no repositório. Vamos criar do zero."* — e coletar username e senha do painel, como se fosse a primeira vez.

---

## PASSO 2 — Definir o que será criado

Perguntar em uma única mensagem:

1. **O que quer criar?**
   - (A) Módulo novo com aulas
   - (B) Aula nova em módulo existente
   - (C) Aula avulsa (sem módulo)

2. **Roteiro** — pedir que cole o conteúdo da aula agora, se ainda não colou

### Se escolher (B) — Aula em módulo existente

Consultar o repositório via GitHub API e listar as pastas dentro de `{usuario}/`:

```
Módulos encontrados:
  1. ia-para-executivos (3 aulas)
  2. growth (1 aula)
  3. Criar módulo novo

Em qual deseja adicionar?
```

### Para qualquer opção, perguntar também:

- **Nome/título** da aula ou módulo
- **Proteção por senha** — regras por tipo:
  - **Módulo novo (opção A):** perguntar "Deseja proteger este módulo com senha?". Se sim, pedir a senha (ex: `delta101`) — ela valerá para todas as aulas do módulo. Avisar: "Guarde essa senha — ela será usada em todas as aulas deste módulo." Se não, o módulo ficará aberto sem autenticação.
  - **Módulo existente (opção B):** **nunca perguntar senha**. Herdar a configuração existente do módulo via `gh api`:
    ```bash
    gh api repos/deltaacademy-ai/tools/contents/{usuario}/{slug-modulo}/index.html \
      --jq '.content' | base64 -d | grep -o 'const SENHA = "[^"]*"' | head -1
    ```
    Se retornar vazio, o módulo não tem senha — manter sem senha. Se retornar erro, perguntar ao professor.
  - **Aula avulsa (opção C):** perguntar "Deseja proteger esta aula com senha?". Se sim, pedir a senha. Se não, a aula ficará aberta sem autenticação.

> **Regra absoluta:** aulas individuais dentro de um módulo **nunca têm senha própria**. A autenticação é sempre feita uma única vez no índice do módulo (`{slug-modulo}/index.html`) e libera todas as aulas automaticamente via `localStorage`.

Confirmar slug (kebab-case, sem acentos — converter automaticamente) e título antes de gerar.

---

## PASSO 3A — Criar/atualizar página de controle do professor

O arquivo `{usuario}/index.html` é o painel pessoal do professor. **Usar `design-system/templates/index.reference.html` como base exata** — não simplificar. Criado na primeira vez e atualizado a cada nova aula ou módulo.

### Comportamento da página

- Gate de senha do painel (definido no Passo 1) — oculta `#main-content` até autenticação
- Lista módulos em cards expansíveis (`.modulo-card` com `.modulo-head` clicável)
- Cada módulo mostra: nome, número de aulas, data da última atualização
- O cabeçalho do módulo (`.modulo-head`) contém **obrigatoriamente** os 3 recursos abaixo
- Cada linha de aula (`.aula-row`) contém **apenas** número + título clicável — sem ações repetidas

### Recursos obrigatórios no `.modulo-head` (cabeçalho de cada módulo)

Os 3 recursos ficam dentro de `.modulo-head-actions` com `onclick="event.stopPropagation()"` para não disparar o toggle do módulo. O chevron `▼` fica **fora** do `.modulo-head-actions`, como filho direto do `.modulo-head`, para permanecer clicável junto com o título.

**Nota sobre `{URL_MODULO}`:** estes 3 botões sempre apontam para o **índice do módulo** (`{slug-modulo}/index.html`), não para o tutorial-01. Isso garante um entry point único e estável.

```js
// Correto
const url = `https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/`;
// Errado — nunca usar
const url = `https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/tutorial-01-*.html`;
```

**1. Olho (ver/ocultar senha do módulo)**

```html
<button class="mod-icon-btn" id="eye-mod-{NN}" title="Ver senha"
  onclick="toggleSenha(this, '{SENHA_DO_MODULO}', 'senha-mod-{NN}')">
  <svg viewBox="0 0 24 24">
    <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/>
    <circle cx="12" cy="12" r="3"/>
  </svg>
</button>
<span class="mod-senha-reveal" id="senha-mod-{NN}" style="display:none">
  <span class="senha-label">Senha</span>{SENHA_DO_MODULO}
</span>
```

**2. Copiar Mensagem** (plain text + HTML para WhatsApp/e-mail)

```html
<button class="mod-action-btn"
  onclick="copyMsg(this, '{Nome do Módulo}', '{URL_MODULO}', '{SENHA_DO_MODULO}')">
  <svg viewBox="0 0 24 24"><path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/></svg>
  Copiar Mensagem
</button>
```

**3. Copiar Link** (link do índice do módulo)

```html
<button class="mod-action-btn" onclick="copyLink(this, '{URL_MODULO}')">
  <svg viewBox="0 0 24 24"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/></svg>
  Copiar link
</button>
```

### IDs únicos por módulo

Cada módulo tem seu próprio índice numérico (`{NN}` = 01, 02, 03…) nos IDs `eye-mod-{NN}` e `senha-mod-{NN}` para que os olhos funcionem independentemente.

### Funções JS obrigatórias no `<script>`

Copiar integralmente do template de referência:
- `checkPwd()` — gate de senha
- `toggleSenha(btn, senha, tagId)` — olho com `EYE_OPEN` / `EYE_CLOSE`
- `toggleModulo(head)` — expande/colapsa módulo
- `copyMsg(btn, titulo, url, senha)` — mensagem rich text + plain text
- `copyLink(btn, url)` — só o link
- `feedback(btn, msg)` — feedback visual "Copiado ✓" por 2 s
- IIFE de palette switcher com localStorage

---

## PASSO 3B — Gerar os HTMLs das aulas

### Regras absolutas do design system (preservar 100%)

1. `<html lang="pt-BR" data-palette="dusk">` — dark mode é o padrão
2. Favicon: `<link rel="icon" href="https://tools.deltaacademy.com.br/icon.png">` — sempre antes do link do Montserrat
3. Montserrat via Google Fonts
4. CSS inline — nunca linkar arquivos externos
5. Conteúdo integral de `palettes.css` dentro do `<style>`
6. JS inline: palette switcher + navegação/progresso/copy
7. Botão toggle de paleta fixo no top-right (`.palette-toggle`) — ícone sol quando em dusk, lua quando em cream; alterna apenas entre dusk e cream
8. Card: head escuro (card-tag-bg) + body branco com desc
9. Blocos de prompt: label verde fora + bloco branco borda verde 4px dentro
10. Section badges todas iguais (verde escuro)
11. Sidebar: grupos 0.82rem 800 creme, subitens 0.74rem 400. Logo Delta obrigatória como primeiro elemento do `.sidebar-header`:
    ```html
    <div class="sidebar-logo"><img src="../../logo_delta.png" alt="Delta Academy"></div>
    ```
    ```css
    .sidebar-logo img { max-width: 148px; height: auto; display: block; }
    ```
12. PKEY único por aula, TOTAL reflete número real de seções
13. Nenhum emoji que não estava no roteiro original
14. CSS do `.header-bar` — sempre usar `color:var(--bg)` sem opacidade:
    ```css
    .header-bar { background:var(--accent2); border-radius:14px; padding:26px 28px 22px; }
    .header-bar .label { font-size:0.73rem; color:var(--bg); text-transform:uppercase;
      letter-spacing:0.12em; font-weight:800; margin-bottom:8px; }
    .header-bar h1 { font-size:1.35rem; font-weight:800; color:var(--bg); letter-spacing:0.01em; }
    .header-bar p  { font-size:0.8rem; color:var(--bg); margin-top:6px; font-weight:700; }
    ```
    > Nunca usar `opacity` em texto de `.header-bar`. Nunca usar `var(--gold)` como `color` no header — em dusk `--accent2 == --gold`, tornando o texto invisível. Usar sempre `color:var(--bg)` sem opacidade.

> **Isolamento de alunos:** Páginas acessadas por alunos (`{modulo}/index.html` e todos os `tutorial-*.html`) **nunca** devem conter link, botão, breadcrumb ou qualquer texto que referencie o painel do professor (`{usuario}/index.html`). O painel é uma URL privada — alunos não devem saber que existe.

### Proteção por senha — autenticação por módulo

O gate de senha fica **somente no `{modulo}/index.html`**. Após autenticar, o acesso é salvo em `localStorage` e todos os tutoriais do módulo ficam desbloqueados automaticamente. **As aulas individuais nunca têm gate próprio.**

**Se o módulo tiver senha — `{modulo}/index.html`:**

```js
const MOD_KEY = "delta-mod-{slug-modulo}";
const SENHA   = "{SENHA_DO_MODULO}";
function checkPwd() {
  if (document.getElementById('pwd').value === SENHA) {
    localStorage.setItem(MOD_KEY, '1');
    unlock();
  } else {
    document.getElementById('pwd-error').style.display = 'block';
    document.getElementById('pwd').value = '';
    document.getElementById('pwd').focus();
  }
}
function unlock() {
  document.getElementById('gate').style.display = 'none';
  document.getElementById('main-content').style.display = 'block';
}
if (localStorage.getItem(MOD_KEY) === '1') unlock();
```

**Se o módulo tiver senha — `tutorial-NN-*.html`:** sem gate HTML, `#main-content` sem `display:none`. Primeira linha do `<script>`:

```js
if (localStorage.getItem('delta-mod-{slug-modulo}') !== '1') {
  window.location.replace('index.html');
}
```

**Se o módulo não tiver senha:** nem o `{modulo}/index.html` nem os `tutorial-NN-*.html` incluem gate ou verificação de localStorage. O `#main-content` fica visível diretamente, sem nenhuma lógica de autenticação.

O gate do painel do professor (`{usuario}/index.html`) segue o mesmo padrão, copiado de `design-system/templates/index.reference.html`, com título fixo **"Painel do Professor"** e `const SENHA = "{SENHA_DO_PAINEL}"` (senha do painel, não do módulo).

**Aula avulsa com senha — `avulsos/tutorial-{slug}.html`:** gate diretamente na página, sem `localStorage`, sem redirect:

```js
const SENHA = "{SENHA_DA_AULA}";
function checkPwd() {
  if (document.getElementById('pwd').value === SENHA) {
    document.getElementById('gate').style.display = 'none';
    document.getElementById('main-content').style.display = 'block';
  } else {
    document.getElementById('pwd-error').style.display = 'block';
    document.getElementById('pwd').value = '';
    document.getElementById('pwd').focus();
  }
}
```

O `<div id="main-content">` começa com `display:none` e é revelado após autenticação.

**Aula avulsa sem senha:** sem gate, sem `localStorage`. O `#main-content` fica visível diretamente.

### Estrutura de pastas no repositório

**Módulo com aulas:**

```
{usuario}/
├── index.html                          ← painel do professor (sempre atualizado)
└── {slug-modulo}/
    ├── index.html                      ← índice do módulo (protegido por senha)
    ├── tutorial-01-{slug-aula}.html
    ├── tutorial-02-{slug-aula}.html
    └── ...
```

**Aula avulsa:**

```
{usuario}/
├── index.html
└── avulsos/
    └── tutorial-{slug}.html
```

Todos os links internos (breadcrumbs, sidebar, nav-buttons) usam caminhos relativos corretos.

### Numeração automática

Ao adicionar aula em módulo existente, consultar o repositório para saber quantas aulas já existem e numerar corretamente a nova. Ex: se já há `tutorial-01` e `tutorial-02`, a nova é `tutorial-03`.

---

## PASSO 4 — Publicar via GitHub

Executar os comandos abaixo no terminal (ou via Claude Code):

**Novo módulo:**

```bash
git add {usuario}/{slug-modulo}/ {usuario}/index.html
git commit -m "Add módulo: {Nome do Módulo} [{usuario}]"
git push origin main
```

**Nova aula em módulo existente:**

```bash
git add {usuario}/{slug-modulo}/tutorial-{NN}-{slug}.html {usuario}/{slug-modulo}/index.html {usuario}/index.html
git commit -m "Add aula: {Título} em {slug-modulo} [{usuario}]"
git push origin main
```

**Aula avulsa:**

```bash
git add {usuario}/avulsos/tutorial-{slug}.html {usuario}/index.html
git commit -m "Add aula avulsa: {Título} [{usuario}]"
git push origin main
```

Só entregar URLs após push retornar sucesso. Se der erro, reportar sem entregar URL.

---

## PASSO 5 — Entregar ao usuário

Após push bem-sucedido, retornar com todos os placeholders substituídos. Usar o template correspondente ao tipo de criação:

**Opção A — Módulo novo com senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/
   🔑 Senha do painel: (você já sabe)

🔗 Link para compartilhar com os alunos:
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/

🔑 Senha do módulo: {SENHA_DO_MODULO}

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
🎨 Paleta default: Dusk (modo escuro). Ícone no canto superior direito alterna para Cream.
```

**Opção A — Módulo novo sem senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/
   🔑 Senha do painel: (você já sabe)

🔗 Link para compartilhar com os alunos:
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
🎨 Paleta default: Dusk (modo escuro). Ícone no canto superior direito alterna para Cream.
```

**Opção B — Aula em módulo existente com senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/

🔗 Link do módulo (entrada para os alunos):
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/

🆕 Nova aula adicionada:
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/tutorial-{NN}-{slug-aula}.html

🔑 Senha do módulo: {SENHA_DO_MODULO} (a mesma de sempre)

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
```

**Opção B — Aula em módulo existente sem senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/

🔗 Link do módulo (entrada para os alunos):
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/

🆕 Nova aula adicionada:
   https://tools.deltaacademy.com.br/{usuario}/{slug-modulo}/tutorial-{NN}-{slug-aula}.html

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
```

**Opção C — Aula avulsa com senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/

🔗 Link para compartilhar com os alunos:
   https://tools.deltaacademy.com.br/{usuario}/avulsos/tutorial-{slug-aula}.html

🔑 Senha da aula: {SENHA_DA_AULA}

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
🎨 Paleta default: Dusk (modo escuro). Ícone no canto superior direito alterna para Cream.
```

**Opção C — Aula avulsa sem senha:**

```
✅ Publicado com sucesso!

📋 Seu painel:
   https://tools.deltaacademy.com.br/{usuario}/

🔗 Link para compartilhar com os alunos:
   https://tools.deltaacademy.com.br/{usuario}/avulsos/tutorial-{slug-aula}.html

⏱ Aguarde 30–60 segundos antes de abrir (GitHub Pages leva alguns segundos).
🎨 Paleta default: Dusk (modo escuro). Ícone no canto superior direito alterna para Cream.
```

---

## CHECKLIST DE VALIDAÇÃO (antes de qualquer commit)

**HTML das aulas (tutorial):**
- [ ] `<html lang="pt-BR" data-palette="dusk">` (dark mode padrão)
- [ ] Favicon presente: `<link rel="icon" href="https://tools.deltaacademy.com.br/icon.png">`
- [ ] Montserrat importado do Google Fonts — única exceção ao CSS inline (além do favicon)
- [ ] CSS de paletas inline: apenas dusk e cream (não Forest)
- [ ] Conteúdo de `palettes.css` inline dentro de `<style>` (paletas dusk e cream)
- [ ] Nenhum `<link href="palettes.css">` ou qualquer outro CSS externo
- [ ] `.palette-toggle` fixo no top-right — ícone sol (dusk) ou lua (cream), alterna apenas entre os dois modos
- [ ] `{modulo}/index.html` tem gate de senha com `MOD_KEY`, `checkPwd()` e `unlock()` corretos
- [ ] `tutorial-NN-*.html` sem gate — `#main-content` sem `display:none`, primeira linha do script redireciona para `index.html` se não autenticado
- [ ] Prompt: label verde fora, bloco branco borda verde 4px dentro
- [ ] Section badges todas iguais (verde escuro `var(--accent2)`)
- [ ] Sidebar: grupos 0.82rem 800, subitens 0.74rem 400
- [ ] PKEY único por aula, TOTAL reflete número real de seções
- [ ] Nenhuma emoji que não estava no roteiro

**Painel do professor (index.html):**
- [ ] Gate de senha com `const SENHA = "{SENHA_DO_PAINEL}"` correta
- [ ] Cada `.modulo-head` tem dentro de `.modulo-head-actions`: olho (`toggleSenha`), tag senha, "Copiar Mensagem" (`copyMsg`), "Copiar link" (`copyLink`)
- [ ] `.aula-row` contém apenas `.aula-top` (número + título clicável) — sem botões de ação
- [ ] IDs únicos por módulo: `eye-mod-{NN}`, `senha-mod-{NN}`
- [ ] Funções JS: `toggleSenha`, `toggleModulo`, `copyMsg`, `copyLink`, `feedback`, palette IIFE
- [ ] URLs das aulas usando domínio `tools.deltaacademy.com.br/{usuario}/...`

**Repositório:**
- [ ] Estrutura de pastas: `{usuario}/{slug-modulo}/tutorial-{NN}-{slug}.html`
- [ ] Painel do professor (`{usuario}/index.html`) atualizado com nova aula
- [ ] Push feito para `deltaacademy-ai/tools` na branch `main`

---

## REGRAS OPERACIONAIS

- Nunca usar caminhos absolutos (`/Users/...`)
- Nunca expor nome completo, senha do painel ou qualquer dado pessoal do professor em páginas acessadas por alunos
- Todos os caminhos são relativos ao repositório
- Se surgir dúvida entre roteiro e design system, perguntar antes de decidir
- O design system é relido a cada execução — nunca assumir que está em cache
- Não existe índice raiz global — cada professor tem apenas seu próprio painel (`{usuario}/index.html`)
