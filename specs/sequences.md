# Sequence Diagrams

## Сценарий 1. Продажа и активация сертификата

1. `OnlineStore` оформляет заказ с товаром типа GiftCertificate.
2. `Orders` публикует событие `All_SalesOrder`.
3. После полной оплаты `Payments` публикует `All_Payments` с деталями транзакции.
4. `GiftCertificateProcessing` получает оба события, сверяет заказ и оплату, переводит сертификат в статус `Active`, публикует `GiftCertificateState` и `GiftCertificateTransaction(Activate)`.
5. Сервис отправляет CVC в SMS через корпоративный SMS-шлюз.

## Сценарий 2. Использование сертификата (частичное списание)

1. Пользователь выбирает оплату сертификатом на сайте/кассе, фронт вызывает `POST /certificates/{code}/authorize`.
2. `GiftCertificateProcessing` проверяет баланс, создаёт запись `authorization_hold`, возвращает `authorizationId` и доступный остаток.
3. После подтверждения платежа фронт или `Payments` вызывает `POST /certificates/{code}/capture`.
4. Сервис уменьшает баланс, фиксирует операцию, публикует `GiftCertificateTransaction(Capture)` и обновлённое состояние.
5. События потребляются `Payments`, `Orders`, `BI Pipeline` для синхронизации.

## Сценарий 3. Возврат/аннулирование списания

1. Инициирующий сервис (`Payments` или поддержка) вызывает `POST /certificates/{code}/void` с идентификатором операции.
2. `GiftCertificateProcessing` проверяет возможность возврата, восстанавливает баланс, создаёт новую запись операции `Void`.
3. Публикуется `GiftCertificateTransaction(Void)` и обновлённый `GiftCertificateState`.
4. `BI Pipeline` и другие подписчики фиксируют событие для отчётности и аудита.
