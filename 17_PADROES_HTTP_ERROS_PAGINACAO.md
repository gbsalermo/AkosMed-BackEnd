# 17 — Padrões HTTP, Erros e Paginação

# 1. Prefixo

```text
/api/v1
```

---

# 2. CRUD administrativo

## Criar

```http
POST /api/v1/pacientes
```

Resposta:

```text
201 Created
```

Body:

`PacienteResponseDTO`.

---

## Buscar

```http
GET /api/v1/pacientes/{id}
```

```text
200 OK
404 Not Found
```

---

## Listar

```http
GET /api/v1/pacientes?page=0&size=20
```

---

## Atualizar

```http
PUT /api/v1/pacientes/{id}
```

ou PATCH se update parcial for intencional.

Para o MVP, preferir PUT em dados administrativos editáveis.

---

# 3. Ações de domínio

Não usar update genérico para status.

Exemplo:

```http
PATCH /api/v1/agendamentos/{id}/confirmar
PATCH /api/v1/agendamentos/{id}/check-in
PATCH /api/v1/agendamentos/{id}/cancelar
```

---

# 4. Deletes

Usar DELETE para relações/removíveis sem histórico crítico.

Exemplo:

```http
DELETE /profissionais/{id}/especialidades/{especialidadeId}
```

Evitar em:

- prontuário;
- evolução;
- atendimento concluído;
- prescrição emitida;
- audit log.

---

# 5. Status HTTP

## 200

Leitura/ação com response.

## 201

Criação.

## 204

Ação sem body, se preferir.

## 400

DTO inválido/sintaxe/validação.

## 401

Não autenticado/token inválido.

## 403

Autenticado, mas sem permissão.

## 404

Recurso inexistente ou não visível no tenant.

Para cross-tenant, 404 é uma boa política para não revelar existência.

## 409

Conflito de negócio:

- duplicidade;
- horário ocupado;
- transição inválida;
- recurso em estado incompatível.

---

# 6. ApiErrorDTO

Padrão recomendado:

```json
{
  "timestamp": "2026-08-23T20:10:00-03:00",
  "status": 409,
  "code": "AGENDAMENTO_CONFLITO",
  "message": "O profissional já possui atendimento no período informado.",
  "path": "/api/v1/agendamentos",
  "correlationId": "..."
}
```

---

# 7. Field errors

```json
{
  "timestamp": "...",
  "status": 400,
  "code": "VALIDATION_ERROR",
  "message": "Existem campos inválidos.",
  "fieldErrors": [
    {
      "field": "nome",
      "message": "não deve estar em branco"
    }
  ]
}
```

---

# 8. Códigos de erro

Criar códigos estáveis.

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
ACCESS_DENIED
```

Frontend/app pode reagir ao `code` sem depender da frase.

---

# 9. Paginação

Para listas grandes, usar:

```text
page
size
sort
```

Limites:

```text
default size = 20
max size = 100
```

---

# 10. PageResponseDTO

Para não acoplar o contrato ao `PageImpl` do Spring:

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

Pode ser implementado quando entrar a primeira listagem grande.

---

# 11. Filtros

Exemplo:

```http
GET /agendamentos?dataInicio=&dataFim=&profissionalId=&status=
```

Não criar endpoint separado para cada combinação.

Mas não fazer filtro genérico complexo cedo demais.

Começar pelos filtros realmente usados.

---

# 12. Ordenação

Agenda:

```text
inicio ASC
```

Pacientes:

```text
nome ASC
```

AuditLog:

```text
timestamp DESC
```

Definir default no Service/Repository.

---

# 13. Data/hora na API

Usar ISO 8601.

Exemplo:

```text
2026-08-23T14:30:00-03:00
```

Evitar formatos locais como:

```text
23/08/2026 14:30
```

no contrato REST.

---

# 14. Booleans

Preferir nomes claros:

```text
ativo
acessoTodasUnidades
usoContinuo
seNecessario
```

---

# 15. Compatibilidade com app

Endpoints do paciente:

```text
/api/v1/me/*
```

Evitar pedir:

```text
pacienteId
```

para o próprio paciente.

---

# 16. Swagger

Na ETAPA 12, documentar estes padrões exatamente.

---

# 17. Checklist HTTP

- [ ] status correto;
- [ ] DTO correto;
- [ ] erro padronizado;
- [ ] tenant não aparece como input livre;
- [ ] lista paginada quando necessário;
- [ ] datas ISO;
- [ ] ações de domínio explícitas;
- [ ] códigos de erro estáveis.
