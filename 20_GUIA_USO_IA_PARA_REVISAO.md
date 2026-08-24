# 20 — Guia para Usar IA como Segunda Revisão

A IA externa pode ajudar bastante, mas deve revisar contra regras explícitas do AkosMed.

Nunca pedir apenas:

> “esse código está bom?”

Forneça contexto e critérios.

---

# 1. Arquivos mínimos por revisão

## CRUD simples

Enviar:

```text
Entity
CreateDTO
UpdateDTO
ResponseDTO
Repository
Service
Controller
Exceptions relacionadas
Testes
```

E informar:

```text
tenant-scoped ou global
regras do domínio
relacionamentos esperados
```

---

# 2. Prompt-base de revisão de CRUD

```text
Revise esta implementação do AkosMed.

Contexto:
- Java + Spring Boot + JPA
- desenvolvimento atual em H2
- PostgreSQL será migrado posteriormente
- arquitetura multi-tenant
- tenantId nunca deve ser confiado ao request
- entidades tenant-scoped devem ser buscadas com tenantId
- Controller não pode conter regra de negócio
- Service concentra validações e @Transactional
- DTOs não podem expor Entity diretamente
- Fetch LAZY por padrão
- não usar CascadeType.ALL sem justificativa

Verifique:
1. relacionamentos JPA
2. DTOs
3. repository
4. regras do service
5. isolamento multi-tenant
6. status HTTP
7. riscos de N+1
8. exceptions
9. testes faltantes
10. segurança
11. aderência ao roadmap

Não reescreva tudo automaticamente.
Primeiro liste problemas por severidade:
CRÍTICO / IMPORTANTE / MELHORIA.
```

---

# 3. Revisão de Entity

Perguntar:

```text
- FK está no lado correto?
- precisa ser bidirecional?
- cascade é seguro?
- fetch está correto?
- constraint única está faltando?
- nullable está coerente?
- money/date types estão adequados?
- há risco de delete em cascata?
```

---

# 4. Revisão de Service

Perguntar:

```text
- tenant é validado?
- relacionamentos pertencem ao mesmo tenant?
- @Transactional está correto?
- transição de status está protegida?
- exception é adequada?
- método faz responsabilidade demais?
- há race condition?
```

---

# 5. Revisão de agendamento

Sempre fornecer também:

```text
DisponibilidadeProfissional
BloqueioAgenda
ProfissionalProcedimento
Agendamento
```

Pedir análise de:

- overlap;
- duração;
- status que ocupam agenda;
- reagendamento;
- concorrência;
- unidade;
- tenant.

Lembrar:

concorrência definitiva só é validada em PostgreSQL/Testcontainers.

---

# 6. Revisão de prontuário

Pedir explicitamente:

```text
verifique risco de alteração/destruição de histórico clínico
```

Checar:

- hard delete;
- cascade remove;
- update destrutivo;
- retificação;
- autorização.

---

# 7. Revisão de prescrição

Checar:

- item editável apenas em rascunho;
- emissão imutabiliza;
- cancelamento preserva histórico;
- paciente/profissional/atendimento mesmo tenant;
- resposta do paciente não expõe além do permitido.

---

# 8. Revisão de testes

Pedir:

```text
Liste casos faltantes sem gerar testes ainda.
Separe:
- happy path
- validation
- not found
- conflict
- tenant isolation
- status transition
- security
```

Depois gerar apenas os testes aprovados.

---

# 9. Não aceitar cegamente sugestões de IA

Desconfiar se a IA sugerir:

- microsserviços cedo;
- Kafka para CRUD;
- Redis sem problema real;
- `CascadeType.ALL` em tudo;
- `FetchType.EAGER`;
- GenericRepository;
- retornar Entity;
- Lombok `@Data` em Entity sem análise;
- MapStruct obrigatório;
- 10 interfaces extras sem necessidade;
- arquitetura hexagonal completa no MVP.

Comparar sempre com os documentos oficiais.

---

# 10. Auditoria de etapa completa

Ao finalizar uma etapa, enviar à IA:

```text
CONTINUIDADE
trecho da etapa do ROADMAP
lista de arquivos criados
principais Entities
Services
Controllers
Testes
```

Pedir:

```text
Compare a implementação com o roadmap e identifique:
- item faltante
- inconsistência
- risco multi-tenant
- relação JPA problemática
- teste ausente
- endpoint divergente
- dívida técnica que deve impedir avanço
```

---

# 11. Critério de confiança

IA é segunda revisão.

A fonte de verdade continua sendo:

```text
regras documentadas
+
testes executados
+
comportamento real
```

Se IA e teste discordarem, investigar.

Não “corrigir” código só porque a IA afirmou algo sem evidência.
