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

```java
@PostMapping
public Paciente salvar(@RequestBody Paciente paciente)
```

Correto:

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
        .findByPublicIdAndTenantId(dto.pacientePublicId(), tenantId)
        .orElseThrow(...);

    ProfissionalSaude profissional = profissionalRepository
        .findForUpdateByPublicIdAndTenantId(dto.profissionalPublicId(), tenantId)
        .orElseThrow(...);

    Unidade unidade = ...;
    Procedimento procedimento = ...;

    validarRelacionamentos(...);
    validarDisponibilidade(...);
    validarConflito(...);

    Agendamento agendamento = ...;
    agendamentoRepository.save(agendamento);
    registrarEvento(...);

    return toResponse(agendamento);
}
```

---

# 6. Repository simples

Preferir derived query/JPQL legível.

Na fronteira pública:

```text
findByPublicIdAndTenantId
findAllByTenantId
existsByCodigoAndTenantId
```

`findById` pode existir para uso estritamente interno, depois que o recurso já foi resolvido/autorizado.

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

DTOs relacionais recebem UUIDs públicos:

```text
pacientePublicId
profissionalPublicId
unidadePublicId
procedimentoPublicId
```

Nunca receber PK `Long id` do cliente.

---

# 8. Actions em vez de PUT genérico

```http
PATCH /agendamentos/{publicId}/confirmar
PATCH /agendamentos/{publicId}/check-in
PATCH /agendamentos/{publicId}/cancelar
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
21_PUBLIC_ID_E_CONCORRENCIA.md
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
- ID `Long` exposto ao cliente;
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

---

# 15. Public ID

Toda Entity persistida nasce com:

```text
Long id
UUID publicId
```

`id` é PK/FK interna.

`publicId` é usado por Controller, DTO, URL, integrações e apps.

Exemplo:

```java
@GetMapping("/{publicId}")
public PacienteResponseDTO buscar(@PathVariable UUID publicId) {
    return service.buscarPorPublicId(publicId);
}
```

O Service combina:

```text
publicId + tenantId interno do contexto
```

Nunca permita que CreateDTO escolha o `publicId`.

---

# 16. Concorrência

Nunca confundir:

```text
"o slot apareceu disponível"
```

com:

```text
"o slot está garantido"
```

A garantia ocorre apenas dentro da transação de criação/reagendamento.

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
↓
commit
```

Dois pacientes disputando o mesmo horário:

```text
1 sucesso
1 AGENDAMENTO_CONFLITO
```

No PostgreSQL, validar com Testcontainers e duas transações reais. Avaliar também exclusion constraint como segunda barreira.
