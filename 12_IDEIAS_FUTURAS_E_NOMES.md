# 12 — Visão Futura do Produto

> Este arquivo registra direção de produto **sem alterar o Core nem antecipar implementação**.  
> Ordem oficial: `11_ROADMAP_ETAPAS.md`. Estado real: `CONTINUIDADE.md`.

---

# Marca

Nome definido:

# AkosMed

Ecossistema possível:

```text
AkosMed
├── Akos Core
├── AkosMed Web
├── AkosMed Patient
├── Akos Assistant
├── Akos Dental
├── Akos Psi
├── Akos Vision
└── Akos Enterprise
```

Backend oficial:

```text
gbsalermo/AkosMed-BackEnd
```

Package:

```text
br.com.akosmed
```

Antes de uso comercial, validar disponibilidade de marca/domínio e requisitos legais/comerciais aplicáveis.

---

# Akos Core

O Core deve permanecer genérico o suficiente para atender diferentes especialidades sem criar uma Entity diferente para cada profissão.

Princípio:

```text
Core estável
+
módulos opcionais
+
interfaces específicas
=
produto extensível
```

Não transformar visão futura em complexidade antecipada no MVP.

---

# AkosMed Patient

App planejado em:

```text
Kotlin / Android
```

Entra **depois do Backend MVP 1.0** e depois da API `/me` correspondente estar estável.

## Primeiro corte

- login;
- próximas consultas;
- detalhes de consulta;
- prescrições;
- notificações.

## Evolução

- horários disponíveis;
- autoagendamento;
- cancelamento/reagendamento;
- documentos;
- orientações do profissional;
- vídeos/materiais terapêuticos;
- solicitações administrativas.

O app consome regras do backend; não mantém uma segunda implementação de agenda, Tenant ou prescrição.

---

# Orientações e vídeos do profissional

Recurso planejado:

```text
OrientacaoPaciente
```

Caso inicial:

```text
Fisioterapeuta
→ envia vídeo demonstrando exercício correto
→ adiciona instruções
→ Paciente visualiza no AkosMed Patient
```

Tipos previstos:

```text
VIDEO
DOCUMENTO
LINK
TEXTO
```

A escolha por um conceito genérico permite atender depois:

- fisioterapia;
- pós-operatório;
- fonoaudiologia;
- enfermagem;
- nutrição;
- outras orientações de acompanhamento.

Posição oficial:

```text
ETAPA 15 → API + OrientacaoPaciente
ETAPA 16 → app Kotlin + visualização de orientações/vídeos
```

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# Akos Assistant

Objetivo:

- agenda;
- próximos pacientes;
- check-in;
- alertas;
- pendências;
- resumo diário.

A primeira entrega útil não precisa ser um bot.

O backend pode oferecer Services/endpoints de resumo consumidos por:

- web;
- mobile;
- push;
- bot futuro.

Regra:

```text
interface/adapter não contém regra de negócio crítica
```

Akos Assistant entra depois do app/API planejados no roadmap, salvo decisão futura oficialmente registrada.

---

# Lembretes de medicamento

A estrutura `ItemPrescricao` pode permitir derivar:

- tratamento ativo;
- data de término;
- frequência;
- intervalos teóricos.

Futuro:

- lembrete de dose;
- aviso de fim do tratamento;
- adesão opcional.

Não criar tabela de lembrete/horários antes da prescrição e da experiência mobile estarem estáveis.

---

# Contato com a clínica

Preferir primeiro solicitações estruturadas.

Exemplo futuro:

```text
SolicitacaoPaciente
- tipo
- mensagem
- status
```

Tipos possíveis:

```text
REAGENDAMENTO
DOCUMENTO
DUVIDA_ADMINISTRATIVA
OUTRO
```

Chat em tempo real só entra com demanda concreta.

---

# IA

Possíveis usos futuros:

- resumo administrativo;
- classificação de solicitações;
- busca assistida;
- apoio à documentação;
- apoio operacional não-autônomo.

Não deixar IA:

- decidir regra crítica do domínio;
- substituir autorização;
- substituir validação clínica/profissional;
- gravar alteração clínica sem fluxo/autorização definidos.

---

# Especializações

Possíveis:

```text
Akos Dental
Akos Psi
Akos Vision
```

Devem reutilizar:

```text
Paciente
ProfissionalSaude
Prontuario
Atendimento
```

Não duplicar o Core por especialidade.

Uma especialização só entra com necessidade real, requisitos e etapa própria.

---

# Enterprise

Possíveis recursos:

- instância dedicada;
- banco dedicado;
- SSO;
- auditoria avançada;
- integrações corporativas;
- limites/configurações próprias.

Implementar somente com necessidade comercial real.

---

# Outros módulos futuros

Possíveis:

- financeiro clínico;
- convênios;
- estoque;
- laboratório;
- TISS;
- RNDS;
- assinatura digital;
- integrações de mensagem;
- SaaS billing.

Critério de entrada definido em `09_MODULOS_OPCIONAIS.md`.

---

# Ordem futura preservada

A visão deste arquivo não altera o roadmap.

Prioridade atual:

```text
Backend MVP 1.0
↓
API do paciente + OrientacaoPaciente
↓
AkosMed Patient / Kotlin
↓
Akos Assistant
↓
Especializações
↓
Módulos adicionais/Enterprise
```

Se essa ordem mudar, atualizar primeiro `CONTINUIDADE.md` e `11_ROADMAP_ETAPAS.md`.
