# StorM Wallet — API de pagamentos (API Key)

Guia curto para integrar **criação** e **consulta** de cobranças PIX nas suas aplicações usando o header **`x-api-key`**.

**URL base (produção):** `https://wallet.stormapplications.com`  
Todas as rotas abaixo usam esse host + o caminho indicado (ex.: `https://wallet.stormapplications.com/api/v1/payments/create`).

---

## Obter a API Key

1. Acesse a [StorM Wallet](https://wallet.stormapplications.com) e faça login.
2. Na área da conta, crie uma **API Key** com as permissões:
   - **Criar pagamento**
   - **Ler pagamento**  
3. Guarde a chave com segurança — ela só é mostrada na criação.

---

## Autenticação

Em **toda** requisição, envie uma das opções:

```http
x-api-key: SUA_CHAVE_AQUI
```

Ou, se a chave começar com `sk_live_`:

```http
Authorization: Bearer SUA_CHAVE_AQUI
```

---

## Formato das respostas

**Sucesso:** o payload útil vem dentro de `data`:

```json
{
  "success": true,
  "data": { ... }
}
```

**Erro:**

```json
{
  "success": false,
  "error": "mensagem"
}
```

**Validação (CPF inválido, valor fora do limite, etc.):** HTTP `400`, com lista em `details`:

```json
{
  "success": false,
  "error": "Erro de validação",
  "details": [{ "field": "payerDocument", "message": "CPF do pagador inválido" }]
}
```

No seu código, use sempre `response.data` após verificar `response.success === true`.

---

## 1. Criar pagamento

`POST /api/v1/payments/create`  
`Content-Type: application/json`

| Campo | Tipo | Obrigatório | Observação |
|--------|------|-------------|------------|
| `amount` | number | Sim | Valor em **reais**, até 2 casas (ex.: `29.90`). Máximo **100.000**. |
| `payerName` | string | Sim | 3–100 caracteres. |
| `payerDocument` | string | Sim | CPF (com ou sem pontuação; a API normaliza). |
| `description` | string | Sim | 1–200 caracteres. |
| `externalId` | string | Não | Até 100 caracteres — seu ID de pedido, assinatura, etc. |
| `metadata` | object | Não | Dados extras (chave/valor). |

### Exemplo (curl)

```bash
curl -s -X POST "https://wallet.stormapplications.com/api/v1/payments/create" \
  -H "Content-Type: application/json" \
  -H "x-api-key: SUA_CHAVE_AQUI" \
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
    "status": "pending"
  }
}
```

- **`data.id`** — use no GET para acompanhar o status.
- **`data.pixCode`** — PIX copia e cola.
- **`data.qrCode`** — QR em base64 (data URL) para exibir na tela.

---

## 2. Consultar pagamento

`GET /api/v1/payments/:id`

Substitua `:id` pelo `id` retornado em `data.id` ao criar.

### Exemplo (curl)

```bash
curl -s "https://wallet.stormapplications.com/api/v1/payments/ID_DO_PAGAMENTO" \
  -H "x-api-key: SUA_CHAVE_AQUI"
```

### Resposta (HTTP 200)

```json
{
  "success": true,
  "data": {
    "id": "...",
    "externalId": "pedido-12345",
    "amount": 29.9,
    "netAmount": 29.41,
    "status": "completed",
    "pixCode": "00020126...",
    "createdAt": "2025-02-25T12:00:00.000Z",
    "completedAt": "2025-02-25T12:05:00.000Z"
  }
}
```

### Valores de `status`

| `status` | Significado |
|----------|-------------|
| `pending` | Aguardando pagamento |
| `completed` | Pago |
| `expired` | Expirado |
| `cancelled` | Cancelado |

---

## Fluxo sugerido no seu app

1. Chame **POST** `/api/v1/payments/create` com valor, nome, CPF, descrição e, se quiser, `externalId`.
2. Mostre o **QR** (`data.qrCode`) ou o **PIX copia e cola** (`data.pixCode`).
3. Faça **polling** com **GET** `/api/v1/payments/:id` a cada poucos segundos até `data.status === "completed"` (ou trate `expired` / `cancelled`).
4. Ao confirmar pagamento, use `externalId` (e/ou seu próprio banco) para liberar o produto ou acesso.

**Segurança:** não exponha a API Key no front-end público; chame a Wallet a partir do **seu backend**.

---

## Erros frequentes

| HTTP | Causa provável |
|------|----------------|
| **401** | API Key ausente, inválida ou revogada. |
| **403** | Chave sem permissão de criar/ler pagamento. |
| **404** | Pagamento inexistente ou de outra conta. |
| **400** | Body inválido (veja `details`). |

---

## Exemplo mínimo (Node / fetch)

```javascript
const BASE = 'https://wallet.stormapplications.com';
const API_KEY = process.env.STORM_WALLET_API_KEY;

async function criarPagamento() {
  const res = await fetch(`${BASE}/api/v1/payments/create`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY,
    },
    body: JSON.stringify({
      amount: 10.5,
      payerName: 'Maria Souza',
      payerDocument: '52998224725',
      description: 'Teste de integração',
      externalId: 'meu-pedido-1',
    }),
  });
  const json = await res.json();
  if (!json.success) throw new Error(json.error || res.statusText);
  return json.data; // id, pixCode, qrCode, status, ...
}

async function statusPagamento(id) {
  const res = await fetch(`${BASE}/api/v1/payments/${id}`, {
    headers: { 'x-api-key': API_KEY },
  });
  const json = await res.json();
  if (!json.success) throw new Error(json.error || res.statusText);
  return json.data;
}
```