# WordPress no CapRover

[![GitHub stars](https://img.shields.io/github/stars/srssaulo/simple-wp-caprover?style=social)](https://github.com/srssaulo/simple-wp-caprover/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/srssaulo/simple-wp-caprover?style=social)](https://github.com/srssaulo/simple-wp-caprover/fork)
[![GitHub followers](https://img.shields.io/github/followers/srssaulo?style=social)](https://github.com/srssaulo)

Projeto mínimo para publicar um WordPress no CapRover a partir de um
repositório Git, usando a [imagem oficial do Docker Hub](https://hub.docker.com/_/wordpress).

O banco MySQL já existe no CapRover; este projeto não cria banco.

> 🍴 **Para usar no seu site, comece fazendo um
> [fork](https://github.com/SEU-USUARIO/SEU-REPO/fork) deste repositório.**
> O fork será o repositório do seu site, de onde o CapRover fará o deploy.

## Arquivos

| Arquivo                  | Para que serve                                      |
|--------------------------|-----------------------------------------------------|
| `captain-definition`     | Diz ao CapRover para usar o `Dockerfile`            |
| `Dockerfile`             | Define a versão do WordPress                        |

## Quem faz o quê

| Etapa | Responsável |
|-------|-------------|
| A. Gerar o token de acesso e enviar os dados à infra | **Dono do repositório** |
| B. Criar e configurar o app no CapRover (passos 1 a 5) | **Equipe de infra** |
| C. Cadastrar o webhook no GitHub | **Dono do repositório** |
| Uso no dia a dia e renovação do token | **Dono do repositório** |

Fluxo resumido:

```
Dono do repositório                       Equipe de infra
──────────────────                        ───────────────
A. gera token (somente leitura)
   envia usuário + token + URL + branch ─►  B. cria e configura o app
                                               gera o webhook
C. cadastra webhook no GitHub          ◄─  envia URL do webhook
   faz o primeiro push
```

---

## A. Dono do repositório: gerar o token e enviar os dados à infra

O CapRover precisa de permissão para baixar o código do repositório. Para
isso usamos um **token de acesso pessoal do tipo fine-grained**, com:

- acesso **somente a este repositório**;
- permissão **somente de leitura** do código;
- **data de expiração**.

Nenhuma senha ou chave SSH sua é compartilhada. O token pode ser revogado a
qualquer momento sem afetar mais nada.

### A.1 Gerar o token no GitHub

1. Acesse <https://github.com/settings/personal-access-tokens>
   (ou: foto do perfil → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens**).
2. Clique em **Generate new token**.
3. Preencha:
   - **Token name:** `caprover-meu-site`
   - **Expiration:** escolha um prazo (ex.: 1 ano). Anote a data, pois o
     token precisará ser renovado (ver "Renovar o token").
   - **Resource owner:** o dono do repositório (seu usuário ou a organização).
   - **Repository access:** **Only select repositories** → selecione
     **apenas** o repositório do site.
   - **Permissions → Repository permissions → Contents:** `Read-only`.
     (*Metadata: Read-only* é marcado automaticamente.) Não marque mais nada.
4. Clique em **Generate token** e **copie o token** (começa com
   `github_pat_...`). O GitHub mostra o token uma única vez.

> Se o repositório pertence a uma **organização**, um administrador da
> organização pode precisar **aprovar** o token antes de ele funcionar.

> Evite o token *classic* (<https://github.com/settings/tokens>): ele não
> pode ser restrito a um único repositório e dá acesso a todos os seus repos.

### A.2 Enviar os dados para a infra

Abra um chamado para a equipe de infra informando:

- **URL do repositório** (ex.: `github.com/minha-org/meu-site`);
- **Branch de deploy** (ex.: `main`). Todo push nesse branch vai para produção;
- **Domínio** desejado para o site;
- **Usuário do GitHub** que gerou o token;
- **Token** gerado no passo A.1.

> 🔒 O token é um segredo: envie **apenas pelo canal indicado pela infra**
> (chamado restrito, cofre de senhas etc.), nunca por e-mail aberto ou chat.

Aguarde a infra devolver a **URL do webhook** (etapa B).

---

## B. Equipe de infra: configuração no CapRover

> Esta etapa é de responsabilidade **exclusiva da equipe de infra**.
> O dono do repositório não precisa acessar o painel do CapRover.

### 1. Criar o app

**Apps → Create New App** → informe o nome e **marque "Has Persistent Data"**.

### 2. Variáveis de ambiente

**App Configs → Environmental Variables → Bulk Edit**:

```
WORDPRESS_DB_HOST=<srv-mysql fornecido pela infra>
WORDPRESS_DB_USER=<usuário gerado pela infra>
WORDPRESS_DB_PASSWORD=<senha gerada pela infra>
WORDPRESS_DB_NAME=<banco informado no chamado>
WORDPRESS_TABLE_PREFIX=wp_
```

As credenciais do banco ficam só no CapRover, nunca no repositório.

### 3. Volume

**App Configs → Persistent Directories**:

| Path in App                | Label                      |
|----------------------------|----------------------------|
| `/var/www/html/wp-content` | `<nome-do-app>-wp-content` |

Guarda imagens enviadas, temas e plugins instalados pelo painel. Sem ele,
tudo some a cada deploy. Clique em **Save & Update**.

### 4. Domínio e HTTPS

**HTTP Settings** → conecte o domínio → **Enable HTTPS** → marque **Force HTTPS**.

### 5. Ligar ao repositório e gerar o webhook

**Deployment → Method 3: Deploy from Github/Bitbucket/Gitlab**:

- **Repository:** URL recebida (ex.: `github.com/minha-org/meu-site`)
- **Branch:** branch informado no chamado
- **Username:** usuário do GitHub informado no chamado
- **Password:** o token recebido (`github_pat_...`)
- **SSH Key:** deixe em branco

Clique em **Save & Update**. O CapRover exibe a **URL do webhook**: envie-a
ao dono do repositório pelo chamado.

---

## C. Dono do repositório: cadastrar o webhook no GitHub

O webhook avisa o CapRover a cada push, e o CapRover faz o deploy sozinho.

1. No repositório: **Settings** → **Webhooks** → **Add webhook**.
2. **Payload URL:** cole a URL recebida da infra.
3. **Content type:** `application/json`.
4. **Secret:** deixe em branco.
5. **Which events…:** *Just the push event*.
6. Marque **Active** e clique em **Add webhook**.

Para conferir: faça um push no branch de deploy. Em **Settings → Webhooks**,
clique no webhook → aba **Recent Deliveries**: deve aparecer um envio com ✔
(resposta 200). Após alguns minutos, acesse o domínio e conclua a instalação
do WordPress (nome do site, usuário administrador etc.).

---

## Uso no dia a dia (dono do repositório)

- **Temas e plugins:** instale e atualize pelo painel do WordPress.
- **Atualizar o WordPress:** troque a tag no `Dockerfile`
  (ex.: `wordpress:7.1-apache` → `wordpress:7.2-apache`) e faça push no
  branch de deploy. Peça à infra um backup do banco antes.

### Renovar o token

Quando o token expira, os deploys param de funcionar (o site continua no ar).
Antes da data de expiração:

1. Em <https://github.com/settings/personal-access-tokens>, abra o token e
   clique em **Regenerate token** (ou gere um novo com as mesmas opções do A.1).
2. Envie o novo token à infra pelo chamado, para atualizar o campo
   **Password** no passo 5.

Se suspeitar que o token vazou, **revogue-o** na mesma página e gere outro.

## Problemas comuns

| Sintoma | Provável causa | Quem resolve |
|---------|----------------|--------------|
| Push não gera deploy | Webhook não cadastrado, URL errada ou push em outro branch | Dono (confira *Recent Deliveries*) |
| Erro de build "Authentication failed" ou "Repository not found" | Token expirado, sem acesso ao repositório, sem permissão *Contents*, ou pendente de aprovação da organização | Dono (gera novo token) + infra (atualiza) |
| "Error establishing a database connection" | Variáveis do banco erradas ou banco/usuário não criados | Infra |
| Imagens ou plugins sumiram depois de um deploy | Volume não configurado ou caminho errado | Infra |

---

## ⭐ Gostou?

Se este projeto te ajudou:

- deixe uma ⭐ **star** no repositório: <https://github.com/srssaulo/simple-wp-caprover>
- faça um 🍴 **fork** e use como base para o seu site: <https://github.com/srssaulo/simple-wp-caprover/fork>
- **siga** no GitHub para acompanhar novos projetos: <https://github.com/srssaulo>

Obrigado! 🙌
