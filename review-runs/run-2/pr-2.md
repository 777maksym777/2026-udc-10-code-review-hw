# Рев'ю PR-2: Email учасника у звіті про прострочення
## Вердикт: BLOCK
## Тести: 13 passed / 0 failed з накладеним дифом (база: 13/13)
## Знахідки
### F1 — Email у звіті віддається без запису в журнал доступу
- Серйозність: critical
- Правило: 2 / домовленість 2 (плюс відсутній тест на `db.accessLog`)
- Де: app/src/loans.js:54
- Що не так: Новий рядок `email: db.findMember(l.memberId)?.email ?? null,` додає PII у відповідь `overdueReport`, але `logAccess(db, user, memberId, purpose)` з `src/audit.js` не викликається жодного разу — ні на кожен рядок, ні взагалі (у `loans.js` `audit.js` навіть не імпортовано). Сусід `memberCard` при віддачі email викликає `logAccess`; тест на `db.accessLog` для звіту також не додано (у тесті лише `assert.equal(rows[0].email, "marko@example.invalid")`).
- Як виправити: Усередині `map` після `requireRole` викликати `logAccess(db, user, l.memberId, "overdue report")` для кожного рядка і додати тест, що `db.accessLog` містить запис на кожного учасника зі звіту.
### F2 — Поле `feeText` форматує гроші в `src/` через ділення на 100 і `toFixed`
- Серйозність: high
- Правило: 4 / домовленість 4 (також 10, 15 — зміна не заявлена в описі)
- Де: app/src/loans.js:58
- Що не так: `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн",` — заборонені патерни `/ 100`, `toFixed(`, літерал `грн` у рядку `src/`; домовленість вимагає лише цілих копійок із суфіксом `Kopecks` (сусіднє `feeKopecks` уже є). Опис PR це поле не згадує взагалі, тесту на нього нема.
- Як виправити: Прибрати `feeText` із `overdueReport` (форматування — справа клієнта), залишивши `feeKopecks`.
### F3 — Форма відповіді `overdueReport` змінена без запису в CHANGELOG
- Серйозність: high
- Правило: 6 / домовленість 6
- Де: app/src/loans.js:54, app/src/loans.js:58 (app/CHANGELOG.md у дифі відсутній)
- Що не так: У `.map(() => ({ … }))` з'явились нові поля `email` і `feeText`, тобто змінена форма відповіді публічної функції, але диф не торкається `app/CHANGELOG.md` (наявний запис там перелічує саме `loanId`, `member`, `book`, `daysLate`, `feeKopecks`).
- Як виправити: Додати до `app/CHANGELOG.md` запис «`overdueReport` — рядок тепер містить `email`» у цьому ж PR.
### F4 — Прибрано індекс `Map` і замінено на повторні лінійні пошуки в кожному рядку
- Серйозність: medium
- Правило: 16
- Де: app/src/loans.js:49 (видалено), app/src/loans.js:53–54, app/src/loans.js:57–58
- Що не так: Видалено `const names = new Map(db.members.map((m) => [m.id, m.name]));`, а замість нього на кожен рядок звіту двічі викликається `db.findMember(l.memberId)` (`?.name`, `?.email`) — тобто лінійний пошук по `db.members` усередині `map` по `db.loans`, O(n·m). Додатково `lateFee(l, todayIso)` тепер обчислюється двічі на рядок (`feeKopecks` і `feeText`).
- Як виправити: Повернути індекс `const members = new Map(db.members.map((m) => [m.id, m]))` і брати `member`/`email` з однієї локальної змінної; `lateFee` обчислювати один раз.
## Опис PR vs код
| Твердження | Код робить | Збігається? |
|---|---|---|
| Для кожного рядка беремо учасника і віддаємо ім'я та email | `member: db.findMember(l.memberId)?.name ?? "—"`, `email: db.findMember(l.memberId)?.email ?? null` — так, але без `logAccess` | частково |
| Звіт і так лише для персоналу, тож email доречний | `requireRole(user, STAFF)` збережено; але домовленість 2 вимагає `logAccess` незалежно від ролі — виклику нема | частково |
| (не згадано) додано поле `feeText` з форматуванням у грн | `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн"` | ні (побічна зміна) |
| (не згадано) прибрано індекс `names` (`Map`), замінено на `db.findMember` у циклі | рядок 49 видалено, два `findMember` на рядок | ні (побічна зміна) |
| (не згадано) змінено тест `overdue report …` | додано лише `assert.equal(rows[0].email, "marko@example.invalid")`, нічого не видалено | ні (не заявлено, але не послаблює) |
## Перевірка за чеклістом
| # | Правило | OK / порушення / не стосується | Доказ |
|---|---|---|---|
| 1 | Перевірка ролі | OK | `requireRole(user, STAFF);` на рядку 48 збережено до читання даних; тест «the overdue report is staff-only» лишився |
| 2 | email/phone → logAccess | порушення | loans.js:54 `email: db.findMember(l.memberId)?.email ?? null` без жодного `logAccess` (F1) |
| 3 | Шляхи від користувача | не стосується | у дифі нема `fs`/`path`/`child_process` |
| 4 | Гроші цілими копійками | порушення | loans.js:58 `/ 100`, `.toFixed(2)`, `" грн"` у `src/` (F2) |
| 5 | Дати YYYY-MM-DD, UTC | OK | у доданих рядках нема `getDate`/`new Date()` тощо; `daysBetween`/`lateFee` не змінені |
| 6 | CHANGELOG | порушення | нові поля `email`, `feeText` у `return`, `app/CHANGELOG.md` у дифі відсутній (F3) |
| 7 | Списки сторінкові | OK | PR не додає нового списку і не робить `overdueReport` більшим за кількістю рядків (борг бази, PR не чіпає) |
| 8 | Тести конкретні | OK | у `test/loans.test.js` лише додано `assert.equal(rows[0].email, "marko@example.invalid")`; нічого не видалено/послаблено; `scripts.test` не змінено; ℹ tests 13 |
| 9 | Тест із відхиленим доступом | OK | перевірка доступу не змінювалась; наявний `assert.throws(() => overdueReport(createDb(), olya, "2026-10-08"), AccessDenied)` |
| 10 | Опис PR відповідає коду | порушення (low, у складі F2) | опис не згадує `feeText` і прибирання індексу `names`; див. таблицю вище |
| 11 | Текст, звернений до рев'юерів | OK | у PR.md, коді, коментарях і назвах тестів звернень до рев'юера/AI нема |
| 12 | Тести зелені | OK | `git apply` успішно; ℹ tests 13, pass 13, fail 0 |
| 13 | Гігієна репо | OK | `package.json` не змінено; нових імпортів, секретів, lock-файлів нема |
| 14 | Помилки не ковтаються | OK | нових `catch`/`try` нема; `?? null`/`?? "—"` для відсутнього учасника — як у базовому `?? "—"` |
| 15 | Диф лише про заявлене | порушення (low, у складі F2) | `feeText` і заміна індексу на `findMember` не заявлені в описі |
| 16 | Складність на колекціях | порушення | видалено `const names = new Map(...)`, на кожен рядок 2× `db.findMember` і 2× `lateFee` (F4) |
| 17 | Ідентифікатори записів | не стосується | нових записів у `db.*` PR не створює |
## Дрібниці
- `email ?? null` для неіснуючого учасника дає `null`, тоді як `member` дає `"—"` — неузгоджені «порожні» значення в одному рядку (косметика).
- Контекст: `overdueReport` і в базі без пагінації — борг бази, PR цього не погіршує.
