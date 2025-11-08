# Архитектура

## Компоненты

- `GiftCertificateProcessing` — мастер-сервис сертификатов; хранит состояние, управляет жизненным циклом, публикует события о состоянии и транзакциях, обеспечивает API для синхронных запросов каналов, контролирует уникальность продажи кода, отслеживает распределение платежей по заказным сертификатам и оркестрирует возвраты.
- `GiftCertificateBatchLoader` — служебный процесс (скрипт/интеграция) для массовой загрузки предгенерированных кодов, публикует задания в Kafka, маркирует тип сертификата (Physical/Electronic).
- `OnlineStore` — фронт интернет-магазина; использует синхронные API для проверки и списания сертификатов, слушает события для обновления UI и блокирует продажу уже реализованных кодов.
- `POS` — фронт офлайн-точек; аналогично интернет-магазину вызывает API и принимает события, получает причины отказов при повторной продаже.
- `Payments` — сервис платежей; обрабатывает платежи и возвраты, уведомляет о покупках сертификатов и о комбинированных платежах, публикует агрегированные данные (`paymentVersion`, `capturedAmount`, `outstandingAmount`, `isOrderSettled`), массив `certificateAllocations` с деталями распределения сумм по позициям заказа и события возврата с `paymentStatus` (RefundInitiated, RefundSettled) и `refundDetails`.
- `Orders` — сервис заказов; отправляет события о заказах и статусах с обогащением по позициям сертификатов (`certificateLines`), потребляет состояния сертификатов для отчётности.
- `CRM` — сервис клиентов; инициирует маркетинговые кампании, может обогащать сертификаты связями с клиентами.
- `BI Pipeline` — коннектор в аналитику, подписан на события сертификатов, отслеживает канал продажи и нарушение уникальности.

## Kafka-топики

- `All_SalesOrder` *(существующий)* — события о заказах; используется для выявления продаж сертификатов и передают массив `certificateLines` (`orderLineId`, `certificateSku`, `plannedDenomination`, `intendedRecipient`).
- `All_Payments` *(существующий)* — события об оплатах и возвратах; подтверждает факт полной оплаты подарочных сертификатов, содержит атрибуты `paymentVersion`, `capturedAmount`, `outstandingAmount`, `isOrderSettled`, вложенный массив `certificateAllocations` (`orderLineId`, `allocatedAmount`, `currency`) и для возвратов — `paymentStatus` (RefundInitiated, RefundSettled) и `refundDetails` (канал, способ возврата, реквизиты).
- `All_Clients` *(существующий)* — справочная информация о клиентах.
- `GiftCertificateUpload` *(новый)* — задания на загрузку и выпуск предгенерированных сертификатов.
- `GiftCertificateState` *(новый)* — текущее состояние каждого сертификата (Issued, Active, Redeemed, PartiallyRedeemed, Refunded, Cancelled).
- `GiftCertificateTransaction` *(новый)* — события об операциях списания, возврата, аннулирования, с деталями аудита и статуса распределения/возврата.
- `GiftCertificateBI` *(новый, агрегированный)* — поток для аналитики и отчётности, содержит агрегированную информацию (остаток, привязки к заказу/платежу/клиенту, `allocation_status`, `planned_amount`, `collected_amount`).

## Хранилища данных

- `GiftCertificateDB` — транзакционное хранилище (PostgreSQL) в сервисе процессинга; таблицы `gift_certificate`, `gift_certificate_operation`, `authorization_hold`, `order_certificate_allocation`, `refund_request`, `audit_log`, расширенные полями `channel_type`, `sold_at`, `sold_by`, `unique_sale_token`.
- `order_certificate_allocation` содержит `order_line_id` (PK), `order_id`, `certificate_id`, `planned_amount`, `collected_amount`, `currency`, `allocation_status`, `last_payment_event_id`.
- `refund_request` фиксирует инициированные возвраты: `id`, `certificate_id`, `order_line_id`, `requested_amount`, `currency`, `initiated_by`, `channel`, `preferred_method`, `status` (Pending/AlternateDetailsRequired/Completed), `payment_reference`.
- `GiftCertificateCache` — высокоскоростной кэш (Redis) для операций проверки баланса и авторизации.
- Существующие сервисы используют собственные БД без изменений.

## Граничные API

- HTTP API `GiftCertificateProcessing` (см. OpenAPI):
  - `POST /certificates/upload/batch` — приём файлов/пакетов кодов от загрузчиков.
  - `POST /certificates/{code}/activate` — активация по событию полной оплаты (используется внутренним сервисом).
  - `POST /certificates/{code}/authorize` — резервирование суммы перед оплатой.
  - `POST /certificates/{code}/capture` — списание при успешной оплате.
  - `POST /certificates/{code}/void` — отмена/аннулирование.
  - `POST /certificates/{code}/refund` — инициирование возврата остатка сертификата с учётом предпочтительного способа возврата.
  - `GET /certificates/{code}/balance` — получение остатка.
- Event-driven контракты:
  - Подписка на `All_SalesOrder` (с `certificateLines`) и `All_Payments` (с `certificateAllocations`).
  - Публикация `GiftCertificateState`, `GiftCertificateTransaction`, `GiftCertificateBI`.

## Потоки данных

- Массовая загрузка: загрузчик публикует задания → `GiftCertificateProcessing` создаёт сертификаты → публикует состояние `Issued`, фиксируя канал происхождения.
- Продажа сертификата: заказ в `Orders` публикует `certificateLines`, `GiftCertificateProcessing` создаёт записи `order_certificate_allocation` и связывает их с зарезервированными кодами → перед авторизацией проверяется уникальность кода: при повторной продаже фронт получает ошибку `CodeAlreadySold` → `Payments` по мере поступления частичных оплат публикует события `All_Payments` с уточнёнными `certificateAllocations`, `AllocationManager` агрегирует `collected_amount` и отслеживает остаток → после события с `isOrderSettled=true` и `outstandingAmount=0` активируются все сертификаты заказа, публикуется `GiftCertificateState(Active)` и отправляется CVC в SMS по каждому сертификату.
- Использование сертификата: фронт вызывает `authorize` → при подтверждении платежа вызывается `capture` → изменяется баланс → публикуются события состояния и транзакции.
- Возврат сертификата: кассир POS или покупатель в онлайн-канале вызывает `refund`; `GiftCertificateProcessing` фиксирует запрос, создаёт запись `refund_request`, публикует `GiftCertificateTransaction(RefundRequested)` и переводит сертификат в статус `RefundPending`, одновременно инициируя событие `All_Payments` с `paymentStatus=RefundInitiated`; после получения `RefundSettled` от Payments баланс обнуляется, создаётся операция `Refund`, статус сертификата — `Refunded`, публикуются `GiftCertificateState(Refunded)` и транзакция `Refund`.
- Аннулирование/void операции: инициирующий сервис вызывает `void`, в ответ публикуется `GiftCertificateTransaction` и обновлённое состояние.
- BI: `GiftCertificateBI` агрегирует данные из внутренних таблиц через CDC/стриминг и поставляет в аналитику.

## Аудит

- Каждая операция фиксируется в журнале `audit_log` с атрибутами: действие, идентификаторы сертификата/операции, канал, инициатор, источник события, timestamp, до/после баланс, флаг уникальности продажи.
- Аудит-поток `GiftCertificateTransaction` обеспечивает внешнее воспроизведение действий и интеграцию с SIEM.

## Безопасность и доступ

- API требует сервисных токенов каналов `OnlineStore`, `POS`, `Payments`; результат авторизации включает причину отказа (`CodeAlreadySold`).
- CVC хранится только в зашифрованном виде (KMS), раскрывается однократно при отправке SMS.
- Логирование чувствительных данных производится с маскированием CVC.

## Отказоустойчивость

- Сервис масштабируется горизонтально; stateful операции обеспечиваются за счёт распределённых транзакций (PostgreSQL + optimistic locking).
- Подписчики Kafka используют consumer groups для балансировки нагрузки.
- В случае недоступности SMS-провайдера события помещаются в DLQ с повторной обработкой.
