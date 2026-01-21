# Memory Bank: VLMHyperBench

## Философия (Core Principles)

1.  **Viral Modularity (Вирусная модульность)**: Система строится из атомарных, независимых компонентов. Каждый компонент (метрика, датасет, итератор) — это полноценный Python-пакет, полезный сам по себе. Это способствует переиспользованию кода в сторонних проектах (идея Андрея Карпаты).
2.  **Composability (Композируемость)**: Сложные бенчмарки собираются из независимых модулей, как из конструктора LEGO.
3.  **Dynamic Composition**: Отказ от статических workspaces и submodules. Зависимости разрешаются динамически через YAML-реестры и устанавливаются JIT (Just-in-Time) в контейнеры.

## Обзор проекта
VLMHyperBench — это open-source фреймворк для оценки возможностей Vision Language Models (VLM), с акцентом на распознавание документов (включая русский язык). Инструмент позволяет исследователям и инженерам сравнивать модели, запускаемые на различных фреймворках (Hugging Face, vLLM, SGLang), в изолированных окружениях.

## Архитектура (v0.2.0)
VLMHyperBench трансформировался в полноценную платформу с разделением на Management, Execution и Inference слои.

### Основные сущности и компоненты

#### 1. Management Plane
*   **Backend (FastAPI)**: Центр управления экспериментами, база данных и BFF для аналитики.
*   **Web UI (React)**: Интерактивный дашборд для мониторинга в реальном времени и визуализации метрик (Plotly).

#### 2. Execution Plane (Orchestrator)
*   **BenchmarkPlanner**: Создает граф задач и управляет жизненным циклом бенчмарка.
*   **TaskTracker**: Мониторит состояния этапов (Planning, Inference, Evaluation, Reporting). Поддерживает инкрементальные запуски.
*   **EventBus**: Обеспечивает real-time стриминг событий выполнения, включая прогресс по объектам датасета и потребление ресурсов.

### 2. Management Layer (Backend)
*   **Backend (FastAPI)**: Центральный узел управления состоянием и данными. Обеспечивает API для создания экспериментов и агрегации результатов.

#### 3. Configuration Management (v0.2.0)
*   **Task Registry**: Система атомарных YAML-конфигураций в `vlmhyperbench/registries/`.
    *   `packages/`: Описание источников плагинов (`DependencySource`).
    *   `tasks/`: Определение типов задач (`MLTaskSchema`).
    *   `metrics/`, `reports/`, `datasets/`: Конфигурации конкретных инстансов.
    *   `runs/`, `experiments/`: Планы выполнения.
*   **RegistryManager**: Компонент для загрузки, мерджинга оверлеев (`RUN_MODE`) и кросс-валидации схем через Pydantic v2.

#### 4. Inference Layer (API Wrapper)
*   **API Wrapper**: FastAPI Proxy, унифицирующий доступ к vLLM, SGLang и HF через OpenAI-совместимый протокол.
*   **Structured Output**: Поддержка JSON Schema и Regex через `response_format` и `constrained decoding`.
*   **PromptManager**: Динамический выбор промптов на основе типа документа (`doc_type`) и конфигурации модели.
*   **Monitoring**: Двухуровневый мониторинг ресурсов (ML-метрики эффективности + системный Watchdog).

#### 5. Evaluation Layer
*   **DataParser**: Валидация JSON-ответов через **Pydantic**.
*   **Метрики**: Иерархия текстовых, структурных, классификационных и ресурсных метрик. Поддержка версионирования алгоритмов в `MetricRegistry`.
*   **Structural Fidelity**: Метрика валидности формата вывода.

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
*   **Язык**: Python 3.13+
*   **Пакетный менеджер**: uv 0.9.26 (uv.build).
*   **Валидация**: Pydantic v2.
*   **Инфраструктура**: Docker.
*   **Данные**: YAML (реестры) и Pandas (отчеты).
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
