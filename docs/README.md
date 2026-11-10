## Навігація архітектурного контуру (Docs-as-Code)
* [Специфікація вимог SRS (ISO/IEC/IEEE 29148)](./srs.md)
* [Розділ 2: Архітектурне проєктування системи (C4 Model)](./2_design.md)
* [Реєстр архітектурних рішень (ADR Repository)](./adr/0001-choice-of-technology.md)

---
### Посилання на зведений Розділ 1 (Google Docs)
- [Переглянути монолітний Розділ 1 диплома (F2 Diploma Project)](https://docs.google.com/document/d/1NPzdy1CthtalK_e4KKUvTy2_soYByN0HtYWEU4l7nXc/edit?usp=sharing)

### Зведена матриця нефункціональних вимог (NFRs Matrix)
| Категорія | Параметр / Вимога | Технічне значення |
| :--- | :--- | :--- |
| **Продуктивність** | Цільовий RPS API | Не менше 500 RPS |
| **Швидкодія** | Затримка (Latency p95) | Менше 200 мс |
| **Безпека** | Аутентифікація / Протоколи | OAuth 2.0 / OIDC, JWT, TLS 1.3 |
| **Архітектура** | Стек і брокер подій | FastAPI, RabbitMQ, PostgreSQL, Docker |
