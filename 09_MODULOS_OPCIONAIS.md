# 09 — Módulos Opcionais

Este arquivo existe principalmente para impedir o Core de crescer sem controle.

## Regra

Um módulo só entra quando:

1. Core está estável;
2. existe caso de uso real;
3. regras foram levantadas;
4. não pode ser resolvido com estrutura atual.

---

# Financeiro clínico

Possíveis entidades:

```text
Cobranca
Pagamento
FormaPagamento
```

Não confundir com cobrança do SaaS.

Entrar somente quando houver necessidade de caixa/faturamento.

---

# Convênios

```text
Operadora
Plano
PacienteConvenio
Autorizacao
```

TISS deve ser uma etapa própria.

---

# Estoque

```text
Produto
Lote
MovimentacaoEstoque
```

Só vale entrar para clínicas que realmente controlam materiais/medicamentos.

---

# Laboratório

```text
Exame
SolicitacaoExame
ResultadoExame
```

Não implementar apenas porque sistemas hospitalares possuem esse módulo.

---

# Documentos clínicos

O primeiro documento útil é a receita gerada a partir de `Prescricao`.

Depois:

```text
Atestado
Relatorio
Laudo
SolicitacaoExame
```

Assinatura digital deve ser tratada separadamente.

---

# App do paciente

O backend necessário é a API `/me`.

O app Kotlin deve começar somente quando:

- auth estiver estável;
- agenda estiver pronta;
- prescrição estiver pronta;
- endpoints `/me` estiverem testados.

---

# Akos Assistant

Não criar backend separado.

Fase 1:

- endpoints de resumo;
- notificações internas.

Fase 2:

- push/app profissional.

Fase 3:

- bot Telegram/WhatsApp, se útil.

---

# Comunicação com clínica

Começar com solicitação estruturada, não chat.

Futuro:

```text
SolicitacaoPaciente
- tipo
- mensagem
- status
```

Tipos:

```text
REAGENDAMENTO
DOCUMENTO
DUVIDA_ADMINISTRATIVA
OUTRO
```

---

# SaaS billing

Depois de existir produto comercial:

```text
PlanoSaas
AssinaturaSaas
FaturaSaas
```

Não incluir antes de haver planos/regras reais.

---

# Integrações

Futuras:

- WhatsApp;
- e-mail;
- SMS;
- RNDS;
- TISS;
- gateway de pagamento;
- assinatura digital;
- calendário externo.

Cada integração deve ficar atrás de adapter/interface.


---

## Regra para entrada de módulo

Nenhum módulo opcional entra no roadmap principal apenas porque pode ser útil.

Ele deve ter:

1. necessidade concreta;
2. definição de escopo;
3. entidade/regra que não possa ser atendida pelo Core;
4. etapa própria;
5. testes próprios.

Até lá, permanece somente como planejamento.


---

## Banco e documentação dos módulos futuros

Módulos opcionais só entram depois do MVP e seguem a mesma ordem:

```text
domínio em H2
↓
teste
↓
PostgreSQL migration
↓
Swagger
↓
Postman
```

Não misturar módulo futuro ao Core apenas para antecipar possibilidades.
