# 05 — Banco de Dados

# Estratégia oficial

O AkosMed não começará em PostgreSQL.

A persistência será trabalhada em duas fases.

---

# FASE A — Desenvolvimento com H2

Etapas:

```text
0 até 10
```

Objetivo:

validar:

- domínio;
- JPA;
- relacionamentos;
- regras;
- autenticação;
- multi-tenancy;
- agenda;
- prontuário;
- prescrição.

Configuração sugerida:

## dev

```text
H2
ddl-auto=update
```

## test

```text
H2 memória
ddl-auto=create-drop
```

Sem Flyway inicialmente.

---

# Cuidados mesmo no H2

Não escrever código dependente de comportamento específico do H2.

Usar:

- JPA;
- JPQL;
- derived queries;
- SQL nativo somente se realmente necessário.

Evitar deixar para PostgreSQL problemas de:

- ownership;
- FK;
- nullability;
- unique lógico;
- tenant isolation.

---

# FASE B — PostgreSQL definitivo

Entrada:

```text
ETAPA 11
```

Adicionar:

- PostgreSQL Driver;
- Flyway;
- Testcontainers PostgreSQL.

Trocar produção para:

```text
ddl-auto=validate
```

---

# Migrations

Flyway só será introduzido quando a modelagem estiver estabilizada.

Estrutura sugerida:

```text
V1__create_tenant_unidade.sql
V2__create_identity.sql
V3__create_professional_patient.sql
V4__create_scheduling.sql
V5__create_clinical.sql
V6__create_prescription_storage.sql
V7__create_notification_audit.sql
```

Não é obrigatório uma migration por Entity.

Preferir migrations por bloco lógico.

---

# Tipos PostgreSQL

## IDs

```text
BIGINT
```

## Dinheiro

```text
NUMERIC(15,2)
```

Java:

```text
BigDecimal
```

Nunca `double/float` para valores monetários.

## Datas

### Data sem horário

```text
DATE
LocalDate
```

### Horário sem data

```text
TIME
LocalTime
```

### Instante

Preferir:

```text
TIMESTAMP WITH TIME ZONE
OffsetDateTime/Instant
```

ou padronizar explicitamente com timezone do sistema.

---

# Timezone do Tenant

`Tenant.timezone` deve existir.

Exemplo:

```text
America/Bahia
```

Agenda é exibida no timezone da organização.

Definir uma estratégia única antes de PostgreSQL.

---

# Constraints importantes

Exemplos:

```text
tenant.slug UNIQUE

unidade UNIQUE(tenant_id, codigo)

pessoa UNIQUE(tenant_id, cpf) quando CPF existir

usuario.email_login UNIQUE

usuario_tenant UNIQUE(usuario_id, tenant_id)

usuario_unidade UNIQUE(usuario_tenant_id, unidade_id)

especialidade UNIQUE(tenant_id, codigo)

procedimento UNIQUE(tenant_id, codigo)

paciente UNIQUE(tenant_id, numero_prontuario)

prontuario UNIQUE(tenant_id, paciente_id)
```

---

# Índices

Criar com base em consultas reais.

Prioridades iniciais:

```text
agendamento(tenant_id, inicio)
agendamento(tenant_id, profissional_id, inicio)
agendamento(tenant_id, paciente_id, inicio)
agendamento(tenant_id, unidade_id, inicio)

atendimento(tenant_id, prontuario_id, inicio)

audit_log(tenant_id, timestamp)
```

Não criar dezenas de índices preventivamente.

---

# Double booking

No H2:

validar regra funcional.

No PostgreSQL:

validar concorrência real.

Estratégia inicial:

```text
@Transactional
↓
lock no profissional
↓
consulta por overlap
↓
salvar
```

Se Testcontainers mostrar falha, ajustar nessa etapa.

---

# JSONB

Não usar no Core sem necessidade.

Possíveis usos futuros:

- formulários clínicos customizáveis;
- metadata de auditoria;
- configurações do tenant.

O modelo principal permanece relacional.

---

# Arquivos

Não armazenar anexos grandes como BLOB no banco.

Guardar metadata:

```text
id
tenantId
prontuarioId
atendimentoId
nomeOriginal
mimeType
tamanho
hash
storageKey
```

Arquivo real no StorageService.

---

# Checklist antes de migrar para PostgreSQL

- [ ] nullability revisada;
- [ ] `BigDecimal` em dinheiro;
- [ ] strings com tamanho coerente;
- [ ] `TEXT` onde conteúdo puder crescer;
- [ ] uniques revisados;
- [ ] índices baseados em consultas;
- [ ] FKs revisadas;
- [ ] cascade revisado;
- [ ] orphanRemoval revisado;
- [ ] nomes de tabelas/colunas revisados;
- [ ] nenhum dado clínico importante depende de H2-specific behavior.
