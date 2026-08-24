# 15 — Padrões de Entidades, DTOs e Repositories

Este documento serve como referência durante cada CRUD.

---

# 1. Padrão de Entity

Uma Entity deve:

- representar estado persistido;
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

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "tenant_id", nullable = false)
    private Tenant tenant;

    @Column(nullable = false, length = 120)
    private String nome;

    @Column(nullable = false, length = 40)
    private String codigo;

    @Column(nullable = false)
    private boolean ativo = true;
}
```

---

# 2. Campos de auditoria simples

Para entidades administrativas pode existir:

```text
createdAt
updatedAt
```

Não criar uma `BaseEntity` enorme no início.

Se repetição ficar realmente incômoda, criar uma base pequena depois.

---

# 3. Enum

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

# 4. CreateDTO

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
tenantId livre
createdAt
ativo arbitrário
```

Tenant vem do contexto.

---

# 5. UpdateDTO

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

---

# 6. ResponseDTO

Pode conter IDs e summaries.

Exemplo:

```java
public record UnidadeResponseDTO(
    Long id,
    String nome,
    String codigo,
    boolean ativo
) {}
```

Não retornar Tenant inteiro dentro de Unidade se não houver necessidade.

---

# 7. SummaryDTO

Útil para compor respostas.

Exemplo:

```java
public record PacienteSummaryDTO(
    Long id,
    String nomeCompleto,
    String numeroProntuario
) {}
```

Agendamento pode retornar:

```text
PacienteSummaryDTO
ProfissionalSummaryDTO
ProcedimentoSummaryDTO
UnidadeSummaryDTO
```

Evita ciclos e respostas gigantes.

---

# 8. Repository por tenant

Padrão:

```java
Optional<Paciente> findByIdAndTenantId(Long id, Long tenantId);
Page<Paciente> findAllByTenantId(Long tenantId, Pageable pageable);
boolean existsByNumeroProntuarioAndTenantId(String numero, Long tenantId);
```

---

# 9. Não usar `findById()` em domínio tenant-scoped

Evitar:

```java
repository.findById(id)
```

em Services tenant-scoped.

Preferir:

```text
findByIdAndTenantId
```

Isso reduz chance de IDOR/cross-tenant.

---

# 10. Listagens

Sempre perguntar:

> essa tabela pode crescer?

Se sim, usar paginação desde o começo.

Certamente paginar:

- pacientes;
- profissionais;
- agendamentos;
- atendimentos;
- audit logs;
- notificações.

Pode listar sem paginação inicialmente:

- enums;
- pequenas listas de especialidades do tenant;
- unidades se o volume for muito baixo.

---

# 11. Repository não retorna DTO por padrão

Começar simples:

```text
Repository → Entity
Service → ResponseDTO
```

Projection/DTO query só entra quando houver problema real de performance.

---

# 12. Query de conflito de agenda

Conceitualmente:

```text
existe agendamento do profissional
com status que ocupa agenda
e início < novoFim
e fim > novoInicio
```

Overlap:

```text
existing.start < newEnd
AND
existing.end > newStart
```

Status cancelado não bloqueia horário.

---

# 13. Campos monetários

Java:

```text
BigDecimal
```

DTO:

```text
BigDecimal
```

Nunca:

```text
double
float
```

---

# 14. Strings clínicas

Não limitar conteúdo clínico a `varchar(255)` por padrão.

Exemplo:

```text
EvolucaoClinica.conteudo → TEXT
Prescricao.observacoes → TEXT
ItemPrescricao.instrucoes → TEXT
```

---

# 15. Checklist Entity

Antes de concluir:

- [ ] tabela tem nome claro;
- [ ] nullability coerente;
- [ ] tenant está presente se necessário;
- [ ] unique lógico definido;
- [ ] enum STRING;
- [ ] dinheiro BigDecimal;
- [ ] datas com tipo adequado;
- [ ] relacionamentos LAZY por padrão;
- [ ] cascade revisado;
- [ ] não há Entity retornando JSON diretamente.
