# 21 — Public ID e Concorrência

Este documento define duas convenções obrigatórias do AkosMed.

---

# PARTE A — PUBLIC ID

## 1. Padrão

Toda Entity persistida:

```text
id       Long
publicId UUID
```

`id` é somente interno.

`publicId` é o identificador externo.

---

## 2. BaseEntity mínima

É permitido criar uma `BaseEntity` somente para estes campos:

```java
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, updatable = false)
    private UUID publicId;

    @PrePersist
    protected void ensurePublicId() {
        if (publicId == null) {
            publicId = UUID.randomUUID();
        }
    }
}
```

Não adicionar automaticamente:

- tenant;
- status;
- timestamps;
- soft delete;
- version.

---

## 3. Banco

Interno:

```text
id BIGINT PK
FKs BIGINT
```

Externo:

```text
public_id UUID NOT NULL UNIQUE
```

Não usar publicId como FK no MVP.

---

## 4. DTOs e URLs

Correto:

```http
GET /api/v1/pacientes/{publicId}
```

```text
pacientePublicId
profissionalPublicId
unidadePublicId
procedimentoPublicId
```

Errado:

```text
pacienteId Long vindo do cliente
```

---

## 5. Repository

Tenant-scoped:

```text
findByPublicIdAndTenantId(publicId, tenantIdInterno)
```

Depois que o recurso é resolvido/autorizado, o backend pode usar `entity.getId()` internamente.

---

## 6. Segurança

UUID não substitui autorização.

Ainda é obrigatório:

- tenant isolation;
- perfil;
- unidade;
- ownership.

Um UUID de outro tenant deve resultar em 404 conforme a política adotada.

---

## 7. JWT

Preferir claims públicos:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfil
```

O filtro pode resolver os IDs internos e guardá-los apenas no contexto do servidor.

---

## 8. Testes publicId

Obrigatórios:

- publicId gerado;
- publicId único;
- publicId não muda em update;
- ResponseDTO não expõe Long id;
- request usa publicId;
- UUID inexistente → 404;
- UUID de outro tenant → 404.

---

# PARTE B — CONCORRÊNCIA

## 9. Problema

Duas requisições podem observar o mesmo horário como livre.

```text
14:00 livre

Paciente A lê → livre
Paciente B lê → livre

A salva
B salva
```

Uma consulta `exists` isolada não evita essa corrida.

---

## 10. Recurso disputado

Para o MVP, o recurso compartilhado é:

```text
ProfissionalSaude
```

Reservas do mesmo profissional são serializadas por lock nessa linha.

---

## 11. Lock

Repository conceitual:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from ProfissionalSaude p where p.publicId = :publicId and p.tenant.id = :tenantId")
Optional<ProfissionalSaude> findForUpdateByPublicIdAndTenantId(
    UUID publicId,
    Long tenantId
);
```

---

## 12. Fluxo de criação

```text
AgendamentoService.criar(dto)
@Transactional
│
├── resolve tenant
├── lock profissional
├── resolve paciente
├── resolve unidade
├── resolve procedimento
├── valida vínculos
├── calcula fim
├── valida disponibilidade
├── valida bloqueios
├── valida overlap
├── salva agendamento
└── registra evento
```

Lock é liberado no commit/rollback.

---

## 13. Overlap

Conflito:

```text
existing.inicio < new.fim
AND
existing.fim > new.inicio
```

Intervalo:

```text
[inicio, fim)
```

Logo:

```text
14:00–14:30
14:30–15:00
```

é permitido.

---

## 14. Status que ocupam agenda

Inicialmente:

```text
SOLICITADO
CONFIRMADO
CHECK_IN
EM_ATENDIMENTO
```

Não ocupam:

```text
CANCELADO_PACIENTE
CANCELADO_CLINICA
FALTOU
```

Centralizar esse conjunto para não duplicar regra.

---

## 15. Reagendamento

Reagendamento usa a mesma proteção:

```text
@Transactional
→ lock profissional de destino
→ validar slot
→ overlap excluindo o próprio agendamento
→ atualizar
→ evento
```

Se no futuro for necessário travar dois profissionais, adquirir locks em ordem determinística para reduzir risco de deadlock.

---

## 16. H2

Durante o Core:

- implementar o lock JPA;
- testar overlap;
- criar teste concorrente básico.

A garantia final será revalidada em PostgreSQL.

---

## 17. PostgreSQL

Na ETAPA 11:

- Testcontainers;
- duas transações reais;
- validar `PESSIMISTIC_WRITE`;
- adicionar exclusion constraint.

Defesa:

```text
Service validation
+
DB lock
+
DB exclusion constraint
```

---

## 18. Exclusion constraint candidata

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE agendamentos
ADD CONSTRAINT ex_agendamento_profissional_periodo
EXCLUDE USING gist (
    profissional_id WITH =,
    tstzrange(inicio, fim, '[)') WITH &&
)
WHERE (
    status IN ('SOLICITADO', 'CONFIRMADO', 'CHECK_IN', 'EM_ATENDIMENTO')
);
```

A migration final deve refletir os nomes/tipos reais.

---

## 19. HTTP

Em disputa:

```text
1 request → 201 Created
1 request → 409 Conflict
```

Erro:

```text
code = AGENDAMENTO_CONFLITO
```

Conflito esperado nunca deve virar 500.

---

## 20. Matriz de testes

### Mesmo profissional + mesmo slot

```text
1 sucesso
1 conflito
```

### Mesmo profissional + horários diferentes

```text
2 sucessos
```

### Profissionais diferentes + mesmo slot

```text
2 sucessos
```

### Reagendamentos concorrentes para mesmo slot

```text
1 sucesso
1 conflito
```

### Limite exato

```text
14:00–14:30
14:30–15:00
```

permitido.

### Overlap parcial

```text
14:00–14:30
14:15–14:45
```

conflito.

---

## 21. Escalabilidade futura

Lock por profissional é simples e adequado para o MVP.

Se o volume crescer, avaliar:

- advisory locks;
- reservation holds;
- idempotency keys;
- lock mais granular.

Não antecipar agora.

---

# Regra final

```text
publicId → protege o contrato externo
tenant isolation → protege acesso
lock + constraint → protege concorrência
```

São responsabilidades diferentes.
