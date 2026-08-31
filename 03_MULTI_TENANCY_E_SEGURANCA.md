# 03 — Multi-Tenancy e Segurança

> Este documento define o modelo de isolamento, autenticação e autorização.  
> Para saber **em qual etapa o projeto está**, leia primeiro `CONTINUIDADE.md` e `00_DOSSIE_PROJETO_AKOSMED.md`.

---

# 1. Estratégia multi-tenant

MVP:

```text
Shared Database + Shared Schema
```

Toda entidade de domínio que pertence a uma organização deve possuir vínculo com:

```text
Tenant
```

No banco isso resulta em:

```text
tenant_id BIGINT
```

O `tenant_id` é interno. O contrato HTTP não recebe esse identificador livremente do cliente.

---

# 2. Tenant

Campos conceituais principais:

```text
id Long
publicId UUID
nome
nomeFantasia nullable
slug
documento nullable
status
timezone
createdAt
updatedAt
```

`slug` é único globalmente e pode participar do fluxo de autenticação.

Identificação:

```text
id       → somente backend/banco
publicId → API, DTOs, JWT quando aplicável e integrações
```

---

# 3. Login tenant-scoped

Request sugerida:

```json
{
  "email": "usuario@clinica.com",
  "senha": "...",
  "tenantSlug": "clinica-vida"
}
```

Processo:

1. localizar/validar `Usuario` pela credencial;
2. localizar `Tenant` pelo `slug`;
3. validar status do Tenant;
4. localizar `UsuarioTenant` ativo;
5. validar credencial;
6. emitir token tenant-scoped.

O `tenantSlug` serve para selecionar o contexto de autenticação. Ele não autoriza acesso por si só.

---

# 4. JWT

## Token tenant-scoped

Preferir claims externas baseadas em `publicId`:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfilTenant
superAdmin=false
```

O `sub` pode representar `usuarioPublicId`, desde que a convenção seja única e documentada.

Não usar como contrato externo:

```text
usuarioId Long
tenantId Long
usuarioTenantId Long
```

O filtro de segurança resolve os UUIDs públicos para as entidades/IDs internos necessários e mantém esses IDs apenas no contexto do servidor.

## Administração global

Um token administrativo específico pode conter:

```text
superAdmin=true
```

sem Tenant ativo, conforme o fluxo global definido para a plataforma.

## Nunca incluir no token

- dados clínicos;
- prontuário;
- CPF;
- lista extensa de unidades;
- senha/hash;
- informações que não sejam necessárias para autenticação/autorização.

Referência obrigatória: `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 5. TenantContext

O contexto é preenchido pelo filtro de autenticação por request.

Conceitualmente pode manter:

```text
tenantPublicId
usuarioPublicId
usuarioTenantPublicId
perfilTenant
```

Após resolver/autenticar essas referências, também pode manter internamente:

```text
tenantId Long
usuarioId Long
usuarioTenantId Long
```

Esses IDs internos **não fazem parte do contrato HTTP**.

O Service pode consultar:

```text
tenantContext.getTenantId()
```

para montar queries eficientes, desde que esse valor tenha sido resolvido pelo servidor e nunca recebido diretamente do DTO/request.

O Controller não recebe `tenantId` para escolher o tenant de uma operação tenant-scoped.

---

# 6. Repository tenant-scoped

Na fronteira de recursos acessados pela API, o padrão é:

```text
findByPublicIdAndTenantId(publicId, tenantIdInterno)
findAllByTenantId(tenantIdInterno, pageable)
existsBy...AndTenantId(..., tenantIdInterno)
```

Exemplo conceitual:

```java
Optional<Paciente> findByPublicIdAndTenantId(UUID publicId, Long tenantId);
```

Evitar em entrada pública:

```text
findById(Long id)
findByIdAndTenantId(Long id, ...)
```

`findById(Long)` pode existir para uso estritamente interno quando o recurso já foi resolvido e autorizado e o ID não veio do cliente.

---

# 7. Regra de erro cross-tenant

Quando um UUID existe, mas não pertence ao tenant atual, responder preferencialmente como recurso não encontrado:

```text
404 Not Found
```

Isso evita confirmar a existência de recursos externos.

Teste obrigatório:

```text
Tenant A cria recurso A
Tenant B tenta buscar o publicId de A
→ 404
```

UUID não substitui autorização.

---

# 8. Usuário global e Pessoa tenant-scoped

```text
Usuario
├── publicId
├── emailLogin
├── passwordHash
├── status
└── superAdmin

UsuarioTenant
├── publicId
├── usuario
├── tenant
├── pessoa
├── perfilTenant
└── acessoTodasUnidades
```

`Pessoa` pertence ao Tenant.

Assim, uma mesma credencial global pode participar de mais de um Tenant sem compartilhar automaticamente os dados de Pessoa entre organizações.

---

# 9. Perfis MVP

Privilégio global:

```text
Usuario.superAdmin
```

Perfis dentro do Tenant:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

Implementação inicial:

```text
PerfilTenant enum
+
Spring Security / @PreAuthorize quando adequado
+
validações de domínio no Service
```

Não criar tabela dinâmica de permissões no MVP.

Na ETAPA 9 consolidar a matriz:

```text
recurso × perfil × ação
```

Referência: `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

---

# 10. Acesso por Unidade

Se:

```text
UsuarioTenant.acessoTodasUnidades = true
```

o usuário pode acessar as unidades permitidas pelo seu perfil dentro do Tenant.

Caso contrário, validar vínculos em `UsuarioUnidade`.

Criar quando a ETAPA 2 chegar:

```text
UnitAccessService
```

Responsabilidade conceitual:

```text
assertCanAccess(unidadePublicId ou Unidade já resolvida)
```

Internamente o Service pode trabalhar com IDs numéricos depois da resolução/autorização.

Nunca aceitar `unidadeId Long` livre do cliente.

---

# 11. Paciente autenticado

Para perfil `PACIENTE`:

```text
UsuarioTenant.pessoa
   ↓
Paciente.pessoa
```

A API `/me` resolve o paciente a partir do usuário autenticado.

Nunca pedir ao app:

```text
pacienteId
pacientePublicId
```

para acessar os próprios dados do paciente.

Essa regra também vale para futuras orientações/materiais em:

```text
GET /api/v1/me/orientacoes
```

---

# 12. Profissional autenticado

Mesma ideia:

```text
UsuarioTenant.pessoa
   ↓
ProfissionalSaude.pessoa
```

Endpoints como:

```text
/me/agenda
/me/pacientes/aguardando
/me/resumo-diario
```

resolvem o profissional atual pelo contexto autenticado.

---

# 13. Senhas

Usar encoder forte fornecido/configurado pelo Spring Security.

Nunca:

- armazenar senha em plaintext;
- usar SHA simples como armazenamento de senha;
- retornar hash;
- registrar senha em log.

---

# 14. Refresh token

MVP prático:

```text
access token curto
+
refresh token persistido e revogável
```

Fluxo:

- refresh associado ao contexto autorizado;
- armazenar hash quando possível;
- logout revoga refresh token;
- rotação/hardening entra na ETAPA 9.

Não começar com arquitetura distribuída de sessão sem necessidade real.

---

# 15. Autorização de recursos clínicos

Toda operação sensível deve validar, conforme o caso:

1. tenant;
2. perfil;
3. ownership/vínculo do usuário;
4. acesso à unidade;
5. vínculo profissional-paciente/atendimento quando exigido pela regra;
6. estado do recurso.

A autorização não deve existir apenas no Controller.

Regras de domínio críticas permanecem no Service.

---

# 16. Acesso ao prontuário

Toda leitura de prontuário deve verificar:

1. tenant;
2. perfil;
3. unidade/regra aplicável;
4. vínculo necessário;
5. auditoria quando a ação for classificada como crítica.

Cross-tenant nunca deve revelar a existência do prontuário.

---

# 17. Auditoria

Registrar explicitamente eventos críticos.

Padrão inicial:

```text
AuditService.record(...)
```

chamado pelos Services relevantes.

Evitar AOP genérico na primeira versão se ele esconder quais operações realmente precisam ser auditadas.

---

# 18. Logs e correlation ID

Toda request deve aceitar ou gerar:

```http
X-Correlation-Id
```

O correlation ID acompanha logs e erros.

Não registrar conteúdo de:

- evolução clínica;
- prescrição completa;
- prontuário completo;
- CPF sem necessidade;
- JWT completo;
- refresh token;
- senha;
- anexos/mídias clínicas.

---

# 19. Segurança de produção

Planejar/validar nas etapas corretas:

- HTTPS;
- CORS restrito;
- headers de segurança;
- rate limit de autenticação;
- secrets fora do repositório;
- storage privado;
- limites/MIME de upload;
- backups e restore;
- observabilidade mínima.

Detalhes: `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

---

# 20. PostgreSQL RLS

Row Level Security pode ser uma defesa adicional futura.

Ela **não substitui**:

```text
queries tenant-aware
+
autorização
+
testes cross-tenant
```

Não adicionar RLS durante o Core H2.

Avaliar somente depois que a modelagem e os testes de isolamento estiverem estáveis no PostgreSQL.

---

# 21. Ordem prática de implementação

## ETAPA 2

Implementar o necessário para operar com isolamento:

- `Pessoa`;
- `Usuario`;
- `UsuarioTenant`;
- `UsuarioUnidade`;
- autenticação;
- JWT;
- refresh token;
- `TenantContext`;
- acesso por unidade;
- testes cross-tenant.

## ETAPA 9

Hardening:

- auditoria crítica;
- matriz de autorização;
- rate limiting;
- headers/CORS;
- refresh rotation/revogação;
- revisão de logs;
- storage privado;
- bateria final de isolamento.

Isso evita antecipar infraestrutura avançada sem adiar a segurança fundamental do tenant.

---

# 22. Desenvolvimento em H2 e revalidação PostgreSQL

O isolamento multi-tenant deve existir desde o H2.

PostgreSQL não é motivo para adiar:

- tenant context;
- `publicId`;
- queries tenant-scoped;
- validação de relacionamentos;
- autorização;
- testes cross-tenant.

Na ETAPA 11, a bateria crítica será repetida em PostgreSQL/Testcontainers.

---

# 23. Checklist mínimo antes de produção

- [ ] nenhum endpoint tenant-scoped aceita `tenantId Long` como seleção livre;
- [ ] nenhum recurso público usa PK `Long` em URL/DTO;
- [ ] JWT tenant-scoped usa identificadores públicos como contrato;
- [ ] nenhum Service público busca entidade tenant-scoped diretamente por `findById(idDoCliente)`;
- [ ] testes cross-tenant de leitura e escrita;
- [ ] refresh revogável;
- [ ] matriz de autorização testada;
- [ ] logs revisados;
- [ ] storage privado;
- [ ] backups e restore testado;
- [ ] HTTPS;
- [ ] política de retenção/LGPD avaliada com responsável adequado antes de uso real.

---

# Regra final

```text
publicId
→ identifica externamente

tenant context
→ determina organização ativa

autorização
→ determina o que o usuário pode fazer

IDs Long
→ permanecem internos ao backend/banco
```

Essas responsabilidades são complementares e não substituem umas às outras.
