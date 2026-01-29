README.md
=========
# Teste Prático Backend Sênior (Cakto)
Confidencial – uso exclusivo do candidato.

## Visão geral
A Cakto é uma fintech especializada em pagamentos para criadores digitais (infoprodutos). Este desafio simula um recorte real do core de pagamentos: cálculo de taxas, split de recebíveis, precisão financeira, idempotência, rastreabilidade e um padrão simples de evento.

O Objetivo deste teste é avaliar a sua senioridade com foco em lógica de negócio e mindset de produção, com escopo curto.

## Regras do desafio
- Tempo máximo sugerido de execução: 1 hora.
  - Faça o melhor recorte possível dentro desse tempo.
  - Se não der para finalizar tudo, priorize o núcleo e descreva no README o que faltou e como faria. 
- Entrega: repositório público no GitHub, com commits e PR.
- Stack: Python + Django + Django REST Framework.
  - Se optar por outra stack, ok, mas explique no README.
- Pode usar IA: sim.
  - Descreva no README como usou (ex: “usei para rascunhar testes, fazer a lógica core, listar edge cases, etc.”).

## Contexto e requisitos de pagamentos reais
Pagamentos em infoprodutos exigem:
- Precisão de centavos, sem divergências.
- Idempotência, retries não podem duplicar registros.
- Auditoria e reconciliação, precisa existir rastro do que foi calculado e repassado.
- Arquitetura orientada a eventos, mesmo que simulada.

==============================
DESAFIO: Mini Split Engine + Ledger + Outbox
==============================

Você vai implementar uma API pequena que:
1) Calcula taxas e split.
2) Persiste um pagamento confirmado.
3) Gera lançamentos de ledger por recebedor.
4) Registra um evento outbox do tipo "payment_captured".

-------------------------
Regras de negócio
-------------------------

1) Taxas da plataforma
- PIX: 0%
- CARD 1x: 3,99%
- CARD 2x a 12x: 4,99% + 2% por parcela extra
  Exemplos:
  - 2x = 4,99% + 2%
  - 3x = 4,99% + 4%
  - 12x = 4,99% + 22%

2) Split de recebíveis
- O split é aplicado sobre o valor líquido (após taxa da plataforma).
- Deve suportar de 1 a 5 recebedores.
- A soma dos percentuais do split deve dar 100%.
- O total distribuído deve bater exatamente com o líquido.

3) Precisão e arredondamento
- Use Decimal, com 2 casas decimais.
- Não use float para dinheiro.
- Regra para sobras/faltas por arredondamento:
  - Se sobrar ou faltar centavos ao distribuir, a diferença vai para o recebedor com role="producer".
  - Se não existir producer, a diferença vai para o recebedor com maior percentual.

4) Idempotência
- O endpoint de pagamento deve aceitar o header "Idempotency-Key".
- Regras:
  - Mesma Idempotency-Key e mesmo payload: retornar o mesmo resultado, sem duplicar registros.
  - Mesma Idempotency-Key e payload diferente: retornar 409 Conflict (ou equivalente), explicando o motivo.

-------------------------
Endpoints
-------------------------

Obrigatório: Confirmar pagamento (persiste + ledger + outbox)
POST /api/v1/payments

Header:
- Idempotency-Key: <string>

Body (exemplo):

```json
{
  "amount": "297.00",
  "currency": "BRL",
  "payment_method": "card",
  "installments": 3,
  "splits": [
    { "recipient_id": "producer_1", "role": "producer", "percent": 70 },
    { "recipient_id": "affiliate_9", "role": "affiliate", "percent": 30 }
  ]
}
```

Response (exemplo):

```json
{
  "payment_id": "pmt_123",
  "status": "captured",
  "gross_amount": "297.00",
  "platform_fee_amount": "26.70",
  "net_amount": "270.30",
  "receivables": [
    { "recipient_id": "producer_1", "role": "producer", "amount": "189.21" },
    { "recipient_id": "affiliate_9", "role": "affiliate", "amount": "81.09" }
  ],
  "outbox_event": {
    "type": "payment_captured",
    "status": "pending"
  }
}
```

Opcional: Simular cálculo (não persiste)
POST /api/v1/checkout/quote
Mesma request do /payments, mas apenas retorna os valores calculados, sem persistir.

-------------------------
Persistência mínima esperada
-------------------------

Você pode modelar como preferir, mas esperamos pelo menos:

Payment
- status ("captured")
- gross_amount, platform_fee_amount, net_amount
- payment_method, installments
- idempotency_key (ou tabela separada)
- created_at

LedgerEntry
- payment_id
- recipient_id, role
- amount
- created_at

OutboxEvent
- type ("payment_captured")
- payload (json)
- status ("pending", "published")
- created_at
- published_at (opcional)

-------------------------
Validações mínimas
-------------------------
- splits: 1..5 recebedores
- percent: 0 < percent <= 100
- soma dos percentuais = 100
- PIX não aceita installments (parcelamento)
- CARD aceita installments 1..12
- amount > 0
- currency suportada (BRL)

-------------------------
Testes automatizados
-------------------------
Escolha e faça pelo menos 3:
1) PIX com taxa zero, split 100%, soma bate certinho.
2) CARD 3x, split 70/30, soma receivables = net.
3) Caso com arredondamento, sobra/falta 0.01, regra do centavo funciona.
4) Idempotência: mesma key não duplica.
5) Idempotência: mesma key e payload diferente retorna conflito.

-------------------------
Entrega no GitHub
-------------------------

Obrigatório:
- Repositório público no GitHub.
- Histórico com múltiplos commits (evite um commit só com tudo).
- Criar pelo menos 1 PR:
  Exemplo:
  - branch: feature/payment-split-ledger
  - PR abrindo para main

Inclua no README o link do PR que você abriu.

Sugestão valorizada:
- Commits no estilo Conventional Commits:
  feat: add payment endpoint
  test: add rounding tests
  docs: explain idempotency strategy

-------------------------
Como avaliamos
-------------------------
- Corretude financeira e edge cases (40%)
- Idempotência, auditabilidade (ledger) e outbox (30%)
- Qualidade de código e testes (20%)
- Processo no GitHub: commits e PR (10%)

-------------------------
O que esperamos ver em um Sênior
-------------------------
- Escolhas pragmáticas, sem overengineering.
- Dinheiro com Decimal e invariantes claras.
- Idempotência bem pensada, incluindo conflito.
- Ledger e outbox como sinais de mentalidade de produção.
- Testes que miram risco real, não só fluxo feliz.

-------------------------
Estrutura sugerida do repositório
-------------------------
Você pode organizar como preferir. Sugestão:

cakto-mini-split-engine/
  README.md
  requirements.txt (ou pyproject.toml)
  manage.py
  app/
    api/
      serializers.py
      views.py
      urls.py
    services/
      split_calculator.py
    models.py
    tests/
      test_payments.py

-------------------------
Seção obrigatória no README: decisões técnicas
-------------------------
Adicione ao final do README:

1) Como garantiu precisão e arredondamento.
2) Regra de centavos e por quê.
3) Estratégia de idempotência.
4) Métricas que colocaria em produção (lista curta).
5) Se tivesse mais tempo, o que faria a seguir.

Boa sorte, e que seus centavos nunca fujam do ledger.
