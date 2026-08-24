# 06 — APIs, CRUDs e Regras

## Padrão

```text
/api/v1
```

Evitar endpoints CRUD puros quando existe ação de domínio.

---

# Auth

```http
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

---

# Tenant — superAdmin global

```http
POST  /api/v1/admin/tenants
GET   /api/v1/admin/tenants
GET   /api/v1/admin/tenants/{id}
PUT   /api/v1/admin/tenants/{id}
PATCH /api/v1/admin/tenants/{id}/suspender
PATCH /api/v1/admin/tenants/{id}/ativar
```

Sem hard delete.

---

# Unidade

```http
POST  /api/v1/unidades
GET   /api/v1/unidades
GET   /api/v1/unidades/{id}
PUT   /api/v1/unidades/{id}
PATCH /api/v1/unidades/{id}/ativar
PATCH /api/v1/unidades/{id}/desativar
```

---

# Usuários

```http
POST  /api/v1/usuarios
GET   /api/v1/usuarios
GET   /api/v1/usuarios/{id}
PUT   /api/v1/usuarios/{id}
PATCH /api/v1/usuarios/{id}/bloquear
PATCH /api/v1/usuarios/{id}/ativar
PATCH /api/v1/usuarios/{id}/perfil
```

Criação deve gerar/ligar `Usuario`, `Pessoa` e `UsuarioTenant` conforme payload.

---

# Profissionais

```http
POST  /api/v1/profissionais
GET   /api/v1/profissionais
GET   /api/v1/profissionais/{id}
PUT   /api/v1/profissionais/{id}
PATCH /api/v1/profissionais/{id}/ativar
PATCH /api/v1/profissionais/{id}/desativar
```

Vínculos:

```http
POST   /api/v1/profissionais/{id}/unidades/{unidadeId}
DELETE /api/v1/profissionais/{id}/unidades/{unidadeId}
POST   /api/v1/profissionais/{id}/especialidades/{especialidadeId}
DELETE /api/v1/profissionais/{id}/especialidades/{especialidadeId}
POST   /api/v1/profissionais/{id}/procedimentos/{procedimentoId}
DELETE /api/v1/profissionais/{id}/procedimentos/{procedimentoId}
```

---

# Especialidades

```http
POST /api/v1/especialidades
GET  /api/v1/especialidades
PUT  /api/v1/especialidades/{id}
```

Ativar/desativar conforme necessidade.

---

# Procedimentos

```http
POST /api/v1/procedimentos
GET  /api/v1/procedimentos
GET  /api/v1/procedimentos/{id}
PUT  /api/v1/procedimentos/{id}
```

---

# Pacientes

```http
POST  /api/v1/pacientes
GET   /api/v1/pacientes
GET   /api/v1/pacientes/{id}
PUT   /api/v1/pacientes/{id}
PATCH /api/v1/pacientes/{id}/inativar
PATCH /api/v1/pacientes/{id}/ativar
```

Filtros:

```text
nome
cpf
telefone
numeroProntuario
```

Paginar listagens desde cedo.

---

# Disponibilidade

```http
POST   /api/v1/profissionais/{id}/disponibilidades
GET    /api/v1/profissionais/{id}/disponibilidades
PUT    /api/v1/profissionais/{id}/disponibilidades/{dispId}
DELETE /api/v1/profissionais/{id}/disponibilidades/{dispId}
```

---

# Bloqueios

```http
POST   /api/v1/profissionais/{id}/bloqueios
GET    /api/v1/profissionais/{id}/bloqueios
DELETE /api/v1/profissionais/{id}/bloqueios/{bloqueioId}
```

---

# Horários disponíveis

```http
GET /api/v1/profissionais/{id}/horarios-disponiveis
```

Query:

```text
data
unidadeId
procedimentoId
```

Resposta simples:

```json
[
  {"inicio":"2026-09-01T08:00:00-03:00","fim":"2026-09-01T08:30:00-03:00"},
  {"inicio":"2026-09-01T08:30:00-03:00","fim":"2026-09-01T09:00:00-03:00"}
]
```

---

# Agendamento

```http
POST  /api/v1/agendamentos
GET   /api/v1/agendamentos
GET   /api/v1/agendamentos/{id}
PATCH /api/v1/agendamentos/{id}/confirmar
PATCH /api/v1/agendamentos/{id}/cancelar
PATCH /api/v1/agendamentos/{id}/check-in
PATCH /api/v1/agendamentos/{id}/marcar-falta
POST  /api/v1/agendamentos/{id}/reagendar
```

Evitar `PUT /agendamentos/{id}` genérico depois que o agendamento entra em fluxo.

---

# Atendimento

```http
POST  /api/v1/agendamentos/{id}/iniciar-atendimento
POST  /api/v1/atendimentos/avulso
GET   /api/v1/atendimentos/{id}
PATCH /api/v1/atendimentos/{id}/concluir
```

---

# Evolução clínica

```http
POST /api/v1/atendimentos/{id}/evolucoes
GET  /api/v1/atendimentos/{id}/evolucoes
POST /api/v1/evolucoes/{id}/retificar
```

Sem DELETE.

---

# Prontuário

```http
GET /api/v1/pacientes/{id}/prontuario
GET /api/v1/pacientes/{id}/prontuario/atendimentos
GET /api/v1/pacientes/{id}/prontuario/prescricoes
GET /api/v1/pacientes/{id}/prontuario/anexos
```

---

# Prescrição

```http
POST  /api/v1/atendimentos/{id}/prescricoes
GET   /api/v1/prescricoes/{id}
POST  /api/v1/prescricoes/{id}/itens
PUT   /api/v1/prescricoes/{id}/itens/{itemId}    # somente rascunho
DELETE /api/v1/prescricoes/{id}/itens/{itemId}   # somente rascunho
POST  /api/v1/prescricoes/{id}/emitir
POST  /api/v1/prescricoes/{id}/cancelar
```

Depois de `EMITIDA`, não editar silenciosamente.

---

# Anexos

A API pode continuar orientada ao paciente, mas internamente `AnexoClinico` referencia o Prontuario.

```http
POST /api/v1/pacientes/{id}/anexos
GET  /api/v1/pacientes/{id}/anexos
POST /api/v1/atendimentos/{id}/anexos
GET  /api/v1/atendimentos/{id}/anexos
GET  /api/v1/anexos/{id}/download
```

O Service resolve:

```text
Paciente → Prontuario → AnexoClinico
```

---

# Minha agenda — profissional

```http
GET /api/v1/me/agenda/hoje
GET /api/v1/me/agenda?data=...
GET /api/v1/me/pacientes/aguardando
GET /api/v1/me/resumo-diario
```

---

# API do paciente

```http
GET  /api/v1/me/perfil
GET  /api/v1/me/consultas
GET  /api/v1/me/consultas/proximas
GET  /api/v1/me/prescricoes
GET  /api/v1/me/prescricoes/{id}
GET  /api/v1/me/notificacoes
```

Autoagendamento pode entrar depois:

```http
GET  /api/v1/me/horarios-disponiveis
POST /api/v1/me/agendamentos
POST /api/v1/me/agendamentos/{id}/cancelar
POST /api/v1/me/agendamentos/{id}/reagendar
```

---

# Regras críticas

1. Recurso sempre pertence ao tenant ativo.
2. Usuário deve possuir acesso à unidade.
3. Profissional deve pertencer ao tenant e à unidade.
4. Procedimento deve estar habilitado para o profissional.
5. Horário deve estar dentro da disponibilidade.
6. Horário não pode estar bloqueado.
7. Não pode haver overlap.
8. Paciente inativo não cria novo agendamento sem regra explícita.
9. Check-in só ocorre em agendamento válido.
10. Atendimento concluído não volta a `EM_ANDAMENTO` diretamente.
11. Evolução finalizada não é sobrescrita.
12. Prescrição emitida não é editada silenciosamente.
13. Toda leitura sensível deve ser autorizada e auditada quando definida como crítica.


---

## Implementação por etapa

Os endpoints deste documento são catálogo de referência. Eles **não devem ser implementados todos de uma vez**.

Seguir:

```text
ETAPA 1 → Tenant/Unidade
ETAPA 2 → Auth/Usuários
ETAPA 3 → Profissionais/Pacientes/Procedimentos
ETAPA 4 → Disponibilidade
ETAPA 5 → Agendamentos
ETAPA 6 → Atendimento/Prontuário
ETAPA 7 → Prescrições/Anexos
ETAPA 8 → /me operacional
ETAPA 15 → /me paciente
```


---

## Documentação da API somente após PostgreSQL

Os endpoints aqui são o contrato planejado.

Durante o Core:

- implementar;
- testar automaticamente;
- ajustar quando a regra exigir.

Depois da migração para PostgreSQL:

```text
ETAPA 12 → Swagger
ETAPA 13 → Postman
```

Consultar `17_PADROES_HTTP_ERROS_PAGINACAO.md` para manter consistência antes mesmo do Swagger existir.
