# Changelog

## 2025-11-08

- Переведён файл `specs/clarification.md` в формат FAQ с указанием ответственных ролей заказчика.
- Зафиксированы архитектурные решения ADR-001 и ADR-002, связанные с частичными платежами и запретом повторной продажи кодов.
- Обновлены спецификации (`summary`, `architecture`, `erd`, `openapi`, `flows`, `sequences`, `tasks`) для отражения контроля уникальности продажи и обработки агрегированных платежей.
- Сгенерированы обновлённые диаграммы `sequences.png` и `flow.png`, соответствующие новым процессам.
- Добавлены уточнения по распределению платежей между сертификатами, требованиям к событиям `All_Payments`, моменту активации сертификата и структуре позиций заказа (`specs/clarification.md`).
- Интегрированы требования по пользовательскому распределению сумм: обновлены `summary`, `architecture`, `openapi`, `erd`, `tasks`, диаграммы (`architecture.mmd`, `flow.mmd`, `sequences.mmd`) и тексты (`flows.md`, `sequences.md`) для поддержки `certificateLines`, `certificateAllocations`, таблицы `order_certificate_allocation` и задержанной активации до полной оплаты заказа.
- Проработан процесс возврата сертификата: уточнения в `clarification.md`, поддержка возвратов и таблицы `refund_request` в архитектуре и ERD, новый endpoint `refund` в OpenAPI, расширены потоки, диаграммы и задачи для команд Payments, OnlineStore, POS и GiftCertificateProcessing.
