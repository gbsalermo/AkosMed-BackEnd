# 04 — Entidades e Relacionamentos

Este arquivo é o **modelo canônico do domínio MVP**.  
Se outro documento divergir em campos ou relacionamentos, este arquivo deve ser usado junto do `CONTINUIDADE.md` e do roadmap.

---

# Convenções

## Entidades tenant-scoped

Quando uma entidade pertence à organização:

```text
id Long
tenant
```

Ela deve ser consultada pelo tenant ativo.

## Campos comuns

Adicionar apenas quando fizer sentido:

```text
createdAt
updatedAt
ativo/status
```

Não criar `ativo`, `deletedAt` ou timestamps em todas as tabelas por reflexo.

## JPA

Antes de implementar relações, consultar:

- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`
- `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`

---

# 1. Tenant

Representa a organização cliente do AkosMed.

```text
id
nome
nomeFantasia nullable
slug
documento nullable
status
timezone
createdAt
updatedAt
```

Enum:

```text
ATIVO
SUSPENSO
```

Regras:

- `slug` único global;
- sem hard delete no MVP;
- timezone obrigatório.

---

# 2. Unidade

```text
id
tenant
nome
codigo
telefone nullable
email nullable
logradouro nullable
numero nullable
bairro nullable
cidade nullable
estado nullable
cep nullable
ativo
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, codigo)
```

Relação:

```text
Unidade N:1 Tenant
```

---

# 3. Pessoa

Dados pessoais dentro de um Tenant.

```text
id
tenant
nomeCompleto
nomeSocial nullable
cpf nullable
dataNascimento nullable
telefone nullable
emailContato nullable
createdAt
updatedAt
```

Constraint, quando CPF existir:

```text
unique(tenant_id, cpf)
```

Pessoa não contém:

- senha;
- prontuário;
- especialidade;
- agenda;
- prescrição.

---

# 4. Usuario

Credencial global.

```text
id
emailLogin
passwordHash
status
superAdmin
ultimoLogin nullable
createdAt
updatedAt
```

Enum:

```text
ATIVO
BLOQUEADO
```

Constraint:

```text
emailLogin UNIQUE
```

Nunca retornar `passwordHash`.

---

# 5. UsuarioTenant

Vínculo da credencial com uma organização.

```text
id
usuario
tenant
pessoa
perfilTenant
acessoTodasUnidades
ativo
createdAt
```

Enum `PerfilTenant`:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

Constraints:

```text
unique(usuario_id, tenant_id)
unique(tenant_id, pessoa_id)
```

Relações:

```text
UsuarioTenant N:1 Usuario
UsuarioTenant N:1 Tenant
UsuarioTenant N:1 Pessoa
```

---

# 6. UsuarioUnidade

Usado quando:

```text
UsuarioTenant.acessoTodasUnidades = false
```

```text
id
usuarioTenant
unidade
```

Constraint:

```text
unique(usuario_tenant_id, unidade_id)
```

Service deve validar:

```text
usuarioTenant.tenant == unidade.tenant
```

---

# 7. RefreshToken

```text
id
usuario
usuarioTenant nullable
tokenHash
expiresAt
revogadoEm nullable
createdAt
```

Para sessão comum tenant-scoped:

```text
usuarioTenant obrigatório na lógica
```

`usuarioTenant` nullable serve apenas para futuro fluxo administrativo global de `superAdmin`.

---

# 8. Especialidade

Tenant-scoped.

```text
id
tenant
nome
codigo
descricao nullable
ativo
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, codigo)
```

Não criar catálogo global no MVP.

---

# 9. Procedimento

```text
id
tenant
nome
codigo
descricao nullable
duracaoPadraoMinutos
valorPadrao nullable
ativo
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, codigo)
```

Tipos:

```text
duracaoPadraoMinutos → Integer
valorPadrao → BigDecimal
```

`valorPadrao` é referência operacional, não módulo financeiro.

---

# 10. ProfissionalSaude

```text
id
tenant
pessoa
tipoProfissional
conselho nullable
numeroRegistro nullable
ufRegistro nullable
ativo
createdAt
updatedAt
```

Constraint:

```text
unique(tenant_id, pessoa_id)
```

Quando os dados de conselho existirem, avaliar também unique composto:

```text
tenant + conselho + numeroRegistro + ufRegistro
```

Não criar `Medico`, `Dentista`, `Psicologo` separados no Core.

---

# 11. ProfissionalEspecialidade

Entidade de vínculo explícita.

```text
id
profissional
especialidade
principal
```

Constraint:

```text
unique(profissional_id, especialidade_id)
```

Service valida mesmo tenant.

---

# 12. ProfissionalUnidade

```text
id
profissional
unidade
ativo
```

Constraint:

```text
unique(profissional_id, unidade_id)
```

Regra:

profissional só pode ser agendado em unidade vinculada e ativa.

---

# 13. ProfissionalProcedimento

```text
id
profissional
procedimento
duracaoMinutosOverride nullable
valorOverride nullable
ativo
```

Constraint:

```text
unique(profissional_id, procedimento_id)
```

Resolução de duração:

```text
duracaoMinutosOverride
senão
Procedimento.duracaoPadraoMinutos
```

---

# 14. Paciente

```text
id
tenant
pessoa
numeroProntuario
status
observacaoAdministrativa nullable
createdAt
updatedAt
```

Enum:

```text
ATIVO
INATIVO
```

Constraints:

```text
unique(tenant_id, pessoa_id)
unique(tenant_id, numero_prontuario)
```

Não criar `PacienteUnidade` no MVP.

Unidades onde o paciente foi atendido são identificadas via:

```text
Agendamento
Atendimento
```

---

# 15. ResponsavelPaciente — FUTURO / sob demanda

Não implementar no MVP inicial sem necessidade concreta.

```text
id
tenant
paciente
pessoaResponsavel
parentesco
responsavelLegal
```

---

# 16. DisponibilidadeProfissional

Substitui uma entidade `Agenda`.

```text
id
tenant
profissional
unidade
diaSemana
horaInicio
horaFim
vigenciaInicio
vigenciaFim nullable
ativo
```

Exemplo:

```text
SEGUNDA 08:00–12:00
SEGUNDA 14:00–18:00
```

Intervalo é representado por duas linhas.

Service valida:

- profissional mesmo tenant;
- unidade mesmo tenant;
- vínculo profissional-unidade;
- horaInicio < horaFim.

---

# 17. BloqueioAgenda

No MVP, o bloqueio é sempre de uma unidade específica.

```text
id
tenant
profissional
unidade
inicio
fim
tipo
motivo nullable
ativo
criadoPorUsuario
createdAt
```

Tipos:

```text
FERIAS
AUSENCIA
REUNIAO
FERIADO
MANUAL
OUTRO
```

Para bloquear todas as unidades:

criar um bloqueio por unidade.

Isso simplifica o cálculo.

---

# 18. Agendamento

```text
id
tenant
unidade
paciente
profissional
procedimento
inicio
fim
status
origem
observacaoAdministrativa nullable
criadoPorUsuario
createdAt
updatedAt
```

Enums:

```text
StatusAgendamento
OrigemAgendamento
```

`fim` é persistido.

Status sugeridos:

```text
SOLICITADO
CONFIRMADO
CHECK_IN
EM_ATENDIMENTO
CONCLUIDO
FALTOU
CANCELADO_PACIENTE
CANCELADO_CLINICA
```

---

# 19. EventoAgendamento

Histórico append-only.

```text
id
tenant
agendamento
tipoEvento
statusAnterior nullable
statusNovo nullable
motivo nullable
usuario nullable
createdAt
```

Não editar eventos existentes.

---

# 20. Prontuario

```text
id
tenant
paciente
createdAt
```

Constraint:

```text
unique(tenant_id, paciente_id)
```

Um paciente possui um prontuário por tenant.

Sem delete.

---

# 21. Atendimento

```text
id
tenant
prontuario
unidade
profissional
agendamento nullable
inicio
fim nullable
status
tipoAtendimento
createdAt
updatedAt
```

Enums:

```text
StatusAtendimento:
EM_ANDAMENTO
CONCLUIDO
CANCELADO
```

Possível constraint:

```text
agendamento_id UNIQUE
```

quando informado, garantindo no máximo um Atendimento por Agendamento.

## Decisão

Não duplicar `paciente_id`.

Paciente é obtido por:

```text
Atendimento
→ Prontuario
→ Paciente
```

Isso evita inconsistência entre `prontuario` e `paciente`.

---

# 22. EvolucaoClinica

Append-only.

```text
id
tenant
atendimento
profissional
conteudo
retificacaoDe nullable
motivoRetificacao nullable
createdAt
```

`conteudo` deve comportar texto longo.

Sem update destrutivo.

Retificação cria novo registro.

---

# 23. Prescricao

```text
id
tenant
atendimento
profissional
status
observacoes nullable
emitidaEm nullable
validadeAte nullable
createdAt
updatedAt
```

Enum:

```text
RASCUNHO
EMITIDA
CANCELADA
```

## Decisão

Não duplicar `paciente_id`.

Paciente é derivado:

```text
Prescricao
→ Atendimento
→ Prontuario
→ Paciente
```

Isso evita uma Prescricao apontar para paciente diferente do Atendimento.

---

# 24. ItemPrescricao

Sem catálogo farmacológico no MVP.

```text
id
prescricao
nomeMedicamento
concentracao nullable
formaFarmaceutica nullable
dose
unidadeDose nullable
viaAdministracao
frequenciaTexto nullable
vezesAoDia nullable
intervaloHoras nullable
duracaoDias nullable
dataInicio nullable
dataFim nullable
usoContinuo
seNecessario
instrucoes nullable
ordem
```

Exemplo:

```text
Amoxicilina
500 mg
cápsula
1 cápsula
via oral
3 vezes ao dia
8/8h
7 dias
após alimentação
```

Enquanto `Prescricao = RASCUNHO`:

- adicionar;
- editar;
- remover.

Depois de EMITIDA:

não editar silenciosamente.

---

# 25. AnexoClinico

```text
id
tenant
prontuario
atendimento nullable
nomeOriginal
mimeType
tamanho
hash
storageKey
categoria
uploadedByUsuario
createdAt
ativo
```

## Decisão

Não duplicar `paciente_id`.

Paciente é obtido pelo Prontuario.

Arquivo real fica fora do banco.

---

# 26. Notificacao

```text
id
tenant
usuarioTenant
categoria
titulo
mensagem
lida
createdAt
lidaEm nullable
```

Primeira versão:

notificação interna.

---

# 27. AuditLog

Preferir IDs simples e não um grafo de relações JPA.

```text
id
tenantId nullable
unidadeId nullable
usuarioId nullable
usuarioTenantId nullable
acao
recurso
recursoId nullable
ip nullable
userAgent nullable
metadataJson nullable
createdAt
```

AuditLog deve sobreviver mesmo se a modelagem do domínio mudar.

---

# Relações principais

```text
Tenant
├── Unidade
├── Pessoa
│   ├── Paciente
│   │   └── Prontuario
│   │       └── Atendimento
│   │           ├── EvolucaoClinica
│   │           ├── Prescricao
│   │           │   └── ItemPrescricao
│   │           └── AnexoClinico
│   └── ProfissionalSaude
│       ├── ProfissionalEspecialidade
│       ├── ProfissionalUnidade
│       ├── ProfissionalProcedimento
│       └── DisponibilidadeProfissional
│
├── UsuarioTenant
│   └── UsuarioUnidade
│
└── Agendamento
    └── EventoAgendamento
```

---

# O que não deve ser bidirecional por padrão

Não é necessário ter:

```text
Tenant.getUnidades()
Tenant.getPessoas()
Paciente.getAgendamentos()
Profissional.getTodosAgendamentos()
Prontuario.getTodosAtendimentos()
```

Pode consultar via Repository.

Isso evita:

- ciclos;
- coleções grandes;
- N+1 acidental;
- problemas de serialização.

---

# Ordem de implementação

```text
1. Tenant
2. Unidade

3. Pessoa
4. Usuario
5. UsuarioTenant
6. UsuarioUnidade
7. RefreshToken

8. Especialidade
9. Procedimento
10. ProfissionalSaude
11. ProfissionalEspecialidade
12. ProfissionalUnidade
13. ProfissionalProcedimento
14. Paciente

15. DisponibilidadeProfissional
16. BloqueioAgenda
17. Agendamento
18. EventoAgendamento

19. Prontuario
20. Atendimento
21. EvolucaoClinica

22. Prescricao
23. ItemPrescricao
24. AnexoClinico

25. Notificacao
26. AuditLog
```

Cada grupo entra apenas na etapa do roadmap correspondente.

---

# Entidades futuras — não implementar agora

- ResponsavelPaciente, salvo necessidade concreta;
- MedicamentoCatalogo;
- Diagnostico estruturado;
- FormularioClinicoModelo;
- FormularioClinicoResposta;
- Cobranca;
- Pagamento;
- Convenio;
- Produto/Lote;
- Exame/Resultado;
- SolicitacaoPaciente;
- DevicePushToken;
- PlanoSaas/AssinaturaSaas.
