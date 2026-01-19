# Memory Bank: VLMHyperBench

## Обзор проекта
VLMHyperBench — это open-source фреймворк для оценки возможностей Vision Language Models (VLM), с акцентом на распознавание документов (включая русский язык). Инструмент позволяет исследователям и инженерам сравнивать модели, запускаемые на различных фреймворках (Hugging Face, vLLM, SGLang), в изолированных окружениях.

## Архитектура
Система построена на модульной архитектуре, где каждый этап (инференс модели, оценка метрик) выполняется в изолированном Docker-контейнере.

### Основные сущности и компоненты

#### 1. Orchestrator (`BenchmarkOrchestrator`)
*   **Локация**: `VLMHyperBench/benchmark_scheduler/benchmark_orchestrator.py`
*   **Функции**:
    *   Центральный управляющий элемент.
    *   Считывает глобальную конфигурацию и список задач.
    *   Проверяет и скачивает необходимые Docker-образы.
    *   Запускает контейнеры для этапов инференса и оценки, монтируя необходимые директории (volumes).

#### 2. Configuration Management
*   **User Config**: Пользователь задает задачи в CSV файле (например, `user_config_model_eval.csv`), указывая датасет, модель, промпты и метрики.
*   **VLM Base**: Реестр моделей (`vlmhyperbench/vlm_base.csv`) хранит технические детали: имя Docker-образа, python-пакеты и классы для инициализации конкретной модели.
*   **UserConfigReader**: Класс (`VLMHyperBench/benchmark_scheduler/user_config_reader.py`), объединяющий пользовательский конфиг с базой VLM для создания полных конфигураций запуска (`BenchmarkRunConfig`).

#### 3. Этапы выполнения (Stages)
Скрипты этапов находятся в `VLMHyperBench/vlmhyperbench/system_dirs/bench_stages/` и монтируются в контейнеры.

*   **Stage: Run VLM (Инференс)**
    *   **Скрипт**: `run_vlm.py`
    *   **Описание**: Запускается внутри контейнера с GPU. Инициализирует модель через `ModelFactory`, создает итератор по датасету (`IteratorFabric`) и сохраняет ответы модели.
*   **Stage: Eval Metrics (Оценка)**
    *   **Скрипт**: `run_eval.py` (подразумевается архитектурой)
    *   **Описание**: Запускается внутри легковесного контейнера (`metric-evaluator`). Сравнивает ответы модели с эталонными значениями (Ground Truth) и считает метрики.

## Структура директорий (VLMHyperBench)

*   `benchmark_scheduler/` — Логика планирования и управления Docker-контейнерами.
*   `vlmhyperbench/`
    *   `data_dirs/` — Хранилище данных (датасеты, промпты, ответы моделей, отчеты).
    *   `system_dirs/` — Системные ресурсы, пробрасываемые в контейнеры:
        *   `bench_stages/` — Скрипты исполнения (`run_vlm.py`).
        *   `cfg/` — Конфигурационные файлы (`VLMHyperBench_config.json`, requirements).
    *   `vlm_base.csv` — База данных поддерживаемых моделей.
*   `run_benchmark.py` — Точка входа (Entry point) приложения.

## Технологический стек
*   **Язык**: Python 3.10+
*   **Инфраструктура**: Docker (управление через `docker-py`).
*   **Данные**: Pandas (работа с CSV конфигами и отчетами).
*   **Внутренние зависимости**:
    *   `benchmark-run-config` — Управление конфигурациями запусков.
    *   `config-manager` — Управление путями и настройками.

## Поток данных (Data Flow)
1.  Пользователь заполняет `user_config.csv`.
2.  `run_benchmark.py` запускает `BenchmarkOrchestrator`.
3.  Оркестратор формирует список задач (`BenchmarkRunConfig`).
4.  **Loop по задачам**:
    *   Запуск Docker-контейнера модели -> Монтирование данных -> `run_vlm.py` -> Генерация ответов (CSV).
    *   Запуск Docker-контейнера оценки -> Монтирование ответов -> `run_eval.py` -> Расчет метрик (CSV).

## Документация и Решения (ADR)
Основная документация проекта перенесена в `docs_site/` и публикуется через Docusaurus.

*   **Архитектура**:
    *   [`docs_site/docs/architecture/concept.md`](docs_site/docs/architecture/concept.md) — Основная концепция и диаграммы.
    *   [`docs_site/docs/architecture/specification.md`](docs_site/docs/architecture/specification.md) — Детальная спецификация компонентов.
    *   [`docs_site/docs/architecture/adr/`](docs_site/docs/architecture/adr/) — Architecture Decision Records (ADR).
*   **Исследования**:
    *   [`docs_site/docs/research/archive/`](docs_site/docs/research/archive/) — Архив проведенных исследований (S3, EvalScope, Metrics).
    *   [`docs_site/docs/research/comparative-analysis.md`](docs_site/docs/research/comparative-analysis.md) — Сравнительный анализ с аналогами.

## Инструменты (Tools)
Вспомогательные инструменты для разработки и CI/CD находятся в директории `tools/`:

*   **Release Tool** (`tools/release_tool`):
    *   Утилита для автоматизации релизного цикла.
    *   Управляет версиями пакетов, обновляет `uv.lock`, создает git tags.
*   **PlantUML Renderer** (`tools/plantuml-docker-renderer`):
    *   Docker-контейнер для локального рендеринга диаграмм PlantUML.
    *   Используется для генерации изображений для документации.
*   **Limited Tree** (`tools/limited_tree.py`):
    *   Утилита для отображения ограниченного дерева файлов проекта.
