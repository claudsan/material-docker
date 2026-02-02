# Feature Toggles no Spring Boot 3: O Trem Ficou Bão com Togglz e um banco de dados!

Nossa senhora, sô! Se você ainda tá fazendo deploy toda vez que precisa ligar ou desligar uma função no seu sistema, "para o mundo que eu quero descer". Hoje em dia, quem não usa **Feature Toggle** (ou Feature Flags) tá mais perdido que cego em tiroteio. 

Vou te mostrar como montar uma biblioteca genérica usando **Togglz** e **MongoDB** pra você compartilhar com todos os seus microsserviços. É um trem "bão demais da conta", facinho de fazer e que vai te dar uma paz de espírito danada.

---

### Por que esse trem é importante?
Imagine que você subiu aquela feature nova, mas o trem deu ruim lá em produção. Em vez de sair correndo pra fazer rollback (aquele desespero!), você só vai num painel, clica num botão e... **pimba!** A feature desliga na hora, sem precisar de deploy nem nada. **Uai, é mágica?** Não, é engenharia das boa!

### A Receita do Pão de Queijo (A Configuração)

Pra esse trem funcionar no **Spring Boot 3** (com Java 17 ou 21), a gente usa o [Togglz 4.4.0](https://www.togglz.org), que já tá atualizado pro tal do "Jakarta EE". Assim não tem erro de compatibilidade nem "dor de corno".

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

Aqui é onde a mágica acontece. A gente vai configurar o Togglz pra guardar o estado das flags lá no MongoDB. Assim, se sua aplicação cair ou reiniciar, as configurações continuam lá, firmes e fortes.

```java
@Configuration
public class MinhaLibToggleConfig {

    @Bean
    public StateRepository stateRepository(MongoClient mongoClient) {
        // Nuuu! Olha que facilidade: salva tudo no banco 'feature_flags'
        return MongoStateRepository.builder(mongoClient, "feature_flags")
                .collectionName("togglz_store")
                .build();
    }

    @Bean
    public FeatureProvider featureProvider() {
        // Esse trem aqui varre o projeto atrás dos seus Enums de feature
        return new EnumBasedFeatureProvider();
    }
}
```


### Duas Opções de Painel (O Nativo e o Customizado)
Opção A: Console Nativo (Pronto pra usar)
O Togglz já te dá uma interface visual prontinha! Basta acessar http://localhost:8080/togglz-console. 

Lá você consegue ligar e desligar as flags com um clique. É simples e resolve 99% dos casos.
Opção B: Console Personalizado (API REST)


Se você precisa de uma interface com a cara da sua empresa, você pode criar seus próprios endpoints injetando o FeatureManager:

```java
@RestController
@RequestMapping("/api/flags")
public class CustomFlagController {

    @Autowired
    private FeatureManager manager;

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
Para ativar no application.yml:
yaml
management:
  endpoints:
    web:
      exposure:
        include: togglz
togglz:
  console:
    enabled: true
    path: /togglz-console # Você pode mudar esse caminho se quiser!
Use o código com cuidado.




### Como usar na lida (No dia a dia)
O desenvolvedor que usar sua lib só precisa criar um Enum implementando Feature. É simples igual chupar manga:


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

### Protegendo o Curral (Segurança)
Nossa senhora! Você não vai deixar qualquer um mexer nas suas flags, né? No console administrativo (que fica em /togglz-console), a gente bota uma tranca com Spring Security. Só quem for por exemplo ROLE_ADMIN é que manda no pedaço.
```java
@Bean
public UserProvider userProvider() {

    //return () -> new SimpleFeatureUser("admin", true); // -> Assim pode testar local com a porteira aberta (sem segurança)
    return new SpringSecurityUserProvider("ROLE_ADMIN");
}
```

### E o REDIS pode? 

E vou te falar: usar Redis com Feature Toggle é "bão" demais da conta, porque o trem é rápido igual um raio! Como o Redis guarda tudo na memória, a verificação da flag na sua aplicação fica instantânea, sem nem precisar "ir ali" no disco.

No Togglz, o suporte pro Redis é oficial e funciona que é uma beleza.

### 🛠️ O que muda na receita?

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

### 2. Configure o Repositório na Classe
Em vez do MongoStateRepository, você vai usar o RedisStateRepository. O Togglz usa uma lib chamada Jedis pra conversar com o Redis:
```java
@Bean
public StateRepository stateRepository() {
    // Cria o pool de conexão com o Redis local
    JedisPool jedisPool = new JedisPool("localhost", 6379);
    
    // O Togglz vai salvar as flags com o prefixo "togglz:" lá no Redis
    return new RedisStateRepository(jedisPool, "togglz:");
}
```

### 🏎️ Por que usar Redis em vez de Mongo?
- Velocidade Nuuu: O Redis é focado em baixíssima latência. Se sua aplicação checa flags milhares de vezes por segundo, o Redis é o caminho.
- Simplicidade: Se você já usa o Redis pra cache ou sessão, é só aproveitar o "puxadinho" e guardar as flags lá também.


### 🐳 Docker Compose pro Redis
Se quiser testar esse trem agora, troca o serviço do Mongo por esse aqui no seu arquivo:
```yaml
  redis:
    image: redis:latest
    container_name: redis_togglz
    ports:
      - "6379:6379"
```


### Conclusão
Usar Togglz com Mongo no Spring 3 é um trem que compensa demais. Você ganha agilidade, segurança e ainda consegue gerenciar tudo por uma interface bonitinha. Se alguém vier te perguntar por que você demorou tanto pra implementar isso, você só responde: "É que o trem tava devagar, mas agora pegou trilho!"

E aí, uai, o que achou desse guia? Ficou "chique no úrtimo" ou precisa de mais algum detalhe pro seu projeto? Diz aí se você quer que eu te mostre como configurar o Spring Boot Actuator pra monitorar essas flags!
