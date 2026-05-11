# Настройка Vite проекта

Пошаговая инструкция с пояснениями

---

## Чеклист

- ✅ Создание проекта
- ✅ FSD структура папок
- ✅ Алиасы путей (@/...)
- ✅ ESLint + Prettier
- ✅ husky + lint-staged + commit-msg
- ✅ Vitest
- ✅ env-config.js (runtime переменные)
- ✅ nginx.conf
- ✅ Dockerfile

---

## Шаг 1 — Создание проекта

Создаёт базовую структуру с React + TypeScript + Vite.

```bash
npm create vite@latest vite-react-setup-01 -- --template react-ts
cd vite-setup-01
npm install
npm run dev
```

> ⚠ Windows: localhost может не открываться — резолвится в IPv6 ::1 вместо 127.0.0.1

Исправление — добавить в vite.config.ts:

```typescript
server: {
  host: '127.0.0.1',
  port: 3000,
}
```

---

## Шаг 2 — FSD структура папок

Feature Sliced Design — архитектурная методология. Каждый слой имеет чёткую зону ответственности.

**WINDOWS:**
```powershell
mkdir src\app, src\pages, src\widgets, src\features, src\entities, src\shared
mkdir src\shared\ui, src\shared\api, src\shared\lib, src\shared\config
```

**UNIX:**
```bash
mkdir -p src/{app,pages,widgets,features,entities,shared/{ui,api,lib,config}}
```

| Слой | Зона ответственности |
|------|---------------------|
| app/ | Провайдеры, роутер, глобальные стили |
| pages/ | Страницы приложения |
| widgets/ | Крупные блоки UI (хедер, сайдбар) |
| features/ | Бизнес-фичи (авторизация, поиск) |
| entities/ | Бизнес-сущности (User, Product) |
| shared/ | Переиспользуемый код (ui, api, lib) |

---

## Шаг 3 — Алиасы путей

Позволяет писать `@/shared/ui/Button` вместо `../../shared/ui/Button`.

Начиная с Vite 6+ алиасы поддерживаются нативно через опцию `resolve.tsconfigPaths`. Отдельный пакет `vite-tsconfig-paths` больше не нужен.

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  resolve: {
    tsconfigPaths: true
  },
  server: {
    host: '127.0.0.1',
    port: 3000
  }
})
```

### tsconfig.app.json — добавить в compilerOptions

```json
"ignoreDeprecations": "6.0",
"baseUrl": ".",
"paths": { "@/*": ["src/*"] }
```

**Зачем ignoreDeprecations:**

TypeScript 6.0 объявил `baseUrl` устаревшим. Опция будет удалена только в TypeScript 7.0. До этого `ignoreDeprecations: "6.0"` подавляет предупреждение. Без `baseUrl` опция `paths` не работает — TypeScript требует её как якорь для разрешения путей.

---

## Шаг 4 — ESLint + Prettier

| Инструмент | Зона ответственности |
|------------|---------------------|
| ESLint | Качество кода — баги, плохие паттерны, логика |
| Prettier | Форматирование — отступы, кавычки, длина строки |
| eslint-config-prettier | Отключает ESLint правила конфликтующие с Prettier |

### Установка

```bash
npm install -D prettier eslint-config-prettier
```

### .prettierrc

```json
{
  "singleQuote": true,
  "semi": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

### eslint.config.js — добавить импорт и последним в extends

```javascript
import prettierConfig from 'eslint-config-prettier'

export default defineConfig([{
  extends: [
    js.configs.recommended,
    tseslint.configs.recommended,
    reactHooks.configs.flat.recommended,
    reactRefresh.configs.vite,
    prettierConfig,  // всегда последним
  ],
}])
```

Все настройки форматирования — только в `.prettierrc`. ESLint не трогаем.

### Форматирование при сохранении и потере фокуса (WebStorm)

**Цель настройки:**

| Действие | Результат |
|----------|-----------|
| Cmd+S / Ctrl+S | Сохраняет + форматирует ✅ |
| Переход в другое окно | Сохраняет + форматирует ✅ |
| Пауза в печати | Ничего не происходит ✅ |

**1. Отключить автосохранение по таймауту**

Путь: `Settings → Appearance & Behavior → System Settings`

| Параметр | Значение |
|----------|----------|
| Save files automatically if application is idle for N seconds | ☐ ОТКЛЮЧИТЬ |
| Save files on frame deactivation | ☑ ВКЛЮЧИТЬ |

Первая опция вызывает сохранение при паузе в печати — это мешает. Вторая опция сохраняет файл при переключении на другое окно (браузер, терминал и т.д.).

**2. Настроить Prettier**

Путь: `Settings → Languages & Frameworks → Prettier`

| Параметр | Значение |
|----------|----------|
| Prettier package | node_modules/prettier (автоопределение) |
| Run for files | \*\*/\*.{js,ts,jsx,tsx,css,html} |
| Run on save | ☑ ВКЛЮЧИТЬ |
| Run on reformat | ☐ ОТКЛЮЧИТЬ |

WebStorm найдёт Prettier автоматически если он установлен в проекте. Run on save — форматирует файл при каждом сохранении (Cmd+S и потеря фокуса).

> ⚠ Run on reformat лучше отключить — иначе Prettier будет запускаться дважды: один раз через Reformat code и второй раз через Run on save.

**3. Настроить ESLint**

Путь: `Settings → Languages & Frameworks → JavaScript → Code Quality Tools → ESLint`

| Параметр | Значение |
|----------|----------|
| Конфигурация | ● Automatic ESLint configuration |
| Run eslint --fix on save | ☑ ВКЛЮЧИТЬ |

Automatic configuration — WebStorm сам найдёт eslint.config.js в проекте. Run eslint --fix on save — автоматически исправляет ошибки линтера при сохранении.

**4. Actions on Save (альтернативный способ)**

Путь: `Settings → Tools → Actions on Save`

| Параметр | Значение |
|----------|----------|
| Reformat code | ☑ ВКЛЮЧИТЬ |
| Optimize imports | ☑ ВКЛЮЧИТЬ |
| Run eslint --fix | ☑ ВКЛЮЧИТЬ |
| Run Prettier | ☑ ВКЛЮЧИТЬ |

Actions on Save — это единое место где собраны все действия при сохранении. Если настроил шаги 2 и 3, дублировать здесь не обязательно. Но Reformat code и Optimize imports доступны только здесь.

**Важный нюанс: Cmd+S vs потеря фокуса**

"On save" в WebStorm — это не только Cmd+S. Если включена опция "Save files on frame deactivation" (Шаг 1), то переключение на другое окно тоже считается сохранением.

Цепочка событий при потере фокуса:
```
Переключился на браузер
  → WebStorm: "Save files on frame deactivation"
  → Файл сохраняется
  → Срабатывает "on save"
  → ESLint --fix + Prettier запускаются
  → Файл отформатирован ✅
```

Цепочка событий при паузе в печати:
```
Перестал печатать на 3 секунды
  → Опция "Save files automatically" ОТКЛЮЧЕНА
  → Файл НЕ сохраняется
  → Ничего не происходит ✅
```

**Проверка:**
1. Открой любой .tsx файл
2. Напечатай код с нарушением форматирования
3. Сделай паузу — ничего не происходит ✅
4. Нажми Cmd+S — файл отформатировался ✅
5. Переключись на браузер и обратно — файл отформатировался ✅

### Что может и чего не может Prettier и ESLint

**Сводная таблица:**

| Инструмент | Что делает | Синтаксис |
|------------|-----------|-----------|
| Prettier | Форматирование: отступы, пробелы, кавычки, длина строк | Только валидный |
| ESLint --fix | Паттерны: == → ===, var → const, неиспользуемые переменные | Только валидный |
| TypeScript | Подчёркивает ошибки в реальном времени | Любой |

> ⚠ Сломанный синтаксис = Prettier не может распарсить файл и ничего не делает. Исправление синтаксических ошибок — задача разработчика.

**Что исправляет Prettier (форматирование):**

Prettier работает только с синтаксически валидным кодом. Он не меняет логику — только внешний вид.

ДО сохранения:
```typescript
const user={name:"John",age:30,city:"Moscow"}
function   greet(  name:string   ){return `Hello, ${name}`}
const arr=[1,2,
    3,4,
            5]
```

ПОСЛЕ сохранения — Prettier исправит:
```typescript
const user = { name: 'John', age: 30, city: 'Moscow' };
function greet(name: string) {
  return `Hello, ${name}`;
}
const arr = [1, 2, 3, 4, 5];
```

Prettier исправляет: отступы, пробелы, кавычки (двойные → одинарные), запятые в конце, длину строк, скобки.

**Что исправляет ESLint --fix (паттерны):**

ESLint находит плохие паттерны и исправляет только те, которые можно исправить безопасно:

```typescript
// ДО — ESLint исправит
if (x == 1) {}         // → if (x === 1) {}
var name = 'test';     // → const name = 'test'

// ESLint подсветит, но НЕ исправит
let unused = 5;        // предупреждение: unused variable
```

**Что НЕ исправят ни Prettier ни ESLint:**

Синтаксические ошибки — это сломанный код, который невозможно распарсить:

Пропущенные запятые:
```typescript
const obj = {
  name: "John"     // ← нет запятой
  age: 30          // ← нет запятой
  city: "Moscow"
}
```

Незакрытые скобки:
```typescript
function test() {
  console.log("hello"
}
```

Опечатки в именах:
```typescript
const naem = "John";   // хотел name
consol.log("test");    // хотел console
```

Синтаксические ошибки подсвечивает TypeScript в редакторе (красное подчёркивание). Исправлять их — задача разработчика.

> ⚠ Если после переключения на браузер код не изменился — проверь настройки WebStorm: Save files on frame deactivation, Prettier Run on save, ESLint --fix on save.

---

## Шаг 5 — husky + lint-staged + commit-msg

| Инструмент | Назначение |
|------------|-----------|
| husky | Перехватывает git события (pre-commit, commit-msg) |
| lint-staged | Запускает ESLint + Prettier только на staged файлах |
| commit-msg | Проверяет формат сообщения коммита |

### 5.1 — Установка

**Важно:** всегда выполняй `git init` до `npx husky init`. Husky регистрирует хуки внутри `.git/config` — если папки `.git/` нет, хуки не подхватятся и придётся запускать `npx husky` повторно.

```bash
git init
npm install -D husky lint-staged
npx husky init
```

`npx husky init` создаёт папку `.husky/` и добавляет `prepare: husky` в `package.json` автоматически.

### 5.2 — lint-staged в package.json

```json
"lint-staged": {
  "*.{ts,tsx}": [
    "eslint --fix",
    "prettier --write"
  ]
}
```

> ⚠ prettier и eslint-config-prettier должны быть в devDependencies

### 5.3 — Vitest

```bash
npm install -D vitest
```

Добавить в package.json секцию scripts:

```json
"scripts": {
  "test": "vitest",
  "test:run": "vitest --run --passWithNoTests"
}
```

| Команда | Назначение |
|---------|-----------|
| npm run test | watch mode — для локальной разработки |
| npm run test:run | один раз — для хука и CI |

`--passWithNoTests` — не падать если тестовых файлов ещё нет.

### 5.4 — .husky/pre-commit

Создать файл `.husky/pre-commit`. Содержимое — см. приложение.

Что делает скрипт:
1. Запрещает коммит напрямую в main/develop
2. Проверяет имя ветки (feature/, bugfix/, hotfix/, refactor/, release/, chore/, docs/, test/, perf/, ci/)
3. Запускает lint-staged (ESLint + Prettier на staged файлах)
4. Запускает тесты (npm run test:run)
5. Запускает сборку (npm run build), при успехе удаляет dist

### 5.5 — .husky/commit-msg

Создать файл `.husky/commit-msg`. Содержимое — см. приложение.

Проверяет формат сообщения по Conventional Commits:

```
<type>[optional scope]: <description>
```

| Тип | Назначение |
|-----|-----------|
| feat | Новая функциональность |
| fix | Исправление багов |
| docs | Изменения в документации |
| style | Форматирование |
| refactor | Рефакторинг кода |
| perf | Улучшение производительности |
| test | Добавление тестов |
| chore | Обновление сборки, инструменты |
| build | Изменения в системе сборки |
| ci | Изменения в CI/CD |
| revert | Откат предыдущих изменений |

### 5.6 — Порядок выполнения при коммите

```
git commit -m "feat: new feature"
  |
  +-- pre-commit:
  |     1. Проверка имени ветки
  |     2. lint-staged (ESLint + Prettier)
  |     3. npm run test:run (тесты)
  |     4. npm run build (сборка -> удаление dist)
  |
  +-- commit-msg:
        5. Проверка формата сообщения

Любой шаг упал -> коммит отклонён
```

### Ошибка: первый коммит

```
git commit -m "init"
fatal: ambiguous argument 'HEAD': unknown revision or path
not in the working tree.
husky - pre-commit script failed (code 128)
```

**Причина:**

В pre-commit скрипте есть строка:

```bash
branch="$(git rev-parse --abbrev-ref HEAD)"
```

`git rev-parse HEAD` требует хотя бы один коммит в истории. При первом коммите HEAD не существует — Git не знает на что он указывает.

**Решения:**

**Способ 1 — Пропустить проверки для первого коммита (быстрый)**

Флаг `--no-verify` говорит Git пропустить все хуки (pre-commit и commit-msg) только для этого коммита:

```bash
git commit -m "init" --no-verify
```

> ⚠ Используй --no-verify с осторожностью — он полностью отключает все проверки. Подходит только для первого коммита. Не используй его постоянно — это обходит линтер, тесты и проверку сообщения.

**Способ 2 — Добавить проверку в скрипт (правильный)**

Добавить в начало pre-commit скрипта (после определения цветов, до git rev-parse):

```bash
# Пропустить проверку ветки при первом коммите (HEAD ещё не существует)
if ! git rev-parse HEAD > /dev/null 2>&1; then
  echo "Первый коммит — проверка ветки пропущена"
  npx lint-staged
  exit 0
fi
```

Как это работает:

```
git rev-parse HEAD > /dev/null 2>&1
  │
  ├─ HEAD существует → код 0 → ! делает false → блок пропускается
  │   → скрипт продолжает полную проверку
  │
  └─ HEAD не существует → код 1 → ! делает true → блок выполняется
      → запускает только lint-staged и выходит
```

`> /dev/null 2>&1` — подавляет вывод ошибки в терминал. Без этого пользователь увидит "fatal: ambiguous argument" перед сообщением о первом коммите.

**Рекомендация:** Способ 2 предпочтительнее — он решает проблему навсегда. Способ 1 — быстрый обход, но пропускает все проверки.

---

## Шаг 6 — Runtime Environment Injection

### Зачем

Vite вшивает переменные (`VITE_API_URL`) в бандл на этапе сборки. Это значит для каждого окружения (dev, staging, prod) нужна отдельная сборка.

Runtime Environment Injection решает эту проблему — переменные подставляются при запуске контейнера, не при сборке. Один Docker образ для всех окружений.

### Как работает

| Окружение | Как формируется env-config.js |
|-----------|------------------------------|
| Локально | Файл public/env-config.js с захардкоженными значениями |
| Docker | Entrypoint скрипт генерирует файл из ENV переменных |

Приложение не знает откуда взялся файл — оно всегда читает `window._env_`.

**Локальная разработка:**
```
public/env-config.js (захардкожен)
  → браузер загружает файл
  → window._env_ доступен в приложении
```

**Продакшн (Docker):**
```
docker run -e API_URL=https://api.prod.com my-app
  → entrypoint берёт ENV переменные
  → генерирует env-config.js
  → nginx раздаёт файл браузеру
  → window._env_ доступен в приложении
```

### Алгоритм создания (4 файла)

#### Файл 1 — public/env-config.js

```javascript
window._env_ = Object.freeze({
  API_URL: 'http://localhost:3000',
});
```

#### Файл 2 — index.html (подключить перед main.tsx)

```html
<script src="/env-config.js"></script>
<script type="module" src="/src/main.tsx"></script>
```

#### Файл 3 — src/shared/config/env.ts (хелпер)

```typescript
const env = window._env_ ?? {
  API_URL: 'http://localhost:3000',
};

export const API_URL = env.API_URL;
```

Зачем: вместо `window._env_.API_URL` пишешь `import { API_URL } from '@/shared/config/env'`. Автокомплит, одно место для изменений, дефолтные значения.

#### Файл 4 — src/global.d.ts (типы)

```typescript
export type {};

declare global {
  interface EnvConfig {
    API_URL: string;
  }

  interface Window {
    _env_?: EnvConfig;
  }
}
```

Зачем: TypeScript не знает про `window._env_`. Этот файл объявляет тип, иначе редактор покажет ошибку.

### Почему файл типов называется global.d.ts а не env.d.ts

TypeScript обрабатывает `.d.ts` файлы в двух режимах:

**Companion declaration (типы модуля):**
Когда в одной папке лежат `env.ts` и `env.d.ts`, TypeScript считает что `env.d.ts` — это типы для модуля `env.ts`. Глобальные декларации внутри такого файла игнорируются.

```
shared/config/
  env.ts      ← модуль
  env.d.ts    ← TypeScript думает: "это типы для env.ts" ❌
```

**Ambient declaration (глобальные типы):**
Когда `.d.ts` файл не совпадает по имени ни с одним `.ts` файлом — TypeScript считает его глобальным.

```
src/
  global.d.ts   ← нет global.ts → файл глобальный ✅
  shared/config/
    env.ts
```

**Правило:**
```
foo.ts + foo.d.ts  → d.ts привязан к модулю (не глобальный) ❌
foo.ts + bar.d.ts  → d.ts глобальный (разные имена) ✅
```

> ⚠ Никогда не называй .d.ts файл так же как .ts файл в той же папке, если хочешь глобальные декларации.

### Почему нужны export type {} и declare global

В `tsconfig.app.json` есть две опции которые влияют на обработку `.d.ts` файлов:

**`moduleDetection: "force"`** — Все файлы считаются модулями, включая `.d.ts`. Интерфейсы внутри файла остаются локальными — не попадают в глобальную область. Чтобы расширить глобальный Window, нужна обёртка `declare global`.

**`verbatimModuleSyntax: true`** — TypeScript требует чтобы экспорты соответствовали модульной системе. Пустой `export {}` не допускается — нужен `export type {}`.

```
Без moduleDetection: "force":
  global.d.ts → ambient → export type {} и declare global не нужны

С moduleDetection: "force":
  global.d.ts → модуль → нужны оба:
    export type {}    → делает файл валидным модулем
    declare global    → расширяет глобальную область из модуля
```

Итого — все три части обязательны:

```typescript
export type {};     // файл = модуль (для verbatimModuleSyntax)
declare global {    // выход в глобальную область (для moduleDetection: force)
  interface Window {
    _env_?: EnvConfig;  // ? = может не существовать
  }
}
```

### Почему _env_? с вопросительным знаком

`?` означает что `window._env_` может быть `undefined`:

```
Скрипт env-config.js ещё не загрузился → undefined
Ошибка загрузки скрипта → undefined
Тесты без env-config.js → undefined
```

Поэтому в `env.ts` стоит fallback с дефолтными значениями:

```typescript
const env = window._env_ ?? {
  API_URL: 'http://localhost:3000',
};
```

`??` и `?` работают в связке как защита.

### Итоговая структура

```
src/
  global.d.ts            ← глобальные типы (Window, EnvConfig)
  shared/
    config/
      env.ts             ← хелпер с дефолтами
public/
  env-config.js          ← переменные окружения
index.html               ← подключение скрипта
```

---

## Шаг 7 — Dockerfile, nginx, entrypoint и кеширование слоёв

### Как работает nginx.conf

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

#### Разбор построчно

- **`server { }`** — Блок виртуального сервера. Nginx может обслуживать несколько сайтов — каждый в своём server блоке.
- **`listen 80`** — Слушать порт 80 (стандартный HTTP).
- **`root /usr/share/nginx/html`** — Корневая папка откуда nginx берёт файлы. Сюда мы копируем собранный dist/ в Dockerfile.
- **`index index.html`** — Если запросили папку (/) — отдать index.html.
- **`location / { }`** — Правило для всех URL начинающихся с / (то есть для всех запросов).
- **`try_files $uri $uri/ /index.html`** — Ключевая строка. Nginx пробует три варианта по очереди:

| Шаг | Что делает | Результат |
|-----|-----------|-----------|
| $uri | Ищет файл с таким именем | Если нашёл — отдаёт |
| $uri/ | Ищет папку с таким именем | Если нашёл — отдаёт |
| /index.html | Отдаёт index.html | Всегда срабатывает |

#### Примеры

**Запрос: /dashboard**
```
1. $uri     → ищет файл /usr/share/nginx/html/dashboard     → нет
2. $uri/    → ищет папку /usr/share/nginx/html/dashboard/   → нет
3. /index.html → отдаёт index.html → React Router показывает страницу ✅
```

**Запрос: /assets/style.css**
```
1. $uri → ищет /usr/share/nginx/html/assets/style.css → нашёл, отдаёт ✅
```

Это называется SPA fallback — все маршруты ведут к одному index.html, а клиентский роутер решает что показать.

### Как работает docker-entrypoint.sh

#### Файл docker-entrypoint.sh

```sh
#!/bin/sh

# Переменные окружения с дефолтами
API_URL="${API_URL:-http://localhost:3000}"

# Генерация env-config.js
cat > /usr/share/nginx/html/env-config.js << EOF
window._env_ = Object.freeze({
  API_URL: '${API_URL}',
});
EOF

exec nginx -g 'daemon off;'
```

#### Как работают плейсхолдеры

`${API_URL}` — это не специальный синтаксис. Shell автоматически подставляет значение переменной окружения.

Цепочка подстановки:
```
docker run -e API_URL=https://api.prod.com my-app
                 ↑
          попадает в ENV контейнера

ENTRYPOINT выполняется при старте:
  shell видит ${API_URL}
  подставляет значение из ENV
  записывает результат в env-config.js
```

#### Дефолтные значения

Синтаксис `${VAR:-default}` — если переменная не передана, берёт default:

```bash
# Переменная передана:
docker run -e API_URL=https://api.prod.com my-app
  → API_URL = "https://api.prod.com"

# Переменная НЕ передана:
docker run my-app
  → API_URL = "http://localhost:3000" (дефолт из :-)
```

#### Результат в файле env-config.js

```javascript
// При docker run -e API_URL=https://api.prod.com
window._env_ = Object.freeze({
  API_URL: 'https://api.prod.com',
});
```

#### exec nginx

`exec` заменяет текущий процесс (sh) на nginx. Без exec: sh (PID 1) → nginx (PID 2) — лишний процесс. С exec: nginx (PID 1) — корректная обработка сигналов (SIGTERM, SIGQUIT).

`daemon off` — nginx работает на переднем плане. Docker требует это — иначе контейнер завершится сразу после старта.

#### Добавление новой переменной

Одна строка в двух местах:

```sh
# 1. Дефолт в начале файла
NEW_VAR="${NEW_VAR:-default_value}"

# 2. В cat блоке
  NEW_VAR: '${NEW_VAR}',
```

### Кеширование слоёв Docker

Docker выполняет Dockerfile построчно. Каждая команда сохраняется как слой (layer). Если входные данные слоя не изменились — Docker берёт результат из кеша.

**Ключевое правило:** Если слой изменился — все слои после него тоже пересобираются.

```
Слой 1: FROM node:20.19.0-alpine      → кешируется
Слой 2: COPY package.json ./           → кешируется пока package.json не менялся
Слой 3: RUN npm ci                     → кешируется пока слой 2 не менялся
Слой 4: COPY . .                       → сбрасывается при любом изменении кода
Слой 5: RUN npm run build              → сбрасывается вместе со слоем 4
```

#### Правильный порядок (наш)

```dockerfile
COPY package.json package-lock.json ./  # меняется редко
RUN npm ci                              # кешируется если package.json не менялся
COPY . .                                # меняется часто (код)
RUN npm run build                       # пересобирается
```

```
Изменил App.tsx → слои 2,3 из кеша → npm ci НЕ запускается → быстро ✅
```

#### Неправильный порядок

```dockerfile
COPY . .                                # меняется при любом изменении
RUN npm ci                              # кеш сброшен, заново
RUN npm run build                       # тоже заново
```

```
Изменил App.tsx → ВСЕ слои сброшены → npm ci заново → медленно ❌
```

#### Разница на практике

| Порядок | Время сборки при изменении кода |
|---------|-------------------------------|
| Правильный (package.json отдельно) | ~15 сек (npm ci из кеша) |
| Неправильный (всё вместе) | ~60 сек (npm ci заново) |

### Dockerfile с комментариями

#### Стейдж 1 — Сборка приложения

```dockerfile
# Базовый образ с Node.js. alpine — минимальный (~50MB)
# AS builder — имя стейджа для ссылки ниже
FROM node:20.19.0-alpine AS builder

# NODE_OPTIONS — лимит памяти для тяжёлых сборок
# HUSKY=0     — отключает husky (git хуки не нужны в Docker)
# CI=true     — говорит инструментам что мы в CI среде
ENV NODE_OPTIONS="--max_old_space_size=8192" \
    HUSKY=0 \
    CI=true

# Рабочая директория внутри контейнера
WORKDIR /app

# Копируем только package.json и lock — отдельный слой для кеша
COPY package.json package-lock.json ./

# --ignore-scripts — пропускает lifecycle скрипты (prepare)
# --include=dev    — ставит devDependencies (vite, typescript)
RUN npm ci --ignore-scripts --include=dev

# Исходный код — после npm ci чтобы не сбрасывать кеш
COPY . .

# Собираем — результат в папке dist/
RUN npm run build
```

#### Стейдж 2 — Продакшн

```dockerfile
# Чистый nginx без Node.js — образ ~40MB вместо ~300MB
FROM nginx:1.27-alpine

# Наш конфиг с try_files для SPA
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Собранные файлы из стейджа builder
# --from=builder — Node.js не попадает в финальный образ
COPY --from=builder /app/dist /usr/share/nginx/html

# Entrypoint скрипт + права на выполнение
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

# Документация — контейнер слушает порт 80
EXPOSE 80

# Точка входа — генерирует env-config.js и запускает nginx
ENTRYPOINT ["/docker-entrypoint.sh"]
```

#### Multi-stage build — зачем два стейджа

| | Один стейдж | Два стейджа |
|---|---|---|
| Финальный образ содержит | Node.js + nginx + исходники | Только nginx + статика |
| Размер образа | ~300MB | ~40MB |
| Безопасность | Исходный код доступен | Только собранные файлы |

Первый стейдж (builder) — собирает. Второй — берёт только dist/ и выбрасывает всё остальное.

### Использование

#### Сборка образа

```bash
docker build -t my-app .
```

#### Запуск контейнера

```bash
# С дефолтами (API_URL = http://localhost:3000)
docker run -p 3000:80 my-app

# С переменными окружения
docker run -p 3000:80 -e API_URL=https://api.prod.com my-app

# Разные окружения — один образ
docker run -p 3000:80 -e API_URL=https://api.dev.com my-app      # dev
docker run -p 3000:80 -e API_URL=https://api.staging.com my-app  # staging
docker run -p 3000:80 -e API_URL=https://api.prod.com my-app     # prod
```

#### .dockerignore

Исключает ненужные файлы из контекста сборки — ускоряет копирование и уменьшает размер:

```
node_modules
dist
.git
.idea
.husky
```

### Итоговая структура файлов

```
vite-setup-01/
  docker-entrypoint.sh  ← генерирует env-config.js при старте
  nginx.conf            ← конфиг nginx с SPA fallback
  Dockerfile            ← multi-stage сборка
  .dockerignore         ← исключения для Docker
```

---

## Где добавлять/изменять переменные окружения

При добавлении новой переменной нужно внести изменения в **4 файла**:

| # | Файл | Что добавить |
|---|------|-------------|
| 1 | `public/env-config.js` | Значение для локальной разработки |
| 2 | `src/global.d.ts` | Тип в интерфейсе `EnvConfig` |
| 3 | `src/shared/config/env.ts` | Экспорт + дефолтное значение |
| 4 | `docker-entrypoint.sh` | Дефолт + строку в cat блоке |

### Пример: добавляем `APP_ENV`

**1. `public/env-config.js`:**

```javascript
window._env_ = Object.freeze({
  API_URL: 'http://localhost:3000',
  APP_ENV: 'development',              // ← добавить
});
```

**2. `src/global.d.ts`:**

```typescript
export type {};

declare global {
  interface EnvConfig {
    API_URL: string;
    APP_ENV: string;                    // ← добавить
  }

  interface Window {
    _env_?: EnvConfig;
  }
}
```

**3. `src/shared/config/env.ts`:**

```typescript
const env = window._env_ ?? {
  API_URL: 'http://localhost:3000',
  APP_ENV: 'development',              // ← добавить
};

export const API_URL = env.API_URL;
export const APP_ENV = env.APP_ENV;    // ← добавить
```

**4. `docker-entrypoint.sh`:**

```sh
#!/bin/sh

API_URL="${API_URL:-http://localhost:3000}"
APP_ENV="${APP_ENV:-development}"        # ← добавить

cat > /usr/share/nginx/html/env-config.js << EOF
window._env_ = Object.freeze({
  API_URL: '${API_URL}',
  APP_ENV: '${APP_ENV}',               // ← добавить
});
EOF

exec nginx -g 'daemon off;'
```

### Использование в коде

```typescript
import { API_URL, APP_ENV } from '@/shared/config/env';

const response = await fetch(`${API_URL}/users`);
console.log(`Environment: ${APP_ENV}`);
```
