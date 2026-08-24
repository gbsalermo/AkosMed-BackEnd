# 18 — Mapa de Métodos dos Services

Este documento ajuda a verificar se cada módulo possui responsabilidade coerente.

Os nomes podem mudar, mas a separação deve permanecer.

---

# TenantService

```text
criar(TenantCreateDTO)
buscarPorId(Long id)
listar(Pageable)
atualizar(Long id, TenantUpdateDTO)
ativar(Long id)
suspender(Long id)
buscarPorSlug(String slug)
```

Cuidados:

- slug;
- status;
- timezone.

---

# UnidadeService

```text
criar(UnidadeCreateDTO)
buscarPorId(Long id)
listar(Pageable)
atualizar(Long id, UnidadeUpdateDTO)
ativar(Long id)
desativar(Long id)
```

Internamente sempre tenant-scoped.

---

# PessoaService

Pode ser mais interno que público.

```text
criar(...)
buscarTenantScoped(...)
atualizar(...)
```

Pessoa não precisa de Controller genérico inicialmente se só nascer via Paciente/Profissional/Usuário.

---

# AuthService

```text
login(LoginDTO)
refresh(RefreshRequestDTO)
logout(RefreshRequestDTO)
```

Responsabilidades:

- credencial;
- tenant;
- vínculo;
- status;
- tokens.

---

# UsuarioService

```text
criarUsuarioTenant(...)
buscarUsuarioTenant(...)
ativarVinculo(...)
desativarVinculo(...)
alterarPerfil(...)
```

Não misturar emissão de token com CRUD administrativo.

---

# UsuarioUnidadeService

```text
adicionarUnidade(...)
removerUnidade(...)
listarUnidadesPermitidas(...)
validarAcessoUnidade(...)
```

---

# EspecialidadeService

```text
criar
buscar
listar
atualizar
ativar
desativar
```

---

# ProcedimentoService

```text
criar
buscar
listar
atualizar
ativar
desativar
```

---

# ProfissionalService

```text
criar(ProfissionalCreateDTO)
buscarPorId
listar
atualizar
ativar
desativar
```

Relacionamentos ficam em métodos próprios ou Services de vínculo.

---

# ProfissionalEspecialidadeService

```text
vincular
desvincular
listar
definirPrincipal
```

---

# ProfissionalUnidadeService

```text
vincular
desvincular
listar
validarVinculo
```

---

# ProfissionalProcedimentoService

```text
vincular
atualizarConfiguracao
desativar
listar
resolverDuracao
resolverValor
```

`resolverDuracao`:

```text
override profissional
senão
duracao padrão procedimento
```

---

# PacienteService

```text
criar
buscarPorId
listar
buscarPorCpf
buscarPorNumeroProntuario
atualizar
ativar
inativar
```

Pode criar Pessoa internamente.

---

# DisponibilidadeService

```text
criarDisponibilidade
atualizarDisponibilidade
remover/desativar
listarPorProfissional
buscarHorariosDisponiveis
```

Não salvar slots.

---

# BloqueioAgendaService

```text
criar
buscar
listarPorPeriodo
cancelar/desativar
```

---

# AgendamentoService

```text
criar
buscarPorId
listarPorPeriodo
listarPorProfissional
listarPorPaciente
listarPorUnidade
confirmar
cancelar
reagendar
checkIn
marcarFalta
```

Responsabilidades:

- disponibilidade;
- conflito;
- transição;
- evento.

---

# EventoAgendamentoService

Idealmente interno.

```text
registrarCriacao
registrarMudancaStatus
registrarReagendamento
```

Não precisa Controller inicialmente.

---

# ProntuarioService

```text
obterOuCriarDoPaciente
buscarDoPaciente
listarHistorico
```

Sem delete.

---

# AtendimentoService

```text
iniciarPorAgendamento
criarAvulso
buscarPorId
listarPorPaciente
concluir
```

Valida status do agendamento quando aplicável.

---

# EvolucaoClinicaService

```text
registrar
listarPorAtendimento
retificar
```

Sem update destrutivo.

---

# PrescricaoService

```text
criarRascunho
buscar
listarPorPaciente
listarPorAtendimento
adicionarItem
atualizarItem
removerItem
emitir
cancelar
```

Pode usar ItemPrescricaoRepository internamente.

Não precisa criar ItemPrescricaoService público se não houver vantagem.

---

# PrescriptionDocumentService

```text
gerar(Prescricao id)
```

No início pode retornar bytes/DTO.

Depois PDF/assinatura.

---

# StorageService

Interface pequena:

```text
salvar
abrir
remover
```

Implementações futuras:

```text
LocalStorageService
MinioStorageService
S3StorageService
```

---

# AnexoClinicoService

```text
upload
listarPorProntuario
listarPorAtendimento
download
inativar/removerLogicamente
```

Valida autorização antes de Storage.

---

# AgendaDiariaService

Query service.

```text
buscarMinhaAgendaHoje
buscarPacientesAguardando
buscarResumoDiario
```

Pode compor dados sem criar novas entidades.

---

# NotificacaoService

```text
criar
listarMinhas
marcarComoLida
marcarTodasComoLidas
```

---

# AuditService

```text
registrar(...)
```

Preferencialmente chamado por Services críticos.

Depois pode evoluir para interceptor/eventos.

---

# Regra de tamanho de Service

Se um Service começar a fazer:

```text
auth + paciente + agenda + prontuario + prescrição
```

ele está grande demais.

Separar por responsabilidade de domínio.

---

# Checklist Service

- [ ] um domínio claro;
- [ ] transação no lugar correto;
- [ ] tenant resolvido internamente;
- [ ] relacionamento validado;
- [ ] status validado;
- [ ] exceptions específicas;
- [ ] Repository não vazado ao Controller;
- [ ] ResponseDTO retornado;
- [ ] testes para regra crítica.
