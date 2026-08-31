# 15 — Padrões de Entidades, DTOs e Repositories

> Referência prática para implementar cada CRUD do AkosMed sem quebrar as convenções de `publicId`, tenant e contratos REST.

---

# 1. Regra de identificação

Toda Entity persistida do AkosMed usa:

```text
Long id
UUID publicId
```

Responsabilidades:

```text
id       → PK/FK e uso interno
publicId → API, DTOs, URLs, integrações e apps
```

A API não recebe nem devolve o `Long id` como identificador de recurso.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 2. Padrão de Entity

Uma Entity deve:

- representar estado persistido;
- possuir `id` interno e `publicId` externo;
- possuir relacionamentos necessários;
- não conhecer HTTP;
- não retornar DTO;
- não acessar Repository;
- evitar setters que permitam quebrar invariantes importantes.

Exemplo conceitual:

```java
@Entity
@Table(
    name = "unidades",
    uniqueConstraints = {
        @UniqueConstraint(
            name = "uk_unidade_tenant_codigo",
            columnNames = {"tenant_id", "codigo"}
        )
    }
)
public class Unidade {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, updatable = false)
    private UUID publicId;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "tenant_id", nullable = false)
    private Tenant tenant;

    @Column(nullable = false, length = 120)
    private String nome;

    @Column(nullable = false, length = 40)
    private String codigo;

    @Column(nullable = false)
    private boolean ativo = true;

    @PrePersist
    void ensurePublicId() {
        if (publicId == null) {
            publicId = UUID.randomUUID();
        }
    }
}
```

Pode existir uma `BaseEntity` mínima somente para `id + publicId`, conforme `21_PUBLIC_ID_E_CONCORRENCIA.md`. Não adicionar nela tenant, status, timestamps ou `@Version` automaticamente.

---

# 3. Campos de auditoria e versão

Para entidades administrativas pode existir:

```text
createdAt
updatedAt
```

Não criar uma base enorme por conveniência.

Para entidades mutáveis em que lost update seja relevante, avaliar:

```java
@Version
private Long version;
```

`@Version` é seletivo, não obrigatório em histórico append-only.

Referência: `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

---

# 4. Enum

Usar:

```java
@Enumerated(EnumType.STRING)
```

Exemplo:

```text
StatusAgendamento.CONFIRMADO
```

Banco:

```text
CONFIRMADO
```

Evitar ordinal.

---

# 5. CreateDTO

Contém apenas campos que o cliente pode informar na criação.

Exemplo:

```java
public record UnidadeCreateDTO(
    @NotBlank String nome,
    @NotBlank String codigo,
    String telefone,
    @Email String email
) {}
```

Não incluir:

```text
id
publicId escolhido pelo cliente
tenantId livre
createdAt
ativo/status arbitrário
version arbitrária
```

O `publicId` é gerado pela aplicação.

O Tenant vem do contexto autenticado, exceto no fluxo administrativo global específico de criação de Tenant.

---

# 6. DTO com relacionamentos

Quando um request precisa referenciar outro recurso, usar `publicId`:

```java
public record AgendamentoCreateDTO(
    @NotNull UUID pacientePublicId,
    @NotNull UUID profissionalPublicId,
    @NotNull UUID unidadePublicId,
    @NotNull UUID procedimentoPublicId,
    @NotNull OffsetDateTime inicio
) {}
```

Nunca usar como contrato:

```text
pacienteId Long
profissionalId Long
unidadeId Long
procedimentoId Long
```

O Service resolve cada UUID dentro do Tenant atual.

---

# 7. UpdateDTO

Somente campos editáveis.

Exemplo:

```java
public record UnidadeUpdateDTO(
    @NotBlank String nome,
    String telefone,
    @Email String email
) {}
```

Se `codigo` for imutável por regra, não entra.

Se o recurso usa optimistic locking e o contrato escolhido exigir versão do cliente, isso deve ser definido explicitamente na etapa da feature. Não adicionar `version` a todos os DTOs sem decisão.

---

# 8. ResponseDTO

Response expõe `publicId`, nunca PK interna.

Exemplo:

```java
public record UnidadeResponseDTO(
    UUID publicId,
    String nome,
    String codigo,
    boolean ativo
) {}
```

Não retornar Tenant inteiro dentro de Unidade sem necessidade.

Não expor:

- `Long id`;
- `tenant.id`;
- `storageKey` interno;
- password hash;
- segredo/token;
- Entity JPA.

---

# 9. SummaryDTO

Útil para compor respostas sem serializar grafos JPA.

Exemplo:

```java
public record PacienteSummaryDTO(
    UUID publicId,
    String nomeCompleto,
    String numeroProntuario
) {}
```

Agendamento pode retornar summaries como:

```text
PacienteSummaryDTO
ProfissionalSummaryDTO
ProcedimentoSummaryDTO
UnidadeSummaryDTO
```

Cada summary usa `publicId` quando o recurso precisar ser referenciável externamente.

---

# 10. Repository por Tenant

Padrão na fronteira externa:

```java
Optional<Paciente> findByPublicIdAndTenantId(UUID publicId, Long tenantId);

Page<Paciente> findAllByTenantId(Long tenantId, Pageable pageable);

boolean existsByNumeroProntuarioAndTenantId(
    String numeroProntuario,
    Long tenantId
);
```

O `tenantId Long` usado nessa query vem do contexto resolvido pelo servidor.

Ele não vem do request.

---

# 11. Não usar `findById()` a partir de entrada pública

Errado:

```java
repository.findById(dto.pacienteId())
```

ou:

```java
repository.findById(pathVariableLong)
```

Correto na fronteira HTTP:

```java
repository.findByPublicIdAndTenantId(publicId, tenantId)
```

Depois que a Entity já foi resolvida/autorizada, o backend pode usar:

```java
entity.getId()
```

para FKs, locks, queries internas e otimizações.

`findById(Long)` pode existir para fluxos internos controlados, mas não deve reintroduzir PK sequencial como contrato externo.

---

# 12. Entidades globais

Nem toda Entity é tenant-scoped.

Exemplo principal:

```text
Usuario
```

Mesmo assim, sua identificação externa continua usando:

```text
UUID publicId
```

Busca administrativa pública deve preferir:

```text
findByPublicId(...)
```

E-mail/login pode existir como chave de autenticação conforme regra própria, sem transformar `Long id` em contrato.

---

# 13. Listagens

Sempre perguntar:

> esta tabela pode crescer?

Se sim, usar paginação desde o início.

Certamente paginar:

- pacientes;
- profissionais;
- agendamentos;
- atendimentos;
- audit logs;
- notificações;
- orientações do paciente quando a etapa futura entrar e o volume justificar.

Pode listar sem paginação inicialmente:

- enums;
- pequenas listas de especialidades do Tenant;
- unidades quando o volume esperado for baixo e a decisão estiver explícita.

Contrato recomendado: `PageResponseDTO`, conforme `17_PADROES_HTTP_ERROS_PAGINACAO.md`.

---

# 14. Repository não retorna DTO por padrão

Começar simples:

```text
Repository → Entity
Service → mapper simples → ResponseDTO
```

Projection/DTO query entra quando houver necessidade real de performance/consulta específica.

Não otimizar todos os CRUDs antecipadamente.

---

# 15. Query de conflito de agenda

Conceitualmente existe conflito quando há agendamento do mesmo profissional, em status que ocupa agenda, com:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

O intervalo adotado é:

```text
[inicio, fim)
```

Portanto:

```text
14:00–14:30
14:30–15:00
```

não é overlap.

Status cancelados não bloqueiam horário.

A query funcional **não é suficiente sozinha** contra corrida concorrente. A criação/reagendamento deve seguir `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 16. Locks

Quando uma query precisar serializar uma disputa de agenda:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("... p.publicId = :publicId and p.tenant.id = :tenantId")
Optional<ProfissionalSaude> findForUpdateByPublicIdAndTenantId(
    UUID publicId,
    Long tenantId
);
```

O lock acontece dentro da transação de Service.

Não criar lock em Controller.

---

# 17. Campos monetários

Java/DTO:

```text
BigDecimal
```

Nunca:

```text
double
float
```

No PostgreSQL, revisar precisão/escala na ETAPA 11.

---

# 18. Strings clínicas

Não limitar conteúdo clínico a `varchar(255)` por padrão.

Exemplos:

```text
EvolucaoClinica.conteudo → TEXT
Prescricao.observacoes → TEXT
ItemPrescricao.instrucoes → TEXT
```

Definir limites funcionais/segurança quando o conteúdo vier de usuário, sem truncar silenciosamente informação clínica.

---

# 19. Datas e Clock

Tipos:

```text
LocalDate       → data sem horário
LocalTime       → horário semanal
Instant         → instante absoluto
OffsetDateTime  → contrato/evento com offset quando adequado
```

Services temporais recebem `Clock`.

Evitar:

```java
LocalDateTime.now()
Instant.now()
```

espalhados pelo domínio.

---

# 20. Soft delete / status

Não adicionar `deletedAt` ou `ativo` automaticamente em todas as Entities.

Quando histórico precisa ser preservado, preferir a regra de status/inativação definida no domínio.

Nunca fazer hard delete destrutivo de:

- prontuário;
- evolução clínica;
- atendimento concluído;
- prescrição emitida;
- EventoAgendamento;
- AuditLog.

---

# 21. OrientacaoPaciente — somente pós-MVP

Quando a ETAPA 15 chegar, `OrientacaoPaciente` também seguirá:

```text
Long id interno
UUID publicId externo
Tenant obrigatório
Paciente e Profissional tenant-scoped
```

Response não expõe:

```text
storageKey
caminho físico
Long id
```

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

Não implementar essa Entity durante o Core.

---

# 22. Checklist de Entity

Antes de concluir:

- [ ] `id Long` é interno?
- [ ] `publicId UUID` existe e é imutável?
- [ ] tabela possui nome claro?
- [ ] nullability coerente?
- [ ] tenant está presente se necessário?
- [ ] unique lógico definido?
- [ ] enum é STRING?
- [ ] dinheiro é BigDecimal?
- [ ] datas usam tipo adequado?
- [ ] relacionamentos são LAZY por padrão?
- [ ] cascade foi justificado?
- [ ] `@Version` foi avaliado?
- [ ] Entity não é retornada diretamente como JSON?

---

# 23. Checklist de DTO/API

- [ ] CreateDTO não aceita `id`?
- [ ] CreateDTO não aceita `publicId` escolhido pelo cliente?
- [ ] Request não aceita `tenantId` livre?
- [ ] relações usam `...PublicId`?
- [ ] ResponseDTO não expõe `Long id`?
- [ ] ResponseDTO não expõe segredo/storageKey?
- [ ] paths públicos usam UUID?

---

# 24. Checklist de Repository

- [ ] busca pública tenant-scoped usa `publicId + tenantId interno`?
- [ ] listagens tenant-scoped filtram tenant?
- [ ] não existe `findById(idDoCliente)` no fluxo público?
- [ ] paginação existe onde o volume cresce?
- [ ] query de overlap usa a fórmula correta?
- [ ] lock está definido apenas quando necessário?
- [ ] SQL nativo foi evitado quando JPA/JPQL resolve?

---

# Regra final

```text
HTTP conhece publicId
Service resolve autorização/tenant
Repository consulta Entity
Banco usa Long para PK/FK
```

Não inverter essas fronteiras.
