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
GET   /api/v1/admin/tenants/{publicId}
PUT   /api/v1/admin/tenants/{publicId}
PATCH /api/v1/admin/tenants/{publicId}/suspender
PATCH /api/v1/admin/tenants/{publicId}/ativar
```

Sem hard delete.

---

# Unidade

```http
POST  /api/v1/unidades
GET   /api/v1/unidades
GET   /api/v1/unidades/{publicId}
PUT   /api/v1/unidades/{publicId}
PATCH /api/v1/unidades/{publicId}/ativar
PATCH /api/v1/unidades/{publicId}/desativar
```

---

# Usuários

```http
POST  /api/v1/usuarios
GET   /api/v1/usuarios
GET   /api/v1/usuarios/{publicId}
PUT   /api/v1/usuarios/{publicId}
PATCH /api/v1/usuarios/{publicId}/bloquear
PATCH /api/v1/usuarios/{publicId}/ativar
PATCH /api/v1/usuarios/{publicId}/perfil
```

Criação deve gerar/ligar `Usuario`, `Pessoa` e `UsuarioTenant` conforme payload.

---

# Profissionais

```http
POST  /api/v1/profissionais
GET   /api/v1/profissionais
GET   /api/v1/profissionais/{publicId}
PUT   /api/v1/profissionais/{publicId}
PATCH /api/v1/profissionais/{publicId}/ativar
PATCH /api/v1/profissionais/{publicId}/desativar
```

Vínculos:

```http
POST   /api/v1/profissionais/{publicId}/unidades/{unidadePublicId}
DELETE /api/v1/profissionais/{publicId}/unidades/{unidadePublicId}
POST   /api/v1/profissionais/{publicId}/especialidades/{especialidadePublicId}
DELETE /api/v1/profissionais/{publicId}/especialidades/{especialidadePublicId}
POST   /api/v1/profissionais/{publicId}/procedimentos/{procedimentoPublicId}
DELETE /api/v1/profissionais/{publicId}/procedimentos/{procedimentoPublicId}
```

---

# Especialidades

```http
POST /api/v1/especialidades
GET  /api/v1/especialidades
PUT  /api/v1/especialidades/{publicId}
```

Ativar/desativar conforme necessidade.

---

# Procedimentos

```http
POST /api/v1/procedimentos
GET  /api/v1/procedimentos
GET  /api/v1/procedimentos/{publicId}
PUT  /api/v1/procedimentos/{publicId}
```

---

# Pacientes

```http
POST  /api/v1/pacientes
GET   /api/v1/pacientes
GET   /api/v1/pacientes/{publicId}
PUT   /api/v1/pacientes/{publicId}
PATCH /api/v1/pacientes/{publicId}/inativar
PATCH /api/v1/pacientes/{publicId}/ativar
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
POST   /api/v1/profissionais/{publicId}/disponibilidades
GET    /api/v1/profissionais/{publicId}/disponibilidades
PUT    /api/v1/profissionais/{publicId}/disponibilidades/{disponibilidadePublicId}
DELETE /api/v1/profissionais/{publicId}/disponibilidades/{disponibilidadePublicId}
```

---

# Bloqueios

```http
POST   /api/v1/profissionais/{publicId}/bloqueios
GET    /api/v1/profissionais/{publicId}/bloqueios
DELETE /api/v1/profissionais/{publicId}/bloqueios/{bloqueioPublicId}
```

---

# Horários disponíveis

```http
GET /api/v1/profissionais/{publicId}/horarios-disponiveis
```

Query:

```text
data
unidadePublicId
procedimentoPublicId
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
GET   /api/v1/agendamentos/{publicId}
PATCH /api/v1/agendamentos/{publicId}/confirmar
PATCH /api/v1/agendamentos/{publicId}/cancelar
PATCH /api/v1/agendamentos/{publicId}/check-in
PATCH /api/v1/agendamentos/{publicId}/marcar-falta
POST  /api/v1/agendamentos/{publicId}/reagendar
```

Evitar `PUT /agendamentos/{publicId}` genérico depois que o agendamento entra em fluxo.

---

# Atendimento

```http
POST  /api/v1/agendamentos/{publicId}/iniciar-atendimento
POST  /api/v1/atendimentos/avulso
GET   /api/v1/atendimentos/{publicId}
PATCH /api/v1/atendimentos/{publicId}/concluir
```

---

# Evolução clínica

```http
POST /api/v1/atendimentos/{publicId}/evolucoes
GET  /api/v1/atendimentos/{publicId}/evolucoes
POST /api/v1/evolucoes/{publicId}/retificar
```

Sem DELETE.

---

# Prontuário

```http
GET /api/v1/pacientes/{publicId}/prontuario
GET /api/v1/pacientes/{publicId}/prontuario/atendimentos
GET /api/v1/pacientes/{publicId}/prontuario/prescricoes
GET /api/v1/pacientes/{publicId}/prontuario/anexos
```

---

# Prescrição

```http
POST   /api/v1/atendimentos/{publicId}/prescricoes
GET    /api/v1/prescricoes/{publicId}
POST   /api/v1/prescricoes/{publicId}/itens
PUT    /api/v1/prescricoes/{publicId}/itens/{itemPublicId}    # somente rascunho
DELETE /api/v1/prescricoes/{publicId}/itens/{itemPublicId}   # somente rascunho
POST   /api/v1/prescricoes/{publicId}/emitir
POST   /api/v1/prescricoes/{publicId}/cancelar
```

Depois de `EMITIDA`, não editar silenciosamente.

---

# Anexos

A API pode continuar orientada ao paciente, mas internamente `AnexoClinico` referencia o Prontuario.

```http
POST /api/v1/pacientes/{publicId}/anexos
GET  /api/v1/pacientes/{publicId}/anexos
POST /api/v1/atendimentos/{publicId}/anexos
GET  /api/v1/atendimentos/{publicId}/anexos
GET  /api/v1/anexos/{publicId}/download
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
GET  /api/v1/me/prescricoes/{publicId}
GET  /api/v1/me/notificacoes
```

Autoagendamento pode entrar depois:

```http
GET  /api/v1/me/horarios-disponiveis
POST /api/v1/me/agendamentos
POST /api/v1/me/agendamentos/{publicId}/cancelar
POST /api/v1/me/agendamentos/{publicId}/reagendar
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

---

# Referência obrigatória

Consultar `21_PUBLIC_ID_E_CONCORRENCIA.md` para padrão de UUID, lock, overlap e testes concorrentes.

---

# Concorrência na API de agendamento

Duas requisições simultâneas para o mesmo profissional/período:

```text
requisição A → 201 Created
requisição B → 409 Conflict
```

Erro:

```json
{
  "code": "AGENDAMENTO_CONFLITO",
  "message": "O horário acabou de ser reservado por outro paciente."
}
```

O cliente deve atualizar os horários disponíveis após receber 409.
