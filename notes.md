# Node.js e JWT: Autenticação com posts

## Protegendo as senhas no banco de dados

### Função de espalhamento ou função de hashing

- Transforma a senha do usuário em um valor aleatório
- Não é possível saber a senha 
- Proteção ao usuário

### Função de hashing mais adequada

- Opções: SHA-256 (tem vulnerabilidades, modifica ela, adiciona `salt` uma string pseudo-aleatória de uso único) e MD5
- função(senha, salt, custo); salt gerado automaticamente usando libs
- função bcrypt.hash

### Implementar a proteção na aplicação

1. Modificações no Modelo do Usuário (`usuarios-modelo.js`)

- adicionar um método de hash

```javascript
static gerarSenhaHash(senha) {
    const custoHash = 12;
    return bcrypt.hash(senha, custoHash);
}
```

- criar novo método de adicionar a senha hash

```javascript
async adicionaSenha(senha) {
    this.senhaHash = await Usuario.gerarSenhaHash(senha);
}
````

- trocar `senha` por `senhaHash` no construtor do usuário

`this.senhaHash = usuario.senhaHash;``

- incluir validações de senha para dentro do método `adicionaSenha`

```javascript
    validacoes.campoStringNaoNulo(senha, "senha");
    validacoes.campoTamanhoMinimo(senha, "senha", 8);
    validacoes.campoTamanhoMaximo(senha, "senha", 64);
```

2. Modificações na DAO do Usuário (`usuarios-dao.js`)

- trocar `senha` por `senhaHash`

3. Modificações no banco de dados (`database.js`)

- na parte do schema dos usuários, trocar `senha` por `senhaHash`

4. Modificar o controlador (`usuarios-controlador.js`)

- quando instanciamos um novo `Usuario` não passamos mais sua senha, então devemos retirar este campo do objeto `usuario`
- devemos receber a senha usando o método `adicionaUsuario`

```javascript
const usuario = new Usuario({
        nome,
        email
      });

await usuario.adicionaSenha(senha);
```

5. Deletar a base de dados `db-sqlite` do `src`

- pois ela ainda está com as configurações antigas
- ao darmos um `npm start` o node cria uma nova base de dados zerada

6. Testar no Insomnia

- ao criarmos um usuário via POST o retorno deve ter uma `senhaHash`, uma string aparentemente aleatória que não dá nenhuma informação sobre a senha original

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

- incluir verificação para quem está criando e deletando tanto post quanto usuário
- criar um sistema de login para isso

### Diferentes métodos de login

- sessões
    - momento de login, envia email e senha
    - servidor cria na memória uma sessão
    - devolve um id_sessao para o usuário
    - nas próximas requisições que ele fizer, ele envia esse id junto
        - exemplo: `criar post + id_session`
    - dessa forma, o servidor vai saber qual usuário está se comunicando com ele
    - e que ele foi autenticado via login anteriormente
    - problemas:
        - se muitos usuários fizerem login ao mesmo tempo, ele guarda muitos estados/sessões, o que sobrecarrega o servidor
        - prejudica a escalabilidade da aplicação
        - vai contra os princípios do rest ("usuários não devem depender de um estado guardado para fazer as requisições")

- o que fazer?
    - já que o estado não pode ser guardado no servidor, vamos guardar no cliente
    - dessa forma o cliente faz um login e se estiver tudo correto, o servidor devolve uma info que identifica o usuário (exemplo: `id`)
    - o usuário guarda o `id` e o envia junto quando fizer uma nova requisição
    - problema:
        - como saber se este usuário é realmente o usuário do `id`?
        - e não algum ataque...
        - se certificar de que apenas os usuários vão conseguir enviar os próprios `ids`
        - uma "assinatura" do servidor no `id`, guardando o `id` assinado

- como aplicar esse modelo de "assinatura" na nossa aplicação?
    - JWT (Json Web Token)

### JWT

- essa informação assinada chama-se `token`
- como o `token` é gerado?

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

- o servidor gera uma assinatura baseada nas informações que o usuário deu e a `senha secreta` guardada
- assim faz uma comparação: a assinatura ideal = assinatura que o usuário enviou?
- se sim, então tem certeza de que esse `token` é verdadeiro e o usuário é `váliado`

[jwt.io](https://jwt.io/)

### Configurações



