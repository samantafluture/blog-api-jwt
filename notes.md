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

