# Настройка пользовательского пространства для работы с ai_model_hub

## Этап 1: Скачивание базы для тестирования

Выполните следующую команду:
```bash
git clone https://www.modelscope.cn/models/cix/ai_model_hub_25_Q3/
mkdir utils
BASE="https://modelscope.cn/models/cix/ai_model_hub_25_Q3/resolve/master/utils"
wget "$BASE/NOE_Engine.py" -P utils/
wget "$BASE/image_process.py" -P utils/
wget "$BASE/tools.py" -P utils/
wget "$BASE/__init__.py" -P utils/
```
Перенесите папку utils в папку ai_model_hub_25_Q3 с изменением всех файлов. Далее откройте requirements.txt и сделайте следующее:
- Удалите указание версии numpy, но не сам numpy;
- Добавьте следующие названия модулей через перенос строки: tqdm matplotlib streamlit.

## Этап 2: Создание userspace

Во время подготовки рабочего пространства вы установили uv. Теперь с помощью этого инструмента создайте виртуальное пространство с Python 3.12 (это важно, поскольку некоторые модули на Python 3.14, который идёт с ALT Linux, не установятся). Войдите в консоль, в папке, где будете создавать userspace, запустите следующую команду:
```bash
uv venv --seed --python 3.12
```
Затем войдите в пользовательское пространство:
```bash
source .venv/bin/activate
```

## Этап 3: Установка модулей

Установите модули для работами с моделями. Зайдите в ai_model_hub_25_Q3 и выполните следующее:
```bash
python3 -m pip install -r requirements.txt
```
Это займёт несколько десятков минут в среднем, наберитесь терпения.
После установки модулей перейдите в папку cix/pypi и установите все .whl файлы:
```bash
pip install --no-index --find-links="/путь/к/папке" /путь/к/папке/*.whl
```

После выполнения этих этапов будет создана конфигурация, готовая к работе с cix-моделями.
