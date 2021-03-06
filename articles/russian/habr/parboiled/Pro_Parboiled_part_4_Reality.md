Про Parboiled
=============
**Часть 4. Суровая действительность**

Как заставить Parboiled работать еще быстрее? Каких ошибок лучше не допускать? Что делать с наследством в
виде Parboiled1? На эти, а так же другие вопросы призвана ответить заключающая статья серии.

**Структура цикла:**

 - [Часть 1. Почему Parboiled?][part1]
 - [Часть 2. Сопоставление текста][part2]
 - [Часть 3. Извлечение данных][part3]
 - Часть 4. Суровая действительность

[part1]: http://habrahabr.ru/post/270233
[part2]: http://habrahabr.ru/post/270531
[part3]: http://habrahabr.ru/post/270609

<habracut>

# Производительность
Parboiled2 работает быстро, но иногда он может работать еще быстрее. В этом разделе мы поговорим о доступных
микрооптимизациях. Главное при выполнении оптимизаций — своевременность. Но если есть возможность сразу написать
чуть более оптимальный код, не потеряв при этом в выразительности — этой возможностью обязательно следует
воспользоваться.

## Разворачивайте `n.times` для малых n <= 4
Вы можете выиграть в производительности, если для малых *n* вместо оператора повторения `n.times` просто соедините
несколько повторяющихся правил в цепочку. Сколько повторений имеет смысл разворачивать — зависит от обстоятельств,
но едва ли это число больше четырех.

    // Медленно
    rule { 4 times Digit }

    // Быстро
    rule { Digit ~ Digit ~ Digit ~ Digit }

Актуальность этой оптимизации [объявлена][issue-101] самим Матиасом, хотя, гипотетически, оператор `n.times`
мог бы и сам ее выполнять.

[issue-101]: https://github.com/sirthias/parboiled2/issues/101

## Ускорение операций со стеком для `n.times`
Использование подобной техники позволит вам выжать немножко производительности и при извлечении данных
со стека значений. Например, так ее можно применить к предыдущему правилу:

    def Digit4 = rule {
      Digit ~ Digit ~ Digit ~ Digit ~
        push(
          #(charAt(-4))*1000 +
          #(charAt(-3))*100 +
          #(charAt(-2))*10 +
          #(lastChar)
        )
    }

## Не пересоздавайте `CharPredicate`
Совершенно нормально радоваться новым возможностям класса `CharPredicate`, но создавать свои экземпляры типа
`CharPredicate` внутри блока `rule` совершенно не стоит: ваш предикат будет пересоздаваться каждый раз, когда
выполняется правило, что драматически испортит производительность вашего парсера. Поэтому, вместо того чтобы
создавать символьные предикаты каждый раз, определите их внутри вашего парсера как константу:

    class MyParser(val input: ParserInput) extends Parser {
      val Uppercase = CharPredicate.from(_.isUpper)
      ...
    }

или, что еще лучше, отправьте это объявление в объект-компаньон вашего парсера:

    class MyParser(val input: ParserInput) extends Parser {
      ...
    }

    object MyParser {
      val Uppercase = CharPredicate.from(_.isUpper)
    }

## Используйте семантические предикаты
Особенность данных правил состоит в том, что они не взаимодействуют со стеком значений. Подробно, они описаны
в документации, но вот самое главное, что вы должны о них знать:

> При использовании семантических предикатов парсер не совершает прогресса, то есть не перемещает
> свой курсор на следующий символ. Поэтому при их бездумном использовании парсер может зациклиться.

Помните пример с объявлением символьного предиката для символов верхнего регистра? Вы можете сделать тоже
самое, используя семантический предикат `test`:

    def JavaUpperCase = rule { oneOrMore(test(currentChar.isUpper) ~ ANY) }


## Используйте `ANY` там, где хотели бы видеть `CharPredicate.All`
Увы, `CharPredicate.All` работает медленно для больших диапазонов символов, `ANY` работает быстрее. Воспользуйтесь
этим знанием.

## Используйте инвертирующий предикат
Представьте, что ваш парсер должен захватывать все символы до перевода строки (для определенности, в стиле Unix).
Конечно, это можно сделать при помощи `noneOf`, но инвертирующий предикат будет быстрее:

    def foo = rule { capture(zeroOrMore(noneOf("\n"))) }

    // Быстрее?
    def foo = rule { capture(zeroOrMore(!'\n')) }

К сожалению, этот замечательно выглядящий пример зациклит, потому что парсер не будет совершать прогресса. Чтобы это
исправить, необходимо правило, передвигающее курсор парсера, но при этом не изменяющее стек. Например, вот такое:

    def foo = rule { capture(zeroOrMore( !'\n' ~ ANY )) }

Теперь правило `foo` поглотит абсолютно все, кроме `EOI` и перевода строки.


# Отчеты об ошибках
Не думаю, что вам захочется работать с парсером, выдающим бессмысленные сообщения при любых некорректных входных
данных. Parboiled2 способен вполне внятно рассказывать об ошибках, если вы ему в этом поможете.

## Форматирование
Итак, если что-то навернулось, парсер передаст в ваше распоряжение объект типа `ParseError`, который можно
привести в читаемый вид посредством метода `formatError`:

    val errorMessage = parser formatError error

Если форматирование по умолчанию вас по каким-то причинам не устраивает, свои пожелания следует передать парсеру
явным образом:

    val errorMessage parser.formatError(error, new ErrorFormatter(showTraces = true))

Если вы захотите написать свой `ErrorFormatter`, вам придется самостоятельно разобраться со структурой
класса `ParseError`, который объявлен в глубине Parboiled таким образом:

    case class ParseError(position: Position, charCount: Int, traces: Seq[RuleTrace]) extends RuntimeException

Также стоит отметить наличие нескольких схем доставки сообщений об ошибке до пользователя: по вашему желанию
`ParseError` может быть представлен не только в виде объекта `Try`, а, например, в виде полиморфного типа или
`Either`. Подробнее можно ознакомиться [здесь][delivery-schemes].

[delivery-schemes]: https://github.com/sirthias/parboiled2/blob/master/README.rst#alternative-deliveryschemes

    def Foo = rule { "foo" | fail("Я упаль!") }

## Тонкая настройка
Существует опция, позволяющая обойти встроенный механизм формирования сообщений об ошибках.
Для этого нужно использовать правило `fail` с сообщением, которое вы хотите увидеть в случае ошибки:

    def Goldfinger = rule { "talk" | fail("to die") }

Тогда при удобном случае вы получите назад свое сообщение об ошибке примерно в такой форме:

    Invalid input 'Bond', expected to die. (line 1, column 1):

## Именованные правила
Использование подобного типа правил бывает весьма полезным не только в целях отлова ошибок.
Данный механизм подробно описан в разделе «Best Practices».

## atomic
Parboiled2 генерирует парсеры, основанные на PEG. Это означает, что парсеры оперируют символами, а не строками
(как многие могли подумать), поэтому и ошибки вам будут показываться на символьном уровне. Согласитесь — сообщение
вида «У вас тут X, мы ожидали Y или Z» потребует больше мысленных усилий, чем «У вас тут XX, а мы ожидали увидеть XY
или XZ». Для того, чтобы видеть строки в отчетах об ошибках целиком, существует маркер `atomiс`, всего-то и нужно
обернуть в него правило:


    def AtomicRuleTest = rule { atomic("foo") | atomic("fob") | atomic("bar") }

Чтобы при лисичках (`foxes`) на входе получить

    Invalid input "fox", expected "foo", "fob" or "bar" (line 1, column 1):
    foxes
    ^

## quiet
Когда вариантов для выбора слишком много, не всегда хочется уведомлять пользователя о всех возможных альтернативах.
Например, в определенном месте ваш парсер ожидает множество пробельных символов в совокупности с неким правилом.
Для устранения избыточности в отчете, вы, возможно, захотите умолчать о пробелах. С использованием маркера `quiet`
это очень просто:

    def OptionalWhitespaces = rule { quiet(zeroOrMore(anyOf(" \t\n"))) }

Честно признаюсь — ситуаций, поощряющих использования этого правила, я не встречал. Так же, как и `atomic`, оно
подробно [описано в документации][doc-quiet].

[doc-quiet]: https://github.com/sirthias/parboiled2/blob/master/README.rst#the-quiet-marker

## Восстановление после ошибок
Практически единственный эпизод, где Parboiled1 выигрывает, а у Parboiled2 дела обстоят не очень хорошо: парсер
падает уже только от вида первой же встреченной им ошибки. Для большинства сценариев это отлично подходит:
это, например, не мешает парсить логи, текстовые протоколы, конфигурационные файлы (для ряда случаев),
однако разработчикам DSL или IDE-подобных инструментов такое положение дел будет не по душе.
[Матиас обещает это исправить][issue-42], поэтому если вам эта функциональность очень сильно нужна уже сегодня
— напишите на баг-трекер, возможно это ускорит процесс разработки.

[issue-42]: https://github.com/sirthias/parboiled2/issues/42

В Parboiled1 имеется [огромное число ParserRunnerов][runners] на все случаи жизни. Посмотрите в сторону
`RecoveringParserRunner`, если вам нужно продолжать парсинг в случае ошибок.

[runners]: https://github.com/sirthias/parboiled/wiki/Parse-Error-Handling


## Тестирование
Разработчики Parboiled используют для тестирования фреймворк [specs2][specs2], который они дополнили своим
вспомогательным классом [TestParserSpec][tps]. Он покажется неудобным тем, кто использует scalatest, но основную
его идею можно и перенять. По секрету от Матиаса, его решение не отличается особенной аккуратностью, так как
полагается на изменяемое состояние. Возможно, в будущем нас будет ждать что-то похожее на полноценный каркас
для тестирования.

[specs2]: https://etorreborre.github.io/specs2/
[tps]:    http://bit.ly/1Y5iZ9t

Правила можно тестировать как по отдельности, так и вместе. Лично я предпочитаю писать тесты не на каждое правило,
а проверять только главное правило в «особых» случаях:

> Во многих форматах, даже стандартизованных, могут встречаться весьма интересные моменты.
> Например, в BSD-подобном формате сообщений [RFC 3164][rfc3164] под число месяца *всегда* отводится две позиции,
> даже если само число имеет один разряд. Вот пример из самого RFC:
>
> > If the day of the month is less than 10, then it MUST be represented as a space and then the number. For
> > example, the 7th day of August would be represented as `"Aug  7"`, with two spaces between the `"g"` and
> > the `"7"`.

[rfc3164]: https://www.ietf.org/rfc/rfc3164.txt

Помимо подобного рода «интересных моментов» можно скармливать парсеру строки с незакрытыми скобками, недопустимыми
символами, проверять порядок операций со стеком значений.

В тестировании есть еще одна тонкость, с которой вы сразу же столкнетесь. Предположим, вы хотите оттестировать
следующее правило:

    def Decimal: Rule0 = rule {
      ("+" | "-").? ~ Digit.+ ~ "." ~ Digit.+
    }

Для этого отправим парсеру заведомо некорректный ввод и будем ждать на выходе ошибку:

    // Я еще не видел десятичных дробей с двумя разделителями.
    val p = new MyParser("12.3.456").Decimal.run()  // Success(())
    p.isFailure shouldBe true  // тест упадет

Но при прогоне теста окажется, что парсер вернул удачный результат. Почему так? В нашем правиле нет `EOI`, но
если если мы добавим в него `EOI`, то испортим все правила, которые используют `Decimal`. Поэтому придется
создать специальное тестирующее правило, например, при помощи хитрого механизма [мета-правил][metarule].
Давайте добавим EOI в конце нашего предыдущего примера, и убедимся в том, что парсер упал с ошибкой:

    Failure(ParseError(Position(5,1,6), Position(5,1,6), <2 traces>))

[metarule]: https://github.com/sirthias/parboiled2/blob/master/README.rst#advanced-techniques



# Недостатки Parboiled

## Parboiled2
Если недостатки есть у людей, почему бы их не иметь библиотекам? Здесь Parboiled2 не является исключением.

 - Длинные, слишком общие и совершенно непонятные сообщения компилятора об ошибках, в лучших традициях C++.
   Наглядный пример приведен на рисунке ниже (в правиле нечаянно пропущен оператор `~`). Причина связана с
   выполнением продвинутых проверок на типах, которые [обещают убрать][issue-106] в будущих версиях.

![Компилятор грязно ругается](https://hsto.org/getpro/habr/post_images/ca6/fe6/0a4/ca6fe60a4692d96f4d7e2034bc4eaa0c.png)

[issue-106]: https://github.com/sirthias/parboiled2/issues/106

 - Эта проблема относится больше не к Parboiled2, а к scalac. Компилятору может снести крышу, если у лямбды,
   захватывающей значения со стека, явно (не)определены типы аргументов:

        // Может не сработать
        def MyRule = rule { oneOrMore(Visible) ~> {s => "[" + s + "]"} }

        // Скорее всего сработает
        def MyRule = rule { oneOrMore(Visible) ~> {s: String => "[" + s + "]"} }
   Что сработает, а что нет — зависит от версии вашего компилятора.

 - Многие IDE еще не научились поддерживать макровыражения, а Parboiled2 был построен не без их помощи. Поэтому
   не стоит верить подчеркиваниям вашей среды разработки. Однажды я, забыв об этом, потратил целый день на поиск
   несуществующей ошибки буквально на ровном месте.

 - Отсутствие механизма восстановления при неудачном разборе. Проектирующих предметно-ориентированные языки, или же
   тех, кто хочет использовать Parboiled2 в качестве фронтэнда к своему компилятору, это сильно разочарует. Но
   над этим [работают][issue-42]. Если вы хотите видеть эту возможность — пишите, это ускорит разработку.

 - Я думаю, что многим разработчикам своих небольших IDE и текстовых редакторов хотелось бы видеть более гибкие
   сообщения об ошибках, чем те, что предоставляются сейчас. На данный [момент][issue-96] существует всего два
   способа повлиять на них:
    - именованные правила,
    - именованные вложенные правила.

[issue-96]: https://github.com/sirthias/parboiled2/issues/96


## Parboiled1
Большинство проектов все еще написаны на Parboiled1, и вряд-ли что-то изменится резко и кардинально (в энтерпрайзе),
поэтому может быть полезным знать, как научиться мириться с его недостатками, коих у Parboiled1 немало. Помимо
весьма ограниченного DSL, у Parboiled существует проблема «Rule8», которая усложняет написание парсера для логов.
Parboiled1 построен так, что на каждое правило с N элементами имеется по классу, по аналогии со скаловскими кортежами
(tuples): есть `Rule0`, `Rule1`, вплоть до `Rule7`. Этого вполне достаточно, чтобы распарсить сложные языки
программирования, такие как Java, да и вообще не вызывает существенных проблем при разборе древовидных структур.
А вот если нужно извлечь данные из линейной структуры, например, сообщения лога-файла, то в это ограничение очень
несложно упереться. Решается это использованием кортежа вместо одного результирующего правила. Вот пример:

    def Event: Rule1[LogEvent] = rule {
      Header ~ " " ~ UserData ~ " " ~ Message ~~> {
        (header, data, message) => SyslogEvent (
          header._1, header._2, header._3, header._4, header._5, data._1, data._2, message
        )
      }
    }

Пусть выглядит убого, зато проблема решена.


# Best practices
В этом разделе я расскажу о прописных истинах, работающих для любого комбинатора парсеров, а так же о нюансах,
специфичных для Parboiled2.

## CharUtils
Есть один полезный объект, не затронутый в документации: [CharUtils][CharUtils]. Он содержит ряд статических методов,
способных облегчить вашу жизнь, например: изменение регистра символов, экранирование, преобразование целочисленных
значений в соответствующие им символы (строки). и др. Его использование, возможно, сэкономит ваше время.

[CharUtils]: http://bit.ly/1NJJ2kd

## Пишите модульные тесты
Одно небольшое неудачное изменение может сломать вам грамматику и обеспечить острую ректальную боль. Это банальный
совет, которым многие пренебрегают. Парсер не так сложно протестировать, как, скажем IO: вам не нужны Mock-объекты и
другие ухищрения для этой рутинной, но очень ценной работы. У нас была целая инфраструктура парсеров. И поверьте,
первое, что я делал при поиске ошибок — садился и писал тесты, в случае их отсутствия.

## Делайте парсеры и правила маленькими
Разделяйте ваши парсеры на подпарсеры. Каждый компонент должен делать что-то вполне определенное. Например если вы
парсите LogEvent, у которого определено поле Timestamp (особенно если этот Timestamp соответствует
какому-нибудь Rfc),то не поленитесь и вынесите его отдельно.

  - Во-первых, это уменьшит код вашего основного прасера, и сделает его нагляднее.
  - Во-вторых, это заметно облегчит тестирование. Вы покроете модульными тестами ваш подпарсер.
    А после этого приступите к разработке главного парсера

Существуют разные подходы:

 - Разбивать парсер на трейты и использовать self-typed reference (предпочитаю этот способ).
 - Объявлять парсеры как самостоятельные сущности и использовать композицию.
 - Использовать встроенный механизм для создания subParsers.

Правила должны быть максимально компактными, но не компактней. Чем меньше ваши правила, тем легче найти ошибку в
грамматике. Очень сложно понять логику разработчика, если он делает правила длинными, и при этом многократно
использует `capture`. Усугублять ситуацию может неявный захват. Указание типа правила также помогает при поддержке.


## Отправляйте case objects вместо строк в Value stack
Данный совет можно отнести и к оптимизациям, так как это заставит парсер работать быстрее.
Отправляйте в Value stack значимые объекты, а не строки. Это сделает ваш парсер быстрее, а код нагляднее.

Плохо:

    def logLevel = rule {
      capture("info" | "warning" | "error") ~ ':’
    }

Хорошо:

    def logLevel = rule {
        “info:”   ~ push(LogLevel.Info)
      | “warning" ~ push(LogLevel.Warning)
      | “error"   ~ push(LogLevel.Error)
    }


## Используйте упрощенный синтаксис для сборки объекта
Этот красивый способ появился еще в Parboiled1. Никакой магии, просто конструктор case classа вызывается неявно.
Главное, чтобы количество и тип аргументов, помещаемых на Value Stack, совпадали с сигнатурой конструктора case classа.

Плохо:

    def charsAST: Rule1[AST] = rule { capture(Characters) ~> ((s: String) => AText(s)) }

Хорошо:

    def charsAST = rule { capture(Characters) ~> AText }

## Именованные правила (named rules)
Именованные правила заметно упрощают жизнь при получении отчетов об ошибках, так как дают возможность вместо
невнятного имени правила использовать псевдоним. Или же помечать правила определенным тегом — «Это выражение»
или «Модифицирует стек». В любом случае знать о данной функции будет полезно.

Многие пользователи Parboiled1 уже полюбили эту возможность. Например разработчики Neo4J, использующие
Parboiled для разбора языка [Cypher][cypher].

[cypher]: http://neo4j.com/docs/2.2.3/cypher-introduction.html

Как это выглядит в Parboiled1:

    def Header: Rule1[Header] = rule("I am header") { ... }

В Parboiled2:

    def Header: Rule1[Header] = namedRule("header is here") { ... }

Так же есть возможность давать имена вложенным правилам:

    def UserName = rule { Prefix ~ oneOrMore(NameChar).named("username") ~ PostFix }


# Миграция
Миграция — процесс, чаще всего, несложный, но времени отнимает немало. Поэтому я постараюсь хотя бы немного
сэкономить драгоценные часы вашей жизни и описать основные подводные камни.

## Classpath
Для того, чтобы избежать конфликтов с первой версией, Parboiled2 использует classpath `org.parboiled2` (тогда
как classpath для первой версии `org.parboiled`). Мавеновский `groupId`, однако, остался старым: `org.parboiled`.
Благодаря этому, можно иметь обе зависимости в одном проекте и осуществлять постепенный переход на новую версию.
Что, кстати, работает весьма неплохо при наличии нескольких автономных парсеров. Если же ваши парсеры состоят
из множества модулей, переиспользуемых в разных местах (как это было в моем случае) — вам придется делать миграцию
сразу и для всех модулей.

## Проверка тестов
Убедитесь в наличии и работоспособности модульных тестов. Они же у вас есть? Нет? Напишите их. В процессе миграции
мне приходилось уточнять некоторые грамматики из-за того, что новый DSL стал мощнее, и нечаянные изменения ломали
грамматики. Падающие тесты экономили много времени.
С серьезными проблемами, вроде поломки всей грамматики целиком, при миграции я не сталкивался. Может быть кто-то
поделится опытом, если с ним это произошло.

## Код вокруг парсера
Теперь парсер будет пересоздаваться каждый раз, что не всегда удобно. С PB1 я очень любил создавать парсер единожды,
а потом многократно его использовать. Теперь этот номер не пройдет. Поэтому вам придется изменить конструктор
парсера и немного переписать использующий его код, и не бойтесь, что это ухудшит производительность.

> **Предупреждение** Parboiled1 позволяет генерировать правила во время выполнения. Поэтому если
> у вас имеется подобный парсер, то вам, скорее всего, придется его переписать: Parboiled2 использует макровыражения
> которые делают динамику весьма затруднительной, взамен давая лучшую производительность.

## Композиция
Подход к композиции элементов парсера не изменился, это хорошая новость для мигрирующих. Однако `Parser` теперь не
трейт, а абстрактный класс. Трейты (traits) — удобнейшее средство композиции програмных компонентов, в PB1 это
позволяло подмешивать `Parser` в любые модули, смешивая модули между собой. Изменение в пользу абстрактного
класса на эту возможность никак не повлияло, но теперь для этого нужно использовать
[self-typed reference][self-typed-ref]:

[self-typed-ref]: http://docs.scala-lang.org/tutorials/tour/explicitly-typed-self-references.html

    trait Numbers { this: Parser =>
      // Ваш код
    }

Тем, кто подобную возможность языка не использовал и каждый раз подмешивал трейт `Parser`, придется изменить свои
вкусовые предпочтения.

В качестве альтернативного способа, вы можете сделать из ваших трейтов полноправные парсеры и импортировать
из них нужные правила (как методы) в ваш основной парсер. Я, правда, все равно предпочитаю использовать
композиции трейтов, потому как нахожу их более наглядными: мне понятней видеть парсер собранный из кусков, вместо
множественных импортов.

## Избавляемся от примитивов
В процессе миграции обязательно устройте ревизию своей личной библиотечки примитивных правил: удалите все что
имеется в `CharPredicate`. Ваша библиотечка похудеет, однако не исчезнет совсем. Многие хотели бы добавить в
parboiled поддержку различных форматов дат, грамматику описывающую электронную почту, заголовки HTTP. Parboiled
просто комбинатор парсеров: он таковым был, таким и останется. Однако согласитесь, что выбрасывать старый код
очень приятно.


# Заключение
В этой серии статей я попытался рассказать вам про самый прогрессивный и перспективный инструмент парсинга,
существующий для языка scala. Сделал небольшой туториал и рассказал о проблемах, с которыми мне пришлось
столкнуться на практике. Надеюсь, что эта статья в худшем случае окажется для вас полезной,
а в лучшем — станет руководством к действию.

# Использованные источники
 - [Список рассылки проекта Parboiled][mail-list]
 - [Презентация Александра Мыльцева][myltsev-presentation] и [слайды к ней][myltsev-slides]
 - [Примеры кода из репозитория Parboiled][pb-examples]
 - [Парсер языка scala, написаный при помощи Parboiled2][scala-parser]

[mail-list]:            https://groups.google.com/forum/#!topic/parboiled-user/Ygb_M6XU5P8
[myltsev-presentation]: http://www.youtube.com/watch?v=qZg4D62K4aQ
[myltsev-slides]:       http://myltsev.name/ScalaDays2014/#/
[pb-examples]:          http://bit.ly/1H2ZQ3A
[scala-parser]:         https://github.com/sirthias/parboiled2/tree/master/scalaParser/src

# Благодарности
Хочу выразить благодарность Александру и Матиасу за то, что у меня появился повод для статьи, а так же удобный
инструмент. Яна, спасибо тебе за вычитку и правку моих многочисленных ошибок, обещаю буду писать грамотнее.
Спасибо @firegurafiku и Too Taboo за помощь в вертске первой статьи, вычитку, многочисленные исправления, и
идеи для примеров в последующих. Спасибо Владу Ледовских, за вычитку и исправления в последней статье серии.
Спасибо @nehaev, за найденную в коде статьи ошибку, и Игорю Кустову за идею разбить огромнейшую статью на части
(я долго не хотел этого делать). Особая благодарность Арсению Алесандровичу @primetalk, за найденные неточности и
полезные предложения. Спасибо и всем тем, кто следил за циклом статей и дошел до последней. Надеюсь работа не была
проделана зря.

