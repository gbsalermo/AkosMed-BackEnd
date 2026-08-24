# 03 — Multi-Tenancy e Segurança

## Estratégia

MVP:

```text
Shared Database + Shared Schema
```

Toda entidade de domínio relevante possui:

```text
tenant_id
```

## Tenant

Campos práticos:

```text
id
nome
slug
status
timezone
criado_em
```

`slug` é único e pode ser usado na autenticação.

## Login

Request sugerida:

```json
{
  "email": "usuario@clinica.com",
  "senha": "...",
  "tenantSlug": "clinica-vida"
}
```

Processo:

1. validar `Usuario`;
2. localizar `Tenant` pelo slug;
3. validar `UsuarioTenant` ativo;
4. gerar token com tenant ativo.

## Token

Claims mínimas em token tenant-scoped:

```text
sub/userId
tenantId
usuarioTenantId
perfilTenant
superAdmin=false
```

Para administração global, um token específico pode conter `superAdmin=true` sem tenant ativo.

Não incluir dados clínicos ou lista enorme de unidades.

## TenantContext

Implementação simples:

```text
TenantContext.getTenantId()
```

Pode ser preenchido pelo filtro JWT por request.

O Service não recebe `tenantId` do DTO.

## Repository

Padrão obrigatório:

```text
findByIdAndTenantId
existsBy...AndTenantId
findAllByTenantId
```

## Regra de erro cross-tenant

Quando um recurso não pertence ao tenant atual, responder preferencialmente como não encontrado.

```text
404
```

Isso evita confirmar a existência de recursos externos.

## Usuário global e pessoa tenant-scoped

```text
Usuario
├── email
├── passwordHash
├── ativo
└── superAdmin

UsuarioTenant
├── usuario
├── tenant
├── pessoa
├── perfil
└── acessoTodasUnidades
```

`Pessoa` pertence ao tenant.

Assim, a mesma conta pode atuar em dois tenants sem compartilhar automaticamente dados pessoais entre eles.

## Perfis MVP

Privilégio global:

```text
Usuario.superAdmin
```

Perfis dentro do tenant:

```text
ADMIN_TENANT
SECRETARIA
PROFISSIONAL
PACIENTE
AUDITOR
```

Implementar `PerfilTenant` como enum + `@PreAuthorize`/regras no Service.

Não criar tabela de permissões agora.

## Unidade

Se `acessoTodasUnidades = true`, o usuário pode acessar qualquer unidade do tenant.

Caso contrário, validar `UsuarioUnidade`.

Criar:

```text
UnitAccessService
```

Responsabilidade:

```text
assertCanAccess(unidadeId)
```

## Paciente autenticado

Para perfil PACIENTE:

```text
UsuarioTenant.pessoa
   ↓
Paciente.pessoa
```

A API `/me` resolve o paciente a partir do usuário autenticado.

Nunca aceitar `pacienteId` do app para acessar dados próprios.

## Profissional autenticado

Mesma ideia:

```text
UsuarioTenant.pessoa
   ↓
ProfissionalSaude.pessoa
```

Endpoints `/me/agenda` resolvem o profissional atual.

## Senhas

Usar encoder forte fornecido pelo Spring Security.

Nunca:

- plaintext;
- SHA simples;
- senha em log.

## Refresh token

MVP prático:

- access token curto;
- refresh token persistido e revogável;
- logout revoga refresh token.

Não precisa começar com arquitetura complexa de sessões distribuídas.

## Auditoria

Registrar explicitamente eventos críticos.

`AuditService.record(...)` chamado nos Services relevantes.

Evitar AOP genérico na primeira versão porque dificulta enxergar o que está sendo auditado.

## Acesso ao prontuário

Toda leitura de prontuário deve verificar:

1. tenant;
2. perfil;
3. unidade/regra aplicável;
4. vínculo necessário.

E gerar audit log.

## Logs

Não registrar conteúdo de:

- evolução clínica;
- prescrição completa;
- CPF desnecessariamente;
- JWT;
- senha;
- anexos.

## Headers e produção

Planejar:

- HTTPS;
- CORS restrito;
- headers de segurança;
- rate limit em auth;
- secrets fora do repositório.

## RLS

PostgreSQL Row Level Security é uma camada futura útil, mas não deve substituir queries tenant-aware.

Adicionar somente depois que o Core estiver estável e os testes cross-tenant estiverem completos.

## Checklist mínimo antes de produção

- [ ] nenhum endpoint sem filtro tenant;
- [ ] nenhum `findById(id)` direto em entidade tenant-scoped;
- [ ] testes cross-tenant;
- [ ] refresh revogável;
- [ ] logs revisados;
- [ ] storage privado;
- [ ] backups;
- [ ] HTTPS;
- [ ] validação de LGPD/políticas de retenção com responsável adequado antes de uso real.


---

## Ordem prática de implementação

Segurança será construída em duas fases:

### ETAPA 2

Implementar o necessário para operar com segurança:

- autenticação;
- JWT;
- TenantContext;
- vínculo usuário-tenant;
- acesso por unidade;
- testes cross-tenant.

### ETAPA 9

Hardening:

- auditoria completa crítica;
- rate limiting;
- revisão de headers/CORS;
- refresh rotation;
- revisão de logs;
- storage privado;
- bateria final de isolamento.

Isso evita bloquear os CRUDs iniciais com infraestrutura avançada, sem adiar o isolamento de tenant.


---

## Desenvolvimento em H2 e revalidação

O isolamento multi-tenant deve existir desde o H2.

PostgreSQL não será usado como desculpa para adiar:

- `tenantId`;
- queries tenant-scoped;
- validação de relacionamentos;
- testes cross-tenant.

Na ETAPA 11, a bateria será repetida no PostgreSQL/Testcontainers.
