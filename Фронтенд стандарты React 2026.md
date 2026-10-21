# Фронтенд «Очага»: стандарты React 2026

21.10.2026

## Стек

Фронт «Очага» строится на Next.js 16 (App Router) с TypeScript в строгом режиме, код организован по фичам по образцу [Bulletproof React](https://github.com/alan2207/bulletproof-react). Мажорные версии указаны по памяти и сверяются при создании проекта.

| Задача | Библиотека | Версия | Аналог на бэке |
| --- | --- | --- | --- |
| Язык | TypeScript, `strict: true` | 5.x | Java |
| UI | React | 19 | — |
| Фреймворк, маршруты, SSR | Next.js, App Router | 16 | Spring Boot + `@RestController` |
| Клиент API | openapi-typescript + openapi-fetch | — | `openapi-generator` в модуле `api` |
| Данные с сервера | TanStack Query | 5 | Spring Cache + вызов сервиса |
| Формы | React Hook Form | 7 | — |
| Схемы и валидация | Zod | 4 | Bean Validation |
| Стили | Tailwind CSS | 4 | — |
| Компоненты | shadcn/ui на Radix UI | — | — |
| Перетаскивание | dnd-kit (`core` + `sortable`) | — | `PositionsUtils` |
| Unit-тесты | Vitest + Testing Library | — | JUnit |
| E2E-тесты | Playwright | — | интеграционные тесты |
| Линтер, формат | ESLint (flat config) + Prettier | 9 / 3 | — |
| Пакеты | npm, Node.js LTS | — | Maven, JVM |

## Структура папок

В React стандарт — организация **по фичам**, а не по слоям, как на бэке. Папка `app/` только задаёт маршруты и собирает страницы из фич; вся логика лежит в `features/`. Это совмещает [конвенции Next.js](https://nextjs.org/docs/app/getting-started/project-structure) и подход [Bulletproof React](https://github.com/alan2207/bulletproof-react).

```
web/
├─ src/
│  ├─ app/                        # маршруты: только сборка страниц
│  │  ├─ (public)/               # страницы для гостей
│  │  │  ├─ page.tsx             # главная мемориала
│  │  │  ├─ sections/[sectionId]/page.tsx
│  │  │  └─ items/[itemId]/page.tsx
│  │  ├─ (admin)/manage/          # редактор владельца
│  │  ├─ layout.tsx               # общий каркас и провайдеры
│  │  ├─ error.tsx, not-found.tsx
│  │  └─ globals.css              # Tailwind
│  ├─ features/                   # по папке на область
│  │  ├─ sections/
│  │  │  ├─ api/                  # запросы и хуки TanStack Query
│  │  │  ├─ components/           # компоненты области
│  │  │  ├─ schemas/              # схемы Zod для форм
│  │  │  └─ index.ts              # публичный вход фичи
│  │  ├─ items/
│  │  ├─ contents/
│  │  └─ media/
│  ├─ components/
│  │  ├─ ui/                      # shadcn/ui, сгенерированные компоненты
│  │  └─ layout/                  # шапка, подвал, навигация
│  ├─ lib/
│  │  ├─ api/
│  │  │  ├─ schema.d.ts           # сгенерировано из openapi.yaml
│  │  │  └─ client.ts             # настроенный openapi-fetch
│  │  └─ utils.ts                 # cn() и прочие утилиты
│  └─ hooks/                      # общие хуки, не привязанные к фиче
├─ e2e/                          # тесты Playwright
├─ public/                       # статика: иконки, шрифты
├─ package.json, package-lock.json
└─ tsconfig.json, eslint.config.mjs, next.config.ts
```

Правила структуры:

- **Импорты идут в одну сторону:** `app` → `features` → `components`, `lib`, `hooks`. Общий код не импортирует фичи.
- **Фичи не импортируют друг друга.** Если странице нужны раздел и его записи, их соединяет `app/.../page.tsx`. Аналог: `Process → Service` без циклов.
- **Внутренности фичи закрыты:** снаружи импортируется только то, что вынесено в её `index.ts`.
- **Файлы — в `kebab-case`:** `section-list.tsx`, компонент внутри — `SectionList`. Так именует файлы shadcn/ui.
- **Абсолютные импорты через `@/`:** `import { Button } from "@/components/ui/button"`, а не `../../../`.
- **Папка появляется, когда в ней есть файлы.** Пустых `schemas/` «на будущее» не заводить.

## Принципы

SOLID в React работает, но через компоненты и хуки, а не через классы и интерфейсы. Классов в современном React нет: только функции.

| Принцип | Как выглядит в React |
| --- | --- |
| Одна ответственность (S) | Компонент делает одно. Загрузка данных — в хуке (`useSection`), отображение — в компоненте (`SectionView`). Компонент длиннее ~150 строк — повод разделить |
| Открытость/закрытость (O) | Композиция через `children` и составные компоненты (`<Card><CardHeader/>…</Card>`) вместо десятка флагов `showHeader`, `isCompact` |
| Замена подтипов (L) | Компонент-обёртка принимает те же props, что и оборачиваемый элемент (`Button` принимает все атрибуты `<button>`) |
| Разделение интерфейса (I) | В props — только то, что нужно: `name` и `photoUrl`, а не весь `ItemDto` |
| Инверсия зависимостей (D) | Компоненты не вызывают `fetch` напрямую: только хуки из `features/*/api`, которые ходят через `lib/api/client.ts` |

Принципы, специфичные для React:

1. **Серверные компоненты по умолчанию.** В Next.js компонент выполняется на сервере и приходит в браузер готовым HTML. `"use client"` ставится только на те компоненты, где есть клики, состояние или хуки, и как можно ниже по дереву. Публичные страницы мемориала почти целиком серверные, редактор — клиентский.
2. **Один источник правды для данных сервера.** Данные API живут в кэше TanStack Query и не копируются в `useState`. Локальное состояние — только для UI: открыт ли диалог, выбрана ли вкладка.
3. **Состояние держится как можно ближе к месту использования.** Поднимается выше, только когда нужно двум компонентам. Глобальный стор (Redux, Zustand) здесь не нужен.
4. **Вычисляемое не хранится.** Если значение считается из props или данных, его считают при рендере, без `useState` + `useEffect`.
5. **`useEffect` — последнее средство.** Он нужен для синхронизации с внешним миром, не для загрузки данных и не для реакции на клик. См. [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect).
6. **Компоненты — чистые функции.** Одни и те же props дают один и тот же результат, побочные эффекты — в обработчиках событий.
7. **`useMemo`/`useCallback` вручную не расставляются.** В React 19 это делает React Compiler, его включает настройка Next.js.
8. **Типы из контракта, а не руками.** DTO берутся из сгенерированного `schema.d.ts`; свои дублирующие интерфейсы не пишутся. `any` запрещён.

## Правила по библиотекам

Путь запроса фиксирован: компонент → хук фичи → клиент API → бэк. Это аналог твоего «контроллер → ProcessService → Service → репозиторий».

**Next.js**

- `page.tsx` и `layout.tsx` только собирают страницу из компонентов фич, без логики. Как твои контроллеры.
- Публичные страницы загружают данные на сервере и задают `generateMetadata`: заголовок, описание и картинку для превью ссылок.
- Серверные запросы к бэку передают исходный `Host`, иначе бэк не определит мемориал.
- Состояния загрузки и ошибки — файлами `loading.tsx`, `error.tsx`, `not-found.tsx`, а не ручными проверками в каждом компоненте. `404` бэка → `notFound()`.

**openapi-fetch**

- Типы генерируются командой `npm run api:generate` из `api/src/main/resources/openapi/openapi.yaml`. Сгенерированный файл руками не правится.
- Один клиент в `lib/api/client.ts`. Там же единое превращение Problem Details в ошибку.

**TanStack Query**

- На каждую операцию — хук в `features/*/api`: `useSections()`, `useCreateSection()`. Компоненты вызывают только их.
- Ключи кэша — из фабрики ключей фичи (`sectionKeys.detail(id)`), строки вручную не пишутся.
- После изменения — `invalidateQueries` по ключу фичи. Перестановки — оптимистично с откатом при ошибке.
- Версия сущности для `If-Match` берётся из загруженных данных; на `412` показывается «данные изменились, обновите».

**React Hook Form + Zod**

- Схема Zod — в `features/*/schemas`, тип формы — `z.infer` от неё.
- Формы собираются из компонента `Form` shadcn/ui.
- Ошибки валидации с бэка (`400`) выводятся у полей через `setError`.

**shadcn/ui и Tailwind**

- Компоненты добавляются только командой `npx shadcn@latest add`, в `components/ui`.
- Цвета и шрифты — только через токены темы в `globals.css` (`bg-background`, `text-muted-foreground`), без произвольных `#hex`.
- Условные классы — через `cn()`, варианты компонента — через `cva`.

**dnd-kit**

- Один общий компонент `SortableList` для разделов, записей и контента; фича передаёт ему элементы и мутацию `…/order`.
- Датчики мыши, касаний и клавиатуры включены всегда.

## Контроль качества

Правила выше проверяет не человек, а сборка: одна команда `npm run verify` — аналог `./mvnw verify`.

| Инструмент | Что ловит |
| --- | --- |
| `tsc --noEmit`, `strict: true`, `noUncheckedIndexedAccess` | Ошибки типов, `null` без проверки, расхождение с контрактом |
| ESLint + `eslint-config-next` | Правила хуков, лишние эффекты, доступность (`jsx-a11y`) |
| `eslint-plugin-boundaries` | Импорты не в ту сторону: фича → фича, общий код → фича |
| Prettier + `prettier-plugin-tailwindcss` | Форматирование и порядок классов Tailwind |
| Vitest + Testing Library | Логика хуков и компонентов; тестируется поведение глазами пользователя, а не внутреннее состояние |
| Playwright | Сценарии спеки в реальном браузере против запущенного бэка |

Именование:

- Компоненты и типы — `PascalCase` (`SectionCard`), функции и переменные — `camelCase`, файлы — `kebab-case`.
- Хуки начинаются с `use` (требование React), обработчики — `handleX`, props-колбэки — `onX`.
- Тест лежит рядом с файлом: `section-card.tsx` → `section-card.test.tsx`.
- Экспорты — именованные; `export default` — только там, где его требует Next.js (`page.tsx`, `layout.tsx`).

## Скиллы для ревью

Стандарт — официальные скиллы Vercel (автор Next.js) из репозитория `vercel-labs/agent-skills`. Они ставятся в проект и работают в Claude Code при написании и ревью кода.

| Скилл | Что проверяет |
| --- | --- |
| [vercel-react-best-practices](https://www.infoq.com/news/2026/02/vercel-react-best-practices/) | Производительность React и Next.js: десятки правил в 8 категориях от CRITICAL до LOW, сверху — цепочки последовательных запросов и размер бандла |
| web-design-guidelines | Интерфейс: доступность, формы, состояния, типографика |

```bash
npx skills add vercel-labs/agent-skills
```

Кроме скиллов в Claude Code уже есть `/code-review` (ошибки в диффе) и `/simplify` (упрощение). Сами правила из этого документа нужно положить в `web/CLAUDE.md`: тогда любое ревью сверяет код с ними, как сейчас с правилами бэка.

## Источники

- [Next.js: Project Structure](https://nextjs.org/docs/app/getting-started/project-structure)
- [Bulletproof React](https://github.com/alan2207/bulletproof-react)
- [React: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [InfoQ: Vercel React Best Practices](https://www.infoq.com/news/2026/02/vercel-react-best-practices/)
- [Robin Wieruch: React Folder Structure](https://www.robinwieruch.de/react-folder-structure/)
