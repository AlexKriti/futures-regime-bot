# futures-regime-bot

Локальная Python-платформа для исследования и разработки **regime-aware** торговой системы по фьючерсам.

Проект ориентирован на практический workflow:
- загрузка и хранение исторических данных,
- построение признаков,
- выделение рыночных режимов,
- генерация сигналов,
- realistic backtesting,
- paper trading,
- журналирование и отчёты.

В качестве основного источника данных используется Databento, у которого есть Python quickstart, historical API и поддержка futures market data, включая CME datasets [1][2][3][4].

## Цели проекта

- Построить локальную исследовательскую среду для работы с фьючерсами.
- Реализовать pipeline: `data -> features -> regimes -> signals -> backtest -> paper trading`.
- Проверять идеи честно: с учётом комиссий, slippage и walk-forward валидации, поскольку realistic cost modeling критичен для прикладного trading research [5][6][7].
- Сделать архитектуру, которую можно расширять под будущие курсовые, отчёты и более серьёзную инженерную реализацию .

## Стек

- Python 3.11+
- pandas, numpy, scikit-learn, scipy
- Databento Python client [3][4]
- matplotlib / plotly
- pydantic, pyyaml, python-dotenv
- typer для CLI
- pytest для тестов
- parquet/pyarrow для локального хранения данных

## Архитектура

```text
futures-regime-bot/
├─ README.md
├─ pyproject.toml
├─ requirements.txt
├─ .gitignore
├─ .env.example
├─ configs/
├─ data/
├─ notebooks/
├─ src/
│  └─ futures_regime_bot/
│     ├─ data/
│     ├─ features/
│     ├─ regimes/
│     ├─ signals/
│     ├─ backtest/
│     ├─ paper/
│     ├─ reporting/
│     └─ cli/
├─ tests/
└─ outputs/
```

Для поддерживаемого Python-проекта удобно использовать `src`-layout, отдельные тесты и централизованную конфигурацию через `pyproject.toml`, потому что такой подход упрощает масштабирование и уменьшает путаницу между кодом, ноутбуками и экспериментами [8][9][10][11].

## MVP

Первая рабочая версия проекта должна быть максимально узкой:
- один рынок — CME futures,
- один источник данных — Databento [1][3],
- один инструмент — например ES или MES,
- один таймфрейм — 1H или 4H,
- один pipeline regime detection,
- базовый signal engine,
- backtest с costs/slippage,
- paper trading journal.

Такой scope полезнее, чем попытка сразу поддержать много инструментов и сложную инфраструктуру, потому что сначала важнее стабилизировать data pipeline и validation layer [5][6][7].

## Структура модулей

### `data/`

Отвечает за загрузку данных из внешнего источника, нормализацию timestamp, хранение parquet-файлов и построение continuous series для фьючерсов. Databento отдельно документирует работу с historical API, futures datasets и continuous/futures conventions, что делает этот слой особенно важным [12][2][13].

### `features/`

Содержит расчёт доходностей, rolling volatility, трендовых и объёмных признаков. Здесь же собирается единый feature pipeline для downstream-моделей.

### `regimes/`

Ключевой исследовательский слой: кластеризация и интерпретация рыночных режимов. Этот блок логично опирается на уже проделанную пользователем работу по классификации рыночных режимов через unsupervised learning .

### `signals/`

Генерирует `long / short / flat` сигналы на основе признаков и режима рынка. На старте лучше использовать простой, интерпретируемый подход, а не сразу усложнять модель.

### `backtest/`

Реализует historical simulation, расчёт метрик, cost modeling и walk-forward validation. Walk-forward и realistic transaction cost modeling считаются важными защитами от переобучения и слишком оптимистичных бэктестов [5][6][7].

### `paper/`

Нужен для paper trading и сравнения “идеального” сигнала с симулированным исполнением. Это делает проект практичнее и приближает его к реальному использованию [14].

### `reporting/`

Формирует таблицы, графики, summary-отчёты и trade journal. Этот слой особенно полезен для последующих курсовых и исследовательских отчётов .

## Установка

### 1. Создать виртуальное окружение

```bash
python -m venv .venv
```

### 2. Активировать окружение

Linux/macOS:

```bash
source .venv/bin/activate
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

### 3. Установить зависимости

```bash
pip install -U pip
pip install -r requirements.txt
pip install -e .
```

### 4. Создать `.env`

```bash
cp .env.example .env
```

### 5. Указать API key Databento

Databento quickstart использует API key для доступа к historical и live market data [3][4].

```env
DATABENTO_API_KEY=your_api_key_here
```

## Пример `.env.example`

```env
DATABENTO_API_KEY=your_api_key_here
DEFAULT_DATASET=GLBX.MDP3
DEFAULT_SYMBOL=ES.c.0
DEFAULT_SCHEMA=ohlcv-1h
DEFAULT_START=2024-01-01
DEFAULT_END=2024-12-31
```

## Пример конфигов

### `configs/data.yaml`

```yaml
provider: databento
dataset: GLBX.MDP3
symbol: ES.c.0
schema: ohlcv-1h
start: "2024-01-01"
end: "2024-06-01"
timezone: UTC
store_format: parquet
```

### `configs/regimes.yaml`

```yaml
method: kmeans
n_clusters: 4
standardize: true
features:
  - ret_1
  - ret_4
  - vol_10
  - vol_20
  - trend_20
```

### `configs/backtest.yaml`

```yaml
initial_cash: 100000
commission_per_trade: 2.5
slippage_bps: 1.0
position_size: 1
walk_forward:
  train_bars: 500
  test_bars: 100
```

## Типовой workflow

1. Скачать исторические данные.
2. Сохранить их локально в parquet.
3. Построить признаки.
4. Запустить выделение режимов.
5. Сгенерировать сигналы.
6. Прогнать backtest.
7. Посчитать метрики.
8. Запустить paper trading loop.
9. Сохранить trade log и отчёт.

Именно поэтапный pipeline удобнее для отладки и расширения, чем один “магический” скрипт, который делает всё сразу [8][9].

## CLI-команды

Минимальный набор команд для первой версии:

```bash
python -m futures_regime_bot.cli.main download-data
python -m futures_regime_bot.cli.main build-features
python -m futures_regime_bot.cli.main detect-regimes
python -m futures_regime_bot.cli.main generate-signals
python -m futures_regime_bot.cli.main backtest
python -m futures_regime_bot.cli.main run-paper
python -m futures_regime_bot.cli.main make-report
```

## Первая неделя разработки

### День 1
- создать репозиторий;
- добавить `pyproject.toml`, `requirements.txt`, `.gitignore`, `.env.example`;
- создать базовую структуру папок.

### День 2
- подключить Databento client;
- проверить API key;
- скачать один исторический датасет и сохранить локально. Databento предоставляет quickstart и примеры Python-запросов для historical data [2][3].

### День 3
- реализовать модуль хранения и нормализации данных;
- привести колонки и timestamps к единому формату.

### День 4
- реализовать базовые фичи: returns, volatility, trend;
- сохранить processed dataset;
- написать первый unit test.

### День 5
- реализовать baseline regime detection, например K-Means;
- визуально проверить, как режимы выглядят на истории .

### День 6
- собрать базовую signal logic;
- добавить краткое reason text для каждого сигнала.

### День 7
- реализовать простой backtest;
- сразу добавить комиссии и slippage, потому что без них оценка стратегии обычно становится слишком оптимистичной [6][7].

## Правила проекта

- Не хардкодить параметры экспериментов в ноутбуках.
- Всё, что влияет на результат, хранить в `configs/`.
- Все прогоны должны сохранять артефакты: equity curve, trade log, summary metrics.
- Сначала простая и прозрачная логика, потом усложнение модели.
- Сначала backtest + paper trading, а не реальная торговля .

## Что можно добавить позже

- HMM или Gaussian Mixture для regime detection.
- Градиентный бустинг для signal engine.
- SHAP для explainability.
- HTML/Markdown-отчёты.
- Telegram-уведомления.
- Лёгкий локальный dashboard.

## Статус

Текущая цель проекта — не “автономный денежный бот”, а практичная исследовательская система для futures trading, которую можно развивать шаг за шагом и использовать как базу для дальнейших учебных и инженерных работ .
