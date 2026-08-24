# 02 — Arquitetura

## Estilo

**Monólito modular por feature.**

Evitar package global do tipo:

```text
controller/
service/
repository/
```

Preferir:

```text
br.com.akosmed
├── shared
├── tenant
├── identity
├── professional
├── patient
├── scheduling
├── clinical
├── prescription
├── notification
└── audit
```

Cada feature:

```text
feature/
├── controller
├── dto
├── entity
├── repository
└── service
```

## Shared

Deve conter somente elementos realmente compartilhados:

```text
shared/
├── config
├── exception
├── security
├── tenant
└── storage
```

Evitar criar `BaseService`, `BaseController` e abstrações genéricas cedo demais.

## Fluxo de request

```text
HTTP
 ↓
Security Filter
 ↓
TenantContext
 ↓
Controller
 ↓
DTO + Validation
 ↓
Service
 ↓
Repository
 ↓
Banco
```

Durante o Core, o banco é H2. PostgreSQL entra somente na etapa definida no roadmap.

## DTOs

Não retornar entidade JPA diretamente.

Padrão simples:

```text
PacienteCreateDTO
PacienteUpdateDTO
PacienteResponseDTO
```

Relacionamentos recebidos pela API usam UUID público:

```text
pacientePublicId
profissionalPublicId
unidadePublicId
procedimentoPublicId
```

Não criar dezenas de DTOs se dois ou três bastarem.

## Mapper

Começar manualmente.

Se o projeto acumular muito boilerplate, avaliar MapStruct depois.

## IDs

Padrão obrigatório desde o primeiro CRUD:

```text
id        Long
publicId  UUID
```

`id`:

- PK técnica;
- FKs internas;
- locks e operações internas.

`publicId`:

- URLs;
- DTOs;
- respostas;
- integrações;
- app Kotlin;
- assistentes.

Exemplo JPA:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Column(nullable = false, unique = true, updatable = false)
private UUID publicId;
```

O `publicId` é gerado uma única vez antes do primeiro persist.

Nunca expor o `Long id` na API pública.

Relacionamentos no banco continuam usando IDs numéricos.

## Datas

- `LocalDate` para datas sem horário;
- `LocalTime` para disponibilidade semanal;
- `OffsetDateTime` ou `Instant` para eventos reais;
- Tenant possui timezone configurável.

Evitar `LocalDateTime` sem contexto em agendamentos de produção multi-timezone.

## Status

Usar enums Java persistidos como string:

```java
@Enumerated(EnumType.STRING)
```

Evitar códigos `1,2,3` no banco.

## Erros

Criar `GlobalExceptionHandler` desde a ETAPA 0.

Formato:

```json
{
  "timestamp": "2026-08-23T21:00:00-03:00",
  "status": 404,
  "code": "PACIENTE_NOT_FOUND",
  "message": "Paciente não encontrado",
  "path": "/api/v1/pacientes/7b2f4b2c-8d8f-4f45-8e48-39ad9aa84aac"
}
```

Exceções mínimas:

```text
ResourceNotFoundException
BusinessRuleException
AccessDeniedException
ConflictException
```

## Profiles

```text
application.yml
application-dev.yml
application-test.yml
application-prod.yml
```

## Configuração

Secrets sempre por ambiente.

```text
DB_URL
DB_USER
DB_PASSWORD
JWT_SECRET
STORAGE_*
```

## Transactions

Service é a fronteira transacional.

- leitura: `@Transactional(readOnly = true)`;
- escrita: `@Transactional`.

## Repositories

Para entidades tenant-scoped chamadas pela API, preferir métodos explícitos por UUID público:

```java
Optional<Paciente> findByPublicIdAndTenantId(UUID publicId, Long tenantId);
Page<Paciente> findAllByTenantId(Long tenantId, Pageable pageable);
```

O `tenantId` interno vem do contexto já resolvido pelo servidor.

`findById(Long id)` pode existir apenas para uso interno quando o recurso já foi resolvido/autorizado.

## Concorrência

Operações que disputam o mesmo recurso devem ser tratadas dentro da transação.

Caso principal do MVP:

```text
dois pacientes
+
mesmo profissional
+
mesmo intervalo
```

Fluxo:

```text
@Transactional
↓
lock pessimista no profissional
↓
revalidar disponibilidade
↓
consultar overlap
↓
salvar
```

A disponibilidade mostrada ao cliente nunca é garantia de reserva.

Detalhamento obrigatório em `21_PUBLIC_ID_E_CONCORRENCIA.md`.

## Dependências entre módulos

Direção desejada:

```text
tenant       ← identity
professional ← scheduling
patient      ← scheduling
patient      ← clinical
professional ← clinical
clinical     ← prescription
```

Evitar ciclos entre Services.

Quando dois módulos precisam conversar, preferir um Service público bem definido.

## Storage

Criar contrato:

```text
StorageService
- save
- load
- deleteLogical/reference
```

Implementações:

```text
LocalStorageService (dev)
S3StorageService (prod/futuro)
```

## Não implementar agora

- DDD completo;
- CQRS;
- event sourcing;
- hexagonal formal em todas as features;
- microsserviços;
- mensageria.

A arquitetura deve ser limpa sem virar um projeto de infraestrutura.

---

## Regra prática de implementação

Embora o backend seja modular, o desenvolvimento será vertical:

```text
feature pequena
→ camada completa
→ testes
→ documentação
→ próxima feature
```

Não criar todas as camadas do sistema antecipadamente.

A separação por módulo serve para organização, não para transformar cada módulo em um microsserviço.

---

## Documentação como guardrail

Durante a implementação, os seguintes documentos funcionam como guardrails:

```text
Entity/DTO/Repository → 15
JPA/relacionamentos → 16
HTTP/API → 17
Services → 18
Qualidade → 19
IA externa → 20
Public ID/Concorrência → 21
```

A intenção é que a estrutura da API permaneça consistente mesmo quando implementada em sessões isoladas ou com auxílio de diferentes ferramentas.
