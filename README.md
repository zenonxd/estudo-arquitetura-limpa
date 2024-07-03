# Como fazer arquitetura limpa protegendo regras de negócio

[Video de Referência](https://www.youtube.com/watch?v=hit0XHGt4WI)

## 1. Criar um pacote (camada) Domain e colocar nele, as entidades do sistema.

Essas entidades são regras de negócio, elas não podem ter dependência com framework.

Um user com anotações e importando spring, não será uma entidade. Precisaremos, portanto,
criar outro User desacoplado da estrutura de banco de dados/framework.

Esse User criado (pode ser um Book ou qualquer outra coisa), será um record, e nele passaremos
como parâmetro o que desejarmos ```(String username, String password, String email)```, por
exemplo.

## 2. Criar pacote (camada) application, aqui ficarão: casos de uso ou interactors.

Cada operação que fizermos, será um caso de uso. Dentro de application criamos outro
pacote chamado "usecases". **Nele, ficará alocado cada interface dos casos de uso**.

E assim, criaremos um interactor (basicamente um service) chamado [CreateUserInteractor](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/application/usecases/CreateUserInteractor.java#L6). 
Ele ficará responsável por criar um Usuário.
```java
public class CreateUserInteractor(User user) {
    public User createUser() {
        
    }
}
```
**Esse User utilizado como parâmetro não é o User do modelo inicial, e sim a entidade do Domain conforme criado no item 1.**

Agora, para realizarmos a criação desse User no interactor, precisaremos fazer um desacoplamento.
No caso, criar uma porta (interface) pra gente se comunicar com as camadas externas (neste caso),
de persistência.

## 3. Dentro de application, criaremos uma pasta chamada gateways.

Essa pasta terá uma classe chamada [UserGateway](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/application/gateways/UserGateway.java#L5) (interface). Gateway, nada mais é, do que um portão de entrada. Um UserGateway, portanto, será uma estrutura genérica que utilizaremos para criar um Usuário.
```java
public interface UserGateway() {
    User createUser(User user);
}
```
Essa interface terá a User (do pacote entitity) dentro dela [Veja aqui](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/application/gateways/UserGateway.java#L3).

## 4. Voltamos para o Interactor e importamos a interface criada acima: 
```private UserGateWay;```. Dessa forma, a gente não usa nenhuma implementação. [Veja aqui](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/application/usecases/CreateUserInteractor.java#L7)


## 5. Agora, dentro do Interactor, faremos uma injeção de dependência.

Criaremos um construtor (do Interactor) passando o UserGateWay dentro dele.
E assim, o createUser (lá de cima) poderá dar o ```return userGateway.createUser(user)```.

## 6. Implementação

### Infrastructure

Criaremos um pacote (infraestructure). Dentro dele teremos dois pacotes: **gateways** e **persistence**.

- Gateway: Criaremos uma classe chamada [UserRepositoryGateway](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/gateways/UserRepositoryGateway.java#L8) e daremos implements com UserGateWay.

Essa classe irá interagir com o Repository do SpringData (aquele que a gente geralmente cria
no pacote repositores que extends JpaRepository). Basicamente, daremos um CTRL C + CTRL V. Fazendo uma copia do UserRepository para dentro do pacote **persistence**.
  
Assim que colocaremos o Repository do SpringData dentro do pacote persistence, poderemos importar ele para dentro da classe UserRepositoryGateway e criaremos
um construtor para injetá-lo.

Para que o repository do SpringData não dependa do **User inicial (entidade do banco)**,
dentro do pacote Persistence criaremos um UserEntity contendo todas as características do 
User inicial, e assim poderemos usar anotações tanto do JPA quanto do SpringData.

Então o UserRepository de persistence, não apontará mais para ```<User>```, e sim ```<UserEntitity>```.

No gateway, precisaremos gerar essa UserEntity a partir de um Usuário de domínio (a classe User inicial).

Para fazermos isso e centralizar esse código, criaremos uma classe Mapper [Veja aqui](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/gateways/UserEntityMapper.java#L6)

**Nele, teremos um método que retornará um UserEntity e um User.**
- [UserEntity](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/gateways/UserEntityMapper.java#L7) -
O parâmetro passado, é o User inicial (classe domain).

Como passaremos os atributos no retorno desse método, criaremos um construtor na classe UserEntity. Mas sem o Id, pois ele será gerado automaticamente pelo Banco.

- [User](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/gateways/UserEntityMapper.java#L11) -
O parâmetro passado é o UserEntity (do pacote persistence)

E retornaremos um novo User passando os atributos do UserEntity.

### Voltando a classe do GateWay para fazer a conversão

Importaremos a classe Mapper criada.

Esse método createUser retornará um **User** e receberá como parâmetro o User inicial (domain).

Instanciaremos um **UserEntity** e passeremos o ```userEntityMapper.toEntity(userDomainObj)```. 
Transformando esse User em um UserEntity.

Instanciaremos também outro **UserEntity**, chamado savedObj. Aqui, entraremos no repository para usar o .save().

Por fim, daremos o return. Nele, usaremos o .toDomainObj do mapper, que receberá o UserEntity, retornando um **new User**.

- Persistence: Uma camada de persistência (adapter).

Terá UserEntity (com os atributos do User do pacote model), porém sem anotações.
E o UserRepository que é um Ctrl C + Ctrl V do repositorie para essa pasta mesmo.
<hr>

### Controller
Criaremos um pacote (controller). Dentro dele criaremos um [UserController](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/controllers/UserController.java#L13) com sua anotação @RestController.

Fazer o import do CreateUserInteractor, juntamente com seu construtor e um método create.

No nosso @PostMapping, trabalharemos com padrão DTO, para que possamos mudar a regra de API, sem mudar a regra de negócio.

Isso é ótimo até mesmo para não fazer informação de negócio que não seja necessária para a API.

Usaremos, portanto, Request e Response.

Criaremos duas classes, onde ambas serão records.

- [CreateUserRequest](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/controllers/CreateUserRequest.java#L3) -
Terá username, password e email como parâmetro.
 

- [CreateUserResponse](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/controllers/CreateUserResponse.java#L3).
Terá username e email como parâmetro, (justamente para não retornamos a senha).

E na classe UserController, nosso [endpoint](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/controllers/UserController.java#L24) retornará um CreateUserResponse e vai receber um CreateUserRequest chamada request.

Chamaremos o iteractor para fazermos a operação que a gente precisa. O problema é: o createUserIteractor recebe um User. **Como criar uma forma de converter essa request em um user?**

Dentro do pacote Controller, criaremos um [UserDTOMapper](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/infrastructure/controllers/UserDTOMapper.java#L5), onde terá a mesma lógica do EntityMapper. Terá dois métodos:

1. Recebendo um CreateUserResponse(User user) e retornando esse **new CreateUserResponse** com username e email.

2. Uma função final que retonará um User. Nela receberemos um CreateUserRequest request que será transformada em um **new User**.

Voltando a classe Controller, injetaremos o DTO importando ele e passando dentro do construtor.

Já no endpoint, criaremos um User chamado userBusinessObj, passando o DTOMapper (importado acima) e converter o request para um User.

E a resposta (UserResponse), será um User cahamdo user, utilizando o método createUserInteractor.createUser (a função que está dentro dessa classe), passando o userBusinessObj criado acima.

O que retornaremos por fim, sera o userDtoMapper.toResponde(user).
<hr>

### Até aqui fizemos tudo sem usar nenhuma anotação Spring, nada de injeção.
Criaremos então um package main e colocaremos toda essa configuração.

Criaremos uma classe chamada [UserConfig](https://github.com/giuliana-bezerra/spring-boot-cleanarch/blob/8901e67d7c3e59dea21418e14f111034c3aba842/src/main/java/br/com/giulianabezerra/springbootcleanarch/main/UserConfig.java#L14).

Lá terá todos os Beans a serem injetados. Ou seja, tudo que fizemos: CreateUseInteractor, UserGateWay, UserEntityMapper...