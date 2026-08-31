# Política de Segurança

Este repositório contém o **Fechamento de Caixa** — um app de página única (HTML/CSS/JS), sem servidor próprio, hospedado no GitHub Pages e sincronizado por um documento único no Firebase Firestore.

## Reportando uma vulnerabilidade

Se você encontrar um problema de segurança neste projeto, entre em contato diretamente com o mantenedor (Willhams) antes de abrir uma issue pública — isso dá tempo de corrigir antes que o problema fique visível pra qualquer pessoa que passe pelo repositório.

Inclua, se possível: o que você encontrou, os passos pra reproduzir, e o impacto que você imagina que isso teria.

## O que este projeto é (e não é)

É importante entender a arquitetura pra avaliar risco corretamente: não existe backend, banco de dados relacional, nem servidor de aplicação. Todo o código roda no navegador de quem abre a página, e o único dado persistido vive num documento do Firebase Firestore. Por isso, boa parte das categorias clássicas de vulnerabilidade web (SQL Injection, Command Injection, Path Traversal, SSRF) **não se aplicam** — não existe consulta a banco relacional, não existe execução de comando de sistema, nem chamada de rede feita a partir de um servidor a mando de uma entrada do usuário.

## Modelo de ameaça

- A "porta de entrada" pública é a leitura/escrita do Firestore, protegida por Firestore Security Rules + autenticação anônima do Firebase (ver `firestore.rules`) — qualquer sessão do app precisa de um token válido emitido pelo Firebase antes de ler ou escrever o documento.
- A separação entre "quem pode fazer o quê" dentro do app (fechar caixa de uma loja, ver o painel gerencial, administrar usuários) é feita pelas senhas de cada loja/conta, com bloqueio automático depois de tentativas erradas seguidas e log de acessos — ver a seção de segurança do `README.md`.
- Como não existe backend, a "lógica de negócio" (cálculo de divergência, validações, etc.) roda no navegador de quem usa o app. Alguém com acesso ao DevTools do próprio navegador consegue inspecionar esse código — isso é uma limitação inerente a um app sem servidor, não um bug específico deste projeto.

## Práticas em vigor

- Toda entrada de texto renderizada na tela passa por uma função de escape (`esc()`) antes de virar HTML, protegendo contra XSS armazenado.
- Exportações em CSV blindam contra "CSV Injection" (um valor começando com `=`, `+`, `-` ou `@` não vira fórmula executável ao abrir no Excel/Sheets).
- Senhas nunca são armazenadas em texto puro — só o hash (SHA-256) fica no Firestore.
- Content-Security-Policy restringe de onde o app pode carregar script/estilo/fonte e para onde pode enviar dados de rede.
- Dependências: o projeto não usa gerenciador de pacotes (não há `package.json`) — a única dependência externa é o Firebase JS SDK, carregado direto do CDN oficial da Google (`gstatic.com`), com a versão fixada no próprio HTML.

Veja `SECURITY_REPORT.md` para o relatório detalhado de uma revisão de segurança feita neste projeto, incluindo o que foi corrigido e o que ainda depende de uma ação manual (Firebase Console, configurações do repositório no GitHub).
