# PEG -> monadic parsers on Haskell

Проект реализует проверку принадлежности строки языку, заданному PEG-грамматикой. Описание грамматики парсится в AST, валидируется, затем синтаксически транслируется в монадический PEG-парсер. Для рекурсивных нетерминалов используется packrat-мемоизация по паре `(имя правила, позиция во входе)`.

## Требования

- GHC и Cabal
- Проверено на GHC `9.14.1` и `cabal-install 3.16.1.0`
- Внешние Haskell-зависимости: `containers`, `parsec`

Если Cabal на новой машине сообщает, что нет индекса пакетов, один раз выполните:

```bash
cabal update
```

## Сборка и проверка

Минимальная проверка:

```bash
cabal build
cabal test all
```

Ожидаемый результат тестов:

```text
All tests passed
1 of 1 test suites (1 of 1 test cases) passed.
```

Полная проверка проекта одной командой:

```bash
make check
```

Она выполняет:

1. `cabal build`
2. `cabal test all`
3. CLI smoke-тесты на примерах из `examples/`
4. печать нормализованной грамматики через `--ast`
5. генерацию Haskell-модуля через `--emit-hs`
6. typecheck сгенерированного модуля командой `ghc -fno-code`

Если `make` недоступен, эти шаги можно выполнить вручную:

```bash
cabal build
cabal test all
cabal exec peg-check -- examples/anbn.peg aabb
cabal exec peg-check -- examples/anbn.peg aaabb
cabal exec peg-check -- examples/anbncn.peg aaabbbccc
cabal exec peg-check -- examples/simple_expression.peg "2+3*4"
cabal exec peg-check -- --ast examples/anbncn.peg
cabal exec peg-check -- --emit-hs /tmp/GeneratedPEG.hs examples/anbncn.peg
cabal exec ghc -- -fno-code /tmp/GeneratedPEG.hs
```

Для отрицательных примеров `peg-check` печатает `REJECTED` и завершает работу с кодом `2`.

## Быстрый запуск CLI

```bash
cabal run peg-check -- examples/anbn.peg aabb
# ACCEPTED

cabal run peg-check -- examples/anbn.peg aaabb
# REJECTED, код выхода 2

cabal run peg-check -- examples/anbncn.peg aaabbbccc
# ACCEPTED
```

Выбор стартового правила:

```bash
cabal run peg-check -- examples/anbn.peg aabb --start S
```

Печать нормализованного AST/грамматики:

```bash
cabal run peg-check -- --ast examples/anbncn.peg
```

Генерация Haskell-модуля, где каждое PEG-правило является монадическим парсером:

```bash
cabal run peg-check -- --emit-hs GeneratedPEG.hs examples/anbncn.peg
cabal exec ghc -- -fno-code GeneratedPEG.hs
```

## Поддерживаемый синтаксис PEG

```peg
Rule      = Expression
Choice    = Sequence ("/" Sequence)*
Sequence  = Prefix*
Prefix    = "&" Prefix / "!" Prefix / Suffix
Suffix    = Primary ("*" / "+" / "?")*
Primary   = Identifier / "literal" / 'literal' / [a-z] / . / "(" Expression ")"
```

Комментарии: `# ...`, `// ...`, `-- ...` до конца строки. Правила можно разделять переводом строки, пробелами или точкой с запятой. Первое правило считается стартовым, если явно не передан `--start`.

Семантические действия, метки Peggy и JavaScript-хуки намеренно не реализованы: задача проекта - распознавание языка, а не построение пользовательского значения.

## Структура

```text
src/PEG/AST.hs            AST PEG-выражений
src/PEG/GrammarParser.hs  парсер описания грамматики на Parsec
src/PEG/Validator.hs      статические проверки грамматики
src/PEG/Runtime.hs        монадический парсер, ordered choice, lookahead, memoization
src/PEG/Matcher.hs        компиляция AST -> Parser ()
src/PEG/Codegen.hs        генерация Haskell-модуля с парсерами
app/Main.hs               CLI peg-check
test/Spec.hs              простые регрессионные тесты без внешнего test framework
examples/*.peg            примеры грамматик
FORMAL.md                 формальная часть для защиты
report/                   PDF/Markdown-версия отчета
Makefile                  короткие команды build/test/smoke/check
```

## Что проверяет валидатор

1. Дубликаты правил.
2. Ссылки на несуществующие нетерминалы.
3. Отсутствие стартового правила.
4. Левую рекурсию, включая косвенную.
5. Повторы `e*` и `e+`, где `e` может успешно завершиться без потребления символов. Такие повторы опасны для greedy-семантики, потому что могут зациклиться.

## Коды выхода CLI

- `0` - строка принята;
- `2` - строка корректно разобрана как не принадлежащая языку;
- `1` - ошибка CLI, синтаксиса грамматики, валидации или выполнения.

## Что покрывают тесты

`test/Spec.hs` не использует внешний test framework и проверяет:

- языки `{a^n b^n | n >= 1}` и `{a^n b^n c^n | n >= 1}`;
- дополнение к `{a^n b^n | n >= 1}` над алфавитом `{a,b,c}`;
- выражения с `+`, `*`, скобками и классами символов;
- PEG ordered choice: `"a" / "ab"` на строке `ab` не откатывается ко второй ветке после успеха первой;
- lookahead-предикаты `&` и `!`;
- дополнительные regex-like примеры из `examples/`: идентификаторы, числа, hex colors, даты, IPv4-like, email-like, URL-like;
- ошибки валидатора: дубликаты правил, неизвестные нетерминалы, прямая и косвенная левая рекурсия, zero-width повтор;
- генерацию Haskell-модуля.

## Примеры

### Примеры из материалов по PEG

`examples/anbn.peg`:

```peg
S = ANBN
ANBN = "a" ANBN "b" / "a" "b"
```

`examples/anbncn.peg`:

```peg
S = &(ANBN "c") As BNCN
ANBN = "a" ANBN "b" / "a" "b"
As = "a" As / "a"
BNCN = "b" BNCN "c" / "b" "c"
```

`examples/not_anbn.peg`:

```peg
S = !(ANBN EOF) ANYe
ANBN = "a" ANBN "b" / "a" "b"
ALPH = "a" / "b" / "c"
ANYe = ALPH ANYe / ""
EOF = !.
```

### Дополнительные regex-like примеры

Эти грамматики демонстрируют, что синтаксис PEG покрывает многие привычные
шаблоны из регулярных выражений. Суффикс `like` означает, что пример проверяет
учебную форму строки, а не полный внешний стандарт вроде RFC 5322 или RFC 3986.

| Файл | Принимает | Отвергает | Что демонстрирует |
|---|---|---|---|
| `examples/identifier.peg` | `user_1`, `_tmp`, `ifx` | `if`, `1abc`, `has-dash` | идентификатор и запрет ключевых слов через `!Keyword` |
| `examples/number.peg` | `0`, `+42`, `-3.14` | `3.`, `.5`, `1.2.3` | ordered choice: `Float / Integer` |
| `examples/hex_color.peg` | `#0F8`, `#ff00AA` | `#abcd`, `ff00AA`, `#xyz` | CSS-like `#RGB` и `#RRGGBB` |
| `examples/iso_date_like.peg` | `2026-05-14`, `1999-12-31` | `2026-13-14`, `2026-00-01` | формат `YYYY-MM-DD` с простыми диапазонами |
| `examples/ipv4_like.peg` | `192.168.0.1`, `255.255.255.255` | `256.0.0.1`, `01.2.3.4` | октеты `0..255` через упорядоченные альтернативы |
| `examples/email_like.peg` | `user.name+tag@example.com` | `user@example`, `user@@example.com` | компактная email-like форма |
| `examples/url_like.peg` | `http://example.com`, `https://example.com/a/b` | `ftp://example.com`, `https://localhost` | http/https URL-like форма |
| `examples/balanced_parentheses.peg` | `ε`, `()`, `(()())` | `(`, `())`, `(()` | рекурсивная PEG-грамматика вне регулярных языков |

Например:

```bash
cabal exec peg-check -- examples/ipv4_like.peg 192.168.0.1
# ACCEPTED

cabal exec peg-check -- examples/ipv4_like.peg 256.0.0.1
# REJECTED, код выхода 2

cabal exec peg-check -- examples/email_like.peg user.name+tag@example.com
# ACCEPTED
```
