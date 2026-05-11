# DB-Schema-Investor-Diary

**Категория**: #concept/infrastructure  
**Уровень**: intermediate  
**Статус**: active  
**Связи**: [[ML-Pipeline-Investor-Diary]], [[API-Investor-Diary]], [[Deployment-Options-Investor-Diary]]

##  Определение
Нормализованная PostgreSQL-схема для учёта сделок, денежных потоков и портфельной аналитики с поддержкой мультиброкерности, JSONB-метаданных и точного расчёта доходности.

## 🎯 Зачем нужно
- Без корректной структуры невозможны точный расчёт TWRR/XIRR и агрегация по тикерам/секторам
- Позволяет отделить сырые сделки от денежных потоков (депозиты/выводы/дивиденды)
- JSONB-поля хранят гибкие метки (цель входа, риск-факторы, выводы) без изменения схемы
- Индексация по `exec_dt` и `ticker` ускоряет отчёты в 10-50 раз

## 🔧 Как работает

### Таблицы ядра
| Таблица | Ключевые поля | Назначение |
|---------|--------------|------------|
| `accounts` | id, broker_name, base_currency, status | Поддержка нескольких брокеров/счетов |
| `instruments` | ticker, asset_type, sector, currency, isin | Справочник инструментов |
| `transactions` | id, account_id, type, exec_dt, qty, price, fee, tax, comment_json, labels_json | Ядро дневника. Фиксирует покупку/продажу |
| `corporate_actions` | id, instrument_id, type, dt, amount_per_unit | Дивиденды/купоны/сплиты отдельно |
| `cashflows` | id, account_id, dt, amount, type (deposit/withdraw/div/fee) | Чистые потоки для XIRR |
| `portfolio_snapshots` | id, account_id, dt, total_value, benchmark_value | Ежедневная оценка для TWRR |
| `ml_features` | id, transaction_id, feature_vector (JSON/ARRAY), label | Готовая матрица для ML |

### Индексы и безопасность
- `B-tree (exec_dt)`, `GIN (comment_json, labels_json)`, составной `(account_id, instrument_id, exec_dt)`
- Цены хранятся в валюте сделки + пересчёт в `base_currency` по курсу ЦБ на дату
- Шифрование: `pgcrypto` для чувствительных полей или LUKS на уровне ФС

## 📊 Примеры применения
- Импорт CSV из Тинькофф → маппинг в `transactions` + `cashflows` → автопересчёт `portfolio_snapshots`
- Агрегация по `labels_json -> goal` → отчёт "Доля сделок с горизонтом > 1 года"

## ⚠️ Ограничения и риски
- ❌ JSONB требует валидации на уровне приложения (иначе "мусор" в метках)
- ❌ Сложность миграций при изменении структуры `labels_json`
- ❌ Требует регулярной очистки дублей при импорте выписок

## 📚 Источники и ссылки
- [[raw/papers/postgresql-jsonb-best-practices.md]]
- [[entities/Finam-Tinkoff-CSV-Formats]] — специфика экспорта брокеров
- Внешние: PostgreSQL Docs, XIRR/TWRR математика (CFA Institute)

##  История изменений
| Дата | Изменение | Автор | Ссылка на решение |
|------|-----------|-------|------------------|
| {{date}} | Создана на основе архитектуры QWEN | Oleg | — |

## 🔗 Связанные страницы
- [[ML-Pipeline-Investor-Diary]] ← потребляет `ml_features`
- [[API-Investor-Diary]] ← эндпоинты для CRUD операций
- [[Cashflow-Tracking]] ← детализация расчёта XIRR
