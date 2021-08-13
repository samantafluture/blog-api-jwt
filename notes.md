# Node.js e JWT: Autenticação com posts

## Protegendo as senhas no banco de dados

### Função de espalhamento ou função de hashing

-   Transforma a senha do usuário em um valor aleatório
-   Não é possível saber a senha
-   Proteção ao usuário

### Função de hashing mais adequada

-   Opções: SHA-256 (tem vulnerabilidades, modifica ela, adiciona `salt` uma string pseudo-aleatória de uso único) e MD5
-   função(senha, salt, custo); salt gerado automaticamente usando libs
-   função bcrypt.hash

### Implementar a proteção na aplicação

1. Modificações no Modelo do Usuário (`usuarios-modelo.js`)

-   adicionar um método de hash

```javascript
static gerarSenhaHash(senha) {
    const custoHash = 12;
    return bcrypt.hash(senha, custoHash);
}
```

-   criar novo método de adicionar a senha hash

```javascript
async adicionaSenha(senha) {
    this.senhaHash = await Usuario.gerarSenhaHash(senha);
}
```

-   trocar `senha` por `senhaHash` no construtor do usuário

`this.senhaHash = usuario.senhaHash;``

-   incluir validações de senha para dentro do método `adicionaSenha`

```javascript
validacoes.campoStringNaoNulo(senha, 'senha');
validacoes.campoTamanhoMinimo(senha, 'senha', 8);
validacoes.campoTamanhoMaximo(senha, 'senha', 64);
```

2. Modificações na DAO do Usuário (`usuarios-dao.js`)

-   trocar `senha` por `senhaHash`

3. Modificações no banco de dados (`database.js`)

-   na parte do schema dos usuários, trocar `senha` por `senhaHash`

4. Modificar o controlador (`usuarios-controlador.js`)

-   quando instanciamos um novo `Usuario` não passamos mais sua senha, então devemos retirar este campo do objeto `usuario`
-   devemos receber a senha usando o método `adicionaUsuario`

```javascript
const usuario = new Usuario({
    nome,
    email,
});

await usuario.adicionaSenha(senha);
```

5. Deletar a base de dados `db-sqlite` do `src`

-   pois ela ainda está com as configurações antigas
-   ao darmos um `npm start` o node cria uma nova base de dados zerada

6. Testar no Insomnia

-   ao criarmos um usuário via POST o retorno deve ter uma `senhaHash`, uma string aparentemente aleatória que não dá nenhuma informação sobre a senha original

```json
[
    {
        "id": 1,
        "nome": "Alissa",
        "email": "alissa@gmail.com",
        "senhaHash": "$2b$12$qoTXdKks0u8mvDeJGfSYH.YRsla7XRjOh1Mv71swtd8mfeWbzVGV6"
    }
]
```

## Criando um sistema de login escalável

-   incluir verificação para quem está criando e deletando tanto post quanto usuário
-   criar um sistema de login para isso

### Diferentes métodos de login

-   sessões

    -   momento de login, envia email e senha
    -   servidor cria na memória uma sessão
    -   devolve um id_sessao para o usuário
    -   nas próximas requisições que ele fizer, ele envia esse id junto
        -   exemplo: `criar post + id_session`
    -   dessa forma, o servidor vai saber qual usuário está se comunicando com ele
    -   e que ele foi autenticado via login anteriormente
    -   problemas:
        -   se muitos usuários fizerem login ao mesmo tempo, ele guarda muitos estados/sessões, o que sobrecarrega o servidor
        -   prejudica a escalabilidade da aplicação
        -   vai contra os princípios do rest ("usuários não devem depender de um estado guardado para fazer as requisições")

-   o que fazer?

    -   já que o estado não pode ser guardado no servidor, vamos guardar no cliente
    -   dessa forma o cliente faz um login e se estiver tudo correto, o servidor devolve uma info que identifica o usuário (exemplo: `id`)
    -   o usuário guarda o `id` e o envia junto quando fizer uma nova requisição
    -   problema:
        -   como saber se este usuário é realmente o usuário do `id`?
        -   e não algum ataque...
        -   se certificar de que apenas os usuários vão conseguir enviar os próprios `ids`
        -   uma "assinatura" do servidor no `id`, guardando o `id` assinado

-   como aplicar esse modelo de "assinatura" na nossa aplicação?
    -   JWT (Json Web Token)

### JWT

-   essa informação assinada chama-se `token`
-   como o `token` é gerado?

1. servidor pega a informação do usuário `id_usuario` e o codifica em `json`
2. essa sessão é chamada de `payload`
    - tradução literal: carga que gera lucro
    - são dados que realmente importam na mensagem, em comparação com cabeçalhos e assinaturas, que apenas existem para permitir a transmissão da mensagem
3. servidor cria uma sessão codificada em `json` chamada `cabeçado` ou `header`
4. o `header` possui:
    - o algoritmo utilizado (ex: HS256)
    - o tipo do token (ex: JWT)
5. resultado: a sessão de assinatura
    - resultado da função de hash
    - recebe como primeiro argumento o `cabeçalho` e o `payload` codificados na `base64Url`, concatenados separados por um `.`
    - recebe como segundo argumento uma `senha secreta` que é guardada apenas no servidor
6. pega essas 3 informações (`header`, `payload` e `assinatura`), codifica de novo em `base64Url`, e concatena separados por um `.`
7. resultado é apenas uma `string` com todas essas informações
8. assim temos o `token` que será enviado ao usuário

-   o servidor gera uma assinatura baseada nas informações que o usuário deu e a `senha secreta` guardada
-   assim faz uma comparação: a assinatura ideal = assinatura que o usuário enviou?
-   se sim, então tem certeza de que esse `token` é verdadeiro e o usuário é `váliado`

[jwt.io](https://jwt.io/)

### Configurações

Devemos configurar nosa primeira estratégia de autenticação, a estratégia local, que é a estratégia onde recebemos o email e a senha do nosso cliente.

Para facilitar a criação dos Middlewares de autenticação, usaremos dois módulos abaixo.

1. instalar `passport` e o `passport-local`

`npm i passport@0.4.1`
`npm i passport-local@1.0.0`

2. configurar a estratégia de autenticação

-   na pasta `usuarios/`, criar o arquivo `estrategias-autenticacao.js`

3. instanciar uma nova estratégia local e fazer algumas modificações, passando nomes de campos que usamos no lugar do padrão

```javascript
passport.use(
    new.LocalStrategy({
        usernameField: 'email',
        passwordField: 'senha',
        session: false
    })
)
```

4. criar a função de verificação

-   recebe 3 argumentos: email, senha e `done` (uma função callback do `passport-authenticate`)
-   objetivo: se dados estiverem válidos, a função devolve os dados para a função callback

5. importar o módulo de estratégia de autenticação no `index.js`

`estrategiasAutenticacao: requite('./estrategias-autenticacao.js')`

6. inicializar a estratégia de autenticação no `app.js`

`const { estrategiasAutenticacao } = require('./src/usuarios')`

7. criar um controlador que vai implementar a resposta depois de um login bem-sucedido

-   vá em `usuarios-controlador.js`
-   se o login for bem sucedido, devolve 204 (página vazia): deu tudo certo e não tem nada na página

```javascript
login: (req, res) => {
    res.status(204).send();
  },
```

8. fazer as rotas de login em `usuarios-rotas.js`

-   criar uma nova rota de login

```javascript
app.route('/usuario/login').post(usuariosControlador.login);
```

-   inserir um método do `passport` para autenticação
-   importar o `passport`

```javascript
app.route('/usuario/login').post(
    passport.authenticate('local', { session: false }),
    usuariosControlador.login
);
```

9. testar no Insomnia

-   criar uma nova requisição
-   chamar de "Efetua login"
-   tipo `POST` e via `form`
-   endereço: localhost:3000/usuario/login
-   preencher com email e senha cadastrados anteriomente
-   sucesso: `204`

## Implementando autenticação com JWT

-   ainda há duas rotas desprotegidas: `adicionar post` e `deletar usuário`
-   como garantir a autenticação do usuário sem ter que ficam enviando email/senha toda hora?
-   vai realizar por meio de `tokens`
-   em vez de usar email/senha, o cliente terá que enviar token
-   onde que ele vai conseguir este token?
-   no momento que se autentica com a estratégia local, ele recebe um token de volta
-   vai guardar e enviar pro servidor nas próximas requisições

### Gerando tokens

-   implementar a criação do token e o seu envio para o usuário
-   vamos usar o pacote `json web token`

`npm i jsonwebtoken@8.5.1`

-   implementar isso no `usuarios-controlador.js` fazendo uma função

```javascript
const jwt = require('jsonwebtoken');

function criaTokenJWT(usuario) {
    const payload = {
        id: usuario.id,
    };

    const token = jwt.sign(payload, 'senha-secreta');
    return token;
}
```

-   modifica a função `login` para usar o `jwt`, criando o `token`
-   em seguida, envia este `token` ao usuário no `header`, mais especificamente, no `authorization`
-   o status http `204` não só emite uma resposta ok em branco como indica que os `headers` podem ser úteis

```javascript
login: (req, res) => {
    const token = criaTokenJWT(req.user);
    res.set("Authorization", token);
    res.status(204).send();
  },
```

-   testar fazer o login no Insomnia
-   mandar o email e senha corretos
-   checar o status e a assinatura no `header authorization`

### Senha segura para JWT

-   rodar um programa do node para gerar uma senha secreta segura

`node -e "console.log( require('crypto').randomBytes(256).toString('base64'))"`

-   guardar essa senha gerada (uma string) dentro de uma variável de ambiente
-   dessa forma, você não publica em controle de versão
-   e também fica acessível em todos os pontos do seu programa
-   mais fácil de manter

-   criar um arquivo `.env` na raiz do projeto
-   colocar essa senha lá como `CHAVE_JWT="senha gerada"`
-   baixar um pacote para tornar essa senha legível pela aplicação node

`npm i dotenv@8.2.0`

-   configurar o módulo no arquivo `server.js`
-   inserir o seguinte na primeira linha de código

`require('dotenv').config();`

-   esta linha vai configurar todas as variáveis de ambiente da aplicação
-   em seguida, no `usuarios-controlador.js`
-   inserir a variável de ambiente que queremos no lugar de "senha-secreta", usando o método `process` do `env`

`const token = jwt.sign(payload, process.env.CHAVE_JWT);`

### Estratégia para JWT

-   estratégia **Bearer** significa token de autenticação
-   instalar o módulo

`npm i passport-http-bearer@1.0.1`

-   editar o arquivo `estrategias-autenticacao.js`
-   configurar a função de verificação para estratégia bearer token

```javascript
passport.use(
    new BearerStrategy(async (token, done) => {
        try {
            const payload = jwt.verify(token, process.env.CHAVE_JWT);
            const usuario = await Usuario.buscaPorId(payload.id);
            done(null, usuario);
        } catch (erro) {
            done(erro);
        }
    })
);
```

-   implementar a estratégia nas rotas
-   modificar em `posts-rotas.js`, adicionando o middlaware do `passport` na rota post de criar posts (não esquecer de importat o passport acima)

```javascript
.post(
      passport.authenticate("bearer", { session: false }),
      postsControlador.adiciona
    );
```

-   modificar o `usuarios-rotas.js`, adicionando o middleware na rota de deletar usuário

```javascript
    .delete(
      passport.authenticate("bearer", { session: false }),
      usuariosControlador.deleta
    );
```

-   testar no Insomnia:
    -   sem autenticação = deve barrar
        -   criar post sem autenticação deve gerar status `401`
    -   fazer login, com autenticação = deve funcionar
        -   copiar o token do header ao fazer login
        -   ir na requisição de criar post
        -   ir no header desta requisição
        -   colocar o tipo "Authorization"
        -   colocar o conteúdo "Bearer (token gerado)"
        -   ou ir na aba `Auth` do tipo `Bearer` e colar o token lá
        -   testar a requisição, que deve ser `201`

### Tratamento de erros do login

-   tratar o comportamento da resposta quando algo dá errado no login
-   criar um novo arquivo `middlewares-autenticacao.js` e colocar lá todo o processo de tratamento de erro (if...)
-   depois voltar ao arquivo `usuarios-rotas.js`: importar este middleware e usá-lo no lugar do `passport` na rota de login

```javascript
app.route('/usuario/login').post(
    middlewaresAutenticacao.local,
    usuariosControlador.login
);
```

### Tratamento de erros do token

-   vamos fazer a mesma coisa que fizemos para a estratégia local para a estratégia bearer no `middlewares-autenticacao.js`
-   depois, adicionar este arquivo no `index.js`

`middlewaresAutenticacao: require("./middlewares-autenticacao")`

-   ir no `posts-rotas.js` e importar lá também além de usar na rota post

```javascript
const { middlewaresAutenticacao } = require('../usuarios').post(
    middlewaresAutenticacao.bearer,
    postsControlador.adiciona
);
```

-   voltar no `usuarios-rota.js` e adicionar o middlewares na rota de deletar

```javascript
app.route('/usuario/:id').delete(
    middlewaresAutenticacao.bearer,
    usuariosControlador.deleta
);
```

## Fazendo logout com tokens

-   incluir como terceiro parâmetro o tempo de expiração do token, no arquivo controller

`const token = jwt.sign(payload, process.env.CHAVE_JWT, { expiresIn: "15m" });`

-   criar um novo `if` no arquivo de middlewares

```javascript
if (erro && erro.name === 'TokenExpiredError') {
    return res
        .status(401)
        .json({ erro: erro.message, expiradoEm: erro.expiredAt });
}
```

-   sistema de logout: quando clicar em `logout`, o token é invalidado
-   para fazer isso, vamos ter que guardar uma info sobre o token em nosso sistema
-   faremos uma lista de tokens inválidos por logout
-   quando experirarem, podemos removê-los da lista
-   essa lista chamaremos de `blacklist`
-   uma opção é usar `redis`
-   colocar o token como chave de busca para a base
-   token como chave e nada como valor

### Usando o Redis

[Instalar Redis no macOs](https://phoenixnap.com/kb/install-redis-on-mac)

-   criar pasta /redis/ na raiz do projeto
-   criar arquivo blacklist.js dentro da pasta
-   criar o cliente com um prefixo na base (boa prática) de acordo com o objeto que ele estará tratando

`module.exports = redis.createClient({ prefix: "blacklist:" });`

-   feito isso, podemos instanciar este cliente redis no nosso server.js

`require("./redis/blacklist");`

-   depois, abrir o terminar e rodar `redis-server`
-   aí sim rodar o app `npm start`
-   agora vamos criar funções para manipular a `blacklist`
-   criar o arquivo manipula-blacklist.js dentro da pasta /redis/
-   serão duas funções que farão interação com a blacklist (adicionar e saber se tem token)

### Implementando logout

-   recuperar o token
-   adicionar o token como terceiro parâmetro no arquivo estrategias-autencicacao.js, dentro de passport.use()
-   no arquivo de middlewares, criar uma requisição para o token, para recebê-lo na estratégia bearer (`req.token = info.token;`)
-   no usuario-controlador, criar a função de `logout` que recupera o token da requisição

```javascript
logout: async (req, res) => {
    try {
        const token = req.token;
        await blacklist.adiciona(token);
        res.status(204).send();
    } catch (erro) {
        res.status(500).json({ erro: erro.message });
    }
},
```

- ir no arquivo de rotas do usuário e criar a rota de logout

```javascript
app.route('/usuario/logout').get(
        middlewaresAutenticacao.bearer,
        usuariosControlador.logout
    );
```

- testar a rota no Insomnia (fazer login primeiro, copiar o token e usar o token na estratégia de bearer em autenticação para fazer logout)
- dá certo, porém se eu tento usar o token de novo em outra requisição, ele deixa pois ainda não tem a confirmação de que o token está na blacklist, ou seja, invalidado
- precisamos implementar esta verificação

### Verificando se o token está na blacklist

- em estrategias-autenticacao.js, criar uma função para verificar o token na blacklist

```javascript
async function verificaTokenNaBlacklist(token) {
    const tokenNaBlackList = await blacklist.contemToken(token);
    if (tokenNaBlackList) {
        throw new jwt.JsonWebTokenError('Token inválido por logout!');
    }
}
```

- chamar essa função de verificação dentro da função `passport.use` de estratégia bearer

`await verificaTokenNaBlacklist(token);`

- assim, ele verifica se o token está na blacklist ou não antes de fazer todo o processo

- testar!

