# 22 — Concorrência Geral, Idempotência, Clock e Operação

Este documento complementa `21_PUBLIC_ID_E_CONCORRENCIA.md` e define guardrails técnicos do AkosMed.

---

# 1. Três problemas diferentes

```text
PESSIMISTIC_WRITE
→ disputa por recurso escasso
→ dois pacientes disputando o mesmo horário

@Version / optimistic locking
→ duas alterações concorrentes do mesmo registro
→ duas secretárias editando o mesmo paciente

Idempotency-Key
→ repetição da mesma requisição
→ retry após timeout/rede instável
```

Não tratar os três como o mesmo problema.

---

# 2. Optimistic Lock com @Version

Usar `@Version` seletivamente em entidades administrativas mutáveis onde lost update seja relevante.

```java
@Version
private Long version;
```

Candidatas iniciais:

```text
Tenant
Unidade
Pessoa
Usuario
UsuarioTenant
Especialidade
Procedimento
ProfissionalSaude
Paciente
DisponibilidadeProfissional
BloqueioAgenda
Agendamento
Prescricao enquanto RASCUNHO
```

Não aplicar por padrão em históricos append-only:

```text
EventoAgendamento
EvolucaoClinica
AuditLog
```

Conflito esperado:

```text
409 Conflict
RESOURCE_VERSION_CONFLICT
```

O frontend/app deve recarregar o recurso em vez de sobrescrever silenciosamente.

---

# 3. Idempotência

Algumas operações críticas podem ser repetidas por timeout, retry automático, duplo clique ou reconexão mobile.

Preparar a arquitetura para:

```http
Idempotency-Key: <uuid>
```

Candidatas:

```text
POST /agendamentos
emissão de prescrição
pagamentos futuros
documentos futuros
```

Não criar infraestrutura genérica na ETAPA 0.

Comportamento futuro:

```text
mesma key + mesmo payload
→ mesmo resultado
→ sem novo efeito

mesma key + payload diferente
→ 409 IDEMPOTENCY_KEY_REUSED
```

Entidade técnica futura possível:

```text
IdempotencyRecord
id
publicId
tenantId
key
operation
requestHash
responseStatus
responseBodyJson
createdAt
expiresAt
```

Só criar quando uma operação realmente precisar.

---

# 4. Clock centralizado

Evitar `Instant.now()`, `LocalDate.now()` e `LocalDateTime.now()` espalhados nos Services.

Criar bean:

```java
@Bean
Clock clock() {
    return Clock.systemUTC();
}
```

Services temporais recebem `Clock`:

```java
Instant agora = Instant.now(clock);
```

Benefícios:

- testes determinísticos;
- regras de check-in/falta;
- validade de token;
- prescrições;
- notificações;
- auditoria.

Timezone:

```text
persistência de instantes → UTC
exibição/cálculo local → Tenant.timezone
```

Nos testes usar `Clock.fixed(...)`.

---

# 5. Correlation ID

Toda request deve ter identificador de rastreio.

```http
X-Correlation-Id
```

Se o cliente não enviar, o backend gera UUID.

Fluxo:

```text
HTTP request
↓
CorrelationIdFilter
↓
MDC/log
↓
Service
↓
ApiErrorDTO
↓
AuditLog quando aplicável
```

Logs relevantes devem incluir correlationId e apenas identificadores seguros.

Nunca logar:

- senha;
- JWT completo;
- refresh token;
- prontuário completo;
- evolução clínica;
- receita completa;
- documentos sensíveis.

---

# 6. Observabilidade mínima

Não precisa de stack complexa no MVP.

Preparar para:

```text
logs estruturados
correlationId
health checks
métricas básicas
```

Spring Boot Actuator pode entrar próximo do fechamento/produção.

Health checks futuros:

```text
liveness
readiness
database
storage
```

Não expor detalhes sensíveis publicamente.

---

# 7. Backup e restore

Backup sem teste de restauração não é recuperação confiável.

Antes de produção:

- frequência definida;
- retenção;
- criptografia;
- cópia em local separado;
- procedimento de restore;
- teste real de restore;
- RPO/RTO registrados, mesmo inicialmente simples.

---

# 8. Secrets

Nunca versionar:

```text
DB_PASSWORD
JWT_SECRET
S3_SECRET
SMTP_PASSWORD
API_KEYS
```

Usar variáveis de ambiente/secret manager.

Pode existir `.env.example`, sem valores reais.

---

# 9. Uploads

Definir no backend:

```text
max file size
tipos MIME permitidos
extensões permitidas
hash
categoria
```

O nome original nunca deve virar diretamente `storageKey`.

---

# 10. Matriz de autorização por perfil

Antes do fechamento do backend, consolidar recurso × perfil × ação.

Base inicial:

| Recurso | ADMIN | SECRETARIA | PROFISSIONAL | PACIENTE | AUDITOR |
|---|---:|---:|---:|---:|---:|
| Tenant/Unidade | ✓ | limitado | - | - | leitura |
| Paciente administrativo | ✓ | ✓ | limitado | próprio | leitura autorizada |
| Agenda | ✓ | ✓ | própria | própria | leitura |
| Prontuário | conforme regra | não por padrão | autorizado | próprio liberado | auditoria |
| Prescrição | - | - | autorizado | própria | leitura autorizada |
| AuditLog | ✓ | - | - | - | ✓ |

A matriz final deve ser refinada e testada na ETAPA 9.

---

# 11. Onde cada guardrail entra

## ETAPA 0

```text
Clock bean
CorrelationIdFilter básico
padrão de exceptions
```

## ETAPAS 1–8

```text
@Version seletivo
publicId
tenant isolation
regras específicas de concorrência
```

## ETAPA 5

```text
PESSIMISTIC_WRITE para agendamento
teste de double booking
```

## ETAPA 7

```text
avaliar idempotência da emissão de prescrição
```

## ETAPA 9

```text
matriz de autorização
logs
secrets
correlationId completo
optimistic locking revisado
```

## ETAPA 11

```text
PostgreSQL
locks reais
constraints
Testcontainers
```

## ETAPA 14 / produção

```text
Actuator
backup/restore
limites definitivos de upload
observabilidade mínima
```

---

# 12. Checklist

Antes de fechar uma feature mutável:

- [ ] precisa de `@Version`?
- [ ] update concorrente gera 409?
- [ ] publicId continua imutável?
- [ ] regra temporal usa `Clock`?
- [ ] teste usa `Clock.fixed` quando necessário?
- [ ] request tem correlationId?
- [ ] logs evitam dados sensíveis?
- [ ] operação pode sofrer retry?
- [ ] precisa de `Idempotency-Key`?
- [ ] upload tem limites?
- [ ] perfil pode executar essa ação?

---

# Regra final

```text
publicId
→ protege o contrato externo

tenant isolation + autorização
→ protegem acesso

pessimistic lock
→ protege disputa de recurso

@Version
→ protege contra lost update

idempotência
→ protege contra retry duplicado

Clock
→ protege a testabilidade temporal

correlationId + observabilidade
→ protegem a rastreabilidade operacional
```
