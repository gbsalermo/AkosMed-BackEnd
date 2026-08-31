# 17 — Padrões HTTP, Erros e Paginação

> Este documento define o contrato REST planejado do AkosMed.  
> **Regra central:** recursos públicos usam `UUID publicId`; PK `Long id` nunca aparece em URL ou DTO HTTP.

---

# 1. Prefixo

```text
/api/v1
```

---

# 2. Identificadores no contrato

Correto:

```http
GET /api/v1/pacientes/{publicId}
GET /api/v1/profissionais/{publicId}
GET /api/v1/agendamentos/{publicId}
```

Correto em relacionamentos:

```text
pacientePublicId
profissionalPublicId
unidadePublicId
procedimentoPublicId
```

Errado:

```text
@PathVariable Long id
pacienteId Long
profissionalId Long
tenantId Long vindo do cliente
```

O Service resolve o `publicId` para a Entity dentro do Tenant atual e pode usar o `Long id` somente internamente depois da autorização.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 3. CRUD administrativo

## Criar

```http
POST /api/v1/pacientes
```

Resposta:

```text
201 Created
```

Body:

```text
PacienteResponseDTO
```

O `Location`, se usado, aponta para o recurso via `publicId`.

---

## Buscar

```http
GET /api/v1/pacientes/{publicId}
```

Respostas:

```text
200 OK
404 Not Found
```

UUID inexistente ou pertencente a outro Tenant deve resultar em 404 conforme a política de isolamento.

---

## Listar

```http
GET /api/v1/pacientes?page=0&size=20
```

---

## Atualizar cadastro editável

```http
PUT /api/v1/pacientes/{publicId}
```

Usar `PATCH` apenas quando update parcial for intencional ou quando a operação representar uma ação de domínio.

Para o MVP, preferir PUT em dados administrativos que possuem atualização integral bem definida.

---

# 4. Ações de domínio

Não usar update genérico para transições importantes.

Exemplos:

```http
PATCH /api/v1/agendamentos/{publicId}/confirmar
PATCH /api/v1/agendamentos/{publicId}/check-in
PATCH /api/v1/agendamentos/{publicId}/cancelar
POST  /api/v1/agendamentos/{publicId}/reagendar
```

A ação deve validar transição no Service.

---

# 5. Vínculos e DELETE

Usar DELETE para vínculos/removíveis quando a remoção não destrói histórico crítico.

Exemplo:

```http
DELETE /api/v1/profissionais/{profissionalPublicId}/especialidades/{especialidadePublicId}
```

Evitar DELETE destrutivo em:

- prontuário;
- evolução clínica;
- atendimento concluído;
- prescrição emitida;
- EventoAgendamento;
- AuditLog.

Para recursos com histórico, usar inativação/cancelamento/retificação conforme regra do domínio.

---

# 6. Status HTTP

## 200 OK

Leitura ou ação que retorna body.

## 201 Created

Criação de recurso.

## 204 No Content

Ação concluída sem body quando essa escolha for consistente para o endpoint.

## 400 Bad Request

- JSON inválido;
- Bean Validation;
- parâmetro de formato inválido.

## 401 Unauthorized

- não autenticado;
- access token ausente/inválido/expirado.

## 403 Forbidden

Usuário autenticado, porém sem autorização para executar a ação.

## 404 Not Found

- recurso inexistente;
- recurso não visível no Tenant atual.

Para cross-tenant, preferir 404 para não revelar existência.

## 409 Conflict

Conflito de domínio/estado/concor­rência, por exemplo:

- código/slug duplicado;
- horário ocupado;
- transição inválida;
- recurso em estado incompatível;
- `RESOURCE_VERSION_CONFLICT`;
- `IDEMPOTENCY_KEY_REUSED` quando essa funcionalidade existir.

Conflito esperado não deve virar 500.

---

# 7. ApiErrorDTO

Padrão recomendado:

```json
{
  "timestamp": "2026-08-31T13:20:00Z",
  "status": 409,
  "code": "AGENDAMENTO_CONFLITO",
  "message": "O horário acabou de ser reservado por outro paciente.",
  "path": "/api/v1/agendamentos",
  "correlationId": "d462f38d-9b0f-40c8-85ca-30eaf3d19cd2"
}
```

O timestamp do erro representa um instante; persistência/logs devem seguir a estratégia UTC do projeto.

A frase pode mudar. O `code` deve permanecer estável.

---

# 8. Field errors

Exemplo:

```json
{
  "timestamp": "2026-08-31T13:20:00Z",
  "status": 400,
  "code": "VALIDATION_ERROR",
  "message": "Existem campos inválidos.",
  "path": "/api/v1/pacientes",
  "correlationId": "d462f38d-9b0f-40c8-85ca-30eaf3d19cd2",
  "fieldErrors": [
    {
      "field": "nome",
      "message": "não deve estar em branco"
    }
  ]
}
```

Nunca retornar stack trace ao cliente como resposta padrão de produção.

---

# 9. Códigos de erro

Criar códigos estáveis e orientados ao domínio.

Exemplos:

```text
RESOURCE_NOT_FOUND
VALIDATION_ERROR
TENANT_SUSPENSO
UNIDADE_INVALIDA
PACIENTE_INATIVO
PROFISSIONAL_INATIVO
AGENDAMENTO_CONFLITO
AGENDAMENTO_STATUS_INVALIDO
PRESCRICAO_NAO_EDITAVEL
RESOURCE_VERSION_CONFLICT
IDEMPOTENCY_KEY_REUSED
ACCESS_DENIED
```

Frontend/mobile deve reagir ao `code`, não depender de comparação da mensagem humana.

---

# 10. Paginação

Para listas grandes:

```text
page
size
sort
```

Defaults planejados:

```text
default size = 20
max size = 100
```

O backend deve limitar `size` para evitar consultas/respostas abusivas.

---

# 11. PageResponseDTO

Não acoplar o contrato HTTP diretamente ao `PageImpl` do Spring.

Exemplo:

```json
{
  "content": [],
  "page": 0,
  "size": 20,
  "totalElements": 120,
  "totalPages": 6,
  "first": true,
  "last": false
}
```

Implementar quando entrar a primeira listagem grande.

---

# 12. Filtros

Exemplo:

```http
GET /api/v1/agendamentos?dataInicio=...&dataFim=...&profissionalPublicId=...&status=CONFIRMADO
```

Quando necessário:

```text
unidadePublicId
pacientePublicId
procedimentoPublicId
```

Não usar nomes `...Id` no contrato para representar PK interna.

Não criar endpoint separado para cada combinação de filtro, nem um motor genérico complexo antes de haver casos reais.

---

# 13. Endpoints `/me`

Quando o recurso pertence ao próprio usuário autenticado, preferir resolver identidade no backend.

Paciente:

```http
GET /api/v1/me/perfil
GET /api/v1/me/consultas
GET /api/v1/me/prescricoes
GET /api/v1/me/notificacoes
```

Profissional:

```http
GET /api/v1/me/agenda/hoje
GET /api/v1/me/pacientes/aguardando
GET /api/v1/me/resumo-diario
```

Não exigir:

```text
pacientePublicId
profissionalPublicId
```

para consultar os próprios dados em `/me`.

---

# 14. Ordenação

Defaults úteis:

```text
Agenda       → inicio ASC
Pacientes    → nome ASC
AuditLog     → createdAt DESC
Notificacoes → createdAt DESC
```

Definir ordenação no Service/Repository e não depender da ordem acidental do banco.

Campos de `sort` aceitos externamente devem ser controlados/whitelisted quando necessário.

---

# 15. Data/hora na API

Usar ISO 8601.

Exemplo:

```text
2026-08-31T14:30:00-03:00
```

Para instante absoluto também é válido:

```text
2026-08-31T17:30:00Z
```

Evitar no contrato REST formatos locais como:

```text
31/08/2026 14:30
```

Regra de domínio:

```text
instantes persistidos → UTC
cálculo/exibição local → Tenant.timezone
```

---

# 16. Booleans

Preferir nomes claros:

```text
ativo
acessoTodasUnidades
usoContinuo
seNecessario
```

Evitar nomes ambíguos como `flag`, `statusBool`, `valor`.

---

# 17. Concorrência de agendamento

Duas requisições simultâneas para o mesmo profissional/período:

```text
request A → 201 Created
request B → 409 Conflict
```

Erro:

```json
{
  "status": 409,
  "code": "AGENDAMENTO_CONFLITO",
  "message": "O horário acabou de ser reservado por outro paciente.",
  "correlationId": "..."
}
```

O cliente deve atualizar os horários disponíveis depois do 409.

Disponibilidade exibida anteriormente não é garantia de reserva.

Referência: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 18. Optimistic locking

Quando `@Version` for usado e outra atualização vencer a corrida:

```text
409 Conflict
code = RESOURCE_VERSION_CONFLICT
```

O cliente deve recarregar o recurso antes de tentar sobrescrever dados mais novos.

O formato exato de transporte da versão deve ser decidido junto da feature; não inventar ETag ou campo DTO diferente em cada módulo sem padronização.

---

# 19. Idempotência

Quando uma operação passar a aceitar:

```http
Idempotency-Key: <uuid>
```

o comportamento deve seguir `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

Em particular:

```text
mesma key + mesmo payload
→ sem efeito duplicado

mesma key + payload diferente
→ 409 IDEMPOTENCY_KEY_REUSED
```

Não exigir esse header em todos os POSTs sem necessidade.

---

# 20. Upload/download

Uploads futuros de anexos/orientações devem:

- autorizar Tenant e ownership antes de acessar storage;
- limitar tamanho;
- validar MIME/extensão;
- não expor `storageKey` interno;
- evitar URL pública permanente;
- usar URL temporária/assinada quando a infraestrutura suportar.

O binário não deve ficar armazenado como BLOB no PostgreSQL por padrão.

---

# 21. Swagger

Swagger/OpenAPI entra somente na ETAPA 12, depois da migração para PostgreSQL e estabilização do contrato.

Quando entrar, documentar exatamente:

- UUIDs públicos;
- DTOs;
- ações;
- filtros;
- paginação;
- status HTTP;
- códigos de erro;
- JWT;
- headers aplicáveis.

---

# 22. Checklist HTTP

- [ ] prefixo `/api/v1`?
- [ ] paths públicos usam `UUID publicId`?
- [ ] nenhum `Long id` no contrato?
- [ ] relações usam `...PublicId`?
- [ ] tenant não aparece como `tenantId` livre?
- [ ] Controller usa DTO e `@Valid`?
- [ ] status HTTP correto?
- [ ] erro possui `code` estável?
- [ ] erro possui `correlationId`?
- [ ] cross-tenant retorna 404 quando aplicável?
- [ ] lista paginada quando necessário?
- [ ] datas são ISO 8601?
- [ ] ações de domínio são explícitas?
- [ ] 409 cobre conflitos esperados, não 500?

---

# Regra final

```text
URL/DTO → publicId
Tenant → contexto autenticado
Long id → interno
Erro → code + correlationId
Lista grande → paginação
Ação de domínio → endpoint explícito
```
