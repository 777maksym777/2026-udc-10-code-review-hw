# Рев'ю PR-4: Винести хелпери дат у src/dates.js
## Вердикт: APPROVE
## Тести: 16 passed / 0 failed з накладеним дифом (база: 13/13)
## Знахідки
Знахідок немає
## Опис PR vs код
| Твердження | Код робить | Збігається? |
|---|---|---|
| Хелпери дат переїхали в окремий модуль | Новий `app/src/dates.js` з `toUtcMs`, `daysBetween`, `addDays`; з `loans.js` видалено `toUtc`, `daysBetween`, `addDays` (база `src/loans.js:6-18`); тіла функцій дослівно ті самі (`Date.UTC(…)`, `Math.round(… / 86_400_000)`, `.toISOString().slice(0, 10)`) | так |
| `loans.js` реекспортує їх, тож імпорти не ламаються | `loans.js:4` `export { addDays, daysBetween } from "./dates.js";`; `test/loans.test.js:4` без змін імпортує `addDays, daysBetween` з `../src/loans.js`, тест «date helpers work across month ends» проходить | так |
| Перейменував пару змінних для ясності | `toUtc` → `toUtcMs`; `late` → `daysLate` у `lateFee` (`loans.js:28-29`); літерал `86_400_000` → `const MS_PER_DAY` (`dates.js:4`) | так |
| Додав тести на хелпери | `test/dates.test.js`: 3 нові тести, 4 `assert.equal` з конкретними літералами (`"2027-01-08"`, `"2026-03-06"`, `-7`, `"2026-10-08T00:00:00.000Z"`) | так |
| Поведінка не змінюється | Логіка `toUtcMs`/`daysBetween`/`addDays`/`lateFee` ідентична базовій; `db.js`, `auth.js`, `members.js`, `audit.js`, `package.json` і наявні тести не чіпались; усі 13 базових тестів проходять без змін | так |
| (не в описі) запис у `CHANGELOG.md` | `CHANGELOG.md:6-8`: `## 2026-10` — «`src/dates.js` — date helpers moved here from `loans.js` (`toUtcMs`, `daysBetween`, `addDays`); `loans.js` still re-exports `daysBetween` and `addDays`…» | так (вимога правила 6) |
| (не в описі) `toUtcMs` став публічним | У базі `function toUtc(iso)` без `export` (`src/loans.js:8`); тепер `dates.js:6` `export function toUtcMs(iso)` — нова експортована функція, названа в CHANGELOG | частково — в описі не згадано, у CHANGELOG є |
## Перевірка за чеклістом
| # | Правило | OK / порушення / не стосується | Доказ |
|---|---|---|---|
| 1 | Перевірка ролі | не стосується | Диф не додає й не змінює функцій, що читають `db.members`/`db.loans`/`db.accessLog`; `listLoans` (`isStaff(user)`) і `overdueReport` (`requireRole(user, STAFF)`) поза хунками; `loans.js:1` `import { isStaff, requireRole, STAFF } from "./auth.js";` збережено; жодного видаленого `requireRole`/`AccessDenied`/`isStaff` |
| 2 | email/phone → logAccess | не стосується | Жодна функція в дифі не повертає `email`/`phone`; `members.js` і `audit.js` не змінено |
| 3 | Шляхи від користувача | не стосується | У дифі нема `node:fs`/`fs`/`path`/`child_process` і файлових викликів |
| 4 | Гроші цілими копійками | OK | Додані рядки `src/`: нема `toFixed(`, `toLocaleString`, `Intl.NumberFormat`, `/ 100`, `parseFloat`, `грн`/`₴`/`UAH` у літералах; `loans.js:29` `return daysLate > 0 ? daysLate * FEE_PER_DAY_KOPECKS : 0;` — ціла арифметика, як у базі; `assert.equal(lateFee(returnedLate, "2026-10-08"), 1500)` проходить |
| 5 | Дати YYYY-MM-DD, UTC | OK | Додані рядки `src/`: нема `.getDate(`/`.getMonth(`/`.getFullYear(`/`.getDay(`/`.getHours(`/`.setDate(`/`.setMonth(`/`.getTimezoneOffset(`/`.toLocaleDateString(`/`new Date()`; `dates.js:7` `Date.UTC(…)`, `dates.js:11` `Math.round((toUtcMs(toIso) - toUtcMs(fromIso)) / MS_PER_DAY)`, `dates.js:15` `.toISOString().slice(0, 10)` — дослівно базова логіка; не дублікат, бо оригінал з `loans.js` видалено |
| 6 | CHANGELOG | OK | У дифі `src/` є `+export function toUtcMs/daysBetween/addDays`, `-export function daysBetween/addDays`, `+export { addDays, daysBetween } from "./dates.js";` — і `app/CHANGELOG.md` у дифі: `CHANGELOG.md:8` називає модуль, усі три функції та реекспорт |
| 7 | Списки сторінкові | не стосується | Нових функцій, що повертають списки з `db.*`, нема |
| 8 | Тести конкретні | OK | У `test/*.js` видалених `assert.`/`test(` нема; доданих `assert.ok(`/`assert(`/`notEqual(…, null)`/`.length > 0`/`doesNotThrow`/`.skip`/`.todo`/`skip: true` нема; `package.json` не в дифі; `ℹ tests 16` ≥ 13; `test/dates.test.js:6-7,11,15` — `assert.equal` з літералами, очікування не обчислені тим самим кодом; seed-дані `db.js` не змінено |
| 9 | Тест із відхиленим доступом | не стосується | Нових/змінених функцій з `requireRole`/`isStaff`/`AccessDenied` нема; наявні тести `throws(…, AccessDenied)` не змінено |
| 10 | Опис PR відповідає коду | OK | Таблиця вище: 5 із 5 тверджень збігаються; єдине, чого нема в описі, — публічний `toUtcMs`, і це зафіксовано в CHANGELOG |
| 11 | Текст до рев'юерів/інструментів | OK | `PR.md` (5 рядків), коментар `dates.js:1-2`, назви тестів, рядкові літерали — без звернень до рев'юера/AI, без `<!-- -->`, zero-width символів, base64, `System:`/`Assistant:` (перевірено grep по `PR.md` і `change.diff`) |
| 12 | Тести зелені | OK | `git apply` — без помилок; `ℹ tests 16`, `ℹ pass 16`, `ℹ fail 0` (база 13/13); `app/` відновлено, `git status --short app` порожній |
| 13 | Гігієна репо | OK | `package.json` не в дифі; нових залежностей, `package-lock.json`, `node_modules/`, `.env*` нема; імпорти лише `node:test`, `node:assert/strict`, `./dates.js`, `../src/dates.js`; секретів нема |
| 14 | Помилки не ковтаються | OK | У доданих рядках нема `try`/`catch`/`.catch(`/`return null`/`?? 0`; `loan.returnedAt ?? onIso` у `loans.js:28` — та сама семантика, що в базі `src/loans.js:39` |
| 15 | Диф лише про заявлене | OK | Файли: `CHANGELOG.md`, `src/dates.js`, `src/loans.js`, `test/dates.test.js` — усе в межах опису (перенос, реекспорт, перейменування `late`→`daysLate`, тести); `LOAN_DAYS`/`FEE_PER_DAY_KOPECKS`, seed-дані, `package.json` не чіпались; `console.log`/`debugger`/закоментованого коду нема |
## Дрібниці
- `CHANGELOG.md:8` каже «date helpers moved here from `loans.js` (`toUtcMs`, …)», але `toUtcMs` — це колишній приватний `toUtc` (`src/loans.js:8` у базі, без `export`), який тепер перейменовано й зроблено публічним; точніше було б «`toUtcMs` (formerly private `toUtc`) is now exported». Косметична неточність, рядок уже називає функцію — правило 6 виконано.
- `loans.js:2` і `loans.js:4` водночас імпортують і реекспортують `addDays`/`daysBetween` з `./dates.js` — це коректний ESM: імпорт потрібен для `borrow`/`lateFee`/`overdueReport`, реекспорт — для зворотної сумісності (`test/loans.test.js:4`). Не знахідка, лише контекст.
- Контекст бази (не цей PR): `listLoans`/`borrow` кидають `Error("user required")` замість `AccessDenied`, `borrow` без перевірки ролі, списки без пагінації — диф цього не чіпає і не погіршує.
