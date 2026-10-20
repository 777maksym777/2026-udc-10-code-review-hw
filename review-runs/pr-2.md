# Рев'ю PR-2: Email учасника у звіті про прострочення
## Вердикт: BLOCK
## Тести: 13 passed / 0 failed з накладеним дифом (база: 13/13)
## Знахідки
### F1 — `overdueReport` віддає email учасників без запису в журнал доступу
- Серйозність: critical
- Правило: 2 / домовленість 2 (відсутній тест на `db.accessLog` — medium, та сама причина; також 10 — опис подає email як «доречний» через staff-only, не згадуючи журнал)
- Де: app/src/loans.js:54 (тест — app/test/loans.test.js:48)
- Що не так: У кожен рядок звіту додано `email: db.findMember(l.memberId)?.email ?? null,`, але жодного виклику `logAccess(db, user, memberId, purpose)` з `src/audit.js` у функції немає — ні на рядок, ні на весь звіт; `audit.js` у `loans.js` навіть не імпортовано. Сусід `memberCard` перед поверненням `email`/`phone` робить `logAccess(db, user, m.id, "member-card")`. Новий тест перевіряє лише `assert.equal(rows[0].email, "marko@example.invalid");` і не звіряє `db.accessLog`.
- Як виправити: після `requireRole` всередині `.map` для кожного рядка викликати `logAccess(db, user, l.memberId, "overdue-report")` і додати тест, що `db.accessLog` містить по запису на кожного учасника зі звіту (або прибрати `email` зі звіту).
### F2 — `feeText`: форматування грошей у `src/` через `/ 100`, `toFixed(2)` і літерал «грн»
- Серйозність: high
- Правило: 4 / домовленість 4; також 10 і 15 (поле не згадане в описі PR)
- Де: app/src/loans.js:58
- Що не так: Додано поле `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн",` — одразу три заборонені патерни в `src/` (`/ 100`, `toFixed(`, `грн` у рядковому літералі); суми в сервісі мають бути лише цілими копійками (`feeKopecks`), а подання — справа клієнта. Поле `feeText` не згадане в описі PR, тесту на нього нема.
- Як виправити: прибрати `feeText` зі звіту (залишити `feeKopecks`) і форматувати суму на рівні презентації.
### F3 — Змінено форму відповіді `overdueReport` без запису в CHANGELOG
- Серйозність: high
- Правило: 6 / домовленість 6
- Де: app/src/loans.js:54 і app/src/loans.js:58; app/CHANGELOG.md — у дифі відсутній
- Що не так: У `.map((l) => ({ … }))` з'явились нові поля `email` і `feeText`, а `app/CHANGELOG.md` диф не торкається. Наявний запис за 2026-09 перелічує саме поля звіту (`loanId`, `member`, `book`, `daysLate`, `feeKopecks`), тож зміна форми відповіді має бути задокументована в тому самому PR.
- Як виправити: додати в `app/CHANGELOG.md` запис про нові поля `overdueReport` у цьому ж PR (про ті, що лишаться після виправлення F1/F2).
## Опис PR vs код
| Твердження | Код робить | Збігається? |
|---|---|---|
| Бібліотекарі просили email у звіті про прострочення, щоб одразу писати нагадування | Мотивація; код додає `email` у кожен рядок `overdueReport` | так |
| «Тепер для кожного рядка беремо учасника і віддаємо ім'я та email» | `member: db.findMember(l.memberId)?.name ?? "—"`, `email: db.findMember(l.memberId)?.email ?? null` — учасник шукається двічі на рядок | так |
| «Звіт і так лише для персоналу, тож email тут доречний» | `requireRole(user, STAFF);` збережено — звіт справді staff-only; але email віддається без `logAccess`, а домовленість 2 вимагає запис у журнал незалежно від ролі | частково |
| (не в описі) нове поле `feeText` — відформатована сума «грн» | `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн"` | ні — в описі відсутнє |
| (не в описі) `Map` імен замінено на `db.findMember(...)` двічі на рядок | видалено `const names = new Map(db.members.map((m) => [m.id, m.name]));` | ні — в описі відсутнє |
| (не в описі) запис у CHANGELOG про нові поля | `app/CHANGELOG.md` у дифі відсутній | ні |
## Перевірка за чеклістом
| # | Правило | OK / порушення / не стосується | Доказ |
|---|---|---|---|
| 1 | Перевірка ролі | OK | `requireRole(user, STAFF);` (app/src/loans.js:48) збережено до читання `db.loans`/`db.members`; список ролей не розширено |
| 2 | email/phone → logAccess | порушення (F1) | app/src/loans.js:54 `email: db.findMember(l.memberId)?.email ?? null,` — `logAccess` у функції відсутній, `audit.js` не імпортовано |
| 3 | Шляхи від користувача | не стосується | у дифі нема `fs`/`path`/`child_process` і файлових операцій |
| 4 | Гроші цілими копійками | порушення (F2) | app/src/loans.js:58 `(lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн"` |
| 5 | Дати YYYY-MM-DD, UTC | OK | нових операцій з датами нема; `daysBetween`/`lateFee` як у базі; `getDate(`/`new Date()` тощо в доданих рядках відсутні |
| 6 | CHANGELOG | порушення (F3) | нові поля `email`, `feeText` у `.map((l) => ({ … }))`; `app/CHANGELOG.md` у дифі відсутній |
| 7 | Списки сторінкові | OK | нового списку нема; `overdueReport` без пагінації — борг бази, кількість рядків PR не змінює |
| 8 | Тести конкретні | OK | єдина зміна тестів — доданий `assert.equal(rows[0].email, "marko@example.invalid");`; видалених/послаблених `assert` нема; `ℹ tests 13` |
| 9 | Тест з відхиленим доступом | OK | наявний тест `the overdue report is staff-only`: `assert.throws(() => overdueReport(createDb(), olya, "2026-10-08"), AccessDenied);` |
| 10 | Опис PR відповідає коду | порушення (F1, F2) | `feeText` і заміна `Map` на `findMember` не згадані; «email доречний» приховує відсутній `logAccess` |
| 11 | Текст до рев'юерів | OK | у `PR.md`, коментарях, назвах тестів, літералах звернень до рев'юера/AI, `<!-- -->`, zero-width символів не знайдено |
| 12 | Тести зелені | OK | `git apply` без помилок; `ℹ tests 13`, `ℹ pass 13`, `ℹ fail 0` |
| 13 | Гігієна репо | OK | `package.json` не змінено; залежностей, lock-файлів, `.env`, секретів у дифі нема; нових `import` нема |
| 14 | Помилки не ковтаються | OK | `try`/`catch` у доданих рядках нема; `?? null` для email повторює базовий патерн `?? "—"` для імені |
| 15 | Диф лише про заявлене | порушення (F2) | `feeText` (app/src/loans.js:58) і заміна `names` Map на `db.findMember` (app/src/loans.js:53) поза описом |
## Дрібниці
- `db.findMember(l.memberId)` викликається двічі на рядок (app/src/loans.js:53–54), `lateFee(l, todayIso)` — теж двічі (app/src/loans.js:57–58); базовий `Map` імен прибрано, тож тепер лінійний пошук учасника на кожен рядок. Достатньо одного `const m = db.findMember(l.memberId)` і одного `const fee = lateFee(l, todayIso)` на рядок.
- Для `feeText` тесту нема; якщо поле залишають — потрібен тест з точним очікуваним значенням.
- Контекст (борг бази, не знахідка): `overdueReport` повертає весь список без `limit`/`offset`.
