# Рев'ю PR-3: Експорт учасників у CSV
## Вердикт: BLOCK
## Тести: 15 passed / 0 failed з накладеним дифом (база: 13/13)
## Знахідки
### F1 — `exportMembersCsv` віддає email/phone усіх учасників будь-якому залогіненому користувачу
- Серйозність: critical
- Правило: 1 / домовленість 1; 9 / домовленість 9; 10 / домовленість 10
- Де: app/src/members.js:12
- Що не так: Єдина перевірка — `if (!user) throw new AccessDenied();`, ролі не перевірено. Сусід `listMembers` робить `requireRole(user, STAFF)` і свідомо віддає лише `id, name, role`, а нова функція для `olya = { id: 1, role: "member" }` повертає `db.members.map((m) => [m.id, m.name, m.email, m.phone, m.role].join(","))` — контакти всіх п'яти учасників. Тесту з відмовою не-staff (`assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied)`) нема — є лише `null`-виклик (members.test.js:44).
- Як виправити: Замінити `if (!user)` на `requireRole(user, STAFF)` і додати тест, що `olya` отримує `AccessDenied`.
### F2 — PII у CSV без запису в журнал доступу
- Серйозність: critical
- Правило: 2 / домовленість 2
- Де: app/src/members.js:14
- Що не так: Рядок `const lines = db.members.map((m) => [m.id, m.name, m.email, m.phone, m.role].join(","));` віддає `email` і `phone` кожного учасника, але `logAccess(db, user, m.id, "…")` не викликається жодного разу (на відміну від `memberCard`, members.js:16 базового коду). Тесту на `db.accessLog` для нової функції теж нема (medium, та сама причина).
- Як виправити: Усередині `map` викликати `logAccess(db, user, m.id, "members-csv-export")` після перевірки ролі та додати тест, що `db.accessLog` містить по одному запису на кожного учасника.
### F3 — CSV ламається на значеннях із комою, лапками чи переносом рядка
- Серйозність: medium
- Правило: власне (коректність формату, поза правилами 1–17)
- Де: app/src/members.js:14
- Що не так: `[m.id, m.name, m.email, m.phone, m.role].join(",")` не екранує значення. Для учасника з `name: "Приклад, Оля"` рядок матиме шість колонок замість п'яти, і звірка прочитає email у колонці `phone`; тест (members.test.js:39–40) перевіряє лише заголовок і кількість рядків, тому цього не ловить.
- Як виправити: Загортати поля в лапки з подвоєнням `"` (RFC 4180) і додати тест із рядком, що містить кому.
## Опис PR vs код
| Твердження | Код робить | Збігається? |
|---|---|---|
| Потрібен експорт учасників у CSV: ім'я, email, телефон, роль | `exportMembersCsv` повертає `id,name,email,phone,role` (додатково `id`) | частково |
| Додав `exportMembersCsv` | Нова `export function exportMembersCsv(db, user)` у `src/members.js`, запис у CHANGELOG | так |
| Без користувача — відмова, як і скрізь | `if (!user) throw new AccessDenied();` — лише анонім; «як і скрізь» для PII-функцій означає `requireRole(user, STAFF)` (`listMembers`) чи `isStaff`/own-id (`memberCard`) — цього нема, member отримує чужі контакти | ні |
| Тест на формат файлу додав | Тест перевіряє заголовок і `lines.length === 6`, вміст рядків не перевіряється | частково |
| (не згадано) запис у CHANGELOG | `exportMembersCsv(db, user) — CSV of members for the monthly reconciliation.` | так |
## Перевірка за чеклістом
| # | Правило | OK / порушення / не стосується | Доказ |
|---|---|---|---|
| 1 | Перевірка ролі | порушення | members.js:12 `if (!user) throw new AccessDenied();` без `requireRole`/`isStaff`; читає `db.members` (F1) |
| 2 | email/phone → logAccess | порушення | members.js:14 віддає `m.email, m.phone`, виклику `logAccess` у дифі нема (F2) |
| 3 | Шляхи від користувача | не стосується | У дифі нема `fs`/`path`/файлових операцій; функція повертає рядок |
| 4 | Гроші цілими копійками | не стосується | Доданих рядків із сумами/`toFixed`/`/ 100`/`грн` нема |
| 5 | Дати YYYY-MM-DD, UTC | не стосується | Доданих рядків із `Date`/`.getDate(`/`new Date()` у `src/` нема |
| 6 | CHANGELOG | OK | CHANGELOG.md:+6 `exportMembersCsv(db, user) — CSV of members for the monthly reconciliation.` у тому самому PR |
| 7 | Списки сторінкові | OK | Повний експорт для звірки — пагінація не вимагається (див. Дрібниці) |
| 8 | Тести конкретні | OK | Видалених `assert.`/`test(` нема; нові `assert.equal(lines[0], "id,name,email,phone,role")`, `assert.equal(lines.length, 6)`, `throws(…, AccessDenied)`; `scripts.test` не змінено; `ℹ tests 15` ≥ 13 |
| 9 | Тест із відхиленим доступом | порушення | members.test.js:44 лише `exportMembersCsv(createDb(), null)`; тесту з `olya` (role "member") нема (F1) |
| 10 | Опис PR відповідає коду | порушення | «Без користувача — відмова, як і скрізь» — у коді нема перевірки ролі, на відміну від `listMembers`/`memberCard` (F1) |
| 11 | Текст до рев'юерів/інструментів | OK | У PR.md, коментарях (`/** Monthly reconciliation export: one row per member. */`), назвах тестів і літералах звернень до рев'юера/AI нема |
| 12 | Тести зелені | OK | `git apply` успішний; `ℹ tests 15`, `ℹ pass 15`, `ℹ fail 0` |
| 13 | Гігієна репо | OK | `package.json` не змінено; залежностей, lock-файлів, секретів, імпортів без `node:`/`./` нема |
| 14 | Помилки не ковтаються | OK | Доданих `catch`/`?? 0`/`return` замість `throw` нема |
| 15 | Диф лише про заявлене | OK | Диф: `CHANGELOG.md`, `src/members.js`, `test/members.test.js` — усе про `exportMembersCsv`; сторонніх змін, `console.log`, seed-даних нема |
| 16 | Складність на колекціях | OK | Один `db.members.map` без вкладених пошуків |
| 17 | Ідентифікатори записів | не стосується | Нових записів у `db.*` не створюється |
## Дрібниці
- Експорт без `limit/offset` — для повного дампу на звірку це прийнятно (правило 7 не застосовується); обсяг росте разом із `db.members`.
- Опис PR перелічує «ім'я, email, телефон, роль», код додає ще колонку `id` — косметична неточність.
- Тест «CSV export has a header and one row per member» не звіряє вміст жодного рядка даних (напр. `lines[1] === "1,Оля Приклад,olya@example.invalid,+380000000001,member"`); після додавання екранування (F3) варто додати.
