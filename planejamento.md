# Mini Banco Digital — Plano de Estudo

> Projeto **educacional**, com dados fictícios. Não movimenta dinheiro real nem se conecta ao Pix ou a qualquer banco.

## 1. Objetivo

Estudar, na prática, os temas pedidos na entrevista:

- **NoSQL** (MongoDB e Redis)
- **Mensageria** (Kafka ou RabbitMQ)
- **Observabilidade** (métricas, logs e tracing)

Tudo em **Java + Spring Boot**, com **Arquitetura Hexagonal**.

## 2. A ideia

O cliente faz uma transferência pela API. O sistema debita e credita as contas e publica um evento `TransferenciaRealizada`. Outros módulos reagem a esse evento.

```
Cliente → API REST → [Domínio] → Postgres (contas e saldos)
                         │
                         └→ evento TransferenciaRealizada
                                  ├→ Extrato      → MongoDB
                                  ├→ Antifraude   → regras simples
                                  └→ Notificação  → simulada (log/e-mail)
```

## 3. Tecnologias

| Tema | Tecnologia | Uso no projeto |
|---|---|---|
| Relacional | PostgreSQL | Contas e saldos, com transações |
| NoSQL (documentos) | MongoDB | Extrato e histórico de movimentações |
| NoSQL (chave-valor) | Redis | Cache de saldo e idempotência |
| Mensageria | Kafka (ou RabbitMQ) | Eventos entre os módulos |
| Observabilidade | Actuator, Micrometer, Prometheus, Grafana | Métricas e dashboards |
| Tracing (opcional) | OpenTelemetry + Jaeger | Rastrear uma transferência de ponta a ponta |
| Testes | JUnit, Mockito, Testcontainers | Unitários e de integração |
| Infra | Docker Compose | Subir tudo localmente |

## 4. Arquitetura Hexagonal

Regra de ouro: **as dependências sempre apontam para dentro**. O domínio não conhece framework nenhum.

```
src/main/java/com/seuprojeto/minibanco
├── domain
│   ├── model          → Conta, Transferencia
│   └── exception      → SaldoInsuficienteException, ContaNaoEncontradaException
├── application
│   ├── port
│   │   ├── in         → RealizarTransferenciaUseCase
│   │   └── out        → ContaRepositoryPort, EventPublisherPort, ExtratoRepositoryPort
│   └── service        → TransferenciaService
└── infrastructure
    ├── adapter
    │   ├── in.web         → TransferenciaController, DTOs
    │   ├── in.messaging   → consumidores (extrato, antifraude, notificação)
    │   ├── out.persistence → adaptador JPA (Postgres)
    │   ├── out.mongo      → adaptador do extrato
    │   ├── out.cache      → adaptador Redis
    │   └── out.messaging  → publicador de eventos
    └── config         → beans e configurações
```

## 5. Fases

Cada fase vive em uma branch `feature/...` (Git Flow) e termina com algo que dá para rodar e testar.

### Fase 0 — Setup
- Projeto criado no Spring Initializr
- `docker-compose` com Postgres
- ✅ Pronto quando: a app sobe e conecta no banco

### Fase 1 — Domínio e transferência (Postgres)
- Domínio puro: `Conta` e `Transferencia`, com regras (valor positivo, saldo suficiente)
- Porta de entrada e caso de uso `RealizarTransferencia`, testado com Mockito
- Adaptador JPA com Flyway e `@Transactional`
- Controller REST
- ✅ Pronto quando: `POST /transferencias` move o saldo e os testes do domínio rodam sem Spring

### Fase 2 — Mensageria
- Publicar o evento `TransferenciaRealizada` (Kafka sugerido)
- Consumidor simples que só loga
- ✅ Pronto quando: cada transferência gera um evento consumido

### Fase 3 — MongoDB (extrato)
- O consumidor grava o extrato como documento
- `GET /contas/{id}/extrato`
- ✅ Pronto quando: o extrato aparece (com pequeno atraso, consistência eventual)

### Fase 4 — Redis
- Idempotência com o header `Idempotency-Key`
- Cache de saldo
- ✅ Pronto quando: repetir a mesma requisição não duplica a transferência

### Fase 5 — Observabilidade
- Actuator + Micrometer + Prometheus + Grafana
- Logs em JSON com `correlationId`
- Métricas próprias (transferências, erros, latência)
- Depois: OpenTelemetry + Jaeger
- ✅ Pronto quando: um dashboard mostra o fluxo em tempo real

### Fase 6 — Resiliência
- Retry + Dead Letter Queue
- Consumidor de antifraude simples
- Outbox pattern
- ✅ Pronto quando: uma mensagem com falha vai para a DLQ sem derrubar nada

### Fase 7 — Acabamento
- README com diagrama da arquitetura e o "porquê" de cada tecnologia
- Testcontainers nos testes de integração
- GitHub Actions rodando os testes

> As fases 1 a 5 já cobrem os três temas da entrevista. As fases 6 e 7 são bônus.

## 6. Fase 0 em detalhe

### 6.1 Criar o projeto

No [start.spring.io](https://start.spring.io):

- **Project:** Maven
- **Language:** Java
- **Spring Boot:** 3.x (versão estável)
- **Java:** 21
- **Packaging:** Jar
- **Dependências:**
  - Spring Web
  - Spring Data JPA
  - PostgreSQL Driver
  - Validation
  - Spring Boot Actuator
  - Flyway Migration

> Dependendo da versão do Spring Boot, o Flyway pode exigir também o módulo `flyway-database-postgresql` no `pom.xml`:
>
> ```xml
> <dependency>
>     <groupId>org.flywaydb</groupId>
>     <artifactId>flyway-database-postgresql</artifactId>
> </dependency>
> ```

### 6.2 `docker-compose.yml`

Na raiz do projeto:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: minibanco-postgres
    environment:
      POSTGRES_DB: minibanco
      POSTGRES_USER: minibanco
      POSTGRES_PASSWORD: minibanco
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U minibanco -d minibanco"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  pgdata:
```

Subir o banco:

```bash
docker compose up -d
```

### 6.3 `application.yml`

Em `src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: minibanco
  datasource:
    url: jdbc:postgresql://localhost:5432/minibanco
    username: minibanco
    password: minibanco
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

> Em projeto real, a senha não fica no arquivo. Aqui é só estudo local.

### 6.4 Testar

```bash
./mvnw spring-boot:run
```

Depois abra `http://localhost:8080/actuator/health`. Deve retornar `{"status":"UP"}`.

### 6.5 Git Flow

```bash
git init
git checkout -b main
git checkout -b develop
git checkout -b feature/setup
```

Faça o commit inicial, abra um Pull Request para `develop` e siga para a Fase 1.

### Checklist da Fase 0

- [ ] Projeto gerado com Maven
- [ ] `docker compose up -d` sobe o Postgres
- [ ] A aplicação conecta no banco sem erros
- [ ] `/actuator/health` retorna `UP`
- [ ] Repositório criado com `main`, `develop` e `feature/setup`

## 7. Ritmo

Sem prazo fixo. Uma fase por vez, em sessões curtas. Ao fim de cada fase, vale anotar no README o que foi decidido e por quê, porque essa explicação é o que mais conta em entrevista.
