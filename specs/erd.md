# ERD

## Сущности и связи

- `gift_certificate`
  - `id` (UUID, PK)
  - `code` (уникальный, индекс)
  - `type` (Electronic/Physical)
  - `status`
  - `channel_type` (nullable до продажи)
  - `sold_at` (nullable)
  - `sold_by` (nullable)
  - `unique_sale_token` (UUID, уникальный, nullable)
  - `nominal_amount`
  - `balance`
  - `currency`
  - `batch_id`
  - `channel`
  - `client_id` (FK -> CRM, nullable)
  - `issued_at`
  - `activated_at`
  - `expires_at` (nullable)
  - `metadata` (JSONB)

- `gift_certificate_operation`
  - `id` (UUID, PK)
  - `certificate_id` (FK -> gift_certificate)
  - `type`
  - `amount`
  - `currency`
  - `channel`
  - `performed_at`
  - `performed_by`
  - `order_id`
  - `payment_id`
  - `authorization_id` (nullable)
  - `balance_before`
  - `balance_after`
  - `audit_id` (FK -> audit_log)

- `authorization_hold`
  - `id` (UUID, PK)
  - `certificate_id` (FK -> gift_certificate)
  - `request_id`
  - `reserved_amount`
  - `currency`
  - `channel`
  - `channel_type`
  - `operator_id` (nullable)
  - `expires_at`
  - `status`

- `audit_log`
  - `id` (UUID, PK)
  - `certificate_code`
  - `operation_id`
  - `action`
  - `actor`
  - `channel`
  - `source`
  - `trace_id`
  - `payload_before`
  - `payload_after`
  - `created_at`

## Особенности моделирования

- Все денежные значения хранятся в минимальной денежной единице (копейки) и пересчитываются в API.
- История операций хранится в `gift_certificate_operation` и связывается с аудитом для обогащения событий.
- Таблица `authorization_hold` обеспечивает идемпотентность и контроль срока действия резервов.
- Уникальность продажи обеспечивается ограничением `gift_certificate.code` + `status in (Sold, Active, Redeemed, Cancelled)` и `unique_sale_token`, который фиксирует канал и оператора продажи.

## Связи

- `gift_certificate`
