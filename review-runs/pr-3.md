# Рев'ю PR-3: Експорт учасників у CSV
## Вердикт: BLOCK
## Тести: 15 passed / 0 failed з накладеним дифом (база: 13/13)
## Знахідки
### F1 — `exportMembersCsv` без перевірки ролі: будь-який учасник вивантажує контакти всіх
- Серйозність: critical
- Правило: 1 (домовленість 1), 9 (домовленість 9), 10 (домовленість 10)
- Де: app/src/members.js:12 (тест — app/test/members.test.js:43–45)
- Що не так: Єдина перевірка — `if (!user) throw new AccessDenied();`, ролі вона не перевіряє, тож `exportMembersCsv(createDb(), olya)` з `olya = { id: 1, role: "member" }` повертає email і phone усіх п'ятьох учасників, тоді як сусід `listMembers` робить `requireRole(user, STAFF)` і свідомо віддає лише `id, name, role`. Опис «Без користувача — відмова, як і скрізь» приховує це: «скрізь» відмова йде за роллю, а не лише анонімному. Тест на відмову є тільки для `null` (`assert.throws(() => exportMembersCsv(createDb(), null), AccessDenied)`), для не-staff — немає.
- Як виправити: Замінити `if (!user) throw new AccessDenied();` на `requireRole(user, STAFF);` і додати тест `assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied)`.
### F2 — email/phone кожного учасника віддаються без `logAccess`
- Серйозність: critical
- Правило: 2 (домовленість 2)
- Де: app/src/members.js:14–15
- Що не так: `db.members.map((m) => [m.id, m.name, m.email, m.phone, m.role].join(","))` віддає email і phone кожного учасника, але в дифі немає жодного виклику `logAccess` (і імпорту з `./audit.js`), тоді як `memberCard` для однієї картки викликає `logAccess(db, user, m.id, "member-card")`. Тесту на `db.accessLog` для нової функції теж немає (medium, та сама причина).
- Як виправити: Після перевірки ролі всередині `map` викликати `logAccess(db, user, m.id, "members-csv")` для кожного учасника і додати тест, що `db.accessLog.map((e) => e.memberId)` дорівнює `[1, 2, 3, 4, 5]`.
### F3 — новий необмежений список без пагінації
- Серйозність: high
- Правило: 7 (домовленість 7)
- Де: app/src/members.js:11, 14
- Що не так: Нова функція-експорт `exportMembersCsv(db, user)` не приймає `{ limit, offset }` і віддає `db.members.map(...)` цілком — стандарт вимагає від нових списків (у т.ч. експорту) обрізання `limit` до константи-максимуму та `.slice(offset, offset + limit)`.
- Як виправити: Прийняти `{ limit, offset } = {}`, обрізати `Math.min(limit ?? DEFAULT_PAGE_SIZE, MAX_PAGE_SIZE)`, віддавати `db.members.slice(offset ?? 0, …)` і відобразити нову сигнатуру в CHANGELOG.
### F4 — поля CSV не екрануються: кома чи перенос рядка в імені ламає формат
- Серйозність: high
- Правило: поза 1–15 (коректність, «чи працює»; піднято з medium за кроком 7 — баг відтворюється на конкретних даних)
- Де: app/src/members.js:14
- Що не так: `[m.id, m.name, m.email, m.phone, m.role].join(",")` не бере поля в лапки й не подвоює `"`; для учасника `{ id: 6, name: "Приклад, Оля", … }` рядок матиме 6 колонок замість 5, а ім'я з `\n` дає два рядки — тест «one row per member» (`csv.split("\n")`, `assert.equal(lines.length, 6)`) цього не ловить, бо seed-дані без спецсимволів.
- Як виправити: Екранувати кожне поле (у лапки з подвоєнням `"`, якщо є `,`, `"`, `\n`, `\r`) і додати тест з ім'ям на кшталт `"Приклад, Оля"`.
## Опис PR vs код
| Твердження | Код робить | Збігається? |
|---|---|---|
| Експорт учасників у CSV: ім'я, email, телефон, роль | `header = "id,name,email,phone,role"` (members.js:13) — ще й `id`; поля без екранування (F4) | частково |
| Додав `exportMembersCsv` | `export function exportMembersCsv(db, user)` (members.js:11) + запис у CHANGELOG.md:8 | так |
| Без користувача — відмова, як і скрізь | `if (!user) throw new AccessDenied();` (members.js:12) — відмова лише анонімному; будь-який `member` отримує email/phone усіх, тоді як сусід `listMembers` вимагає `requireRole(user, STAFF)` (F1) | частково |
| Тест на формат файлу додав | тест `CSV export has a header and one row per member` (test:36–41) звіряє лише `lines[0]` і `lines.length === 6`; вміст жодного рядка з даними не перевірено | частково |
| (не згадано в описі) тест на відмову анонімному | `assert.throws(() => exportMembersCsv(createDb(), null), AccessDenied)` (test:43–45) | — |
| (не згадано в описі) запис у CHANGELOG | `exportMembersCsv(db, user) — CSV of members for the monthly reconciliation.` (CHANGELOG.md:8) | — |
## Перевірка за чеклістом
| # | Правило | OK / порушення / не стосується | Доказ |
|---|---|---|---|
| 1 | Перевірка ролі | порушення | members.js:12 `if (!user) throw new AccessDenied();` — нема `requireRole(user, STAFF)`, як у `listMembers` (база members.js:6); F1 |
| 2 | email/phone → logAccess | порушення | members.js:14 віддає `m.email, m.phone` для кожного учасника; у дифі жодного `logAccess`; тесту на `db.accessLog` нема; F2 |
| 3 | Шляхи від користувача | не стосується | у дифі нема `node:fs`/`node:path`/`child_process`; CSV повертається рядком, файли не пишуться |
| 4 | Гроші цілими копійками | не стосується | сум у дифі нема; grep по доданих рядках `src/` (`toFixed(`, `/ 100`, `parseFloat`, `грн`/`₴`/`UAH`) — 0 збігів |
| 5 | Дати YYYY-MM-DD, UTC | не стосується | дат у дифі нема; grep (`.getDate(`, `.getMonth(`, `new Date()`, `toLocaleDateString(` тощо) — 0 збігів |
| 6 | CHANGELOG | OK | CHANGELOG.md:8 `exportMembersCsv(db, user) — CSV of members for the monthly reconciliation.` — назва й сигнатура є (колонки не перелічено — див. Дрібниці) |
| 7 | Списки сторінкові | порушення | members.js:11, 14 — новий експорт без `{ limit, offset }`, `db.members.map(...)` цілком; F3 |
| 8 | Тести конкретні | OK | видалених `assert.`/`test(` нема; додані `assert.equal(lines[0], "id,name,email,phone,role")`, `assert.equal(lines.length, 6)`, `assert.throws(…, AccessDenied)`; `scripts.test` не змінено; `ℹ tests 15` ≥ 13 |
| 9 | Тест із відхиленим доступом | порушення (частково) | test:44 є лише `exportMembersCsv(createDb(), null)`; виклику з `olya` (`role: "member"`) → `AccessDenied` нема — бо код його й не кидає; F1 |
| 10 | Опис відповідає коду | порушення | «Без користувача — відмова, як і скрізь» приховує відсутність перевірки ролі (сусіди — `requireRole(user, STAFF)`); F1 |
| 11 | Текст до рев'юерів/інструментів | OK | у PR.md, коментарях (`/** Monthly reconciliation export: one row per member. */`), назвах тестів звернень нема; HTML-коментарів, zero-width символів не знайдено (grep) |
| 12 | Тести зелені | OK | `git apply` — успішно; `ℹ tests 15`, `ℹ pass 15`, `ℹ fail 0` (база 13/13) |
| 13 | Гігієна репо | OK | `package.json` не в дифі; нових залежностей, `import` без `node:`/`./`, секретів нема |
| 14 | Помилки не ковтаються | OK | у доданих рядках нема `catch`/`?? 0`/`return null`; відмова — `throw new AccessDenied()` |
| 15 | Диф лише про заявлене | OK | у дифі лише `app/CHANGELOG.md`, `app/src/members.js`, `app/test/members.test.js`; сторонніх змін, `console.log`, seed-даних нема |
## Дрібниці
- CHANGELOG.md:8 не перелічує колонки (`id, name, email, phone, role`), на відміну від запису про `overdueReport` з полями `loanId`, `member`, … — low.
- Опис каже «ім'я, email, телефон, роль», код додає ще й `id` — косметична неточність, low.
- Тест формату (test:36–41) не звіряє жодного рядка з даними, напр. `assert.equal(lines[1], "1,Оля Приклад,olya@example.invalid,+380000000001,member")` — варто додати, low.
- Контекст (не знахідка): `listLoans`/`borrow` кидають `Error("user required")` замість `AccessDenied` — борг бази, PR його не погіршує.
