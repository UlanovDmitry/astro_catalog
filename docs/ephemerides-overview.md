# Системы эфемерид и Python-библиотеки

Обзор основных **систем эфемерид** и **Python-библиотек** для работы с ними.

## Системы эфемерид (источники данных)

### Численные эфемериды (наиболее точные)

| Система | Организация | Диапазон / примечания |
|--------|-------------|------------------------|
| **JPL DE** (DE405, DE421, DE430, **DE440**, **DE441**) | NASA JPL | Стандарт для профессиональной астрономии; DE440/DE441 — актуальные версии |
| **INPOP** (INPOP17a, INPOP19a, INPOP21a) | IMCCE, Observatoire de Paris | Альтернатива JPL, сопоставимая точность |
| **EPM** (EPM2011, EPM2021) | ИПА РАН (Санкт-Петербург) | Российская система, хороша для Солнечной системы |
| **SPICE kernels** (SPK) | NASA NAIF | Файлы `.bsp` для планет, спутников, космических аппаратов |

### Аналитические теории (формулы, не таблицы)

| Система | Назначение |
|--------|------------|
| **VSOP87 / VSOP2000 / VSOP2013** | Планеты (гелиоцентрические координаты) |
| **ELP2000 / ELP/MPP02** | Луна |
| **MOSH / Meeus** | Упрощённые алгоритмы (книга «Astronomical Algorithms») |

### Специализированные

| Система | Назначение |
|--------|------------|
| **Swiss Ephemeris** (Astrodienst) | На базе JPL + собственные расширения; популярна в астрологии, много объектов (астероиды, фиктивные точки) |
| **JPL Horizons** | Онлайн-сервис NASA (не локальная библиотека, но эталон для проверки) |
| **Gaia DR2/DR3** | Звёздные каталоги (не планетные эфемериды, но нужны для звёздных координат) |

---

## Python-библиотеки

### 1. Skyfield + jplephem — рекомендуемый старт для научных расчётов

- **[Skyfield](https://rhodesmill.org/skyfield/)** — высокоуровневая библиотека, чистый Python
- **[jplephem](https://github.com/brandon-rhodes/python-jplephem)** — низкоуровневое чтение файлов JPL (`.bsp`)

```python
from skyfield.api import load

ts = load.timescale()
planets = load('de440s.bsp')  # или de421.bsp, de441.bsp
earth, mars = planets['earth'], planets['mars']
t = ts.utc(2025, 6, 17, 12, 0, 0)
astrometric = earth.at(t).observe(mars)
ra, dec, distance = astrometric.radec()
```

**Плюсы:** точность на уровне Astronomical Almanac, планеты/Луна/спутники, звёзды, TLE для спутников, интеграция с Astropy.

**Минусы:** нужно скачивать файлы эфемерид (десятки–сотни МБ для полных версий).

---

### 2. Astropy — координаты, время, встроенные эфемериды

```python
from astropy.coordinates import get_body, solar_system_ephemeris
from astropy.time import Time

with solar_system_ephemeris.set('jpl'):
    pos = get_body('mars', Time('2025-06-17'))
```

**Плюсы:** экосистема (единицы, системы координат, ERFA), `jpl`/`de430`/`de432s` из коробки.

**Минусы:** менее удобна для сложных сценариев (видимость, затмения), чем Skyfield.

---

### 3. pyswisseph / libephemeris — для астрологии и Swiss Ephemeris API

| Библиотека | Особенности |
|-----------|-------------|
| **[pyswisseph](https://github.com/astrorigin/pyswisseph)** | Опривязка к C-библиотеке Swiss Ephemeris; индустриальный стандарт в астрологии |
| **[libephemeris](https://github.com/g-battaglia/libephemeris)** | Чистый Python, API совместим с pyswisseph, бэкенды: JPL DE440/DE441 через Skyfield, Horizons API |

```python
import swisseph as swe  # pyswisseph

jd = swe.julday(2025, 6, 17, 12.0)
pos, ret = swe.calc_ut(jd, swe.SUN)
```

**Плюсы:** много объектов (астероиды, лунные узлы, лилит и т.д.), зодиакальные/сидereal системы.

**Минусы:** pyswisseph требует сборки C-расширения; лицензия Swiss Ephemeris — AGPL (коммерция платная).

---

### 4. SPICE (spiceypy) — для космических миссий

```python
import spiceypy as spice
spice.furnsh('de440.bsp')
spice.furnsh('naif0012.tls')
state, lt = spice.spkezr('MARS', et, 'J2000', 'NONE', 'EARTH')
```

**Плюсы:** эталон NASA для траекторий, спутников, аппаратов.

**Минусы:** более низкоуровневый API, крутая кривая обучения.

---

### 5. PyEphem — устаревающий вариант

- **[PyEphem](https://github.com/brandon-rhodes/pyephem)** — быстрый, простой, но **устарел** (автор рекомендует Skyfield)
- Точность ниже, чем у JPL DE; подходит только для грубых расчётов

---

### 6. Специализированные и вспомогательные

| Библиотека | Назначение |
|-----------|------------|
| **novas** / **novas_de405** | USNO NOVAS (средняя точность) |
| **poliastro** | Орбитальная механика, Kepler, perturbations |
| **rebound** | N-body симуляции |
| **horizons** (обёртки) | Запросы к JPL Horizons по HTTP |
| **pyerfa** / **ERFA** (через Astropy) | Фундаментальная астрономия (precession, nutation), не полные эфемериды |

---

## Как выбрать

| Сценарий | Рекомендация |
|---------|--------------|
| Каталог небесных тел, координаты планет/Луны | **Skyfield + DE440s** |
| Интеграция с научным стеком (fits, координаты) | **Astropy** |
| Астрологические расчёты, астероиды | **pyswisseph** или **libephemeris** |
| Траектории спутников/зондов | **spiceypy** |
| Без локальных файлов, разовые запросы | **JPL Horizons API** (через libephemeris или свой клиент) |
| Быстрый прототип, низкие требования | **PyEphem** (не рекомендуется для новых проектов) |

---

## Файлы эфемерид JPL (часто используемые)

| Файл | Период | Размер |
|------|--------|--------|
| `de421.bsp` | 1900–2050 | ~16 МБ |
| `de440s.bsp` | 1849–2150 | ~32 МБ (short) |
| `de440.bsp` | −13200 – +17191 | ~115 МБ |
| `de441.bsp` | −13200 – +17191 | ~3 ГБ (максимальный диапазон) |

Skyfield скачивает их автоматически при первом `load('de440s.bsp')`.

---

## Рекомендация для astro_catalog

Для проекта вроде `astro_catalog` обычно оптимален **Skyfield + DE440s**: высокая точность, чистый Python, удобный API для планет, Луны, звёзд и времени. Если нужны астрологические объекты или совместимость с Swiss Ephemeris — смотрите **libephemeris**.
