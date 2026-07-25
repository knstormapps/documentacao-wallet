# StorM Wallet — API de pagamentos (API Key)

Guia para integrar cobranças PIX com **API Key** (`x-api-key`).

**URL base (produção):** `https://wallet.stormapplications.com`

---

## Obter a API Key

1. Acesse a [StorM Wallet](https://stormapplications.com/wallet) e faça login.
2. Em **API Keys**, crie uma chave.
3. (Opcional) Configure o **webhook de pagamentos** na mesma página.
4. Guarde a chave e o secret do webhook com segurança — só aparecem na criação/regeneração.

---

## Autenticação

Em **toda** requisição:

```http
x-api-key: SUA_CHAVE_AQUI
```

Também aceito: `Authorization: Bearer sk_live_...`

## Formato das respostas

**Sucesso** — payload em `data`:

```json
{ "success": true, "data": { ... } }
```

**Erro:**

```json
{ "success": false, "error": "mensagem" }
```

**Validação:** HTTP `400` com `details`.

---

## 0. Validar a API Key

`GET /api/v1/account`

Endpoint leve para checar se a chave é válida (sem criar cobrança).

```bash
curl -s "https://wallet.stormapplications.com/api/v1/account" \
  -H "x-api-key: SUA_CHAVE_AQUI"
```

```json
{
  "success": true,
  "data": {
    "id": "...",
    "name": "Seu nome",
    "email": "voce@email.com",
    "webhookConfigured": true,
    "createdAt": "2025-01-01T00:00:00.000Z"
  }
}
```

---

## 1. Criar pagamento

`POST /api/v1/payments/create`  
`Content-Type: application/json`

| Campo | Tipo | Obrigatório | Observação |
|--------|------|-------------|------------|
| `amount` | number | Sim | Reais, até 2 casas. Máx. **100.000**. |
| `payerName` | string | Sim | 3–100 caracteres. |
| `payerDocument` | string | Sim | CPF (com ou sem pontuação). |
| `description` | string | Sim | 1–200 caracteres. |
| `externalId` | string | Não | Até 100 — seu ID de pedido. Também serve como **idempotência**. |
| `metadata` | object | Não | Dados extras. |

### Idempotência

Envie o header **`Idempotency-Key`** (ou `X-Idempotency-Key`) **ou** o campo `externalId`.

- Mesma chave + mesma conta → a API **reutiliza** o pagamento já criado (não gera novo PIX).
- Sem chave → cada request cria um pagamento novo.

```bash
curl -s -X POST "https://wallet.stormapplications.com/api/v1/payments/create" \
  -H "Content-Type: application/json" \
  -H "x-api-key: SUA_CHAVE_AQUI" \
  -H "Idempotency-Key: pedido-12345" \
  -d '{
    "amount": 29.90,
    "payerName": "João Silva",
    "payerDocument": "12345678900",
    "description": "Assinatura - Plano Pro",
    "externalId": "pedido-12345"
  }'
```

### Resposta (HTTP 201)

```json
{
  "success": true,
  "data": {
    "id": "id-do-pagamento-na-wallet",
    "externalId": "pedido-12345",
    "amount": 29.9,
    "pixCode": "00020126...",
    "qrCode": "data:image/png;base64,...",
    "status": "PENDENTE"
  }
}
```

Use `data.id` nas consultas. Status reais: **`PENDENTE`**, **`COMPLETO`**, **`FALHA`**.

---

## 2. Consultar pagamento

`GET /api/v1/payments/:id`

- ID inexistente ou inválido → **404** (não 500).
- Só retorna pagamentos da sua conta.

```bash
curl -s "https://wallet.stormapplications.com/api/v1/payments/ID_DO_PAGAMENTO" \
  -H "x-api-key: SUA_CHAVE_AQUI"
```

---

## 3. Listar pagamentos

`GET /api/v1/payments`

| Query | Tipo | Default | Observação |
|--------|------|---------|------------|
| `status` | string | — | `PENDENTE` / `COMPLETO` / `FALHA` (ou `pending` / `completed` / `failed`) |
| `startDate` | string | — | ISO ou data parseável |
| `endDate` | string | — | ISO ou data parseável |
| `page` | number | 1 | |
| `limit` | number | 20 | máx. 100 |

```bash
curl -s "https://wallet.stormapplications.com/api/v1/payments?status=COMPLETO&page=1&limit=20" \
  -H "x-api-key: SUA_CHAVE_AQUI"
```

```json
{
  "success": true,
  "data": {
    "items": [ { "id": "...", "status": "COMPLETO", "amount": 29.9, "...": "..." } ],
    "pagination": { "page": 1, "limit": 20, "total": 42, "totalPages": 3 }
  }
}
```

---

## 4. Webhook de pagamento (outbound)

Configure a URL em **Wallet → API Keys → Webhook de pagamentos**.

Quando um pagamento criado via API muda para pago ou falha, a StorM envia:

`POST` na sua URL HTTPS

| Header | Valor |
|--------|--------|
| `Content-Type` | `application/json` |
| `X-Storm-Event` | `payment.completed` ou `payment.failed` |
| `X-Storm-Signature` | HMAC-SHA256 hex do **body bruto** com o secret |

### Body

```json
{
  "event": "payment.completed",
  "data": {
    "id": "...",
    "externalId": "pedido-12345",
    "amount": 29.9,
    "netAmount": 29.41,
    "status": "COMPLETO",
    "completedAt": "2026-07-25T03:00:00.000Z"
  },
  "createdAt": "2026-07-25T03:00:00.000Z"
}
```

### Verificar assinatura (Node)

```javascript
const crypto = require('crypto');

function verifyStormSignature(rawBody, signatureHeader, secret) {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signatureHeader));
}
```

Responda **2xx** rápido. Em falha de rede a StorM **não** garante reenvio automático — combine com polling se precisar de garantia forte.

Você ainda pode usar polling em `GET /api/v1/payments/:id` se preferir.

---

## Fluxo sugerido

1. (Opcional) `GET /api/v1/account` para validar a key.
2. `POST /api/v1/payments/create` com `Idempotency-Key` / `externalId`.
3. Mostre QR / PIX copia e cola.
4. Prefira **webhook**; senão faça polling até `COMPLETO` ou `FALHA`.
5. Liberar produto com base em `id` / `externalId`.

**Segurança:** não exponha a API Key nem o webhook secret no front-end público.

---

## Erros frequentes

| HTTP | Causa |
|------|--------|
| **401** | API Key ausente, inválida ou revogada. |
| **403** | Chave sem permissão. |
| **404** | Pagamento inexistente, ID inválido ou de outra conta. |
| **400** | Body/query inválidos (`details`). |
| **409** | Mesma `Idempotency-Key` ainda em processamento. |

---

## Exemplo mínimo (Node / fetch)

```javascript
const BASE = 'https://wallet.stormapplications.com';
const API_KEY = process.env.STORM_WALLET_API_KEY;

async function criarPagamento(externalId) {
  const res = await fetch(`${BASE}/api/v1/payments/create`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY,
      'Idempotency-Key': externalId,
    },
    body: JSON.stringify({
      amount: 10.5,
      payerName: 'Maria Souza',
      payerDocument: '52998224725',
      description: 'Teste de integração',
      externalId,
    }),
  });
  const json = await res.json();
  if (!json.success) throw new Error(json.error || res.statusText);
  return json.data;
}

async function listarPagos() {
  const res = await fetch(`${BASE}/api/v1/payments?status=COMPLETO&limit=10`, {
    headers: { 'x-api-key': API_KEY },
  });
  const json = await res.json();
  if (!json.success) throw new Error(json.error || res.statusText);
  return json.data;
}
```

### Outras rotas públicas

- `POST /api/v1/withdrawals/create` / `GET /api/v1/withdrawals/:id`
- Rotas de MED Discord-pending (bots StorM)
