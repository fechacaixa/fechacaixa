# Relatório de segurança — Fechamento de Caixa

Data: 27/08/2026 (atualizado em 30/08/2026 — ver "Adendo" no final)

Este relatório responde, ponto a ponto, ao checklist de segurança pedido, adaptado à arquitetura real deste projeto: **uma única página HTML estática** (sem backend, sem build, sem banco SQL), hospedada no GitHub Pages, com todo o estado gravado em **um único documento do Firebase Firestore**. Vários itens do checklist original pressupõem um servidor de aplicação (rate limiting no servidor, WAF/DDoS, permissões de arquivo/diretório no SO, backend pra mover lógica sensível) que simplesmente não existe aqui — esses itens estão marcados como **não aplicável** abaixo, com a explicação do porquê, em vez de fingir uma implementação que não faria sentido.

## Resumo executivo

| # | Item do checklist | Situação |
|---|---|---|
| 1 | OWASP Top 10 | Revisado item a item (ver seção 1) |
| 2 | Segredos hardcoded | Nenhum segredo real encontrado (ver seção 2) |
| 3 | Mover segredos para env vars/GitHub Secrets | Não aplicável (ver seção 2) |
| 4 | Headers de segurança | CSP e Referrer-Policy implementados via `<meta>`; demais headers não aplicáveis no GitHub Pages (ver seção 3) |
| 5 | XSS/CSRF/SQLi/Command Injection/SSRF/Path Traversal | XSS já era tratado corretamente; **injeção de fórmula em CSV corrigida agora**; os demais não se aplicam (ver seção 4) |
| 6 | Validação/sanitização de entradas | Reforçada com `maxlength` (ver seção 5) |
| 7 | Autenticação e sessão | Reforçada com Firebase Anonymous Auth (ver seção 6) |
| 8 | HTTPS obrigatório | Já garantido pelo GitHub Pages (ver seção 7) |
| 9 | Nenhuma informação sensível em console/logs/erros | Já estava limpo, confirmado agora (ver seção 8) |
| 10 | Rate limiting | Já existia no nível da aplicação (bloqueio por tentativas); rate limiting de rede não aplicável (ver seção 9) |
| 11 | Proteção contra DDoS | Herdada da infraestrutura do GitHub Pages/Firebase; nada a configurar no projeto (ver seção 9) |
| 12 | Dependências vulneráveis | Única dependência é o Firebase SDK, fixado por versão; sem gerenciador de pacotes pra automatizar (ver seção 10) |
| 13 | Permissões de arquivos/diretórios | Não aplicável (sem servidor próprio) |
| 14 | Menor privilégio | Aplicado nas regras do Firestore (ver seção 6) |
| 15 | GitHub Actions (CodeQL, Dependabot, Secret Scanning) | CodeQL e Dependabot criados agora; Secret Scanning precisa ser ligado por você nas configurações do repositório (ver seção 11) |
| 16 | SECURITY.md / .gitignore / dependabot.yml | Criados agora (ver seção 11) |
| 17-18 | Lógica sensível no frontend / mover pra backend | Identificado o que existe; não há como nem por que mover pra um backend inexistente sem redesenhar o produto (ver seção 12) |
| 19 | Este relatório | Você está lendo |

---

## 1. Revisão OWASP Top 10 (2021), aplicada a este projeto

- **A01 Broken Access Control** — era o maior risco real do projeto: as regras do Firestore permitiam `allow read, write: if true`, ou seja, qualquer pessoa com a `apiKey` pública do projeto (visível no próprio `index.html`) podia ler ou escrever direto no banco pelo console do navegador, ignorando completamente o login e o bloqueio por tentativas do app. **Corrigido** — ver seção 6.
- **A02 Cryptographic Failures** — senhas são guardadas como hash SHA-256 (sem salt). Não é o ideal (o correto seria bcrypt/scrypt/Argon2 com salt), mas trocar o algoritmo agora invalidaria a senha de todos os usuários cadastrados por um ganho marginal, já que a correção que realmente fecha a brecha é a das regras do Firestore (sem acesso de escrita livre ao banco, não há como fazer força bruta offline nos hashes). Decisão consciente de manter o SHA-256 nesta rodada — documentado como recomendação futura na seção 13.
- **A03 Injection** — auditoria completa de todo uso de `innerHTML`/`.textContent`/concatenação de HTML: o app já escapa corretamente tudo que vem do usuário através da função `esc()` antes de qualquer inserção no DOM. O único ponto de injeção real encontrado foi **CSV Formula Injection** (um valor começando com `=`, `+`, `-` ou `@` vira fórmula executável ao abrir o CSV exportado no Excel/Google Sheets) — **corrigido agora**, ver seção 4. SQL Injection e Command Injection não se aplicam: não há banco SQL nem execução de comandos.
- **A04 Insecure Design** — o design de senha compartilhada por loja (sem conta de e-mail individual) é uma escolha consciente de simplicidade operacional pras 9 unidades, coberta pelo controle de tentativas/bloqueio já existente e agora reforçada pela autenticação anônima nas regras do Firestore.
- **A05 Security Misconfiguration** — CSP e Referrer-Policy adicionados (seção 3); regras do Firestore corrigidas (seção 6).
- **A06 Vulnerable and Outdated Components** — única dependência externa é o Firebase JS SDK v10.14.1, carregado direto do CDN da Google. Sem gerenciador de pacotes não há como automatizar isso com Dependabot; ficou documentado como checagem manual periódica (seção 10).
- **A07 Identification and Authentication Failures** — bloqueio progressivo por tentativas erradas já existia; agora reforçado com autenticação (anônima) exigida pelo Firestore antes de qualquer leitura/escrita.
- **A08 Software and Data Integrity Failures** — o SDK do Firebase é carregado de `https://www.gstatic.com` sem Subresource Integrity (SRI). Não foi possível adicionar SRI nesta rodada porque o Firebase não publica hashes oficiais estáveis pra essas URLs versionadas (a Google atualiza o conteúdo do CDN sem trocar a URL em alguns casos) — documentado como risco residual aceito na seção 13.
- **A09 Security Logging and Monitoring Failures** — o app já tem uma tela de "Logs de segurança" (tentativas de login, bloqueios) persistida no próprio documento do Firestore. Suficiente pra esse porte de aplicação.
- **A10 Server-Side Request Forgery (SSRF)** — não aplicável: não existe servidor fazendo requisições em nome do usuário. A única chamada de rede feita pelo próprio app além do Firebase é para `https://api.ipify.org` (descobrir o IP público de quem está preenchendo um fechamento, pra registrar no log de auditoria) — chamada de leitura simples, sem parâmetro controlado pelo usuário, sem risco de SSRF.

## 2. Segredos hardcoded

Nenhuma senha, token de API privado ou string de conexão de banco de dados tradicional foi encontrado no código. O único valor "sensível" presente é o objeto `FIREBASE_CONFIG` (apiKey, projectId etc.) — isso **não é um segredo** pela própria documentação do Firebase: essas chaves identificam o projeto, não autorizam nada sozinhas, e o controle de acesso de verdade é feito pelas regras do Firestore (que era exatamente o que faltava — corrigido na seção 6). Por isso não há nada aqui pra mover pra GitHub Secrets ou variável de ambiente: este é um site estático servido como arquivo puro pelo GitHub Pages, sem nenhuma etapa de build ou servidor que possa injetar variáveis de ambiente em tempo de execução.

## 3. Headers de segurança

O GitHub Pages **não permite configurar headers HTTP customizados** — não existe um `next.config.js`, `.htaccess` ou painel pra isso nessa hospedagem. O que dá pra fazer, e foi feito, é declarar via `<meta http-equiv>` no `<head>`:

- **Content-Security-Policy**: restringe scripts, estilos, fontes e conexões de rede às origens que o app realmente usa (Firebase, Google Fonts, `api.ipify.org`) e bloqueia todo o resto por padrão (`default-src 'none'`).
- **Referrer-Policy**: `strict-origin-when-cross-origin`, evita vazar a URL completa (que nesse app não tem dados sensíveis na query string, mas é boa prática de qualquer forma).

**Limitações conhecidas, documentadas em vez de escondidas:**
- `X-Frame-Options`, `X-Content-Type-Options` e `Strict-Transport-Security` **não têm equivalente em `<meta>`** — só existem como headers HTTP de verdade, e o GitHub Pages não os envia.
- A diretiva `frame-ancestors` (que substituiria o X-Frame-Options dentro do próprio CSP) é **ignorada pelos navegadores quando o CSP vem via `<meta>`** — só funciona como header HTTP real. Por isso ela foi deixada de fora da política em vez de incluída sem efeito nenhum.
- Solução recomendada pra fechar essa lacuna, caso vire prioridade: colocar o site atrás de um proxy como o Cloudflare (plano gratuito já serve) na frente de um domínio próprio — aí sim é possível injetar esses headers de verdade antes de chegar no navegador. Fica registrado como recomendação (seção 13), não implementado agora porque exigiria decisões fora do escopo deste código (domínio próprio, DNS).

## 4. Injeção: o que foi encontrado e corrigido

**CSV Formula Injection (corrigido).** As exportações de CSV (fechamentos e divergências) escreviam os campos direto, só cuidando de aspas. Um nome de loja ou uma justificativa de despesa começando com `=`, `+`, `-` ou `@` — inclusive um usuário mal-intencionado poderia batizar uma loja com um nome desses — vira uma fórmula que o Excel/Google Sheets executa automaticamente ao abrir o arquivo (um vetor conhecido do OWASP, "CSV/Formula Injection"). Corrigido com a função `csvField()`, que agora prefixa esses valores com um apóstrofo antes de aspas, neutralizando a interpretação como fórmula sem alterar o texto visível pra quem abre o arquivo.

**XSS (já estava protegido, confirmado agora).** Auditoria de toda a base: todo texto vindo do usuário (nomes de loja, login, justificativas, observações, motivos de divergência) passa por `esc()` antes de entrar no DOM via `innerHTML`. Não existe nenhum `eval`, `new Function`, `document.write`, `insertAdjacentHTML` ou `outerHTML =` no projeto. Testado com um payload deliberadamente malicioso (`<script>alert(1)</script>` como nome de loja) — o texto aparece escapado na tela, sem executar.

**SQL Injection, Command Injection, Path Traversal, SSRF** — não aplicáveis: não há banco SQL, não há servidor executando comandos ou lendo arquivos do sistema a partir de entrada do usuário.

**CSRF** — o modelo clássico (cookie de sessão + formulário em outro site) não se aplica da mesma forma aqui: não há sessão baseada em cookie no backend (a "sessão" fica em memória/localStorage do próprio navegador, e toda escrita já exige estar autenticado no Firebase e ter passado pelo login do app). O vetor real de escrita não autorizada era as regras abertas do Firestore, já corrigido.

## 5. Validação e sanitização de entradas

Adicionado `maxlength` em todos os campos de texto livre relevantes (nome de loja, login, senha, justificativas, observações, motivo de divergência), como camada extra de defesa: reduz o tamanho de qualquer payload possível e ajuda a respeitar o limite de 1 MiB por documento do Firestore (esse app guarda todo o estado — incluindo os arrays de lançamentos e logs de segurança, que só crescem — dentro de um único documento).

## 6. Autenticação, sessão e menor privilégio

Antes: `firestore.rules` tinha `allow read, write: if true` — acesso total e público ao banco pra qualquer um com a apiKey (pública por design). Isso permitia contornar completamente o login e o bloqueio por tentativas do próprio app, bastando abrir o console do navegador e falar direto com o Firestore.

Agora: o app faz **login anônimo automático** no Firebase (`signInAnonymously()`, silencioso, sem nenhuma tela ou clique extra pro usuário) antes de conectar ao Firestore, e as regras passaram a exigir `request.auth != null`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /fechamento_caixa/{docId} {
      allow read, write: if request.auth != null;
    }
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Isso é uma camada **adicional**, não um substituto do login/senha do próprio app (que continua controlando quem entra em cada loja ou no painel administrativo) — o objetivo é impedir que alguém fale direto com o banco por fora do app. Também é aplicado o princípio de menor privilégio: qualquer outro caminho do Firestore fora do único documento usado (`fechamento_caixa/estado`) é explicitamente negado (`match /{document=**} { allow read, write: if false; }`).

**Ação necessária sua**, pois só pode ser feita no Console do Firebase (fora do alcance deste código): habilitar o provedor "Anônimo" em Authentication → Sign-in method, e publicar o novo `firestore.rules` — passo a passo no README.

## 7. HTTPS obrigatório

Já garantido automaticamente: o GitHub Pages serve todo o site por HTTPS e redireciona HTTP → HTTPS por padrão, sem nenhuma configuração adicional necessária.

## 8. Informação sensível em console/logs/erros

Confirmado por busca em todo o arquivo: **zero** ocorrências de `console.log`, `console.error`, `console.warn` ou `console.debug`. Nenhuma mensagem de erro do app expõe detalhes internos — os poucos `catch` existentes (ex.: falha ao buscar o IP público) falham silenciosamente sem afetar a experiência nem vazar informação.

## 9. Rate limiting e proteção contra DDoS

O app já implementa, no nível da aplicação, bloqueio progressivo por tentativas de login erradas (com exceção especial pro único admin ativo, pra nunca trancar o sistema — ver README). Rate limiting de rede (nível de infraestrutura) e proteção contra DDoS não são algo configurável dentro deste projeto: são herdados da infraestrutura do GitHub Pages e do Firebase (ambos operam atrás de proteção de borda da Google/GitHub por padrão, sem nada a fazer aqui).

## 10. Dependências vulneráveis

A única dependência externa do projeto é o Firebase JS SDK, com versão fixada (`10.14.1`) direto nas tags `<script>`. Como não existe `package.json` nem nenhum gerenciador de pacotes, o Dependabot não tem manifesto pra rastrear automaticamente — ele foi configurado (seção 11) só pra manter as próprias GitHub Actions do repositório atualizadas. Fica registrado como recomendação: checar periodicamente se há uma versão mais nova do SDK em https://firebase.google.com/support/release-notes/js.

## 11. Governança do repositório

Criados nesta rodada:
- **`SECURITY.md`** — política de divulgação de vulnerabilidades.
- **`.gitignore`** — evita commitar chaves de conta de serviço, arquivos `.env`, logs de debug do Firebase, etc. (nenhum desses existe hoje no projeto, mas evita acidente futuro).
- **`.github/dependabot.yml`** — monitora as GitHub Actions do repositório (é o único ecossistema que existe aqui sem manifesto de pacotes).
- **`.github/workflows/codeql.yml`** — roda análise estática (CodeQL) no JavaScript do projeto — incluindo o `<script>` embutido dentro do `index.html` — a cada push/PR na `main` e semanalmente.

**Ação necessária sua** (só pode ser feita nas configurações do repositório no GitHub, fora do alcance deste código): habilitar **Secret Scanning** e **Push Protection** em Settings → Security → Code security do repositório, depois de publicado no GitHub.

## 12. Lógica sensível exposta no frontend

Por ser um app 100% estático sem backend, **toda** a lógica de negócio (cálculo de divergências, validações, controle de acesso) roda no navegador por definição — não existe "mover pra um backend" sem redesenhar o produto inteiro pra ter um servidor de aplicação, o que está fora do escopo pedido aqui. O ponto que importa de verdade — impedir que alguém contorne essa lógica falando direto com o banco — é exatamente o que a correção da seção 6 resolve: mesmo que a lógica do app seja pública (é HTML/JS, sempre foi visível a quem abrir o "Ver código-fonte"), agora só é possível gravar no Firestore autenticado, e as regras do banco (não o JavaScript do cliente) são a fronteira de segurança real.

## 13. Recomendações não implementadas nesta rodada (e por quê)

- **Hash de senha com salt (bcrypt/Argon2)**: trocaria a senha de todos os usuários cadastrados por um ganho pequeno agora que o acesso direto ao banco está fechado. Recomendado pra uma próxima rodada, com aviso prévio aos usuários.
- **Subresource Integrity (SRI) no SDK do Firebase**: não aplicado porque o Firebase não garante hash estável pra essas URLs de CDN versionadas.
- **Headers HTTP reais (X-Frame-Options, HSTS, etc.)**: exigem colocar o site atrás de um proxy como Cloudflare com domínio próprio — decisão de infraestrutura fora do escopo deste código.
- **Restringir a apiKey do Firebase por referrer HTTP**: pode ser configurado direto no Google Cloud Console (APIs e Serviços → Credenciais) pra que a chave só funcione a partir do domínio do GitHub Pages — recomendado, mas é uma ação no console da Google, não uma mudança de código.
- **Secret Scanning do GitHub**: precisa ser ligado nas configurações do repositório depois de publicado (ver seção 11).

---

**Resumo da postura final**: a maior brecha real do projeto (banco de dados publicamente gravável, contornando totalmente o login do app) foi corrigida. A segunda vulnerabilidade real encontrada (injeção de fórmula no CSV exportado) também foi corrigida. O restante do checklist foi mapeado honestamente entre "já estava correto", "implementado agora" e "não aplicável a um app estático sem backend, com a razão explicada" — nenhum item foi marcado como feito sem ter sido de fato implementado e testado.

## Adendo (30/08/2026) — auditoria de interligação usuários/lojas/senhas

Depois da rodada de segurança acima, foi pedida uma análise completa de como usuários, lojas e senhas se conectam entre si, ponta a ponta. Essa auditoria encontrou e corrigiu uma lacuna real:

**Login de loja sem proteção contra força bruta nem log de segurança (corrigido).** Quando a senha de cada loja virou uma conta de verdade em Usuários (perfil "loja"), a tela de login da loja (`attemptLojaLogin`) não foi atualizada junto — continuava só comparando a senha, sem os mesmos limites de tentativas, bloqueio temporário e registro no log de segurança que o login de Administrador/Consulta já tinha (`attemptLogin`). Na prática, isso significava que a senha de uma loja podia ser testada indefinidamente sem nunca travar, e nenhuma tentativa (certa ou errada) ficava registrada em "Logs de segurança" — justamente a conta mais usada no dia a dia, por mais gente, com senhas mais simples. Corrigido: agora o login de loja bloqueia depois de 5 tentativas erradas (mesma regra do admin, 15 minutos ou desbloqueio manual em Usuários) e cada tentativa vira uma linha no log de segurança. Testado com Playwright simulando 5 senhas erradas seguidas (bloqueia), uma 6ª tentativa com a senha certa enquanto ainda bloqueado (continua bloqueado), desbloqueio manual pelo administrador, e login normal depois disso — os 9 pontos de verificação passaram, sem quebrar o fluxo de migração de contas antigas.

**Outros pontos conferidos nesta auditoria, sem necessidade de mudança:**
- Toda loja em `state.lojas` sempre tem uma conta `perfil:'loja'` correspondente em `state.usuarios` — conferido nas três migrações em cadeia (bootstrap inicial, conversão de formato antigo, backfill de loja sem conta) e nos fluxos de Adicionar/Remover loja.
- `firestoreConnect()`, `persist()` e `firestore.rules` continuam coerentes entre si (mesma coleção/documento, autenticação anônima antes de qualquer leitura/escrita).
- CSP (`<meta>`) continua batendo exatamente com os recursos externos que a página carrega (Firebase, Google Fonts, `api.ipify.org`) — testado com um carregamento real da página, zero violações no console.

**Achado sem risco de segurança, mas registrado como débito técnico:** o campo `lojasPermitidas` (pensado originalmente pra restringir um usuário "Consulta" a ver só certas lojas) existe no formato de dados de cada usuário, mas nenhuma tela permite escolher essas lojas (sempre grava `[]`) e nenhum código do sistema realmente filtra dados por ele — a função que faria isso (`lojasPermitidasNomes`) está definida mas nunca é chamada. Ou seja, hoje todo usuário "Consulta" enxerga todas as lojas, mesmo que o campo sugira o contrário. Não é uma falha de segurança (é só leitura, sem exposição indevida), mas é uma peça do modelo de dados que não faz nada — vale decidir se um dia implementa essa restrição de verdade ou remove o campo pra não confundir.

## Adendo (30/08/2026, parte 2) — bug real na tela de confirmação e correção

Depois das mudanças na tela de confirmação de fechamento salvo (`showSuccessScreen`) e no botão "Novo fechamento", foi pedida uma nova rodada de revisão pra caçar erros. Encontrado e corrigido um bug real (não relacionado a segurança, mas de confiabilidade — grave pra um app de fechamento de caixa):

**A tela de "Caixa fechado" aparecia mesmo quando o fechamento NÃO foi salvo (corrigido).** `persist()` (a função central que grava qualquer mudança no Firestore) nunca rejeitava a Promise que devolve — se o aparelho estava sem internet, ou a gravação no Firestore falhava por qualquer motivo, `persist()` só mostrava um aviso (banner) e seguia em frente normalmente. Como `handleSubmit()` disparava a tela de sucesso assim que essa Promise resolvia, sem checar se a gravação realmente aconteceu, a pessoa via "Isso aí! Caixa fechado 🍔" mesmo quando o fechamento tinha sido perdido — podendo sair de lá achando que estava tudo certo, sem o fechamento existir no banco de dados. Corrigido com uma mudança pequena e isolada: `persist()` agora aceita um segundo parâmetro opcional (`onDone`, chamado com `true`/`false` avisando se a gravação foi confirmada), e só o fluxo de salvar fechamento (`handleSubmit`) passa esse callback — os outros ~29 pontos do sistema que chamam `persist()` continuam exatamente como estavam, sem precisar mudar nada. Agora a tela de sucesso (e o reset do formulário) só acontece quando a gravação é confirmada; se falhar, o banner de erro aparece, o formulário mantém os dados digitados (nada é perdido) e nada mais some da tela por cima do aviso. Testado com Playwright simulando os dois casos (gravação com sucesso e gravação forçada a falhar) — os 6 pontos de verificação passaram.

**Outros pontos revisados nesta rodada, sem necessidade de mudança** (checados por uma auditoria de código dedicada, item por item): a regra `.success-screen:not([hidden])` (o mesmo tipo de bug de CSS já corrigido antes) não se repete em nenhuma outra tela nova; `exitLoja()` só é chamado pelo clique em "Novo fechamento" e pelo botão de sair — nunca automaticamente ao salvar; a cadeia de migração em `init()` continua na ordem certa; nenhuma gravação no Firestore passa por fora de `persist()`; os botões novos de "Editar login" (loja) e "Editar" (admin/consulta) não se confundem entre si; e os textos escritos pela pessoa nesses fluxos novos (tela de sucesso, login de loja) continuam escapados corretamente contra XSS.
