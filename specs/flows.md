# Flowcharts

## Процесс выпусков (массовая загрузка)

- Инициатор запускает `GiftCertificateBatchLoader`, который подготавливает пакет кодов и метаданных.
- Пакет отправляется в API `POST /certificates/upload/batch`, сервис записывает сертификаты в статусе `Issued` и публикует событие `GiftCertificateState`.
- Ошибки валидации формируют DLQ/повторную отправку.

## Процесс активации при продаже

- `Orders` публикует `All_SalesOrder` с массивом `certificateLines`, `GiftCertificateProcessing` фиксирует каждый `orderLineId` в `order_certificate_allocation`.
- Перед авторизацией выполняется проверка уникальности кода: если сертификат уже реализован, фронт получает ошибку `CodeAlreadySold`.
- `Payments` публикует `All_Payments` с секцией `certificateAllocations`, сервис распределяет поступившие суммы и отслеживает статус `allocation_status`.
- После события с `isOrderSettled=true` и `outstandingAmount=0` активируются все сертификаты заказа, инициируется отправка SMS, обновляется состояние и публикуются события.

## Процесс оплаты сертификатом

- Фронт запрашивает авторизацию, получает `authorizationId` и обновлённый доступный баланс.
- После подтверждения платежа выполняется `capture`.
- При откате операции выполняется `void`, восстанавливается баланс.

## Процесс возврата сертификата

- Инициатор (кассир POS или покупатель в веб-интерфейсе) вызывает `POST /certificates/{code}/refund`, передавая канал и предпочтительный способ возврата.
- `GiftCertificateProcessing` проверяет остаток, заносит запрос в `refund_request`, публикует `GiftCertificateTransaction(RefundRequested)` и переводит сертификат в статус `RefundPending`.
- Сервис инициирует возврат в `Payments`, опубликованное событие `All_Payments` имеет `paymentStatus=RefundInitiated`, включает `refundDetails` и `orderLineId`.
- После поступления `RefundSettled` баланс обнуляется, создаётся операция `Refund`, статус обновляется на `Refunded`, публикуются `GiftCertificateState(Refunded)` и `GiftCertificateTransaction(Refund)`.
- Если требуется альтернативный способ перечисления средств, `refund_request` переводится в `AlternateDetailsRequired`, фронт запрашивает реквизиты у клиента, после чего процесс продолжается.

## Процесс интеграции с BI

- CDC/ETL подписывается на `GiftCertificateBI`.
- События агрегируются в витрину отчётности с атрибутами сертификата, операций и аудита.
