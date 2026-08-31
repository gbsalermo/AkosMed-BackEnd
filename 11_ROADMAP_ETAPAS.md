# 11 — ROADMAP DE DESENVOLVIMENTO — AkosMed

> Roadmap executável oficial.  
> Trabalhar **uma subetapa por vez**.  
> Só marcar `[x]` após implementação e teste executado.  
> Estado corrente e próxima tarefa: `CONTINUIDADE.md`.

---

# 0. COMO USAR ESTE ROADMAP

Antes de iniciar uma subetapa:

```text
CONTINUIDADE.md
↓
00_DOSSIE_PROJETO_AKOSMED.md
↓
subetapa deste roadmap
↓
documentos técnicos relacionados
```

Padrão fixo:

```text
1. confirmar regra
2. Entity/Enum quando aplicável
3. DTOs
4. Repository
5. Service
6. Controller
7. validações/exceptions
8. testes automatizados
9. execução local
10. checklist 19
11. atualizar CONTINUIDADE
12. commit
```

Não criar todas as Entities antes das camadas restantes.

Não desenvolver vários CRUDs em paralelo sem concluir a subetapa aberta.

---

# 1. GUARDRAILS QUE VALEM EM TODAS AS ETAPAS

## Identificação

Toda Entity persistida:

```text
Long id       → somente interno
UUID publicId → API/DTO/URL/integrações
```

## Multi-tenancy

```text
Shared Database + Shared Schema + tenant_id
```

Tenant vem do contexto autenticado. Nunca confiar em `tenantId` livre do request.

## API

- Entity não é contrato HTTP;
- Controller recebe/retorna DTO;
- relacionamento externo usa `...PublicId`;
- listas grandes são paginadas;
- ações de domínio têm endpoints explícitos;
- erros têm `code` estável + `correlationId`.

## JPA

- LAZY por padrão;
- sem `ManyToMany` no Core;
- sem `CascadeType.ALL` por conveniência;
- sem hard delete de histórico clínico.

## Consistência

- `Clock` em regras temporais;
- `@Version` seletivo quando houver lost update;
- idempotência apenas quando a operação realmente precisar;
- concorrência de agendamento protegida dentro da transação.

Referências transversais:

```text
15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md
16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md
17_PADROES_HTTP_ERROS_PAGINACAO.md
18_MAPA_METODOS_SERVICES.md
19_CHECKLIST_REVISAO_QUALIDADE.md
21_PUBLIC_ID_E_CONCORRENCIA.md
22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md
```

---

# ETAPA 0 — FUNDAÇÃO H2

## 0.1 Projeto base

- [x] criar repositório `gbsalermo/AkosMed-BackEnd`;
- [ ] gerar Spring Boot Maven;
- [ ] Java 21;
- [ ] package `br.com.akosmed`;
- [ ] dependências iniciais: Web, Data JPA, Validation, H2, Test;
- [ ] executar `mvn test`;
- [ ] iniciar aplicação;
- [ ] atualizar `CONTINUIDADE.md`;
- [ ] commit.

### Não adicionar em 0.1

```text
PostgreSQL Driver
Flyway
Spring Security/JWT
Springdoc/Swagger
mensageria
```

### Aceite

```text
BUILD SUCCESS
Aplicação inicia
Sem PostgreSQL
Sem Swagger
Sem entidades clínicas antecipadas
```

---

## 0.2 H2 e profiles

Criar:

```text
application.yml
application-dev.yml
application-test.yml
```

### dev

```text
H2 local
DDL controlado para desenvolvimento
console H2 opcional e apenas dev
```

### test

```text
H2 em memória
ddl-auto=create-drop ou estratégia equivalente de teste
```

- [ ] SQL logging controlado;
- [ ] nenhuma credencial real versionada;
- [ ] aplicação/testes usam profile correto.

---

## 0.3 Estrutura de packages

Criar somente o necessário, seguindo:

```text
br.com.akosmed
├── shared
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

Não criar dezenas de classes vazias para preencher estrutura.

---

## 0.4 Tratamento global de erros

Criar:

- [ ] `ApiErrorDTO`;
- [ ] `FieldErrorDTO`;
- [ ] `GlobalExceptionHandler`;
- [ ] `ResourceNotFoundException`;
- [ ] `BusinessRuleException`/`ConflictException` conforme padronização.

Testar:

- [ ] 400 validação;
- [ ] 404 recurso inexistente;
- [ ] 409 conflito/regra;
- [ ] `correlationId` no erro.

---

## 0.5 Clock + Correlation ID

- [ ] bean `Clock` centralizado;
- [ ] regra temporal preparada para injeção de Clock;
- [ ] `CorrelationIdFilter`;
- [ ] aceitar ou gerar `X-Correlation-Id`;
- [ ] MDC/log quando configurado;
- [ ] devolver o mesmo ID em `ApiErrorDTO`.

---

## 0.6 Smoke tests

- [ ] `contextLoads`;
- [ ] teste simples de Controller/erro;
- [ ] teste do correlationId;
- [ ] teste com `Clock.fixed`;
- [ ] `mvn test` verde.

### ETAPA 0 concluída quando

A base Spring Boot estiver estável em H2 e preparada para iniciar o primeiro CRUD sem infraestrutura futura antecipada.

---

# ETAPA 1 — TENANT E UNIDADE

## 1.1 Tenant

Campos principais:

```text
id
publicId
nome
nomeFantasia
slug
documento
status
timezone
createdAt
updatedAt
```

Métodos esperados:

```text
criar
buscarPorPublicId
listar
atualizar
ativar
suspender
buscarPorSlug
```

Regras:

- [ ] `id Long` interno;
- [ ] `publicId UUID` imutável;
- [ ] API não expõe `id`;
- [ ] slug obrigatório/único global;
- [ ] timezone obrigatório;
- [ ] sem hard delete;
- [ ] status alterado por ação;
- [ ] `@Version` avaliado.

Testes:

- [ ] criar;
- [ ] publicId gerado/imutável;
- [ ] duplicidade de slug;
- [ ] buscar por UUID;
- [ ] atualizar;
- [ ] suspender/ativar.

---

## 1.2 Unidade

Campos:

```text
id
publicId
tenant
nome
codigo
telefone
email
endereco
ativo
createdAt
updatedAt
```

Regras:

- [ ] código único dentro do Tenant;
- [ ] Unidade sempre possui Tenant;
- [ ] Tenant suspenso não recebe nova Unidade;
- [ ] sem hard delete;
- [ ] publicId externo;
- [ ] `@Version` avaliado.

Repository mínimo:

```text
findByPublicIdAndTenantId
findAllByTenantId
existsByCodigoAndTenantId
```

Testes críticos:

- [ ] mesmo código em Tenants diferentes funciona;
- [ ] mesmo código no mesmo Tenant falha;
- [ ] UUID de outro Tenant não retorna;
- [ ] Response não expõe Long id.

---

## 1.3 Revisão da Etapa 1

- [ ] `19_CHECKLIST_REVISAO_QUALIDADE.md` executado;
- [ ] `mvn test` verde;
- [ ] `CONTINUIDADE.md` atualizado;
- [ ] commit/revisão antes de avançar.

---

# ETAPA 2 — IDENTIDADE, AUTH E MULTI-TENANCY

Adicionar agora:

```text
Spring Security
biblioteca JWT escolhida
```

Não adicionar autenticação antes desta etapa apenas por conveniência.

---

## 2.1 Pessoa

Tenant-scoped.

Campos:

```text
id
publicId
tenant
nomeCompleto
nomeSocial
cpf
dataNascimento
telefone
emailContato
createdAt
updatedAt
```

Regras:

- [ ] CPF pode ser nullable;
- [ ] se informado, único por Tenant;
- [ ] e-mail de contato não é automaticamente credencial;
- [ ] publicId externo;
- [ ] `@Version` avaliado.

---

## 2.2 Usuario

Credencial global.

Campos:

```text
id
publicId
emailLogin
passwordHash
status
superAdmin
ultimoLogin
createdAt
updatedAt
```

Regras:

- [ ] email de login único global;
- [ ] senha com encoder forte;
- [ ] hash nunca retorna;
- [ ] publicId no contrato externo;
- [ ] sem senha/token em logs.

---

## 2.3 UsuarioTenant

Liga:

```text
Usuario
Tenant
Pessoa
PerfilTenant
```

Campos:

```text
id
publicId
usuario
tenant
pessoa
perfilTenant
acessoTodasUnidades
ativo
createdAt
```

Constraint:

```text
unique(usuario_id, tenant_id)
```

Service valida consistência entre Tenant/Pessoa/Usuário.

---

## 2.4 UsuarioUnidade

Usar quando:

```text
acessoTodasUnidades = false
```

Constraint:

```text
unique(usuario_tenant_id, unidade_id)
```

- [ ] publicId conforme convenção global;
- [ ] mesmo Tenant validado;
- [ ] acesso por Unidade testado.

---

## 2.5 Login tenant-scoped

Request:

```json
{
  "email": "user@exemplo.com",
  "senha": "...",
  "tenantSlug": "clinica-vida"
}
```

Fluxo:

```text
Usuario
↓
Tenant por slug
↓
UsuarioTenant ativo
↓
credencial
↓
token
```

Claims preferenciais:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfilTenant
superAdmin=false
```

Não usar IDs Long sequenciais como contrato do JWT.

---

## 2.6 RefreshToken

Campos conceituais:

```text
id
publicId
usuario
usuarioTenant nullable
tokenHash
expiresAt
revogadoEm
createdAt
```

Regras:

- [ ] guardar hash quando possível;
- [ ] revogação no logout;
- [ ] não logar token puro;
- [ ] fluxo global de superAdmin só se necessário.

---

## 2.7 TenantContext

Extrair/resolver contexto pelo token.

O Controller não recebe `tenantId` para escolher o Tenant.

O servidor pode manter `Long tenantId` internamente após resolver `tenantPublicId`.

---

## 2.8 Testes cross-tenant e autorização básica

Criar Tenant A/B e usuários A/B.

Testar:

- [ ] leitura cruzada;
- [ ] escrita cruzada;
- [ ] relacionamento usando UUID de outro Tenant;
- [ ] perfil negado/permitido;
- [ ] acesso por Unidade;
- [ ] refresh revogado.

### ETAPA 2 concluída quando

Isolamento/autenticação estiverem comprovados e a API não depender de PK Long externa.

---

# ETAPA 3 — ESTRUTURA CLÍNICA BASE

## 3.1 Especialidade

Campos:

```text
id
publicId
tenant
nome
codigo
descricao
ativo
```

Regras:

- código único por Tenant;
- inativação em vez de delete;
- publicId externo.

---

## 3.2 Procedimento

Campos:

```text
id
publicId
tenant
nome
codigo
descricao
duracaoPadraoMinutos
valorPadrao
ativo
```

Regras:

- duração > 0;
- dinheiro com `BigDecimal`;
- código único por Tenant.

---

## 3.3 ProfissionalSaude

Campos:

```text
id
publicId
tenant
pessoa
tipoProfissional
conselho
numeroRegistro
ufRegistro
ativo
```

Regras:

- Pessoa no mesmo Tenant;
- não duplicar registro profissional quando os dados necessários existirem;
- publicId externo.

---

## 3.4 ProfissionalEspecialidade

Entidade de vínculo explícita.

Campos:

```text
id
publicId
profissional
especialidade
principal
```

Não usar `@ManyToMany`.

---

## 3.5 ProfissionalUnidade

Entidade de vínculo.

Profissional só pode atender/agendar em Unidade vinculada e ativa.

---

## 3.6 ProfissionalProcedimento

Entidade de vínculo.

Campos extras:

```text
duracaoMinutosOverride
valorOverride
ativo
```

Duração efetiva:

```text
override
senão
duração padrão do Procedimento
```

---

## 3.7 Paciente

Campos:

```text
id
publicId
tenant
pessoa
numeroProntuario
status
observacaoAdministrativa
createdAt
updatedAt
```

Regras:

- Pessoa mesmo Tenant;
- número de prontuário único por Tenant;
- sem `PacienteUnidade` no MVP;
- inativar em vez de apagar;
- publicId externo.

### ETAPA 3 concluída quando

Cadastros clínicos básicos estiverem consistentes, paginados quando necessário e isolados por Tenant.

---

# ETAPA 4 — DISPONIBILIDADE E BLOQUEIOS

## 4.1 DisponibilidadeProfissional

Campos:

```text
id
publicId
tenant
profissional
unidade
diaSemana
horaInicio
horaFim
vigenciaInicio
vigenciaFim
ativo
```

Regras:

- profissional vinculado à Unidade;
- início < fim;
- mesmo Tenant;
- sem slots persistidos;
- `@Version` avaliado.

---

## 4.2 BloqueioAgenda

Campos:

```text
id
publicId
tenant
profissional
unidade
inicio
fim
tipo
motivo
ativo
criadoPorUsuario
createdAt
```

No MVP, Unidade é obrigatória.

Para bloquear todas as Unidades, criar um bloqueio por Unidade.

---

## 4.3 AvailabilityService

Método principal conceitual:

```text
buscarHorariosDisponiveis(
    profissionalPublicId,
    unidadePublicId,
    procedimentoPublicId,
    data
)
```

Considerar:

- disponibilidade;
- procedimento;
- override;
- bloqueios;
- agendamentos que ocupam agenda;
- duração;
- timezone do Tenant.

Não persistir slots.

### Testes

- sem disponibilidade;
- janelas separadas;
- bloqueio parcial/total;
- duração override;
- intervalo adjacente;
- cross-tenant.

---

# ETAPA 5 — AGENDAMENTO

## 5.1 Agendamento

Campos:

```text
id
publicId
tenant
unidade
paciente
profissional
procedimento
inicio
fim
status
origem
observacaoAdministrativa
criadoPorUsuario
createdAt
updatedAt
```

Service de criação deve:

1. resolver Tenant;
2. carregar paciente por `publicId + Tenant`;
3. adquirir lock no profissional dentro da transação;
4. validar vínculo profissional-Unidade;
5. carregar procedimento;
6. determinar duração;
7. validar disponibilidade;
8. validar bloqueios;
9. validar overlap;
10. salvar;
11. registrar `EventoAgendamento`;
12. criar notificação quando a regra exigir.

---

## 5.2 Listagens

Métodos:

```text
buscarPorPublicId
listarPorData
listarPorProfissionalEPeriodo
listarPorPaciente
listarPorUnidadeEData
```

Paginar quando o volume puder crescer.

---

## 5.3 Ações

```text
confirmar
cancelar
reagendar
checkIn
marcarFalta
```

Cada ação valida transição.

Não usar `PUT /agendamentos/{publicId}` genérico para alterar livremente status/horário.

---

## 5.4 EventoAgendamento

Histórico append-only.

Registrar:

- criação;
- mudança de status;
- reagendamento;
- motivo/usuário quando aplicável.

Não editar eventos existentes.

---

## 5.5 Overlap e concorrência

Intervalo:

```text
[inicio, fim)
```

Overlap:

```text
existing.inicio < novoFim
AND
existing.fim > novoInicio
```

Status iniciais que ocupam agenda:

```text
SOLICITADO
CONFIRMADO
CHECK_IN
EM_ATENDIMENTO
```

Cancelados/FALTOU não ocupam.

Centralizar essa regra.

### Fluxo concorrente

```text
@Transactional
→ PESSIMISTIC_WRITE no profissional
→ revalidar tudo
→ consultar overlap
→ salvar
```

Resultado obrigatório:

```text
2 pacientes + mesmo profissional + mesmo slot
→ 1 sucesso
→ 1 HTTP 409 AGENDAMENTO_CONFLITO
```

Criação e reagendamento reutilizam a mesma proteção.

---

## 5.6 Idempotência de criação

Avaliar `Idempotency-Key` no `POST /agendamentos`.

Se adotado:

- mesma key + mesmo payload não duplica efeito;
- mesma key + payload diferente → 409.

Não construir infraestrutura genérica se a decisão da subetapa concluir que ainda não é necessária.

---

## 5.7 Testes H2

- [ ] overlap funcional;
- [ ] adjacência permitida;
- [ ] status que bloqueiam/liberam;
- [ ] reagendamento conflitante;
- [ ] cross-tenant;
- [ ] teste concorrente básico;
- [ ] lock JPA configurado.

A garantia definitiva é revalidada na ETAPA 11 com PostgreSQL/Testcontainers.

---

# ETAPA 6 — PRONTUÁRIO E ATENDIMENTO

## 6.1 Prontuario

Criado sob demanda para Paciente.

Constraint:

```text
unique(tenant_id, paciente_id)
```

Sem endpoint de delete.

---

## 6.2 Atendimento

Campos:

```text
id
publicId
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

Regras:

- profissional/Prontuário no mesmo Tenant;
- Agendamento, se informado, é compatível;
- Atendimento pode ser avulso;
- não duplicar `paciente_id`;
- considerar `agendamento_id UNIQUE` quando informado.

---

## 6.3 EvolucaoClinica

Campos:

```text
id
publicId
tenant
atendimento
profissional
conteudo
retificacaoDe nullable
motivoRetificacao nullable
createdAt
```

Regras:

- append-only;
- original não é sobrescrito;
- retificação cria novo registro;
- sem delete destrutivo;
- autorização/auditoria conforme regra.

---

# ETAPA 7 — PRESCRIÇÃO, DOCUMENTO E ANEXOS

## 7.1 Prescricao

Estados:

```text
RASCUNHO
EMITIDA
CANCELADA
```

Métodos:

```text
criarRascunho
buscarPorPublicId
listarPorPaciente
listarPorAtendimento
emitir
cancelar
```

Paciente é derivado do Atendimento/Prontuário.

---

## 7.2 ItemPrescricao

Manipulável apenas enquanto Prescrição estiver em RASCUNHO.

Métodos:

```text
adicionarItem
atualizarItem
removerItem
```

Após EMITIDA:

- não editar silenciosamente;
- preservar histórico;
- correção segue cancelamento/substituição conforme regra definida.

---

## 7.3 Gerador de receita

No Core, gerar representação a partir de `Prescricao`.

Assinatura digital fica para etapa futura.

Idempotência de emissão:

- [ ] avaliar `Idempotency-Key`;
- [ ] retry não pode emitir efeito duplicado.

---

## 7.4 StorageService

Criar abstração pequena:

```text
salvar
abrir
remover/inativar referência
```

Desenvolvimento:

```text
filesystem local
```

Produção futura:

```text
S3/storage compatível
```

Não acoplar domínio diretamente a SDK específico.

---

## 7.5 AnexoClinico

Banco guarda metadata e `storageKey` interno.

Arquivo real fica no StorageService.

Regras:

- autorização antes de download;
- storageKey não aparece no ResponseDTO;
- sem BLOB grande no banco;
- limites/MIME progressivamente definidos;
- falhas não deixam persistência inconsistente.

Essa infraestrutura será reutilizada depois por `OrientacaoPaciente`, mas **não implementar OrientacaoPaciente agora**.

---

# ETAPA 8 — OPERAÇÃO DIÁRIA E NOTIFICAÇÕES

## 8.1 Agenda diária

Criar Query Service para visão do usuário autenticado.

Exemplos:

```text
buscarMinhaAgendaHoje
buscarMinhaAgenda(data)
```

---

## 8.2 Pacientes aguardando

Baseado em status `CHECK_IN` e autorização do profissional/Unidade.

---

## 8.3 Pendências

Começar calculadas por consulta:

- atendimentos abertos;
- prescrições em rascunho;
- consultas aguardando confirmação;
- pacientes em check-in.

Não criar entidade/motor genérico de Pendência sem necessidade.

---

## 8.4 Notificacao

Primeira versão:

```text
in-app
```

Sem WhatsApp/Telegram/push ainda.

Endpoints `/me` resolvem o usuário autenticado.

---

# ETAPA 9 — AUDITORIA E HARDENING DE SEGURANÇA

## 9.1 AuditLog

Registrar ações críticas.

Preferir IDs internos simples/metadata segura em vez de grafo JPA pesado.

Não logar conteúdo clínico desnecessário.

---

## 9.2 Segurança

- [ ] refresh rotation/revogação;
- [ ] CORS;
- [ ] headers de segurança;
- [ ] rate limit em auth;
- [ ] revisão de logs;
- [ ] secrets;
- [ ] correlationId completo;
- [ ] acesso por Unidade;
- [ ] storage privado;
- [ ] revisão de endpoints.

---

## 9.3 Matriz de autorização

Perfis:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

- [ ] recurso × perfil × ação definido;
- [ ] testes de acesso permitido;
- [ ] testes de acesso negado;
- [ ] cross-tenant permanece 404 quando aplicável.

---

## 9.4 Optimistic locking

Revisar entidades mutáveis:

- [ ] `@Version` onde lost update for relevante;
- [ ] duas atualizações concorrentes testadas;
- [ ] conflito convertido em `409 RESOURCE_VERSION_CONFLICT`;
- [ ] histórico append-only não recebe `@Version` por reflexo.

---

# ETAPA 10 — REVISÃO FUNCIONAL COMPLETA EM H2

Antes de PostgreSQL:

- [ ] todos os testes passam;
- [ ] `mvn test` verde;
- [ ] nenhuma regra depende de SQL H2-specific sem necessidade;
- [ ] relacionamentos revisados;
- [ ] cascades/orphanRemoval revisados;
- [ ] publicId revisado em toda API;
- [ ] nenhum Long id exposto;
- [ ] endpoints revisados;
- [ ] paginação revisada;
- [ ] exceptions/correlationId revisados;
- [ ] Tenant isolation revisado;
- [ ] Clock revisado;
- [ ] `@Version` revisado;
- [ ] código morto removido;
- [ ] TODO crítico resolvido.

Fluxo automatizado mínimo:

```text
Tenant
→ Unidade
→ Usuário/Auth
→ Profissional
→ Paciente
→ Disponibilidade
→ Agendamento
→ Confirmação
→ Check-in
→ Atendimento
→ Evolução
→ Prescrição
→ Conclusão
```

### Gate

Não iniciar PostgreSQL se o domínio ainda estiver mudando por erros básicos do Core.

---

# ETAPA 11 — POSTGRESQL + FLYWAY + TESTCONTAINERS

Agora adicionar:

```text
PostgreSQL Driver
Flyway
Testcontainers PostgreSQL
```

---

## 11.1 Revisão de modelagem

Antes das migrations:

- [ ] nomes de tabelas/colunas;
- [ ] FKs;
- [ ] `public_id UUID NOT NULL UNIQUE`;
- [ ] índices;
- [ ] uniques compostos;
- [ ] nullability;
- [ ] BigDecimal/NUMERIC;
- [ ] timestamps/UTC;
- [ ] varchar/TEXT;
- [ ] constraints de status/integridade quando adequadas.

---

## 11.2 Migrations Flyway

Criar migrations por blocos lógicos a partir do modelo estabilizado.

Produção:

```text
ddl-auto=validate
```

Testar banco limpo executando todas as migrations sem alteração manual.

---

## 11.3 Reexecutar testes em PostgreSQL

Obrigatório:

- multi-tenant;
- constraints;
- publicId;
- paginação;
- queries;
- locks;
- concorrência;
- transações;
- timezone;
- optimistic locking.

---

## 11.4 Double booking real

Testcontainers/PostgreSQL:

- [ ] disparar duas transações simultâneas;
- [ ] mesmo profissional/período;
- [ ] `PESSIMISTIC_WRITE` no profissional;
- [ ] segunda transação espera e revalida;
- [ ] exatamente uma persiste;
- [ ] outra resulta em `AGENDAMENTO_CONFLITO`;
- [ ] repetir reagendamento concorrente;
- [ ] testar profissionais diferentes no mesmo horário;
- [ ] testar slots adjacentes.

A ETAPA 11 não termina se corrida concorrente permitir overlap.

---

## 11.5 Exclusion constraint PostgreSQL

Avaliar/adicionar como segunda barreira:

```text
Service validation
+
PESSIMISTIC_WRITE
+
PostgreSQL exclusion constraint
```

Usar `tstzrange(..., '[)')` e filtrar status que ocupam agenda conforme nomes/tipos reais da migration.

Testar a constraint em Testcontainers.

---

# ETAPA 12 — SWAGGER / OPENAPI

Somente agora adicionar Springdoc.

Documentar:

- tags por módulo;
- UUID publicId;
- DTOs;
- exemplos;
- status HTTP;
- códigos de erro;
- correlationId;
- JWT;
- filtros;
- paginação;
- ações de domínio;
- headers de idempotência onde existirem.

Não documentar endpoint instável como contrato final.

---

# ETAPA 13 — POSTMAN + VALIDAÇÃO PONTA A PONTA

Criar coleção oficial:

```text
00 Auth
01 Tenant
02 Unidade
03 Usuários
04 Especialidades
05 Procedimentos
06 Profissionais
07 Pacientes
08 Disponibilidade
09 Agendamentos
10 Prontuário
11 Atendimento
12 Prescrições
13 Anexos
14 Operação diária
15 Auditoria
```

Environment:

```text
baseUrl
accessToken
tenantSlug
publicIds de teste
```

Nunca depender de Long IDs copiados do banco.

Validar:

- happy paths;
- auth;
- publicId;
- cross-tenant;
- status inválidos;
- conflitos;
- fluxo E2E completo.

---

# ETAPA 14 — FECHAMENTO BACKEND MVP 1.0

## 14.1 Operação/produção

- [ ] Actuator/health;
- [ ] secrets fora do Git;
- [ ] limites definitivos de upload;
- [ ] MIME/extensões permitidas;
- [ ] estratégia de backup;
- [ ] retenção/criptografia;
- [ ] procedimento de restore;
- [ ] teste real de restore;
- [ ] logs/correlationId revisados;
- [ ] observabilidade mínima;
- [ ] HTTPS/configuração de produção documentada.

Backup sem teste de restore não é considerado concluído.

---

## 14.2 Fechamento técnico

- [ ] testes H2;
- [ ] testes PostgreSQL/Testcontainers;
- [ ] migrations;
- [ ] concorrência real;
- [ ] Swagger;
- [ ] Postman;
- [ ] README de execução;
- [ ] revisão de segurança;
- [ ] documentação atualizada;
- [ ] `CONTINUIDADE.md` registra MVP fechado;
- [ ] release/tag `Backend MVP 1.0`.

### Gate pós-MVP

Somente depois desse fechamento avançar para funcionalidades específicas de paciente/mobile já planejadas abaixo.

---

# ETAPA 15 — API DO PACIENTE + ORIENTAÇÕES/MATERIAIS TERAPÊUTICOS

> **Pós-MVP.** O Core já deve estar fechado, PostgreSQL/Flyway/Swagger/Postman em operação.

Objetivo: expor uma API `/me` segura para o paciente e adicionar o domínio `OrientacaoPaciente`, inicialmente motivado por vídeos de exercícios enviados por fisioterapeutas.

Referência principal: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

## 15.1 API `/me` do paciente

Implementar/estabilizar:

```http
GET /api/v1/me/perfil
GET /api/v1/me/consultas
GET /api/v1/me/consultas/proximas
GET /api/v1/me/prescricoes
GET /api/v1/me/prescricoes/{publicId}
GET /api/v1/me/notificacoes
```

Regra:

```text
UsuarioTenant.pessoa
→ Paciente
```

O paciente não envia `pacientePublicId` para acessar os próprios dados.

Testar:

- paciente correto;
- outro paciente não acessa;
- cross-tenant;
- perfil não-paciente;
- paginação quando necessária.

---

## 15.2 Autoagendamento do paciente — se aprovado para esta entrega

Endpoints planejados:

```http
GET  /api/v1/me/horarios-disponiveis
POST /api/v1/me/agendamentos
POST /api/v1/me/agendamentos/{publicId}/cancelar
POST /api/v1/me/agendamentos/{publicId}/reagendar
```

Reutilizar **o mesmo** `AgendamentoService` e as mesmas proteções de concorrência do fluxo administrativo.

Não criar uma segunda regra de agenda exclusiva para o app.

Se o escopo comercial decidir adiar autoagendamento, registrar no `CONTINUIDADE.md` sem bloquear as orientações.

---

## 15.3 OrientacaoPaciente — domínio

Criar Entity apenas agora.

Campos planejados:

```text
id
publicId
tenant
paciente
profissional
atendimento nullable
titulo
descricao
tipo
storageKey nullable
urlExterna nullable
mimeType nullable
duracaoSegundos nullable
visivelAPartirDe nullable
expiraEm nullable
ativo
createdAt
```

Enum:

```text
VIDEO
DOCUMENTO
LINK
TEXTO
```

Primeiro caso de uso:

```text
fisioterapeuta
→ paciente
→ vídeo demonstrando exercício correto
→ instruções
→ visualização no AkosMed Patient
```

Não criar `VideoFisioterapia` específico.

---

## 15.4 Regras de OrientacaoPaciente

- [ ] `id Long` interno + `publicId UUID` externo;
- [ ] paciente/profissional no mesmo Tenant;
- [ ] Atendimento, se informado, compatível com paciente/profissional/Tenant;
- [ ] profissional autorizado para enviar;
- [ ] paciente não altera conteúdo enviado;
- [ ] inativação/remoção lógica quando necessário;
- [ ] criação/remoção auditada conforme matriz;
- [ ] `Clock` para visibilidade/expiração;
- [ ] `@Version` avaliado se houver edição administrativa real.

---

## 15.5 Storage de materiais

Não armazenar binário de vídeo no PostgreSQL.

Banco guarda metadata/referência.

Reutilizar `StorageService`.

Regras:

- [ ] `storageKey` nunca sai no ResponseDTO;
- [ ] autorização antes de gerar acesso;
- [ ] URL temporária/assinada quando infraestrutura suportar;
- [ ] tamanho máximo;
- [ ] MIME/extensões permitidas;
- [ ] não permitir executáveis;
- [ ] nome original não vira chave física diretamente;
- [ ] falhas de upload/persistência não deixam lixo/inconsistência.

---

## 15.6 Endpoints do profissional

Planejados:

```http
POST   /api/v1/pacientes/{pacientePublicId}/orientacoes
GET    /api/v1/pacientes/{pacientePublicId}/orientacoes
DELETE /api/v1/pacientes/{pacientePublicId}/orientacoes/{orientacaoPublicId}
```

Se `DELETE` for usado, representar a semântica definida de inativação/remoção lógica sem destruir histórico necessário.

---

## 15.7 Endpoints do paciente

```http
GET /api/v1/me/orientacoes
GET /api/v1/me/orientacoes/{orientacaoPublicId}
```

O backend resolve o Paciente do contexto autenticado.

Nunca aceitar `pacientePublicId` do app nesse fluxo.

---

## 15.8 Testes de OrientacaoPaciente

Obrigatórios:

- [ ] profissional autorizado cria;
- [ ] publicId gerado/imutável;
- [ ] paciente só lista as próprias;
- [ ] UUID de outro paciente → 404/negado sem vazamento;
- [ ] cross-tenant;
- [ ] Atendimento incompatível;
- [ ] storageKey não exposto;
- [ ] MIME/tamanho inválido;
- [ ] expiração/visibilidade com `Clock.fixed`;
- [ ] remoção lógica/auditoria;
- [ ] acesso ao arquivo depende de autorização.

---

## 15.9 Atualizar documentação final

Como PostgreSQL/Swagger/Postman já existem nessa fase:

- [ ] migration Flyway de `OrientacaoPaciente`;
- [ ] testes Testcontainers;
- [ ] Swagger atualizado;
- [ ] coleção Postman atualizada;
- [ ] `CONTINUIDADE.md` atualizado.

### ETAPA 15 concluída quando

A API do paciente e o fluxo de orientações estiverem seguros/estáveis para serem consumidos pelo app Kotlin.

---

# ETAPA 16 — AKOSMED PATIENT / KOTLIN + ORIENTAÇÕES E VÍDEOS

> Projeto mobile separado. O backend continua sendo a fonte das regras.

Tecnologia planejada:

```text
Kotlin / Android
```

---

## 16.1 Fundação do app

- [ ] projeto separado;
- [ ] configuração de ambientes/baseUrl;
- [ ] cliente HTTP;
- [ ] armazenamento seguro de token;
- [ ] tratamento padronizado de erros por `code`;
- [ ] sessão/logout/refresh;
- [ ] não guardar dados clínicos desnecessários em cache/log.

---

## 16.2 MVP inicial do app

- [ ] login;
- [ ] próximas consultas;
- [ ] detalhes da consulta;
- [ ] prescrições;
- [ ] notificações.

Consumir `/api/v1/me/*`.

O app não implementa regra de Tenant/agendamento/prescrição duplicada localmente.

---

## 16.3 Orientações do profissional

Criar área:

```text
Orientações do profissional
```

Exibir:

- profissional;
- título;
- instrução;
- data;
- tipo/material;
- vínculo com Atendimento quando aplicável.

---

## 16.4 Vídeos

Para `VIDEO`:

- [ ] obter acesso autorizado pelo backend;
- [ ] consumir URL temporária/assinada;
- [ ] player adequado;
- [ ] tratar expiração da URL;
- [ ] não transformar URL privada em link público permanente;
- [ ] mensagens de erro amigáveis sem esconder `code` para diagnóstico.

Exemplo de uso:

```text
Exercício para mobilidade do ombro
Enviado por: profissional responsável
Orientação: instruções definidas pelo profissional
[Assistir vídeo]
```

O paciente não edita o material.

---

## 16.5 Segunda fase do app

Depois do fluxo inicial validado:

- horários disponíveis;
- autoagendamento;
- cancelamento/reagendamento;
- documentos;
- solicitações administrativas.

Não começar por chat em tempo real.

---

# ETAPA 17 — AKOS ASSISTANT

Consumir Services/endpoints já existentes.

Objetivo:

- agenda;
- próxima consulta;
- pacientes aguardando;
- cancelamentos;
- atendimentos abertos;
- pendências;
- resumo diário;
- alertas.

Regra:

```text
bot/app/adapter não contém regra de negócio crítica
```

Fases possíveis:

```text
endpoints/resumos
↓
push/app profissional
↓
bot Telegram/WhatsApp se houver valor real
```

Não criar backend paralelo.

---

# ETAPA 18 — ESPECIALIZAÇÕES CLÍNICAS

Entram somente com necessidade concreta.

Possíveis módulos:

```text
Akos Dental
Akos Psi
Akos Vision
```

Regras:

- reutilizar Paciente/Profissional/Atendimento/Prontuário;
- especialização referencia `Atendimento` como raiz clínica;
- não duplicar Core;
- etapa própria por especialização;
- requisitos/testes próprios.

---

# ETAPA 19 — MÓDULOS ADICIONAIS E INTEGRAÇÕES

Somente com caso real definido.

Possíveis:

```text
Financeiro clínico
Convênios
Estoque
Laboratório
TISS
RNDS
Assinatura digital
WhatsApp/e-mail/SMS
Gateway de pagamento
Calendário externo
SaaS billing
```

Cada módulo deve ter:

1. necessidade concreta;
2. escopo;
3. modelagem;
4. etapa própria;
5. migrations;
6. segurança;
7. testes;
8. Swagger/Postman atualizados.

Não incluir no Core preventivamente.

---

# REGRA DE RETOMADA

Sempre:

```text
1. abrir CONTINUIDADE.md
2. confirmar estado no repositório
3. abrir 00_DOSSIE_PROJETO_AKOSMED.md
4. identificar a próxima subetapa neste arquivo
5. abrir documentos técnicos correspondentes
6. implementar somente essa subetapa
7. executar testes
8. passar checklist 19
9. atualizar CONTINUIDADE
10. commit
11. avançar apenas depois da validação
```

---

# REGRA DE STATUS

Não marcar `[x]` por intenção, documentação ou código não executado.

```text
[x] = implementado + testado + revisado
[ ] = ainda não comprovado
```

Se outra IA assumir o projeto, ela deve preservar essa semântica.

---

# ESTADO NA REVISÃO DOCUMENTAL DE 2026-08-31

Na data desta revisão:

```text
ETAPA 0 aberta
repositório/documentação existentes
Spring Boot ainda não gerado
próxima tarefa = 0.1 Projeto base
```

O estado futuro deve ser consultado em `CONTINUIDADE.md`, que é atualizado a cada subetapa.
