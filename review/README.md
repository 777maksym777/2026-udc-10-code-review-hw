# review/

- `prompt.md` — промпт AI-рев'юера: копія тіла слеш-команди `.claude/commands/review.md` (без YAML-frontmatter), покладена сюди, щоб авто-рев'ю курсу її побачило. Стандарт, який він реалізує, — `docs/review-standard.md`.
- Обидва файли **тримаються ідентичними**: редагуєте команду → перегенеровуєте копію (з кореня репо, Git Bash): `{ head -n 2 review/prompt.md; awk 'BEGIN{f=0} NR==1&&$0=="---"{f=1;next} f==1&&$0=="---"{f=2;next} f==1{next} {print}' .claude/commands/review.md; } > review/prompt.tmp && mv review/prompt.tmp review/prompt.md`.
- Запуск рев'ю одного PR: нова сесія Claude Code (або окремий sub-agent) з кореня репо → `/review N`, де `N` — номер з `review-queue/pr-N/`. Сесія бачить лише `PR.md`, `change.diff` і `app/`; сама накладає диф (`git apply`), запускає `cd app && npm test`, відновлює `app/` (`git checkout -- app && git clean -fd app`) і зберігає вивід дослівно у `review-runs/pr-N.md`.
- Один PR — одна сесія; інші PR, `review-runs/`, scorecard і ключ відповідей рев'юеру не показуються.
