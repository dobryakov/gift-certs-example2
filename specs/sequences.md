# Sequence Diagrams

## Сценарий 1. Продажа и активация сертификата

1. `OnlineStore` запрашивает у покупателя желаемые суммы по каждому сертификату в заказе.
2. `OnlineStore` оформляет заказ с отдельными позициями GiftCertificate и передает их в `Orders`.
3. `Orders` публикует `All_SalesOrder` с массивом `certificateLines` (`orderLineId`, `plannedDenomination`, `intendedRecipient`).
4. Фронт запрашивает `POST /certificates/{code}/authorize`; `GiftCertificateProcessing` проверяет уникальность кода (не продан ли он ранее), резервирует его за каналом и создаёт запись `order_certificate_allocation`.
5. После каждой частичной оплаты `Payments` публикует `All_Payments` с атрибутами `capturedAmount`, `outstandingAmount`, `paymentVersion`, `isOrderSettled` и массивом `certificateAllocations`.
6. `GiftCertificateProcessing` обновляет `collected_amount` в `order_certificate_allocation`, отслеживая покрытие плана.
7. Когда поступает событие с `isOrderSettled=true` и `outstandingAmount=0`, сервис активирует все сертификаты заказа, публикует `GiftCertificateState` и `GiftCertificateTransaction(Activate)` для каждого.
8. Сервис отправляет CVC в SMS через корпоративный SMS-шлюз.

## Сценарий 2. Использование сертификата (частичное списание)

1. Пользователь выбирает оплату сертификатом на сайте/кассе, фронт вызывает `POST /certificates/{code}/authorize`; при повторной продаже возвращается ошибка `CodeAlreadySold`.
2. `GiftCertificateProcessing` проверяет баланс, создаёт запись `authorization_hold`, возвращает `authorizationId` и доступный остаток.
3. После подтверждения платежа фронт или `Payments` вызывает `POST /certificates/{code}/capture`.
4. Сервис уменьшает баланс, фиксирует операцию, публикует `GiftCertificateTransaction(Capture)` и обновлённое состояние.
5. События потребляются `Payments`, `Orders`, `BI Pipeline` для синхронизации.

## Сценарий 3. Возврат сертификата

1. Кассир POS или покупатель в веб-интерфейсе вызывает `POST /certificates/{code}/refund`, указывая канал возврата и предпочтительный способ перечисления средств.
2. `GiftCertificateProcessing` проверяет остаток, создаёт запись `refund_request`, фиксирует операцию `RefundPending`, статус сертификата меняется на `RefundPending`.
3. Сервис публикует `GiftCertificateTransaction(RefundRequested)` и инициирует событие `All_Payments` с `paymentStatus=RefundInitiated`, `refundDetails` и `orderLineId`.
4. После того как `Payments` обработает возврат и опубликует `All_Payments` с `paymentStatus=RefundSettled`, `GiftCertificateProcessing` обнуляет баланс, создаёт операцию `Refund`, статус меняется на `Refunded`.
5. Публикуются `GiftCertificateState(Refunded)` и `GiftCertificateTransaction(Refund)`, `BI Pipeline` фиксирует информацию для отчётности.
