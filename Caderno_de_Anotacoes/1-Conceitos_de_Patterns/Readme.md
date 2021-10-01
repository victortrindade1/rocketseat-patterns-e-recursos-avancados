# Conceitos de Patterns

## O q são design patterns

Patterns são regras de organização do código.

São patterns comuns:
  - Singleton (é pra usar)
  - Repository
  - Service (é pra usar)
  - Observer

### Singleton

No pattern Singleton, uma classe só pode ter uma instância. Isso quer dizer q se vc instanciar, chamar um objeto, o programa só lê o código daquele objeto uma única vez. Nós já fazemos isso naturalmente usando export e import.

### Repository

Abstração da conexão com o banco. Pelo q entendi é separar umas queries muito usadas num local e sempre chamá-las. Só faz sentido qnd não usa ORM. Não é meu caso.

### Service

Abstração da lógica. Até aqui colocamos toda a lógica dentro dos controllers. O pattern service é retirar a lógica do controller e alocar em diretórios separados.

### Observer

Comunicação por eventos. Não entendi, o curso não usou nenhum momento.
