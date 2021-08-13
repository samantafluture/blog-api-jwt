# Blog do Código
> Uma API de blog simples em Node.js

- Node.js
- Express
- SQlite
- JWT
- Redis
- Insomnia

## Como criar autenticação com JWT em Node.js

1. Criar um sistema de login escalável
    - [x] incluir verificação para requisições do tipo `POST`
    - [x] criar um sistema de login 
    - [x] adicionar função de hashing para proteger senha
    - [x] usar os módulos npm `passport` e `passport-local`
    - [x] criar estratégia de verificação local

2. Implementar autenticação com [JWT](https://jwt.io/)
    - [x] usar token para validar usuário 
    - [x] gerar tokens para autenticação
    - [x] modificar função de login adicionando o token
    - [x] usar estratégia **Bearer** via módulo `passport-http-bearer`
    - [x] fazer tratamento de erros do login e do token

3. Implementar logout e expiração do token
    - [x] fazer uma lista de tokens inválidos por logout
    - [x] usar redis para guardar chave valor dos tokens
    - [x] recuperar o token por requisição
    - [x] criar função e rota de logout
    - [x] criar verificação se o token está na blacklist


## Como usar

Rode:

`npm install`
`npm start` 

Isso irá colocar a API no ar no `localhost:3000` e criará um banco de dados zerado (vide arquivo `.dbsqlite`).

Não esqueça de deixar o Redis rodando em sua máquina também (`$ redis-server`).

### Pré-requisitos

- Node.js
- NPM
- Redis 
- Insomnia ou Postman