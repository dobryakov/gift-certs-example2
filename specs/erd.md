# ERD

## Сущности и связи

- `gift_certificate`
  - `id` (UUID, PK)
  - `code` (уникальный, индекс)
  - `type` (Electronic/Physical)
  - `status`
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

## Связи

- `gift_certificate` 1:N `gift_certificate_operation`.
- `gift_certificate` 1:N `authorization_hold`.
- `gift_certificate_operation` 1:1 `audit_log` (опционально).
- Внешние связи: `client_id` → `CRM.All_Clients`, `order_id` → `Orders`, `payment_id` → `Payments` через события и справочные словари.
