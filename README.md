# Наблюдаемость LLM-приложения в Langfuse

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/EkaterinaGrisha/llm-observability-langfuse/blob/main/llm_observability_langfuse.ipynb)
[![nbviewer](https://img.shields.io/badge/render-nbviewer-orange)](https://nbviewer.org/github/EkaterinaGrisha/llm-observability-langfuse/blob/main/llm_observability_langfuse.ipynb)

Трассировка запросов, версионирование промптов, A/B-тестирование, эксперименты с параметрами модели и конвейер оценки качества (dataset, experiment, LLM-as-a-Judge) в Langfuse. Работа выполнена в рамках практического задания курса по работе с LLM (практика 1, Langfuse).

**Стек экспериментов:**
- платформа наблюдаемости — Langfuse Cloud (тариф Hobby);
- модель-генератор — `GigaChat-2` (тариф Freemium), вызывается через LangChain;
- модель-оценщик (LLM-as-a-Judge) — `deepseek-flash` (DeepSeek API).

Все ответы моделей получены в результате реальных запросов. Результаты экспериментов и данные из API Langfuse сохранены в каталоге `results/`.

## Содержание

| № | Задание | Что сделано |
|---|---|---|
| 1 | Базовое подключение и трассировка | 10 запросов с тегами `baseline` / `test` / `experiment1`, `user_id`, `session_id`; анализ длительности, токенов и структуры вызова; публичные трейсы |
| 2 | Версионирование промптов | версии v1 (базовая инструкция), v2 (улучшенная инструкция) и v3 (системная роль) в Prompt Management; 5 задач × 2 повтора; качество, длина, время, токены, стабильность |
| 3 | A/B-тестирование | 40 запросов со случайным выбором варианта 50/50, оценка 1–5, t-критерий Уэлча и U-критерий Манна — Уитни |
| 4 | Параметры модели | серии `temperature` 0,2 / 0,7 / 1,0, `max_tokens` 150 / 400 / 1000 и три системных промпта, параметры в metadata, распределение длительности |
| 5 | Собственный кейс и конвейер оценки | ассистент службы поддержки; датасет из 10 обращений; 4 метрики на основе кода, LLM-as-a-Judge и агрегированные метрики; разбор неудачного трейса; сравнение промптов v1 и v2 |

## Основные результаты

- **Трассировка.** Langfuse фиксирует промпт, ответ, модель с версией (`GigaChat-2:2.0.30.01`), параметры, токены и время каждого шага. На генерацию приходится 99,9% длительности трейса.
- **Учёт токенов.** Обнаружена особенность интеграции: закэшированная часть промпта GigaChat в Langfuse не учитывается во входных токенах, поэтому полный размер промпта пришлось восстанавливать вручную.
- **Версии промпта.**
  - Явные требования к структуре (v2) обеспечили 100% соблюдения формата вместо 0%, сократили ответ на 23% и время генерации на 22%, вдвое снизили разброс длины.
  - Системная роль (v3) дала наибольшее качество (3,74 из 5) и достоверность.
  - Достоверность во всех версиях остаётся низкой: модель дописывает характеристики, которых нет в описании товара.
- **A/B-тест.** По пользовательской оценке (3,52 и 3,53; p = 0,86) и времени ответа (p = 0,30) варианты не различаются. Вариант B расходует на 15% меньше токенов (p < 10⁻⁶), поэтому он статистически лучше по стоимости.
- **Параметры.**
  - `temperature` управляет разнообразием: вариативность растёт с 0,23 до 0,44 при неизменных времени и длине ответа.
  - Системный промпт определяет объём и время ответа: от 0,97 до 4,23 с.
  - `max_tokens` только обрывает ответ.
  - Время ответа растёт с числом выходных токенов: ρ = 0,74, около 5,9 мс на токен.
- **Конвейер оценки.**
  - `pass_rate` = 0,4. Основной тип ошибок — выдумывание фактов и процедур, когда ответа нет в регламенте (например, «поддержка работает круглосуточно»).
  - Промпт v2 изменил характер ошибок, но не их число. Это показало, что версии нужно сравнивать поэлементно, а не только по агрегированным метрикам.

## Публичные трейсы

Трейсы открываются без авторизации, пока данные хранятся в проекте Langfuse:

- [трейс 1 (baseline)](https://cloud.langfuse.com/project/cmv0zn87700w1ad0gzxrqpptb/traces/c944f98b1098a44121b6d7a1f5f29e4b)
- [трейс 2 (baseline)](https://cloud.langfuse.com/project/cmv0zn87700w1ad0gzxrqpptb/traces/255e9345314cbb0e8015b789cd590b3c)

## Структура репозитория

```
llm_observability_langfuse.ipynb   ноутбук с кодом, результатами и выводами
results/                           результаты экспериментов и снимки данных из API Langfuse (JSON)
requirements.txt                   зависимости
.env.example                       шаблон файла с ключами
certs/russian_trusted_root_ca.pem  корневой сертификат Минцифры для проверки SSL GigaChat
```

## Воспроизведение

1. Установить зависимости: `pip install -r requirements.txt` (проверено на Python 3.11).
2. Скопировать `.env.example` в `.env` и указать ключи:
   - Langfuse — `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`;
   - GigaChat — `GIGACHAT_CREDENTIALS`;
   - при необходимости DeepSeek — `DEEPSEEK_API_KEY`. Без него оценщиком будет `GigaChat-2-Max`.
3. Открыть ноутбук и выполнить ячейки.

При наличии каталога `results/` повторный запуск не создаёт новых трейсов и воспроизводит приведённые результаты. Чтобы провести эксперименты заново в своём проекте Langfuse, нужно установить `RERUN = True`.

Сертификат `certs/russian_trusted_root_ca.pem` загружен с официального ресурса https://www.gosuslugi.ru/crt. Его отпечаток SHA-256: `D2:6D:2D:02:31:B7:C3:9F:92:CC:73:85:12:BA:54:10:35:19:E4:40:5D:68:B5:BD:70:3E:97:88:CA:8E:CF:31`. Сертификат используется только клиентом GigaChat.

## Технологии

Python, Jupyter, Langfuse SDK 4, LangChain, `langchain-gigachat`, `langchain-openai`, pandas, SciPy, Matplotlib.

## English summary

LLM observability with Langfuse, using GigaChat as the generator and DeepSeek as an independent LLM judge. Work covered:
- request tracing with users, sessions, tags and metadata;
- prompt versioning in Prompt Management;
- an A/B test of two prompts with statistical tests;
- tracing of generation-parameter experiments;
- a custom evaluation pipeline: a dataset, code-based metrics, LLM-as-a-Judge and run-level metrics.

Key findings:
- Explicit structural instructions had the largest effect on format compliance, length and latency.
- The A/B test detected a 15% token saving at indistinguishable quality.
- Temperature controls diversity, the system prompt controls length and latency, and `max_tokens` only truncates.
- The main failure mode of the support assistant is inventing facts when the policy is silent.

The notebook is written in Russian.
