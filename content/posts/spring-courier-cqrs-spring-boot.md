---
title: "Spring Courier: Implementando CQRS no Spring Boot de forma simples e poderosa"
date: 2026-03-02
draft: false
tags: ["java", "spring-boot", "cqrs", "design-patterns", "open-source", "mediator"]
categories: ["Tutorial"]
description: "Tutorial completo sobre o Spring Courier — uma lib open source que traz o poder do padrão CQRS + Mediator para o ecossistema Spring Boot, inspirada no MediatR do .NET."
---

Se você já trabalhou com **.NET**, provavelmente conhece o [MediatR](https://github.com/jbogard/MediatR) — aquela biblioteca que transforma a forma como organizamos commands, queries e eventos nas aplicações. No mundo **Java/Spring Boot**, frameworks como Axon existem, mas muitas vezes são pesados demais para quem só quer desacoplar a lógica da aplicação sem embarcar em Event Sourcing completo.

Foi exatamente pra resolver isso que criei o **[Spring Courier](https://github.com/Valossa515/spring-courier)** — uma biblioteca CQRS leve, extensível e com **zero configuração** para Spring Boot.

Neste tutorial, vou mostrar na prática como integrar o Spring Courier em um projeto e usar cada recurso que ele oferece.

---

## O que é CQRS e por que usar?

CQRS (Command Query Responsibility Segregation) é um padrão arquitetural que separa as operações de **escrita** (Commands) das de **leitura** (Queries). Em vez de um único service monolítico fazendo tudo, cada operação tem seu próprio handler dedicado.

Os benefícios diretos são:

- **Código desacoplado** — cada caso de uso vive em sua própria classe
- **Testabilidade** — handlers são fáceis de testar isoladamente
- **Escalabilidade** — leitura e escrita podem escalar de forma independente
- **Manutenção** — acabam os "God Services" com 30 métodos e 15 dependências injetadas

O **Mediator Pattern** complementa o CQRS ao fornecer um ponto central que roteia commands e queries para seus respectivos handlers, eliminando o acoplamento direto entre controllers e lógica de negócio.

---

## Conhecendo o Spring Courier

O Spring Courier oferece os seguintes recursos:

- **Command Handlers** e **Query Handlers** — separação clara entre escrita e leitura
- **Notifications/Events** — publique eventos para múltiplos handlers simultâneos
- **Validation Pipeline** — valide requests antes da execução com pipeline behaviors
- **Async Support** — publicação assíncrona de notificações
- **Auto-configuração** — plug and play com Spring Boot, sem XML nem configs manuais
- **Interfaces genéricas** — tipagem forte e flexível via generics

### Pré-requisitos

- **Java 17+**
- **Spring Boot 3.x+**

---

## Mãos à obra: Projeto completo

Vamos construir uma API de gerenciamento de produtos do zero usando Spring Courier.

### 1. Adicionando a dependência

**Maven:**
```xml
<dependency>
    <groupId>io.github.valossa515</groupId>
    <artifactId>spring-courier</artifactId>
    <version>1.1.0</version>
</dependency>
```

**Gradle:**
```groovy
implementation("io.github.valossa515:spring-courier:1.1.0")
```

Pronto. Sem nenhuma configuração adicional — o Spring Courier usa auto-configuração do Spring Boot e descobre automaticamente todos os handlers anotados com `@Service` ou `@Component`.

### 2. Entendendo as interfaces principais

Antes de codar, vamos entender a anatomia da lib:

| Interface | Propósito | Tipo |
|---|---|---|
| `ICommand<R>` | Representa uma operação de escrita que retorna `R` | Command |
| `CommandHandler<C, R>` | Processa um `ICommand` e retorna a resposta | Handler |
| `IQuery<R>` | Representa uma operação de leitura que retorna `R` | Query |
| `QueryHandler<Q, R>` | Processa uma `IQuery` e retorna a resposta | Handler |
| `INotification` | Representa um evento/notificação | Event |
| `NotificationHandler<N>` | Processa uma notificação (pode ter múltiplos handlers) | Handler |
| `Courier` | O mediador central — envia commands/queries e publica eventos | Mediator |
| `Validator<T>` | Valida um request antes da execução | Pipeline |

O fluxo é simples: o **Controller** injeta o `Courier`, envia um command ou query, e o Courier roteia para o handler correto automaticamente.

### 3. Criando Commands e Handlers

Vamos criar o fluxo de criação de produto. Primeiro, o record que representa o command e o DTO de resposta:
```java
// Command
public record CreateProductCommand(
    String name, 
    BigDecimal price, 
    String category
) implements ICommand<CreateProductResponse> {}

// Response DTO
public record CreateProductResponse(
    UUID id, 
    String name, 
    BigDecimal price, 
    String category
) {}
```

Agora o handler que processa esse command:
```java
@Service
@RequiredArgsConstructor
public class CreateProductHandler 
        implements CommandHandler<CreateProductCommand, CreateProductResponse> {

    private final ProductRepository repository;

    @Override
    public CreateProductResponse handle(CreateProductCommand command) {
        var product = new Product(
            UUID.randomUUID(),
            command.name(),
            command.price(),
            command.category()
        );
        
        repository.save(product);
        
        return new CreateProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getCategory()
        );
    }
}
```

Repare que o handler é um bean Spring normal — você pode injetar repositórios, services e qualquer dependência normalmente.

### 4. Criando Queries

Agora a query para buscar um produto por ID:
```java
// Query
public record GetProductByIdQuery(UUID id) 
        implements IQuery<ProductResponse> {}

// Response
public record ProductResponse(
    UUID id, 
    String name, 
    BigDecimal price, 
    String category
) {}
```

E o handler:
```java
@Service
@RequiredArgsConstructor
public class GetProductByIdHandler 
        implements QueryHandler<GetProductByIdQuery, ProductResponse> {

    private final ProductRepository repository;

    @Override
    public ProductResponse handle(GetProductByIdQuery query) {
        var product = repository.findById(query.id())
            .orElseThrow(() -> new ProductNotFoundException(query.id()));
        
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getCategory()
        );
    }
}
```

### 5. O Controller — limpo e enxuto

Aqui é onde a mágica aparece. O controller não conhece nenhum handler diretamente — ele só fala com o `Courier`:
```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final Courier courier;

    @PostMapping
    public ResponseEntity<CreateProductResponse> create(
            @RequestBody CreateProductCommand command) {
        var response = courier.send(command);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getById(@PathVariable UUID id) {
        var response = courier.send(new GetProductByIdQuery(id));
        return ResponseEntity.ok(response);
    }
}
```

O controller tem **zero lógica de negócio**. Ele recebe a request, empacota em um command ou query, e delega ao Courier. Isso torna os controllers extremamente finos e testáveis.

### 6. Notifications — eventos para múltiplos handlers

Um dos recursos mais poderosos é o sistema de notificações. Quando um produto é criado, você pode querer disparar diversas ações: enviar email, atualizar cache, gerar log de auditoria, etc.

Primeiro, defina o evento:
```java
public record ProductCreatedEvent(
    UUID productId, 
    String name, 
    BigDecimal price
) implements INotification {}
```

Crie quantos handlers quiser — todos serão executados quando o evento for publicado:
```java
@Service
@Slf4j
public class SendEmailOnProductCreated 
        implements NotificationHandler<ProductCreatedEvent> {
    
    @Override
    public void handle(ProductCreatedEvent event) {
        log.info("Enviando email de confirmação para produto: {}", 
                 event.name());
    }
}

@Service
@Slf4j
public class UpdateCacheOnProductCreated 
        implements NotificationHandler<ProductCreatedEvent> {
    
    @Override
    public void handle(ProductCreatedEvent event) {
        log.info("Atualizando cache para produto: {}", 
                 event.productId());
    }
}

@Service
@Slf4j
public class AuditLogOnProductCreated 
        implements NotificationHandler<ProductCreatedEvent> {
    
    @Override
    public void handle(ProductCreatedEvent event) {
        log.info("Registrando auditoria: produto {} criado", 
                 event.name());
    }
}
```

Publique o evento no handler do command:
```java
@Service
@RequiredArgsConstructor
public class CreateProductHandler 
        implements CommandHandler<CreateProductCommand, CreateProductResponse> {

    private final ProductRepository repository;
    private final Courier courier;

    @Override
    public CreateProductResponse handle(CreateProductCommand command) {
        var product = new Product(
            UUID.randomUUID(), 
            command.name(), 
            command.price(), 
            command.category()
        );
        repository.save(product);

        // Publica notificação — todos os handlers serão executados
        courier.publish(new ProductCreatedEvent(
            product.getId(), 
            product.getName(), 
            product.getPrice()
        ));

        // Ou de forma assíncrona:
        // courier.publishAsync(new ProductCreatedEvent(...));

        return new CreateProductResponse(
            product.getId(), product.getName(), 
            product.getPrice(), product.getCategory()
        );
    }
}
```

### 7. Validation Pipeline — validação antes da execução

O Spring Courier permite criar validators que são executados **antes** do handler, seguindo o padrão de pipeline behaviors:
```java
public class CreateProductValidator 
        implements Validator<CreateProductCommand> {
    
    @Override
    public ValidationResult validate(CreateProductCommand command) {
        List<ValidationError> errors = new ArrayList<>();

        if (command.name() == null || command.name().isBlank()) {
            errors.add(new ValidationError("name", 
                "O nome do produto é obrigatório"));
        }

        if (command.price() == null 
                || command.price().compareTo(BigDecimal.ZERO) <= 0) {
            errors.add(new ValidationError("price", 
                "O preço deve ser maior que zero"));
        }

        return errors.isEmpty() 
            ? ValidationResult.success() 
            : ValidationResult.failure(errors);
    }
}
```

Registre o behavior como bean:
```java
@Configuration
public class PipelineConfig {

    @Bean
    public ValidationBehavior<CreateProductCommand, CreateProductResponse> 
            createProductValidation() {
        return new ValidationBehavior<>(
            List.of(new CreateProductValidator())
        );
    }
}
```

---

## Estrutura recomendada do projeto
```
src/main/java/com/example/app/
├── product/
│   ├── commands/
│   │   ├── CreateProductCommand.java
│   │   ├── CreateProductHandler.java
│   │   └── CreateProductValidator.java
│   ├── queries/
│   │   ├── GetProductByIdQuery.java
│   │   └── GetProductByIdHandler.java
│   ├── events/
│   │   ├── ProductCreatedEvent.java
│   │   ├── SendEmailOnProductCreated.java
│   │   └── UpdateCacheOnProductCreated.java
│   ├── dto/
│   │   ├── CreateProductResponse.java
│   │   └── ProductResponse.java
│   ├── model/
│   │   └── Product.java
│   └── ProductController.java
└── Application.java
```

---

## Spring Courier vs Alternativas

| Característica | Spring Courier | Axon Framework | spring-mediatR |
|---|---|---|---|
| Peso | Leve (~zero config) | Pesado (requer Axon Server) | Leve |
| Event Sourcing | Não (foco em CQRS puro) | Sim | Não |
| Notifications | Sim (sync + async) | Sim | Limitado |
| Validation Pipeline | Sim | Via interceptors | Não |
| Spring Boot 3.x | Sim | Sim | Parcial |
| Java 17+ | Sim | Sim | Depende da versão |
| Maven Central | Sim | Sim | Bintray (descontinuado) |

---

## Conclusão

O padrão CQRS não precisa ser complicado. Com o **Spring Courier**, você ganha:

- Controllers finos que só delegam
- Handlers focados em um único caso de uso
- Eventos desacoplados com múltiplos subscribers
- Validação como cidadão de primeira classe
- Tudo isso com **uma dependência** e **zero configuração**

O projeto é open source sob licença MIT. Contribuições, issues e stars são muito bem-vindas!

**Links úteis:**

- [GitHub — spring-courier](https://github.com/Valossa515/spring-courier)
- [Maven Central](https://central.sonatype.com/artifact/io.github.valossa515/spring-courier)
- [Diagramas de arquitetura](https://github.com/Valossa515/spring-courier/tree/main/docs/diagrams)

Se tiver dúvidas ou sugestões, me encontre no [GitHub](https://github.com/Valossa515).