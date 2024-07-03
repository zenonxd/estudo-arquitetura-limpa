## Como fazer arquitetura limpa protegendo regras de negócio
[Video de Referência](https://www.youtube.com/watch?v=hit0XHGt4WI)
1. Criar um pacote (camada) Domain e colocar nele, as entidades do sistema.

Esssas entidades são regras de negócio, elas não podem ter dependência com framework.
Um user com anotações e importando spring, não será uma entidade. Precisaremos, portanto,
criar outro User desacoplado da estrutura de banco de dados/framework.

Esse User criado (pode ser um Book ou qualquer outra coisa), será um record, e nele passaremos
como parâmetro o que desejarmos ```(String username, String password, String email)```, por
exemplo.

2. Criar pacote (camada) application, aqui ficarão: casos de uso ou interactors.

Cada operação que fizermos, será um caso de uso. Dentro de application criamos outro
pacote chamado "usecases". **Nele, ficará alocado cada interface dos casos de uso**.

E assim, criaremos um interactor (basicamente um service) chamado CreateUserInteractor. 
Ele ficará responsável por criar um Usuário.
```java
public class CreateUserInteractor(User user) {
    public User createUser() {
        
    }
}
```
**Esse User não é o User do modelo inicial, e sim a entidade do Domain conforme criado no item 1.**

Agora, para realizarmos a criação desse User no interactor, precisaremos fazer um desacoplamento.
No caso, criar uma porta (interface) pra gente se comunicar com as camadas externas (neste caso),
de persistência.

3. Dentro de application, criaremos uma pasta chamada gateways.

Essa pasta terá uma classe chamada UserGateway (interface). Gateway, nada mais é, 
do que um portão de entrada. Um UserGateway, portanto, será uma estrutura genérica que 
utilizaremos para criar um Usuário.
```java
public interface UserGateway() {
    User createUser(User user);
}
```
Essa interface terá a User entitity dentro dela.

4. Voltamos para o Interactor e importamos a interface criada acima: ```private UserGateWay;```. Dessa forma, a gente
não usa nenhuma implementação.


5. Agora faremos uma injeção de dependência.

Criaremos um construtor (do Interactor) passando o UserGateWay dentro dele.
E assim, o createUser (lá de cima) poderá dar o ```return userGateway.createUser(user)```.

6. Implementação

Criaremos um pacote (infraestructure). Dentro dele teremos dois pacotes: gateways e persistence.

- Gateway: colocaremos a implementação do UserGateway, criando uma classe chamada:
UserRepositoryGateway. Então nessa classe usaremos o ```public class UserRepositoryGateway implements UserGateway```.
Essa classe irá interagir com o Repository do springdata (aquele que a gente cria
no pacote repositores que extends JpaRepository). 
  
Assim que colocaremos o Repository do springdata dentro do pacote persistence,
poderemos importar ele para dentro da classe UserRepositoryGateway e criaremos
um construtor para injetá-lo.

Por fim, o método createUser poderá dar o ```return userRepository.save(user)```;

Para que o repository do springdata não dependa do User inicial (entidade do banco),
dentro do pacote Persistence criaremos um UserEntity contendo todas as características do 
User inicial.

Então o UserRepository de persistence, não apontará mais para User e sim UserEntity. 

No gateway, precisaremos gerar essa UserEntity a partir de um dominio.


- Persistence: Uma camada de persistência (adapter).
Nós vamos copiar o Repository do pacote repository e colocaremos 
ele dentro do package persistence.
