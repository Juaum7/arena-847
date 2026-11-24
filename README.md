# Arena 847 — Configuração do banco de dados online (Supabase)

O Arena 847 agora guarda usuários, aulas e inscrições em um banco de dados na internet (Supabase). Todos os aparelhos (computador, celular, tablet) passam a ver as mesmas informações, em tempo real.

O visual, as páginas e a navegação continuam os mesmos. Mudou só a forma de guardar os dados e de fazer login.

Siga os passos **na ordem**. Leva uns 20 minutos.

---

## Resumo do que foi alterado

| Arquivo | O que mudou |
|---|---|
| `supabase/banco.sql` | **Novo.** Cria as tabelas, as regras, as funções e as permissões no Supabase. |
| `js/config.js` | **Novo.** Onde você cola a URL e a chave pública do seu projeto. |
| `js/dados.js` | Saiu o `localStorage`. As mesmas funções agora consultam e gravam no Supabase. |
| `js/comum.js` | A sessão de login agora é a do Supabase Auth. Inclui a atualização automática entre aparelhos. |
| `js/login.js` | Login pelo Supabase Auth. **Saiu** o formulário público "Crie a conta do administrador" (pedido de segurança). |
| `js/aluno.js`, `js/administrador.js` | Ajustados para operações assíncronas (`await`). |
| `login.html`, `aluno.html`, `administrador.html` | Carregam a biblioteca do Supabase e o `config.js`. O cartão de primeiro acesso saiu do login. |
| `aluno.html` | Botão **Cancelar inscrição** em "Minhas aulas" (e a janela de confirmação que já existia no painel). |
| `administrador.html` | Na lista de alunos: botão **Desativar/Reativar** e selo "Desativada". O botão **Excluir** continua. |

Nenhum arquivo CSS foi alterado.

---

## 1. Criar o projeto no Supabase

1. Acesse <https://supabase.com>, crie uma conta e clique em **New project**.
2. Escolha um nome (ex.: `arena-847`), uma senha para o banco (guarde-a, mas ela **não** vai no site) e a região **South America (São Paulo)**.
3. Aguarde o projeto ficar pronto.

## 2. Criar as tabelas e as regras (arquivo `supabase/banco.sql`)

1. No painel do projeto, abra **SQL Editor** → **New query**.
2. Abra o arquivo `supabase/banco.sql` num editor de texto, copie **todo** o conteúdo e cole no SQL Editor.
3. Clique em **Run**. Deve aparecer "Success. No rows returned".

Se rodar de novo, não tem problema: o arquivo pode ser executado mais de uma vez.

## 3. Ajustar o login (Authentication)

O sistema continua com login por **nome de usuário e senha, sem e-mail do aluno**. Como o Supabase Auth identifica as contas por e-mail, o site monta internamente um endereço `nomedeusuario@usuarios.arena847.app`. Esse endereço só existe dentro do sistema: nenhum e-mail é enviado e o aluno nunca o vê.

Para isso funcionar, no painel do Supabase vá em **Authentication → Sign In / Providers** (em algumas versões: **Authentication → Providers → Email**) e confira:

| Opção | Valor | Por quê |
|---|---|---|
| **Enable Email provider** | Ligado | É o login por usuário e senha. |
| **Confirm email** | **Desligado** | Os endereços internos não recebem e-mail. Se ficar ligado, os alunos não conseguem entrar. |
| **Allow new users to sign up** | **Ligado** | Necessário para o administrador criar alunos pelo painel. Está protegido: o banco recusa qualquer cadastro que não venha de um convite gerado pelo administrador logado. |

Salve.

> **Senha mínima:** o site exige 6 caracteres, o mesmo padrão do Supabase. Se você aumentar a exigência em **Authentication → Policies/Passwords**, o site mostra a mensagem do Supabase quando a senha não for aceita.

## 4. Conectar o site ao projeto (`js/config.js`)

1. No Supabase, abra **Project Settings → Data API** (ou **API**) e copie a **Project URL** (ex.: `https://abcdefghijkl.supabase.co`).
2. Em **Project Settings → API Keys**, copie a chave **Publishable** (`sb_publishable_...`). Em projetos mais antigos ela aparece como **anon / public** (começa com `eyJ...`). Qualquer uma das duas serve.
3. Abra `js/config.js` e substitua:

```js
url: 'COLE_AQUI_A_URL_DO_PROJETO',      // → 'https://abcdefghijkl.supabase.co'
chavePublica: 'COLE_AQUI_A_CHAVE_PUBLICA', // → 'sb_publishable_...'
```

> ⚠️ **Nunca** coloque a chave **secret** / **service_role** no `config.js` nem em nenhum arquivo do site. Ela dá acesso total ao banco, e qualquer visitante consegue ler os arquivos JavaScript. O site não precisa dela: todas as operações do administrador são liberadas pelas regras do banco (RLS) para quem está logado como administrador.

**`dominioLogin`:** deixe `usuarios.arena847.app`. Só mude se o Supabase recusar o endereço (veja "Problemas comuns"), e **sempre antes** de criar as contas: quem já tem conta continua usando o domínio com que foi criado.

## 5. Criar o primeiro administrador

Por segurança, **não existe** nenhuma tela pública para criar administrador. O primeiro é criado por você, pelo painel do Supabase:

1. **Authentication → Users → Add user → Create new user**.
   - **Email:** `admin@usuarios.arena847.app`. A parte antes do `@` é o nome de usuário que você vai digitar no login, em letras minúsculas. Depois do `@` vai o mesmo valor de `dominioLogin`.
   - **Password:** a senha do administrador (mínimo de 6 caracteres).
   - Marque **Auto Confirm User**.
   - Clique em **Create user**.
2. **SQL Editor → New query**, cole e rode, trocando o e-mail e o nome se você usou outros:

```sql
select public.criar_perfil_admin('admin@usuarios.arena847.app', 'admin');
```

3. Abra o site e entre com o usuário **admin** e a senha escolhida.

Enquanto o primeiro administrador não existir, a página de login avisa isso.

Depois que o primeiro administrador existe, o banco **recusa** qualquer conta criada sem convite, inclusive pelo botão "Add user" do painel.

**Outro administrador no futuro:** cadastre a pessoa como aluno pelo painel do site e depois rode no SQL Editor:

```sql
select public.criar_perfil_admin('fulano@usuarios.arena847.app', 'fulano');
```

## 6. Criar as contas dos alunos

Igual a antes: no painel do administrador, abra **Alunos**, preencha **Nome de usuário** e **Senha** e clique em **Cadastrar aluno**. Repasse o usuário e a senha ao aluno.

Por trás, o banco gera um convite de uso único (válido por 10 minutos) que só o administrador logado consegue gerar. A conta é criada no Supabase Auth com esse convite, e o banco cria o perfil do aluno. A senha fica só no Supabase Auth, com hash: não é guardada em nenhuma tabela.

> O Supabase limita o número de cadastros por minuto vindos do mesmo endereço de internet (por padrão, cerca de 30 a cada 5 minutos). Se aparecer o aviso de limite, aguarde alguns minutos. Dá para ajustar em **Authentication → Rate Limits**.

Na lista de alunos:

- **Desativar** bloqueia o aluno na hora: ele não consegue entrar, e quem já estiver logado é mandado para o login na próxima atualização da tela (no máximo 30 segundos). As inscrições dele são mantidas. **Reativar** devolve o acesso.
- **Excluir** apaga a conta, o login e as inscrições.

## 7. Publicar o site

Para que os outros aparelhos acessem, o site precisa estar na internet. Qualquer hospedagem de site estático serve, porque são só arquivos HTML, CSS e JS. Por exemplo:

- **Netlify Drop** (<https://app.netlify.com/drop>): arraste a pasta do projeto (a que tem `index.html`). Pronto.
- **GitHub Pages**, **Vercel**, **Cloudflare Pages** ou a hospedagem que você já usa.

Lembre de publicar com o `js/config.js` já preenchido. Para testar só no seu computador, rode `npx serve .` ou `python3 -m http.server` na pasta do projeto e abra `http://localhost:...`.

---

## 8. Como testar em dois ou mais aparelhos

Use o endereço publicado. Para simular vários usuários no mesmo computador, use janelas anônimas ou navegadores diferentes: cada um tem a sua sessão.

1. **Computador:** entre como `admin`. Em **Alunos**, cadastre `aluno1` e `aluno2`. Em **Aulas → Nova aula**, crie uma aula de **Futevôlei** para amanhã, com **2** vagas.
2. **Celular A:** entre como `aluno1`. A aula aparece com "2 vagas disponíveis". Clique em **Inscrever-se**.
3. **Celular B:** entre como `aluno2`. A aula aparece com "1 vaga disponível". Inscreva-se. A aula fica **lotada**.
4. **Computador:** sem recarregar, o resumo do painel passa a mostrar 2 vagas ocupadas (tempo real).
5. **Terceiro aparelho/janela** com outro aluno: a aula aparece como **Sem vagas**.
6. **Duplicidade:** no celular A, a aula aparece como "Inscrito". O botão some, e o banco recusa uma segunda inscrição mesmo que alguém tente forçar.
7. **Cancelamento:** no celular A, abra **Minhas aulas → Cancelar inscrição**. A vaga reaparece nos outros aparelhos.
8. **Participantes:** no computador, **Gerenciar aula** → adicione ou remova alunos.
9. **Editar, cancelar, reativar e excluir** a aula pelo **Gerenciar aula**. Aula cancelada some das "Aulas disponíveis" e o banco recusa inscrição nela.
10. **Sessão:** recarregue a página do aluno. Ele continua logado. Feche e abra o navegador: continua logado até clicar em **Sair**.
11. **Desativação:** no computador, **Desativar** o `aluno2`. Em até 30 segundos (ou ao voltar para a aba), o celular B volta para o login. Ao tentar entrar, aparece "Esta conta está desativada".

**Vaga disputada no mesmo instante:** a inscrição é feita por uma função do banco que trava a aula enquanto confere e grava a vaga. Se duas pessoas clicarem na última vaga ao mesmo tempo, uma é aceita e a outra recebe "Não há mais vagas disponíveis nesta aula".

---

## Segurança (resumo do que o `banco.sql` garante)

- **Row Level Security** ligado em todas as tabelas. Visitantes não logados não leem nada.
- **Aluno:** vê as aulas e só as **próprias** inscrições. Cancela só as próprias, e só de aulas que ainda não aconteceram. Não consegue inserir inscrição direto na tabela: só pela função `inscrever`, que aplica todas as regras. Nas aulas, vê só a **quantidade** de inscritos, nunca quem são os outros.
- **Aluno não vira administrador:** a tabela `perfis` não aceita escrita pelo site. Perfis só são criados pelo banco (com convite) ou pelo SQL Editor.
- **Somente o administrador** cria, edita e exclui aulas, gerencia participantes, cria convites e desativa ou exclui contas.
- **Conta desativada:** todas as regras exigem conta ativa, então ela perde o acesso a tudo imediatamente, mesmo com a sessão ainda aberta.
- **Limite de vagas:** conferido no banco, com trava da aula (`SELECT ... FOR UPDATE`) na função `inscrever`, e de novo num gatilho em `inscricoes`. O limite também não pode ser reduzido abaixo do número de inscritos.
- **Senhas:** só no Supabase Auth, com hash. Nenhuma chave privilegiada no navegador.
- **Horário:** "a aula já aconteceu" é decidido pelo banco no fuso **America/Sao_Paulo**. Para outro fuso, altere a função `public.agora_local` em `banco.sql` e rode o arquivo de novo.

---

## Problemas comuns

| Mensagem / sintoma | Solução |
|---|---|
| "O sistema ainda não está conectado ao banco de dados" | Preencha `url` e `chavePublica` em `js/config.js` (passo 4). |
| "O banco de dados não está configurado" | Rode `supabase/banco.sql` no SQL Editor (passo 2). |
| "O sistema ainda não tem um administrador" | Faça o passo 5. |
| "A conta foi criada, mas o Supabase está exigindo confirmação de e-mail" | Desligue **Confirm email** (passo 3). Para a conta já criada: **Authentication → Users**, abra o usuário e confirme-o, ou exclua e cadastre de novo. |
| "O cadastro de contas está desligado no Supabase" | Ligue **Allow new users to sign up** (passo 3). |
| "O Supabase recusou o endereço interno de login" | Troque `dominioLogin` em `js/config.js` por um domínio seu, por exemplo o domínio do site: `usuarios.seusite.com.br`. Faça isso antes de criar as contas. Se já existirem contas, recrie-as com o novo domínio. |
| "Esta conta ainda não tem acesso ao sistema" | A conta existe no Auth mas não tem perfil. Para administrador, rode o `criar_perfil_admin` do passo 5. |
| "Usuário ou senha incorretos" para o admin recém-criado | Confira se o e-mail criado no painel é exatamente `nome@` + `dominioLogin`, em minúsculas, e se marcou **Auto Confirm User**. |
| A tela não atualiza sozinha | A atualização em tempo real depende de `aulas` e `inscricoes` estarem em **Database → Publications → supabase_realtime**. O `banco.sql` faz isso automaticamente. De qualquer forma, as telas também se atualizam ao voltar para a aba e a cada 30 segundos. |

---

## O que foi testado e o que depende do seu projeto

**Testado** num PostgreSQL 16 local que reproduz o ambiente do Supabase (schema `auth`, papéis `anon`/`authenticated`, `auth.uid()`), executando o `banco.sql` real:

- todas as permissões e regras de RLS;
- convites e bloqueio de cadastro público;
- limite de vagas e duplicidade;
- aula cancelada e aula passada;
- desativação e exclusão de contas;
- 5 rodadas com 10 alunos disputando 1 vaga ao mesmo tempo: sempre exatamente 1 inscrição aceita.

As páginas foram testadas no Chromium, simulando computador, dois celulares e um tablet ao mesmo tempo, com um servidor local que imita a API do Supabase e executa as consultas nesse mesmo banco. Passaram todos os fluxos da lista acima, inclusive login, logout, persistência da sessão e atualização automática, sem erros de JavaScript.

**Depende da sua configuração** e por isso não pôde ser executado aqui:

- a conexão com o seu projeto Supabase real (URL e chave);
- as opções de Authentication do passo 3;
- a criação do primeiro administrador pelo painel;
- o tempo real do Supabase (no teste local ele foi simulado).

Depois de configurar, faça o roteiro da seção 8 para confirmar no seu projeto.
