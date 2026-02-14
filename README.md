# Real-Time Fraud Detection System

Motor de análise de risco para transações financeiras de baixa latência, focado em decidir se uma operação deve ser aprovada ou negada em milissegundos, utilizando mensageria e cache.

---

## 1. Tecnologias Utilizadas

| Tecnologia   | Função                        | Motivo                                          |
|--------------|-------------------------------|-------------------------------------------------|
| **NestJS e spring**   | API & Microservices           | Estrutura modular e suporte nativo a Microservices. |
| **Next.js**  | Admin Dashboard               | SSR para SEO (opcional) e interface reativa para monitoramento. |
| **PostgreSQL** | Banco de Dados Relacional   | Persistência segura de histórico e dados de usuários. |
| **Redis**    | In-memory Data Store          | Validação de limites e trava de velocidade (Velocity Check). |
| **RabbitMQ** | Message Broker                | Processamento assíncrono de escrita em banco e notificações. |

---

## 📋 2. Requisitos Funcionais (RF)

- **[RF01]** Ingestão de Transação: Endpoint de alta performance para receber dados de venda.  
- **[RF02]** Validação de Saldo/Limite: Consultar saldo remanescente no Redis antes de aprovar.  
- **[RF03]** Velocity Check (Anti-Spam): Bloquear usuários que tentarem mais de 3 compras em 60 segundos.  
- **[RF04]** Persistência Assíncrona: O Gateway responde ao cliente e envia os dados para o RabbitMQ salvar no Postgres em background.  
- **[RF05]** Dashboard de Monitoramento: Visualização em tempo real das transações via WebSockets.  
- **[RF06]** Gestão de Regras: Interface para alterar limites e bloquear usuários manualmente.

---

## ⚙️ 3. Regras de Negócio (Business Rules)

| Regra                  | Lógica de Implementação                        | Ação       |
|------------------------|------------------------------------------------|------------|
| **Insufficient Funds** | `amount > user_limit` (Busca no Redis)         | `REJECTED` |
| **Velocity Attack**    | `count_keys(user_id_*) > 3` em 60s             | `REJECTED` |
| **High Ticket Value**  | `amount > 10000.00`                            | `REVISION` |
| **Blacklisted Merchant** | `merchant_id` presente na tabela de bloqueio | `REJECTED` |

---

## 📡 4. Definição da API (Contracts)

### A. Solicitação de Transação

**Endpoint:** `POST /v1/transactions`

**Payload:**
```json
{
  "userId": "uuid-v4-12345",
  "cardToken": "tok_visa_9988",
  "amount": 450.00,
  "merchantId": "m_loja_tech",
  "merchantCategory": "eletronics",
  "location": {
    "lat": -23.55,
    "lon": -46.63,
    "country": "BR"
  }
}

Resposta (Status 201):

```json
{
  "transactionId": "tx_987654",
  "status": "APPROVED",
  "score": 12,
  "processedAt": "2026-02-13T22:50:00Z"
}
```

## 5. Modelagem de Dados

### Estrutura de Cache (Redis)

- **Limite do Usuário**  
  `user:limit:{userId}` → Value: `5000.00` (TTL 24h)

- **Contador de Velocidade**  
  `user:velocity:{userId}:{timestamp}` → TTL 60s


### Esquema Relacional (PostgreSQL)

```sql
CREATE TYPE transaction_status AS ENUM ('APPROVED', 'REJECTED', 'REVISION');

CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  daily_limit DECIMAL(12, 2) DEFAULT 1000.00,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  amount DECIMAL(12, 2),
  status transaction_status,
  merchant_id VARCHAR(100),
  reason TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);