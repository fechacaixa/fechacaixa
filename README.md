# Fechamento de Caixa — versão real (com sincronização entre lojas)

Página única (`index.html`) com o sistema de fechamento de caixa das 9 unidades, com dados sincronizados em tempo real entre todos os dispositivos via Firebase Firestore. Esta é a ferramenta de uso diário — não é a versão de apresentação.

**Importante:** este repositório fica público (o GitHub Pages gratuito exige isso), o que significa que qualquer pessoa com o link consegue abrir a tela de login. Os dados em si só ficam visíveis depois de digitar usuário e senha — mas troque as senhas de fábrica (veja abaixo) antes de divulgar o link pras lojas.

## Layout em tela cheia

O sistema agora ocupa a largura toda da tela (antes ficava numa faixa central de 920px, "encolhido" no meio de monitores grandes) — pensado pra telas 1920px, 2560px e ultrawide. O menu lateral do painel gerencial também ficou um pouco mais largo. O card do formulário de fechamento também é largura total, mas o campo em si (geralmente um por etapa) fica centrado numa coluna confortável de leitura — esticar um campo de dinheiro sozinho pela tela toda ficaria ruim de usar, e nenhum sistema corporativo de verdade faz isso.

## Como publicar no GitHub Pages

1. Crie um repositório novo no GitHub (público — pode ser um nome que não entregue o que é, já que qualquer pessoa pode ver o nome do repositório mesmo sem acessar os dados).
2. Faça upload deste arquivo `index.html` pra raiz do repositório (arraste e solte pela interface do GitHub).
3. No repositório, vá em **Settings → Pages**.
4. Em "Source", selecione a branch `main` e a pasta `/ (root)`. Salve.
5. Em alguns minutos o GitHub mostra o link do site (algo como `https://SEU-USUARIO.github.io/NOME-DO-REPOSITORIO/`).
6. Esse é o link que você compartilha com as 9 lojas — todas vão ver os mesmos dados, atualizados em tempo real.

## Duas portas de entrada: loja e administrador

A tela inicial tem duas abas — **Loja** e **Administrador** — cada uma com sua própria senha. Ela sempre abre na aba Loja, que é o uso do dia a dia:

- **Fechar caixa de uma loja** — escolhe a loja numa lista, digita a senha dela (uma senha só, compartilhada por quem estiver no caixa daquela unidade) e o nome de quem vai fechar agora. Não existe mais conta individual por funcionário aqui: qualquer pessoa da equipe pode fechar o caixa, desde que saiba a senha da loja, e o nome digitado é o que aparece no topo da tela e fica gravado naquele fechamento. Um link "Trocar de loja" no topo deixa trocar de loja (ou passar o aparelho pra outra pessoa) a qualquer momento, sem precisar voltar pra tela inicial.
- **Administrador/Consulta** — login com usuário e senha individuais, do jeito que já era: **Administrador** tem acesso total (painéis, histórico de todas as lojas, gerenciamento de usuários/lojas/motivos); **Consulta** entra no painel gerencial e vê tudo, mas não consegue criar, editar, aprovar ou excluir nada — é o perfil pra quem só precisa acompanhar os números (contador, sócio, etc.). Essas contas continuam com toda a proteção de segurança: bloqueio automático depois de tentativas erradas seguidas e log de acessos (veja "Proteção contra tentativas excessivas de login" abaixo).

Login padrão de fábrica: usuário `admin`, senha `8081` — troque assim que publicar.

## Login e senha de cada loja

Adicionar ou remover uma loja é feito na seção **Lojas** do painel gerencial (Administração → Lojas → **+ Adicionar loja**) — o modal pede o **Nome** da loja e o **Login**, já sugerido automaticamente a partir do nome (dá pra trocar o login sugerido antes de confirmar). Nome e Login não são a mesma coisa: o Nome identifica a loja em todo o sistema (Dashboard, Relatórios, etc.) e não muda mais depois; o Login existe só pra identificar a conta dela — pode ser ajustado a qualquer momento.

A senha, o próprio login e o bloqueio/ativação da conta ficam todos gerenciados junto com as outras contas na seção **Usuários** (Administração → Usuários) — cada loja aparece lá como uma linha com perfil **Loja**, e o menu "⋮" dela tem **Editar login**, **Trocar senha** e **Bloquear/Ativar** (dá pra desativar uma loja temporariamente sem removê-la da lista). Isso mantém tudo num único lugar, junto com as contas de Administrador/Consulta, em vez de espalhado em duas telas.

Uma loja nova já nasce com uma senha (sequência simples, ex. `2026`, `2027`...) assim que é cadastrada — troque em Usuários assim que possível. Recomendo trocar as senhas de fábrica das lojas já existentes antes de divulgar o link.

## Depois de publicar: troque as senhas

1. Entre como administrador (`admin` / `8081`) e troque essa senha primeiro (seção **Usuários**, no menu ⋮ da sua própria conta).
2. Na seção **Lojas**, troque a senha de cada unidade (botão **Trocar senha** em cada linha) antes de divulgar o link pras equipes.
3. Crie um usuário individual (perfil Administrador ou Consulta) só pra quem realmente precisa entrar no painel gerencial — o resto da equipe usa só a senha da própria loja.

## Painel gerencial: o que tem hoje

O painel do administrador tem um botão (« no canto superior do menu) pra esconder o menu lateral e aproveitar a tela inteira — clique de novo (») pra trazer ele de volta. Essa preferência fica salva no aparelho.

O painel do administrador tem um menu lateral agrupado em 4 seções:

**📊 Visão geral**
- **Dashboard Executivo** — tudo numa aba só, recalculado a partir do período escolhido no filtro do topo (veja "Filtro de período" abaixo): resumo do período (faturamento, média diária e divergência total — os totais somam o período inteiro, mesmo que incluam uma loja já removida do cadastro), o ranking de faturamento em formato de tabela (posição, loja, faturamento, % de participação e uma barra de progresso com o percentual embutido), medalha e cor ouro/prata/bronze nos 3 primeiros, o card de líder do período (com a variação % vs. o período anterior), um gráfico de distribuição do faturamento (donut, com as 4 maiores lojas em destaque e o resto agrupado em "Outras lojas") e a evolução diária do faturamento — agora num gráfico de linha (antes era em barras), com os pontos de cada dia, quantidade de fechamentos e divergências no hover, e sinalizador nos dias com divergência. Tudo recalcula sozinho a cada fechamento salvo ou troca de período, sem recarregar a página. (O antigo card "Comparação de períodos", que comparava só um dia contra o dia anterior, foi removido — não fazia sentido fora de um período maior; a comparação com o período anterior continua existindo, só que dentro do card de líder e dos relatórios.)

**💰 Fechamento**
- **Lançamentos** — a lista de todos os fechamentos registrados, com o botão de exportar CSV.
- **Divergências** — agora é a **Central de auditoria financeira** (veja a seção própria abaixo), junto com o gerenciamento dos motivos de divergência cadastrados logo abaixo dela.

**📈 Relatórios**
- **Relatórios** — virou um relatório executivo completo (veja a seção própria "Relatório executivo" abaixo), com atalhos rápidos (Diário/Semanal/Mensal/Trimestral/Personalizado) que aplicam o mesmo filtro de período do topo da tela.
- **Exportação** — os dois arquivos CSV disponíveis hoje (lançamentos e só as divergências), num só lugar.

**⚙️ Administração**
- **Lojas** — adicionar ou remover lojas, e trocar a senha de cada uma (a mesma senha usada em "Fechar caixa de uma loja").
- **Usuários** — criar, editar senha e remover contas de Administrador/Consulta.

O formulário de fechamento agora também mostra em qual das 8 etapas você está ("Etapa 3 de 8 · Dinheiro", por exemplo) — mais fácil de saber quanto falta.

## Filtro de período

O antigo filtro por mês virou um filtro por intervalo de datas (Data Inicial → Data Final), com atalhos rápidos no topo da tela: Hoje, Ontem, Últimos 7/15/30 dias, Este mês, Mês anterior — ou datas personalizadas nos campos "De"/"Até". Trocar o período recalcula na hora o Dashboard, o Ranking, os Relatórios, os gráficos, as Divergências e os líderes, sem precisar recarregar a página — tudo é calculado em cima dos fechamentos que já estão sincronizados do Firestore, sem nenhuma consulta nova ao banco a cada clique.

## Central de auditoria financeira

A antiga tela de "Divergências" (só uma lista) virou um fluxo completo de identificação, análise e aprovação — sem criar nenhuma coleção nova no Firestore: tudo continua gravado dentro do mesmo documento de fechamento, agora só com um campo a mais chamado `divergencia`.

- **Status de cada divergência**: 🔴 **Aberta** (detectada, ainda sem análise), 🟡 **Justificada** (a loja já registrou motivo/observação ao fechar o caixa) ou 🟢 **Resolvida** (a administração já conferiu e aprovou). Uma divergência nova entra como Justificada automaticamente se a loja já marcou um motivo ou escreveu uma observação no fechamento — não existe nenhuma tela nova pra loja preencher, o sistema reaproveita os campos que já existiam no formulário.
- **KPIs no topo**: quantidade de divergências abertas, valor total em divergência, quantidade de lojas com divergência e quantidade já resolvida — tudo já filtrado pelo período escolhido no topo da tela.
- **Filtros rápidos**: Abertas / Justificadas / Resolvidas / Todas.
- **Aprovação individual**: cada card tem um botão "Resolver divergência" que abre uma confirmação com a loja, o valor e um campo de observação — ao aprovar, fica gravado quem aprovou, quando e o que foi observado, e o card passa a mostrar esse histórico permanentemente.
- **Aprovação em massa**: um botão no topo aprova de uma vez todas as divergências ainda pendentes que estão visíveis com o filtro atual (loja + período + status), numa confirmação só com a quantidade e o valor total.
- **Maiores divergências**: ranking das lojas com mais valor em divergência no período, com medalha nas 3 primeiras.
- **Divergências por loja**: gráfico de barras com quantidade e valor por unidade.
- **Fechamentos antigos** (salvos antes dessa central existir) foram migrados automaticamente na primeira vez que o app abriu depois da atualização — nenhuma divergência antiga ficou de fora do histórico.
- A exportação CSV de divergências (aba Exportação) agora também traz o status de auditoria, quem resolveu, quando e a observação.

Detalhe técnico, caso apareça alguma dúvida revisando o código: o Firestore não grava `serverTimestamp()` dentro de um item de array (que é como cada fechamento fica armazenado aqui), então a data/hora da resolução usa o horário do próprio aparelho no momento da aprovação — o mesmo formato que o app já usa em `criadoEm` desde sempre. E como esse sistema guarda tudo num único documento, a "transação" pedida pra aprovação em lote é a mesma transação do Firestore que toda gravação do app já usa — não precisou de nenhuma API nova.

## Sistema de notificações (badges de divergência pendente)

Os avisos de "alerta crítico" espalhados pelos cards das lojas saíram — uma divergência não é uma emergência, é só uma ocorrência que precisa ser revisada. No lugar entrou um badge de notificação único, no estilo Instagram/Teams/Outlook (fundo vermelho `#FF3B30`, texto branco, circular):

- **Menu lateral**: o item "Divergências" ganha um badge vermelho com a quantidade de divergências com status **aberta** — some sozinho quando não sobra nenhuma.
- **Cards das lojas**: em vez do texto "⚠ R$ X de diferença · 🔴 alerta crítico", agora é só um badge pequeno ao lado do nome da loja com a quantidade de divergências abertas dela (some se não houver nenhuma).
- **Topo da tela de Divergências**: um resumo "🔔 Divergências Pendentes" com o badge e o total em R$ das divergências abertas, antes da grade de KPIs e filtros que já existiam.

O contador conta só **aberta** — uma divergência **justificada** (a loja já registrou motivo) não gera notificação nova, e uma **resolvida** nunca aparece. Resolver uma divergência (individualmente ou em "Aprovar todas") atualiza os badges na hora, em todo canto — tudo calculado em memória a partir dos fechamentos já sincronizados via Firestore, sem nenhuma consulta nova ao banco.

## Proteção contra tentativas excessivas de login

Cada usuário agora acompanha suas próprias tentativas de senha errada, pra dificultar um ataque de tentativa e erro:

- **1ª e 2ª tentativas erradas**: só a mensagem de sempre ("Usuário ou senha incorretos").
- **3ª e 4ª**: a mensagem passa a avisar que a conta será bloqueada se continuar errando ("Atenção: 3 tentativas incorretas — após 5 sua conta será bloqueada temporariamente por 15 minutos").
- **5ª tentativa errada**: a conta é bloqueada — "Conta temporariamente bloqueada por excesso de tentativas." — e o login fica recusado mesmo que a senha certa seja digitada depois, até o bloqueio passar.
- **Depois de 15 minutos**, o bloqueio se desfaz sozinho, sem precisar de nenhum administrador — é checado automaticamente na próxima tentativa de login daquela pessoa.
- Um **administrador também pode desbloquear na hora**, pelo menu ⋮ da tela de Usuários ("🔓 Desbloquear usuário") — esse botão só aparece quando a conta está mesmo bloqueada.
- Uma senha certa zera o contador de tentativas.

Isso é um bloqueio de **segurança**, diferente do "Ativo/Bloqueado" que já existia (a decisão do administrador de desligar um acesso) — os dois aparecem em colunas separadas na tela de Usuários (**Status** e **Segurança**), e o KPI "🔒 Bloqueados por segurança" no topo mostra quantas contas estão travadas agora.

Cada tentativa de login (certa, errada, bloqueada, ou um desbloqueio automático/manual) vira uma linha na tabela **🔒 Logs de segurança**, embaixo da lista de usuários: quem, quando, o evento, quantas tentativas e o IP de quem tentou — quando disponível. Como esse é um site estático (sem servidor próprio), o IP não vem de uma requisição de backend; ele é buscado do navegador por um serviço público, em segundo plano, sem nunca atrasar ou travar o login caso a rede falhe ou demore.

## Relatório executivo

A antiga tabela simples de Relatórios virou um relatório completo, no estilo de um informe gerencial pra imprimir ou mandar por e-mail:

- **Capa do relatório**: período selecionado, data/hora de geração e responsável pela conferência (a pessoa logada).
- **4 indicadores no topo**: vendas no período, número de fechamentos, total de divergências em R$ e o status de hoje (quantas lojas ainda estão pendentes).
- **Resumo executivo**: um parágrafo gerado automaticamente com os números principais do período — loja líder, participação % dela no total, valor total em divergências e quantas lojas ainda não fecharam hoje.
- **Vendas x Divergências**: gráfico de barras pareadas comparando o faturamento com o valor de divergência de cada loja, lado a lado.
- **Vendas por loja** e **Ranking de divergências**: duas listas de barras horizontais, ordenadas do maior pro menor valor.
- **Participação das vendas** e **Status dos fechamentos**: dois gráficos de donut — um mostrando a fatia de cada loja no faturamento total, outro mostrando quantas lojas estão com o fechamento concluído, em análise ou pendente hoje.
- **Tabela detalhada**: todas as lojas lado a lado, agora com 7 colunas — vendas no período, participação %, número de fechamentos, divergências em R$, variação vs. o período anterior e o status de hoje.
- **Faturamento por mês/trimestre**: um gráfico de tendência dos últimos 12 meses (ou 8 trimestres, no botão "Trimestral") — clicar numa barra aplica aquele mês/trimestre como período do relatório inteiro, na hora.

## Admin único nunca fica sem acesso

Como esse sistema não tem "esqueci minha senha" (não existe e-mail nem SMS por trás — veja "Segurança das senhas" abaixo), um administrador que fosse bloqueado por excesso de tentativas erradas (seção acima) ficaria trancado pra sempre se não sobrasse nenhum outro administrador ativo pra desbloquear ele — foi exatamente isso que aconteceu no início desta atualização e motivou essa correção.

Agora, sempre que a conta que está tentando logar é a **única conta de administrador ativa** do sistema, ela nunca é bloqueada de verdade: continua vendo os mesmos avisos progressivos de tentativas erradas (e continua sendo registrada nos Logs de segurança normalmente), mas o bloqueio de 15 minutos nunca chega a ser aplicado pra ela — e se por acaso ela já estivesse bloqueada de uma versão anterior, o próprio login com a senha certa já desfaz o bloqueio na hora, sem precisar esperar. Assim que existir um segundo administrador ativo, essa exceção deixa de valer pra ambos — o sistema volta a proteger normalmente, já que aí um pode desbloquear o outro pela tela de Usuários.

## Segurança das senhas

As senhas ficam guardadas com hash (SHA-256) no Firestore, não em texto puro — mas isso ainda não é autenticação de verdade (não existe conta de e-mail/Google por trás, como teria com o Firebase Authentication). Continua valendo o princípio de sempre: não divulgue o link pra quem não deveria ter acesso.

## Firestore

Este arquivo já vem conectado ao projeto Firebase `fechacaixa-de698`. Confirme que as regras de segurança (`firestore.rules`, já publicadas no Console) continuam ativas: Firestore Database → Regras.

## Autenticação anônima (obrigatória a partir desta versão)

Antes desta atualização de segurança, o documento do Firestore ficava com a regra `allow read, write: if true` — qualquer pessoa com a `apiKey` do projeto (que aparece no próprio `index.html`, já que é pública por design do Firebase) conseguia ler ou escrever direto no banco, contornando totalmente o login e o bloqueio por tentativas do app. A partir de agora o app faz login anônimo no Firebase automaticamente, sem nenhuma tela ou clique a mais pro usuário, só pra provar ao Firestore que a requisição veio do app — e as novas regras (`firestore.rules`) passaram a exigir `request.auth != null`.

Isso só funciona se o provedor "Anônimo" estiver habilitado no projeto Firebase (já feito no `fechacaixa-de698`).

1. Abra o [Console do Firebase](https://console.firebase.google.com/) → escolha o projeto.
2. Vá em **Authentication** → aba **Sign-in method**.
3. Habilite o provedor **Anônimo** (Anonymous).
4. Em **Firestore Database** → **Regras**, cole o conteúdo do arquivo `firestore.rules` deste repositório e publique.

Sem esses dois passos, o app carrega normalmente mas fica preso na tela de carregamento (o login anônimo falha e a conexão com o Firestore nunca é liberada pelas novas regras).
