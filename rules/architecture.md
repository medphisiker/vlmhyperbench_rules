# Архитектура VLMHyperBench (v0.2.0)

## Философия (Core Principles)

1.  **Viral Modularity (Вирусная модульность)**: Система строится из атомарных, независимых компонентов. Каждый компонент (метрика, датасет, итератор) — это полноценный Python-пакет, полезный сам по себе. Это способствует переиспользованию кода в сторонних проектах (идея Андрея Карпаты).
2.  **Composability (Композируемость)**: Сложные бенчмарки собираются из независимых модулей, как из конструктора LEGO.
3.  **Dynamic Composition**: Отказ от статических workspaces и submodules. Зависимости разрешаются динамически через YAML-реестры и устанавливаются JIT (Just-in-Time) в контейнеры.

## Обзор архитектуры
VLMHyperBench трансформировался в полноценную платформу с разделением на Management, Execution и Inference слои.

### Основные слои и компоненты

#### 1. Management Plane
*   **Backend (FastAPI)**: Центр управления экспериментами, база данных и BFF (Backend for Frontend) для аналитики. Подготавливает данные для визуализации (Plotly JSON).
*   **Web UI (React)**: Интерактивный дашборд для мониторинга в реальном времени и визуализации метрик.

#### 2. Execution Plane (Orchestrator)
*   **BenchmarkPlanner**: Создает граф задач и управляет жизненным циклом бенчмарка.
*   **TaskTracker**: Мониторит состояния этапов (Planning, Inference, Evaluation, Reporting). Поддерживает инкрементальные запуски.
*   **EventBus**: Обеспечивает real-time стриминг событий выполнения, включая прогресс по объектам датасета и потребление ресурсов.

#### 3. Configuration Management
*   **Task Registry**: Система атомарных YAML-конфигураций в `vlmhyperbench/registries/`.
    *   `packages/`: Описание источников плагинов (`DependencySource`). Поддерживает типы: `local`, `git`, `pypi`.
    *   `tasks/`: Определение типов задач (`MLTaskSchema`).
    *   `metrics/`, `reports/`, `datasets/`: Конфигурации конкретных инстансов.
    *   `runs/`, `experiments/`: Планы выполнения.
*   **RegistryManager**: Компонент для загрузки, мерджинга оверлеев и кросс-валидации схем через Pydantic v2.
    *   **Overlays (Оверлеи)**: Механизм переопределения конфигурации без изменения базовых файлов.
    *   **RUN_MODE**:
        *   `prod`: Используются стабильные источники (Git tags, PyPI).
        *   `dev`: Используются локальные пути из `.dev.yaml` оверлеев для редактируемой установки пакетов.

#### 4. Inference Layer (API Wrapper)
*   **API Wrapper Core**: Легковесное ядро, реализующее Control Plane (Load/Unload) и Data Plane (Inference) API.
*   **Backend Adapters**: Модульные пакеты (`vllm-adapter`, `deepseek-adapter`), загружаемые JIT.
*   **Security**: Авторизация через `X-API-Key`, безопасная инъекция секретов (`env`) и маскировка в логах.
*   **Structured Output**: Поддержка JSON Schema через `response_format`.
*   **Monitoring**: Двухуровневый мониторинг ресурсов (Latency/VRAM + системный Watchdog).

#### 5. Evaluation Layer
*   **DataParser**: Валидация JSON-ответов через **Pydantic**.
*   **Метрики**: Иерархия текстовых, структурных, классификационных и ресурсных метрик. Поддержка версионирования алгоритмов в `MetricRegistry`.
*   **Structural Fidelity**: Метрика валидности формата вывода.

## Поток данных (Data Flow)
1.  Пользователь заполняет `user_config.csv` или работает через UI.
2.  `run_benchmark.py` запускает `BenchmarkOrchestrator`.
3.  Оркестратор формирует список задач (`BenchmarkRunConfig`).
4.  **Loop по задачам**:
    *   **Data Preparation**: `SmartSyncManager` гарантирует наличие датасета в локальном кэше (Sync-before-Run).
    *   Запуск Docker-контейнера модели -> Монтирование данных из кэша -> `run_vlm.py` -> Генерация ответов (CSV).
    *   Запуск Docker-контейнера оценки -> Монтирование ответов -> `run_eval.py` -> Расчет метрик (CSV).
    *   **Artifacts Upload**: Загрузка результатов в удаленное хранилище (S3).

## Принципы проектирования
1.  **Async-First**: Все сетевые вызовы (API Wrapper, DB, WebSockets) должны быть асинхронными.
2.  **BFF (Backend for Frontend)**: Backend подготавливает данные для визуализации на стороне сервера.
3.  **Strict Validation**: Любой структурированный вывод модели должен валидироваться через Pydantic v2.
4.  **Dynamic Composition**: Зависимости плагинов разрешаются динамически через реестры. Пакеты являются автономными и не требуют статической регистрации в корневом проекте.

## Структура директорий (VLMHyperBench)

*   `VLMHyperBench/` — **Legacy Core**: Код в этой директории является устаревшим. При выполнении задач по переходу на v0.2.0 его можно и нужно изменять или переписывать без сохранения обратной совместимости.
*   `tmp_model_eval/` — **Legacy Scripts**: Очень старые реализации скриптов оценки. Будут обновлены или удалены после релиза v0.2.0.
*   `benchmark_scheduler/` — Логика планирования и управления Docker-контейнерами.
*   `packages/` — Директория для локальной разработки автономных пакетов.
    *   `data_layer/` — Реализация Data Layer (Smart Sync, S3/Local Storage).
    *   `dependency_injector/` — Механизм установки пакетов в контейнеры.
    *   `metric_registry/` — Динамический реестр метрик.
    *   `dataset_factory/` — Фабрика итераторов датасетов.
*   `vlmhyperbench/`
    *   `registries/` — YAML-реестры (tasks, runs, experiments, metrics, etc.).
    *   `data_dirs/` — Хранилище данных (датасеты, промпты, ответы моделей, отчеты).
    *   `system_dirs/` — Системные ресурсы, пробрасываемые в контейнеры:
        *   `bench_stages/` — Скрипты исполнения (`run_vlm.py`).
        *   `cfg/` — Конфигурационные файлы (`VLMHyperBench_config.json`, requirements).
    *   `vlm_base.csv` — База данных поддерживаемых моделей.
*   `run_benchmark.py` — Точка входа (Entry point) приложения.