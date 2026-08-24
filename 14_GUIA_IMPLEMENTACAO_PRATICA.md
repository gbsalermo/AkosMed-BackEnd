# 14 — Guia de Implementação Prática

# 1. Trabalhar verticalmente

Correto:

```text
Tenant
├── Entity
├── DTOs
├── Repository
├── Service
├── Controller
├── Tests
└── revisão
```

Depois:

```text
Unidade
```

Evitar criar todas as entities antes dos demais componentes.

---

# 2. Package por feature

```text
br.com.akosmed
├── shared
│   ├── config
│   ├── exception
│   ├── security
│   ├── tenant
│   └── storage
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

Dentro de uma feature:

```text
controller
dto
entity
repository
service
```

`mapper` só quando reduzir repetição de verdade.

---

# 3. Ordem de uma feature

Antes de programar:

1. abrir roadmap;
2. abrir relacionamentos;
3. definir regras;
4. definir DTOs;
5. só então codar.

Implementação:

```text
Entity/Enum
↓
Repository
↓
Service
↓
DTOs/mapper simples
↓
Controller
↓
Tests
```

Pode criar DTO antes do Service se isso ajudar a definir o contrato.

---

# 4. Não usar Entity como contrato HTTP

Errado:

```text
@PostMapping
public Paciente salvar(@RequestBody Paciente paciente)
```

Correto conceitualmente:

```text
PacienteCreateDTO
↓
PacienteService
↓
PacienteResponseDTO
```

---

# 5. Service é o centro da regra

Exemplo conceitual:

```java
@Transactional
public AgendamentoResponseDTO criar(AgendamentoCreateDTO dto) {
    Long tenantId = tenantContext.getTenantId();

    Paciente paciente = pacienteRepository
        .findByIdAndTenantId(dto.pacienteId(), tenantId)
        .orElseThrow(...);

    ProfissionalSaude profissional = ...
    Unidade unidade = ...
    Procedimento procedimento = ...

    validarRelacionamentos(...);
    validarDisponibilidade(...);
    validarConflito(...);

    Agendamento agendamento = ...
    agendamentoRepository.save(agendamento);

    registrarEvento(...);

    return toResponse(agendamento);
}
```

---

# 6. Repository simples

Preferir derived query/JPQL legível.

Exemplo:

```text
findByIdAndTenantId
findAllByTenantId
existsByCodigoAndTenantId
```

Evitar generic repository customizado.

---

# 7. DTOs

Padrão:

```text
CreateDTO
UpdateDTO
ResponseDTO
SummaryDTO
```

Não criar `UpdateDTO` se o recurso não puder ser editado livremente.

Exemplo:

Agendamento usa ações específicas para status.

---

# 8. Actions em vez de PUT genérico

```http
PATCH /agendamentos/{id}/confirmar
PATCH /agendamentos/{id}/check-in
PATCH /agendamentos/{id}/cancelar
```

Isso deixa regra explícita.

---

# 9. H2 até fechar regras

Não adicionar PostgreSQL antes da ETAPA 11.

Não adicionar Swagger antes da ETAPA 12.

Não montar coleção Postman oficial antes da ETAPA 13.

---

# 10. Branches

```text
feat/tenant
feat/unidade
feat/auth
feat/paciente
feat/agendamento
fix/agendamento-conflito
```

Pequenas.

---

# 11. Commits

```text
feat: implementa CRUD de tenant
test: valida isolamento de unidade
fix: impede overlap de agendamento
docs: atualiza continuidade da etapa 5
```

---

# 12. Antes de marcar concluído

Executar:

```text
mvn test
```

Depois usar:

```text
19_CHECKLIST_REVISAO_QUALIDADE.md
```

Se estiver usando IA externa, seguir:

```text
20_GUIA_USO_IA_PARA_REVISAO.md
```

---

# 13. O que evitar

- `CascadeType.ALL` automático;
- `FetchType.EAGER` para resolver LazyInitialization;
- Entity em Controller;
- Repository em Controller;
- Service gigantesco com vários domínios;
- enum como número no banco;
- `double` para dinheiro;
- setter público para tudo;
- `tenantId` confiado ao request;
- hard delete clínico;
- status alterável livremente;
- consultas sem paginação para listas grandes;
- catches genéricos escondendo erro.

---

# 14. Meta

Prioridades:

```text
correto
↓
seguro
↓
testável
↓
legível
↓
otimizado
```

Não inverter essa ordem.
