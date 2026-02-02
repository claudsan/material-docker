# Feature Toggles no Spring Boot: O Trem Ficou Bão com banco de dados!

Nossa senhora, sô! Se você ainda tá fazendo deploy toda vez que precisa ligar ou desligar uma função no seu sistema, "para o mundo que eu quero descer". 

Hoje em dia, quem não usa **Feature Toggle** (ou Feature Flags) tá mais perdido que cego em tiroteio. 

Vou te mostrar como montar uma biblioteca genérica usando **Togglz** e **MongoDB** pra você compartilhar com todos os seus microsserviços. É um trem "bão demais da conta", facinho de fazer e que vai te dar uma paz de espírito danada.


## Por que esse trem é importante?
Imagine que você subiu aquela feature nova, mas o trem deu ruim lá em produção. Em vez de sair correndo pra fazer rollback (aquele desespero!), você só vai num painel, clica num botão e... **pimba!** A feature desliga na hora, sem precisar de deploy nem nada. **Uai, é mágica?** Não, é engenharia das boa!

## A Receita do Pão de Queijo (A Configuração)

Pra esse trem funcionar no **Spring Boot 3** (com Java 17 ou 21), a gente usa o [Togglz 4.4.0](https://www.togglz.org), que já tá atualizado pro tal do "Jakarta EE". Assim não tem erro de compatibilidade prá não "deixar o café aguado".

#### 1. Os Ingredientes (Dependências)
No `pom.xml` da sua lib, você joga essas dependências aqui. É o básico pra o trem não solar:

```xml
<dependencies>
    <!-- Core do Togglz para Spring Boot 3 -->
    <dependency>
        <groupId>org.togglz</groupId>
        <artifactId>togglz-spring-boot-starter</artifactId>
        <version>4.4.0</version>
    </dependency>
    <!-- Repositório MongoDB oficial do Togglz -->
    <dependency>
        <groupId>org.togglz</groupId>
        <artifactId>togglz-mongodb</artifactId>
        <version>4.4.0</version>
    </dependency>
</dependencies>
```

### 2. O Forno (Configurando no Mongo)

Aqui é onde a mágica acontece. Mas ó, presta atenção: se a gente for no banco toda hora perguntar se a flag tá ligada, o banco não aguenta e "abre o bico". Por isso, a gente usa um tal de CachingStateRepository pra guardar a informação na memória por um tempinho. É igual guardar o queijo na prateleira de cima pra não ter que ir no porão toda hora

```java
@Configuration
public class MinhaLibToggleConfig {

    @Bean
    public StateRepository stateRepository(MongoClient mongoClient) {
        // Primeiro a gente conecta no banco
        StateRepository repository = MongoStateRepository.builder(mongoClient, "feature_flags")
                .collectionName("togglz_store")
                .build();

        // Nuuu! Agora sim: Cache de 30 segundos (30000ms), ocê escolhe conforme a necessidade.
        // Assim a aplicação não fica "amolando" o banco a cada milissegundo.
        return new CachingStateRepository(repository, 30000);
    }

    @Bean
    public FeatureProvider featureProvider() {
        // Esse trem aqui precisa saber ONDE procurar. 
        // Não esquece de passar sua classe de Enum aqui dentro, senão ele fica perdido!
        return new EnumBasedFeatureProvider(MinhasFlags.class);
    }
}
```

### Docker Compose pro Mongo

Pra testar esse trem sem ter que instalar o banco na sua máquina (que dá uma "preguiça danada"), usa esse Docker Compose aqui que é tiro e queda:
```yaml
version: '3.8'
services:
  mongo:
    image: mongo:latest
    container_name: mongo_togglz
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_DATABASE=feature_flags
```


### Painel/Console (O Nativo e o Customizado)

#### Opção A: Console Nativo (Pronto pra usar)
O Togglz já te dá uma interface visual prontinha! Basta acessar http://localhost:8080/togglz-console. 

Lá você consegue ligar e desligar as flags com um clique. É simples e resolve 99% dos casos.


#### Opção B: Console Personalizado (API REST)

Se você precisa de uma interface com a cara da sua empresa, você pode criar seus próprios endpoints. 

E pra fazer bonito pro Spring (e pro Sonar), a gente injeta as dependências pelo construtor, que é o jeito "elegante" de fazer:

```java
@RestController
@RequestMapping("/api/flags")
public class CustomFlagController {

    private final FeatureManager manager;

    // Injeção via construtor: O Spring 3 gosta mais assim e evita "dor de cabeça" nos testes
    public CustomFlagController(FeatureManager manager) {
        this.manager = manager;
    }

    @GetMapping
    public List<FeatureState> listar() {
        return manager.getFeatures().stream()
            .map(f -> manager.getFeatureState(f))
            .toList();
    }

    @PostMapping("/{name}/toggle")
    public void mudar(@PathVariable String name, @RequestParam boolean enabled) {
        Feature feature = new NamedFeature(name);
        FeatureState state = manager.getFeatureState(feature);
        state.setEnabled(enabled);
        manager.setFeatureState(state); 
    }
}
```

### Integração com Actuator
O Togglz se integra ao Spring Boot Actuator, permitindo visualizar as flags via JSON padrão em /actuator/togglz.
Para ativar no ``application.yml``:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: togglz
togglz:
  console:
    enabled: true
    path: /togglz-console # Você pode mudar esse caminho se quiser!
```


### Como usar na lida (No dia a dia)

Agora que a "carpintaria" tá pronta, a vida do desenvolvedor fica mansa! Pra criar novas chaves (flags) no sistema, você não precisa mexer no banco de dados nem fazer configuração complicada.

O processo é mamão com açúcar: basta criar um Enum simples implementando a interface Feature. Cada item desse Enum vira automaticamente um botão lá no seu painel administrativo.

Óia como é fácil:


```java
public enum MinhasFlags implements Feature {
    @Label("Pagar com PIX")
    PAGAMENTO_PIX,

    @Label("Promoção de Pão de Queijo")
    PROMO_PAO_DE_QUEIJO;
    
    public boolean taLigado() {
        return FeatureContext.getFeatureManager().isActive(this);
    }
}
```

E pra usar no meio do código? É só fazer um if simples, igual mamão com açucar:

```java
if (MinhasFlags.PAGAMENTO_PIX.taLigado()) {
    // Código novo (Feature ativa)
    pagarComPix();
} else {
    // Código antigo (Feature desligada)
    pagarComBoleto();
}
```


### Protegendo o Curral (Segurança)
Nossa senhora! Você não vai deixar qualquer um mexer nas suas flags, né? No console administrativo (que fica em /togglz-console), a gente bota uma tranca com Spring Security. Só quem for por exemplo ROLE_ADMIN é que manda no pedaço.
```java
@Bean
public UserProvider userProvider() {

    //return () -> new SimpleFeatureUser("admin", true); // -> Assim pode testar local com a porteira aberta (sem segurança)
    return new SpringSecurityUserProvider("ROLE_ADMIN");
}
```


## E o REDIS pode? (adicionar aquele doce leite no pão de queijo)

E vou te falar: usar Redis com Feature Toggle é "bão" demais da conta, porque o trem é rápido igual um raio!

Como o Redis guarda tudo na memória, a verificação da flag na sua aplicação fica instantânea, sem nem precisar "ir ali" no disco.

No Togglz, o suporte pro Redis é oficial e funciona que é uma beleza.

### O que muda na receita?

Pra trocar o MongoDB pelo Redis, você só precisa ajustar dois "tiquim" de coisa:

1. Mude os Ingredientes (Dependências)
No seu pom.xml, tira o do Mongo e coloca o do Redis:
```xml
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-redis</artifactId>
    <version>4.4.0</version>
</dependency>
```

### 1.1 Configuração do Redis

Adicioanar a configuração do REDIS no ``application.yaml`` para se comunicar com o REDIS.

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      # password: sua-senha-aqui (se tiver)
```

### 2. Configure o Repositório na Classe

Em vez do MongoStateRepository, você vai usar o RedisStateRepository. 

O Togglz usa uma lib chamada Jedis pra conversar com o Redis:

```java
@Configuration
public class MinhaLibToggleConfig {

    @Bean
    public StateRepository stateRepository(
            @Value("${spring.data.redis.host:localhost}") String redisHost,
            @Value("${spring.data.redis.port:6379}") int redisPort
    ) {
        JedisPool jedisPool = new JedisPool(redisHost, redisPort);
        
        // O cache de 10s (10000ms) também é importante aqui pra não sobrecarregar o Redis
        StateRepository redisRepo = new RedisStateRepository(jedisPool, "togglz:");
        return new CachingStateRepository(redisRepo, 10000);
    }
    
    // ... bean do featureProvider continua igual
}
```

### Por que usar Redis em vez de Mongo?
- Velocidade Nuuu: O Redis é focado em baixíssima latência. Se sua aplicação checa flags milhares de vezes por segundo, o Redis é o caminho.
- Simplicidade: Se você já usa o Redis pra cache ou sessão, é só aproveitar o "puxadinho" e guardar as flags lá também.


### Docker Compose pro Redis
Se quiser testar esse trem agora, troca o serviço do Mongo por esse aqui no seu arquivo do docker-compose:
```yaml
  redis:
    image: redis:latest
    container_name: redis_togglz
    ports:
      - "6379:6379"
```


### Conclusão
Usar Togglz com Mongo no Spring 3 é um trem que compensa demais. Você ganha agilidade, segurança e ainda consegue gerenciar tudo por uma interface bonitinha. Se alguém vier te perguntar por que você demorou tanto pra implementar isso, você só responde: "É que o trem tava devagar, mas agora pegou trilho!"

E aí, uai, o que achou desse guia? Ficou "chique no úrtimo" comenta ai se já conhecia ou já utiliza o Togglz/outra para gerenciar as feature toggles no seu projeto.

