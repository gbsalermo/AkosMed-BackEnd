# 06 — APIs, CRUDs e Regras

> Catálogo de referência dos contratos REST planejados.  
> **Não significa que todos os endpoints já estejam implementados.** A ordem executável está em `11_ROADMAP_ETAPAS.md` e o estado real em `CONTINUIDADE.md`.

---

# 1. Padrão geral

Prefixo:

```text
/api/v1
```

Identificação externa:

```text
UUID publicId
```

Nunca usar PK `Long id` em URL/DTO público.

Relacionamentos recebidos pela API usam:

```text
...PublicId
```

Tenant vem do contexto autenticado e não de `tenantId` livre do request.

Detalhes: `17_PADROES_HTTP_ERROS_PAGINACAO.md` e `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 2. Auth — ETAPA 2

```http
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

Login tenant-scoped usa `tenantSlug` conforme documentação de segurança.

---

# 3. Tenant — superAdmin global — ETAPA 1

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

# 4. Unidade — ETAPA 1

```http
POST  /api/v1/unidades
GET   /api/v1/unidades
GET   /api/v1/unidades/{publicId}
PUT   /api/v1/unidades/{publicId}
PATCH /api/v1/unidades/{publicId}/ativar
PATCH /api/v1/unidades/{publicId}/desativar
```

Código único dentro do Tenant.

---

# 5. Usuários/vínculos — ETAPA 2

Contrato administrativo planejado:

```http
POST  /api/v1/usuarios
GET   /api/v1/usuarios
GET   /api/v1/usuarios/{publicId}
PUT   /api/v1/usuarios/{publicId}
PATCH /api/v1/usuarios/{publicId}/bloquear
PATCH /api/v1/usuarios/{publicId}/ativar
PATCH /api/v1/usuarios/{publicId}/perfil
```

A criação pode coordenar `Usuario`, `Pessoa` e `UsuarioTenant` conforme DTO/regra da subetapa.

Não retornar `passwordHash`.

Acesso por Unidade usa vínculos tenant-safe e UUIDs públicos.

---

# 6. Especialidades — ETAPA 3

```http
POST /api/v1/especialidades
GET  /api/v1/especialidades
GET  /api/v1/especialidades/{publicId}
PUT  /api/v1/especialidades/{publicId}
PATCH /api/v1/especialidades/{publicId}/ativar
PATCH /api/v1/especialidades/{publicId}/desativar
```

---

# 7. Procedimentos — ETAPA 3

```http
POST /api/v1/procedimentos
GET  /api/v1/procedimentos
GET  /api/v1/procedimentos/{publicId}
PUT  /api/v1/procedimentos/{publicId}
PATCH /api/v1/procedimentos/{publicId}/ativar
PATCH /api/v1/procedimentos/{publicId}/desativar
```

---

# 8. Profissionais — ETAPA 3

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
POST   /api/v1/profissionais/{profissionalPublicId}/unidades/{unidadePublicId}
DELETE /api/v1/profissionais/{profissionalPublicId}/unidades/{unidadePublicId}

POST   /api/v1/profissionais/{profissionalPublicId}/especialidades/{especialidadePublicId}
DELETE /api/v1/profissionais/{profissionalPublicId}/especialidades/{especialidadePublicId}

POST   /api/v1/profissionais/{profissionalPublicId}/procedimentos/{procedimentoPublicId}
DELETE /api/v1/profissionais/{profissionalPublicId}/procedimentos/{procedimentoPublicId}
```

Service valida mesmo Tenant e demais regras de vínculo.

---

# 9. Pacientes — ETAPA 3

```http
POST  /api/v1/pacientes
GET   /api/v1/pacientes
GET   /api/v1/pacientes/{publicId}
PUT   /api/v1/pacientes/{publicId}
PATCH /api/v1/pacientes/{publicId}/inativar
PATCH /api/v1/pacientes/{publicId}/ativar
```

Filtros possíveis:

```text
nome
cpf
telefone
numeroProntuario
```

Paginar listagens.

Paciente inativo não cria novo agendamento sem regra explícita.

---

# 10. Disponibilidade — ETAPA 4

```http
POST   /api/v1/profissionais/{profissionalPublicId}/disponibilidades
GET    /api/v1/profissionais/{profissionalPublicId}/disponibilidades
PUT    /api/v1/profissionais/{profissionalPublicId}/disponibilidades/{disponibilidadePublicId}
DELETE /api/v1/profissionais/{profissionalPublicId}/disponibilidades/{disponibilidadePublicId}
```

Não persistir slots calculados.

---

# 11. Bloqueios — ETAPA 4

```http
POST   /api/v1/profissionais/{profissionalPublicId}/bloqueios
GET    /api/v1/profissionais/{profissionalPublicId}/bloqueios
DELETE /api/v1/profissionais/{profissionalPublicId}/bloqueios/{bloqueioPublicId}
```

No MVP, bloqueio é por Unidade específica.

---

# 12. Horários disponíveis — ETAPA 4

```http
GET /api/v1/profissionais/{profissionalPublicId}/horarios-disponiveis
```

Query:

```text
data
unidadePublicId
procedimentoPublicId
```

Resposta conceitual:

```json
[
  {
    "inicio": "2026-09-01T08:00:00-03:00",
    "fim": "2026-09-01T08:30:00-03:00"
  }
]
```

A lista exibida não reserva o slot.

---

# 13. Agendamentos — ETAPA 5

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

Evitar `PUT /agendamentos/{publicId}` genérico depois que o recurso entra no fluxo de status.

Criação/reagendamento devem revalidar concorrência dentro da transação.

Double booking:

```text
1 request → 201 Created
1 request → 409 AGENDAMENTO_CONFLITO
```

---

# 14. Atendimento — ETAPA 6

```http
POST  /api/v1/agendamentos/{agendamentoPublicId}/iniciar-atendimento
POST  /api/v1/atendimentos/avulso
GET   /api/v1/atendimentos/{publicId}
PATCH /api/v1/atendimentos/{publicId}/concluir
```

Check-in não inicia atendimento automaticamente.

Atendimento avulso continua tenant-safe e precisa de Unidade/Paciente/Profissional conforme DTO.

---

# 15. Evolução clínica — ETAPA 6

```http
POST /api/v1/atendimentos/{atendimentoPublicId}/evolucoes
GET  /api/v1/atendimentos/{atendimentoPublicId}/evolucoes
POST /api/v1/evolucoes/{publicId}/retificar
```

Sem DELETE e sem update destrutivo.

---

# 16. Prontuário — ETAPA 6

```http
GET /api/v1/pacientes/{pacientePublicId}/prontuario
GET /api/v1/pacientes/{pacientePublicId}/prontuario/atendimentos
GET /api/v1/pacientes/{pacientePublicId}/prontuario/prescricoes
GET /api/v1/pacientes/{pacientePublicId}/prontuario/anexos
```

Toda leitura sensível valida Tenant, autorização e auditoria quando aplicável.

---

# 17. Prescrição — ETAPA 7

```http
POST   /api/v1/atendimentos/{atendimentoPublicId}/prescricoes
GET    /api/v1/prescricoes/{publicId}
POST   /api/v1/prescricoes/{prescricaoPublicId}/itens
PUT    /api/v1/prescricoes/{prescricaoPublicId}/itens/{itemPublicId}
DELETE /api/v1/prescricoes/{prescricaoPublicId}/itens/{itemPublicId}
POST   /api/v1/prescricoes/{publicId}/emitir
POST   /api/v1/prescricoes/{publicId}/cancelar
```

Item só é editável/removível enquanto a Prescrição estiver em RASCUNHO.

Após EMITIDA, não editar silenciosamente.

---

# 18. Anexos — ETAPA 7

A API pode ser orientada ao Paciente/Atendimento, enquanto internamente `AnexoClinico` referencia Prontuário.

```http
POST /api/v1/pacientes/{pacientePublicId}/anexos
GET  /api/v1/pacientes/{pacientePublicId}/anexos
POST /api/v1/atendimentos/{atendimentoPublicId}/anexos
GET  /api/v1/atendimentos/{atendimentoPublicId}/anexos
GET  /api/v1/anexos/{publicId}/download
```

Service resolve:

```text
Paciente → Prontuario → AnexoClinico
```

Arquivo fica fora do banco e `storageKey` não é exposto.

---

# 19. Minha agenda — profissional — ETAPA 8

```http
GET /api/v1/me/agenda/hoje
GET /api/v1/me/agenda?data=...
GET /api/v1/me/pacientes/aguardando
GET /api/v1/me/resumo-diario
```

O backend resolve o profissional autenticado; não pedir `profissionalPublicId` em `/me`.

---

# 20. Notificações — ETAPA 8

```http
GET   /api/v1/me/notificacoes
PATCH /api/v1/me/notificacoes/{publicId}/lida
PATCH /api/v1/me/notificacoes/lidas
```

Primeira versão in-app.

---

# 21. Auditoria — ETAPA 9

Endpoints administrativos/auditoria devem ser definidos conforme matriz final de autorização.

Listagem sempre paginada e protegida.

Não retornar metadata sensível desnecessária.

---

# 22. API do paciente `/me` — ETAPA 15, pós-MVP

```http
GET /api/v1/me/perfil
GET /api/v1/me/consultas
GET /api/v1/me/consultas/proximas
GET /api/v1/me/prescricoes
GET /api/v1/me/prescricoes/{publicId}
GET /api/v1/me/notificacoes
```

O usuário autenticado é convertido em Paciente pelo backend.

Nunca aceitar `pacientePublicId` para acessar dados próprios em `/me`.

---

# 23. Autoagendamento do paciente — ETAPA 15, se aprovado

```http
GET  /api/v1/me/horarios-disponiveis
POST /api/v1/me/agendamentos
POST /api/v1/me/agendamentos/{publicId}/cancelar
POST /api/v1/me/agendamentos/{publicId}/reagendar
```

Reutilizar o mesmo `AgendamentoService` e a mesma proteção de concorrência.

Não duplicar regra no módulo mobile.

---

# 24. Orientações/materiais terapêuticos — ETAPA 15, pós-MVP

Primeiro caso:

```text
Fisioterapeuta envia vídeo do exercício correto ao paciente
```

Domínio genérico:

```text
OrientacaoPaciente
```

Tipos:

```text
VIDEO
DOCUMENTO
LINK
TEXTO
```

Paciente autenticado:

```http
GET /api/v1/me/orientacoes
GET /api/v1/me/orientacoes/{orientacaoPublicId}
```

Profissional autorizado:

```http
POST   /api/v1/pacientes/{pacientePublicId}/orientacoes
GET    /api/v1/pacientes/{pacientePublicId}/orientacoes
DELETE /api/v1/pacientes/{pacientePublicId}/orientacoes/{orientacaoPublicId}
```

Regras:

- paciente/profissional/Atendimento no mesmo Tenant;
- paciente só vê as próprias orientações;
- paciente não edita conteúdo enviado pelo profissional;
- mídia fora do PostgreSQL;
- `storageKey` interno;
- acesso por URL temporária/assinada quando suportado;
- MIME/tamanho validados;
- remoção lógica/auditoria conforme regra.

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# 25. Regras críticas transversais

1. Todo recurso tenant-scoped pertence ao Tenant ativo.
2. UUID não substitui autorização.
3. Usuário deve possuir acesso à Unidade quando a regra exigir.
4. Profissional deve pertencer ao Tenant e estar vinculado à Unidade.
5. Procedimento deve estar habilitado quando o fluxo exigir.
6. Horário deve estar dentro da disponibilidade.
7. Horário não pode estar bloqueado.
8. Não pode haver overlap ativo.
9. Paciente inativo não cria novo agendamento sem regra explícita.
10. Check-in só ocorre em agendamento válido.
11. Atendimento concluído não volta livremente a EM_ANDAMENTO.
12. Evolução clínica não é sobrescrita.
13. Prescrição emitida não é editada silenciosamente.
14. Toda leitura sensível é autorizada e auditada quando classificada como crítica.
15. `Clock` governa regras temporais.
16. `@Version` protege lost update onde adotado.
17. Idempotência evita efeito duplicado onde adotada.
18. `correlationId` acompanha erros/rastreabilidade.
19. API não expõe PK `Long`.

---

# 26. Ordem de implementação dos endpoints

```text
ETAPA 1  → Tenant/Unidade
ETAPA 2  → Auth/Usuários
ETAPA 3  → Especialidades/Procedimentos/Profissionais/Pacientes
ETAPA 4  → Disponibilidade/Bloqueios
ETAPA 5  → Agendamentos
ETAPA 6  → Atendimento/Prontuário/Evolução
ETAPA 7  → Prescrições/Anexos/Storage
ETAPA 8  → /me operacional + Notificações
ETAPA 9  → Auditoria/Hardening
ETAPA 11 → PostgreSQL/migrations
ETAPA 12 → Swagger
ETAPA 13 → Postman
ETAPA 14 → fechamento MVP
ETAPA 15 → /me paciente + orientações/materiais
```

Não implementar todos os endpoints deste arquivo de uma vez.

---

# 27. Documentação final da API

Durante o Core:

- implementar;
- testar automaticamente;
- manter contratos coerentes com este catálogo;
- ajustar documentação se a regra mudar oficialmente.

Depois do PostgreSQL:

```text
ETAPA 12 → Swagger/OpenAPI
ETAPA 13 → Postman
```

Swagger/Postman não são motivo para antecipar infraestrutura no início.

---

# 28. Resposta de conflito de agenda

Exemplo:

```json
{
  "status": 409,
  "code": "AGENDAMENTO_CONFLITO",
  "message": "O horário acabou de ser reservado por outro paciente.",
  "correlationId": "..."
}
```

O cliente deve atualizar a disponibilidade depois do 409.
