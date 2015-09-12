# Все, что вы хотели знать о Parboiled

Сегодня, в свете бурного роста популярности функциональных языков, всё чаще находят применение комбинаторы парсеров —
подход, облегчающий разбор текста простым смертным. Такие инструменты, как Parsec (Haskell) и Planck (OCaml) уже успели
хорошо себя зарекомендовать в своих экосистемах. Их удобство и востребованность в своё время подтолкнули создателя
языка Scala, Мартина Одерски, внести в стандартную библиотеку их аналог — Scala Parser Combinators
(ныне вынесены в scala-modules), а знание и умение пользоваться подобными инструментами — отнести к обязательным
требованиям к Scala-разработчикам [уровня A3](http://www.scala-lang.org/old/node/8610).

Эта статья посвящена библиотеке Parboiled — мощной альтернативе и возможной замене для Scala Parser Combinators.
Мы подробно рассмотрим работу с текущей версией библиотеки — Parboiled2, а также уделим внимание Parboiled1,
так как большая часть существующего кода всё ещё использует именно её. Мы затронем следующие вопросы:

 - Почему Parboiled?
 - Введение в Parboiled2 от простого к сложному.
 - Миграция с первой на вторую версию библиотеки.
 - Подводные камни Parboiled1 и Parboiled2.
 - Паттерны и best-practices™ при написании парсеров.

Итак, всех заинтересовавшихся прошу под кат.

------------------------------------------------------------------------------------------------------------------------

# Введение
Parboiled - библиотека позволяющая вам с легкостью разбирать (парсить) языки разметки (такие как HTML, XML или JSON),
конфигурационные файлы, логи, языки программирования, текстовые протоколы и многое другое. Если вы хотите спроектировать
свой предметно-ориентированный язык ([DSL](https://en.wikipedia.org/wiki/Domain-specific_language)) Parboiled придет вам
на помощь. Вы сможете получить древовидное представление вашей предметной области, или используя паттерн
[интерпретатор](https://en.wikipedia.org/wiki/Interpreter_pattern), исполнять команды вашего доменного языка.

На данный момент существует несколько версий данной библиотеки:

  - Parboiled for java - до сих пор пользуются популярностью, хоть и находится в состоянии End of Life. Если по воле
    случая она вам досталась в наследство, или же вы начинаете проект на java, предлагаю обратить ваше внимание на
    [grappa](https://github.com/fge/grappa). Эта библиотека является форком parboiled1, написана на java, и старательно
    поддерживается пользователем с ником fge.
  - Parboiled for scala, далее pb1 - После того как Mathias Doenitz (разработчик библиотеки) проникся скалой, он сделал
    scala-фронтэнд для parboiled, тем самым забросив поддержку java-версии. Потихонечку перестает поддерживаться,
    однако, не смотря на это списывать его со счетов ее тоже не стоит:
      - Parboiled2 не научился поддерживать все фичи Parboiled1
      - На данный момент Parboiled1 используется в большинстве проектов, с которыми возможно вам предстоит столкнуться
  - Parboiled2 - новейшая версия библиотеки, устраняющая ряд недостатков pb1. Работает быстрее и, самое главное:
    поддерживается разработчиками.

Как уже было написано ранее, данна статья написана с упором на Parboiled2. Иногда я буду отвлекаться, чтобы рассказать
об отличиях важных между первой и второй версией библиотеки.


## Основные фичи Parboiled2
  - Parboiled2 основан на [PEG](https://en.wikipedia.org/wiki/Parsing_expression_grammar)
  - Parboiled геренирует однопроходные парсеры. Лексер не требуется
  - Используется DSL являющися подмножеством языка scala
  - Оптимизации выполняются на этапе компиляции


## Почему Parboiled
  - Вам не нужно писать парсер голыми руками.
  - Parboiled основан на PEG. Это позволяет разбирть рекурсивные структуры данных (регулярные выражения не могут это
    [по определению](https://en.wikipedia.org/wiki/Chomsky_hierarchy#The_hierarchy)). Да, регулярными выражениями вы не
    распарсите JSON или даже простенький калькулятор (чего уж говорить о языках программирования). Небезызвестная
    цитата:
> Parse arbitrary HTML is like asking Paris Hilton to write an operating system.
  - Читаемость сравнимая с различными сортами bnf (по моему мнению даже и лучше)
  - Даже если вам нужно разобрать линейную (нерекурсивную структуру) parboiled2 (при использовании должных оптимизаций)
    будет работать быстрее регулярных выражений. Подтверждающие это данные получены
    [отсюда](https://groups.google.com/forum/#!msg/parboiled-user/XATcJRLTXjA/XSmf3n6gZSwJ).

        Бенчмарк:
          parboiled2 (warmup): 1621.212ms
          parboiled2: 409.162ms
          parboiled2 with better types (warmup): 488.919ms
          parboiled2 with better types: 134.676ms
          regex (warmup): 621.95ms
          regex: 620.379ms

  - В отличии от генераторов парсеров, таких как ANTLR, вы освобождены от генерации java кода парсера, который надо в
    последствии скармливать компилятору. Весь код в parboiled пишется на scala, поэтому вы получаете подсветку
    синтаксиса и проверку типов из коробки, а так же отсутствие дополнительных операций над файлами грамматик. Однако,
    в сравнении с Parboiled, ANTLR мощнее, документированее и стабильнее.

  - Скаловские парсер-комбинаторы работают медленно. Очень медленно. Слишком медленно. Mathias Doenitz проводил
    сравнение производительности Jackson и JSON парсеров сгенерированных с помощью библиотек Parboiled, Parboiled2 и
    Scala parser-combinators. С неутешительными результатами для последнего можно ознакомиться
    [здесь](https://groups.google.com/forum/#!topic/parboiled-user/bGtdGvllGgU).

  - В отличии от Language Workbenches, Parboiled - маленькая и простая в использовании библиотека. Вам не нужно
    скачивать плоходокументированного и тормозящего монстра, тратить часы драгоценной жизни на изматывающее общение с
    UI: поиск нужных менющек и кнопочек, всего-навсего для построения небольшого DSL. Да, вы не получите готовый
    текстовый редактор из коробки и, возможно, вам придется самостоятельно писать плагины для текстовых редакторов.
    Однако parboiled вполне достойная альтернатива для маленьких предметно-ориентированных языков.

  - parboiled успешно зарекомендовал себя во
    [многих проектах](https://github.com/sirthias/parboiled/wiki/Projects-using-parboiled).


### Parboiled2 против писанных-прямыми-руками Json парсеров
Небольшой benchmark, проведенный Александром Мыльцевым (разработчиком библиотеки Parboiled2) (взято
[отсюда](http://myltsev.name/ScalaDays2014/#/)), показывает, что скорость разбора приближается к "древним магическим
парсерам эльфийской работы"

                  бенчмарк    mc линейное время выполнения
      Parboiled1JsonParser 85.64 ==============================
      Parboiled2JsonParser 13.17 ====
                  Argonaut  7.01 ==
              Json4SNative  8.06 ==
             Json4SJackson  4.09 =


## Новые возможности Parboiled2
Данный раздел будет в основном полезен тем кто уже работал с первой версией библиотеки.

  - Parboiled2 решает ряд детских болезней первой версии
    - Решена так называемая Rule7-problem. Для этого была использована библиотека shapeless с ее знаменитыми HListами:
      теперь одно правило может оперировать большим количеством значений на стеке.
    - Добавлены недостающие конструкции. Например, В Pb1 нельзя было указать динамическое количество повторений для
      правила nTimes. Для решение приходилось использовать правило более "мягкое" правило oneOrMore, что не давало\
      требуемой точности при описании грамматики.
    - Встроенные примитивные терминалы. Появился класс CharPredicate, который содержит такие поля как AlphaNumeric, Hex,
      Printable, Visible и другие. Многие пользователи pb1 писали свои маленькие стандартные библиотеки терминалов.
    - Добавлена возможность расширения и сужения предиката. Раньше возникала потребность исключить несколько символов из
      правила. Например нам нужны все видимые символы, кроме '=' и ']'. Теперь это с легкостью можно сделать. А не
      создавать "белый список" символов.
  - Parboiled2 использует дополнительную зависимость - библиотеку shapeless.
  - Parboiled2 использует макросы, это позволяет генерировать грамматику на этапе компиляции, а не в рантайме, как это
    сделано для parboiled1. Это многократно увеличивает производительность вашего парсера, и увеличивает количество
    проверок. В связи с этим блок 'rule' стал обязательным. В некоторых случаях код корректно работает без него.
    Теперь это не так.
  - Улучшена система отчета об ошибках
  - Поддержка scala.js. Демо-проект располагается 
    [здесь](https://github.com/alexander-myltsev/parboiled2-scalajs-samples).


# Как работать с библиотекой
В этому разделе вашему вниманию будет представлен небольшой туториал о том, как работать с Parboiled2. Начнем мы
конечно с простого.

## Подготовительные работы
Перед началом работы с библиотекой добавим ее в classpath, например в maven это делается так:

    <dependency>
        <groupId>org.parboiled</groupId>
        <artifactId>parboiled_2.11</artifactId>
        <version>2.1.0</version>
    </dependency>

Я использую scala 2.11, однако существует версия и для 2.10.
А теперь напишем парсер который ничего не делает, что не мешает ему существовать и радовать нас.

    import org.parboiled2._

    class MyParser(val input: ParserInput) extends Parser {
      // I am useless
    }


Конструкции DSL, а так же ряд полезных класов добавляются в зону видимости всего одной строчкой импорта.
Прошу заметить, что наличие поля input в конструкторе является обязательным, Input должен быть уникальным для каждого
экземпляра парсера. Это означает, что на каждый вход нужен новый парсер. Вначале, меня это очень сильно пугало, бояться
я перестал когда увидел как быстро оно работает.

Итак, никчемный парсер у нас уже имеется, теперь давайте опишем то, что нашему парсеру предстоит распознать. Напишем
небольшой распознаватель (recognizer), способный проверять правильность ввода. Если вход оказался валидным (с точки
зрения нашего парсера), нам просигнализируют об успешном выполнении операции, в противном случае скажут "что не так".


## Описываем правила (Rule DSL)
Работавшим с Parboiled1 этот раздел можно просто пролистать, (объяснения могут показаться им уж слишком подробными).
Тем кто знакомится с Parboiled2 - следует прочитать.
И начнем мы с терминалов. Это термин будет использоваться в дальнейшем, поэтмоу определим его.
> Терминалы - символы не требующие дополнительных определений.
Теперь давайте добавим в наш парсер два приведенных ниже правила: одно должно распознать символ, другое строку.

    // Сматчит символ 'a'
    def MyCharRule = rule { ch('a') }

    // Cметчит строку "string"
    def MyStringRule = rule { str("string") }

Каждый раз обозначать свои намерения подобным образом весьма уморительно. Благодаря механизму неявного преобразования
(implicit conversions), следующие правила будут выглядеть так:

    def MyCharRule = rule { 'a' }
    def MyStringRule = rule { "string" }

теперь, давайте проверим работоспособность наших правил, для этого у каждого правила (Rule) определен метод run().
Для этого где-нибудь в коде вызовете:

    // возвращает scala.util.Try
    myParser.MyStringRule("string").run
    myParser.MyStringRule("won't match").run

На выходе мы получим объект типа Try. Так же можно распознавать строки вне зависимости от их регистра:

    // передаваемая в правило строка должна быть в нижем регистре
    def StringWithCaseIgnored = rule { ignoreCase("string") }

Правила (Rule) - особая форма материи (подробнее о них будет рассказано позже). Нам главное знать что Rule0 сопоставляет
вход с правилом и говорит нам совпало или нет. Все приведенные ниже правила имеют данный тип.

    def StringWithCaseIgnored: Rule0 = rule { ignoreCase("string") }

В Parboiled существуют особенные терминалы:
  - EOI (End of Input) - виртуальный символ, котоый вы обязательно захотите добавить в конце главного правила :)
  Определен EOI так:
  
        val EOI = '\uFFFF'
  - ANY - соптавляет все символы, кроме EOI

Так же в Parboiled2 можно объявлять диапазоны символов. Делается это так:

    // Сматчат по одному символу
    def Digit      = rule { '0' - '9' }
    def AlphaLower = rule { 'a' - 'z' }

А вот Parboiled1 заставляет делать это часто - практически каждый раз, когда вы пишете парсер. Поэтому я носил свою
библиотечку примитивов из проекта в проект (Уверен, что не я один так делал). Теперь так делать не надо, так как у нас
есть CharPredicate. Думаю, что из названий будет понятно каким категориям символов они соответствуют.
 - CharPredicate.All (работает как Any, но значительно медленнее)
 - CharPredicate.Digit
 - CharPredicate.Digit19
 - CharPredicate.HexDigit

Для CharPredicate работают правила 'except' и 'union', которых не было в pb1, (и, лично я от этого страдал)
Приходилось замыкать правило "с другой стороны" указвая "черный" или "белый" список, в зависимости от ситуации.

    // Сопоставит любой печатный символ, если это не кавычка
    def AllButQuotes = rule {
      CharPredicate.Visible -- "\"" -- "'"
    }

    // Вполне себе сойдет для определения идентификатора:
    // Alphanumeric объединяется с нижним подчергиванием
    def ValidIndentifier = rule {
      CharPredicate.AlphaNum ++ "_"
    }

Полезно будет рассказать еще о двух правилах: anyOf и noneOf. Работают они очень похоже на except и union для
CharPredicate, однако на всем пространстве символов ANY. И главное - они работают быстрее. Эти функции могут принимать
на вход строку состоящую из перечислений символов. Например:

    // Сопоставит, если символ является цифрой или одной из операций
    def DigitOrOperation = rule { anyOf("1234567890+-=/*") }

    // сопоставит все кроме пробела и EOI
    def EverythingExceptTheWhiteSpace = rule { noneOf(" ") }
    
Конечно же никто не будет матчить единичные символы. Поэтому перейдем к более сложным правилам:

### N.times
Правило которое позволяет сопоставить вполне конкретное число символов, выглядит так:

    def 33Cows = rule { 33 times "Cow" }

Некоторые грамматики требуют жесткого диапазона числа символов символов, например
[от двух до пяти](http://www.chukfamily.ru/Kornei/Prosa/Ot2do5/Ot2do5.htm). Теперь так можно сделать:

    import CharPredicate.Digit
    def FromTwoToFive = rule { 2 to 5 times Digit }

Для Parboiled1 существует правило nTimes, которое сопоставлет жестко заданное число символов.

### Цепочки
#### Правило "Ноль или много" (zero or more)
Делает то же что
[звезда Клини](https://ru.wikipedia.org/wiki/%D0%97%D0%B2%D0%B5%D0%B7%D0%B4%D0%B0_%D0%9A%D0%BB%D0%B8%D0%BD%D0%B8) для
регулярных грамматик.

    def WhiteSpace = rule { anyOf(" \n") }
    // пробелов может быть много, а может и не быть
    def OptionalWhiteSpaces = rule { zeroOrMore(WhiteSpace) }

#### Правило "Один и более" (one or more)
Тоже что ZeroOrMore, однако хотя бы один символ должен присутствовать (идентично Плюсу Клини для регулярных грамматик).

    def UnsignedInteger = rule { oneOrMore(Digit) }

#### Разделители
Зачастую цепочки требуется разделять, и делается это легко и непринужденно:

    def SeparatedNumbers rule { oneOrMore(Number).separatedBy(",") }

### Операция последовательности (Sequence)
Последовательность правил, выглядит как тильда '~'. Опишем электронный адрес некоего пользователя, зарегистрированного
на example.com

    // Предположим, что имя пользователя может включать все печатные символы
    // кроме собаки
    def UserName = rule { oneOrMore(CharPredicate.Visible -- "@") }

    // Правило опрелеляющее Email пользователя example.com
    def ReallySimplifiedEmail = rule { UserName ~ "@" ~ "example.com" }

### "Не обязательное" правило (Optional)
Если бы существовало правило zeroOrOne, то это бы и был optional. Либо есть одно вхождение, либо вхождений нет совсем.
Давайте разберем следующий пример:
У каждой строки текста есть вполне себе newline. В разных семейства операционных систем он кодируется по разному.
Например в Unix-подобных операционных системах нужен всего символ "\n", тогда как в Windows нужна последовательность из
двух симвловов: "\r" и "\n". Поэтому наше правило должно выглядеть так:

    // Сопоставился newline для Linux, OS X, Windows
    def NewLine = rule { optional('\r') ~ '\n' }


### Упорядоченный Выбор (Ordered Choice)
Аналог для Или в языках программирования и '|' в регулярных выражениях.
Итак, предположим, что нам нужно распознать число у которого может быть знак, а может и не может.
Знак бывает двух типов: + и -. разберемся сначала с ним:

    // Знак бывает как положительный так и отрицательный
    def Signum = rule { '+' | '-' }

    // Знака может и не быть
    def MayBeSign = rule { optional(Signum) }

    // Число со знаком
    def Integer = rule { MayBeSign ~ UnsignedInteger }

Порядок следования правил имеет значение. Это исключает возможность появление неоднозначности вашей грамматике. И да,
так работают все PEG парсер-генераторы. Например, мы сопоставляем операции сравнения для числовых величин, у нас есть:
<,=,>,<=,=>. Запишем правило в сематнике Parboiled2.

    def ComparisonOperators = rule { "<" | "==" | "<" | "<=" | "=>" }
Парсер сопоставит правила для `<`, `==` и `>`. А вот если на вход мы подадим `<=`, то сопоставиться только `<`. Парсер
просто не пойдет проверять дальше первых трех правил. Поэтмоу, чтобы "работало" пример должен быть записан так:

    def ComparisonOperators = rule { "==" | ("<" ~ optional("=")) | (">" ~ optional("=")) }


### Немного сахара 
Для optional, oneOrMore и zeroOrMore существует синтаксический сахар, позволяющий сделать определения еще короче.
Пожалуйста, используйте его мудро, ибо если навертеть, ваши правила будут читаться немногим лучше чем регулярки.

    import CharPredicate.Digit

    // + для oneOrMore
    def UnsignedInteger = rule { Digit.+ }

    // ? для optional
    def SignedInteger   = rule { (+ | -).? ~ UnsignedInteger }

    // и * для zeroOrMore
    def Integers  = rule { SignedInteger.*.separatedBy("\n") }
    // Лучше
    def BetterIntegers = rule { zeroOrMore(SignedIntegers) separatedBy "\n" }

### Вложенные структуры данных
То что может Parboiled и не могут регулярные выражения. Да, можно парсить рекусивные выражения. И Это не сложно,
единственное дополнительное усилие, которое от вас требуется - явное объявление типа в в том месте где используется
рекурсивыный вызов.

TODO:
--------------------------------------------------


Какой смысл писать парсеры, если мы можем только сопоставлять входные данные и проверять их корректность?
Конечно же данные нужно извлекать, для этого и существуют так называемые Parser Actions, подробнее о которых будет
рассказанно в следующем разделе.


## Parser Actions
Действия на парсерах позволяют извлекать данные из сопоставившихся правил, а так же преобразовывать в соответствии с
вашими желаниями и потребностями.

TODO:

Выимка данных
TODO:
  правило push
  правило ~>

Не упоминать про ValueStack


-----------------------------------------------------

  if there's recursive call specify type for expression

  Tell about ValueStack show definition
  типизация value stack проверяется на этапе компиляции
     -> push кладет на стек. null класть не надо :)
     -> capture смотри совпадение и если совпадает кладет в стек
     -> ~> (capture обязателен для этой хрени!!) принимает лямбду - cнимает с value stack что-то делает а потом кладет
     обратно
        пример снятия числа и toInt
        А еще можно делать reduction
        value stack растет справа -> у лямбды порядок соответственный
        ~> позволяет вкладывать парсеры внутрь...
        Показать синтаксис (сахарок) для case classes

  рассказать про дохуя операторов в парбойлде типа ~~> и др теперь ихнет


  Rule0 - не аффектит стек -> отвечает на вопрос да нет
  Rule1 - кладет 1 на стек
  Rule2 - кладет 2 на стек
  RuleN - кладет

  import shapeless._
  def root: RuleN[Seq[String] :: String :: Seq[String] :: String :: HNil] = ???Hlist шейплесовский


  Для пользователей parboiled1.
  У нас была известная проблема с Rule7. Нельзя было вытащить большее число параметров. Поэтому извращались как-то так
    def Event: Rule1[SyslogEvent] = rule {
      Header ~ " " ~ UserData ~ " " ~ Message ~~> {
        (header, data, message) => SyslogEvent (
          header._1, header._2, header._3, header._4, header._5, data._1, data._2, message
        )
      }
    }


  run метод класса Rule -> берет с value stack проверяет нет ничего лишнего

  TODO
  -------------------------
  Показать пример вынимания линейной структуры и генерации AST

-----------------

TODO
Рассказать что для pb1 существует всего один? (ПРОВЕРИТЬ) ParserRunner и нет того зоопарка что было в pb1

## Value Stack
TODO: from the b1 documentation
The value stack is a simple stack construct that serves as temporary storage for custom objects

Syntactic Predicates
Test and TestNot rules never affect the value stack. parboiled always resets the value stack back to a saved
snapshot after a Test or TestNot rule has matched. You can therefore be sure that syntactic predicates will never
“mess” with your value stack setup, even if they contain parser actions or reference other rules that do.

## AST
TODO: from the pb1 documentation
Contrary to the parse tree, which is very closely tied to the grammar by its direct relation to the grammar rules,
the Abstract Syntax Tree you might want to construct for your language heavily depends on your exact project needs.
This is why parboiled takes a very open and flexible approach to supporting it.
There are absolutely no restrictions on the type of your AST nodes. parboiled does provide a number of immutable
and mutable base classes you might choose to use, however, there is nothing that forces you to do so. Take a look at
the org.parboiled.trees package to get started.


# Error reporting
TODO: from the pb1 documentation
The proper handling of illegal input (with regard to the language defined by your grammar) is a key feature of any
parser that is to be used in real-world projects, and it’s one of the big drawbacks of regular expressions. For example,
if you have a user provide input in a custom DSL you can be sure he or she will make syntactic and/or semantic mistakes
at some point. The latter ones will have to be caught and reported by higher levels of your application, however, the
syntactic ones can and should be caught, reported and dealt with directly in the parser.

Для Parboiled1 существует гибкий набор ParserRunnerов с которым можно ознакомиться
[здесь](https://github.com/sirthias/parboiled/wiki/Parse-Error-Handling)

## Восстановление после ошибок
TODO: current parboiled2 documentation
Currently parboiled only ever parses up to the very first parse error in the input. While this is all that's required
for a large number of use cases there are applications that do require the ability to somehow recover from parse
errors and continue parsing. Syntax highlighting in an interactive IDE-like environment is one such example.

Future versions of parboiled might support parse error recovery. If your application would benefit from this feature
please let us know in this github [ticket](https://github.com/sirthias/parboiled2/issues/42).


# Тестирование
TODO: from pb usergroup
Well, seeing that the `TestParserSpec` is a mere 25 lines I have not yet deemed it worthwhile to include it.
Also, it is specific to specs2 (others might prefer scalatest) and not necessary the cleanest solution
(due to it relying on mutability).
But I’ll think about cooking something up that can be included, thanks for the hint!
TODO:


# Недостатки Parboiled2
У любой, даже самой замечательной библиотеки есть свои недостатки, Parboiled2 не является исключением.

 - Длинные, слишком общие и совершенно непонятные сообщения от компилятора. Которые не помогают. Пример приведен на
   рисунке ниже (в правиле отсутствует оператор ~):

   ![Потеряшки ~](https://cloud.githubusercontent.com/assets/934140/5181763/7aa8a570-744e-11e4-9011-fea34b07cdc9.png)

   Это связано с выполнением продвинутых проверок "на типах". Данные проверки
   [обещают](https://github.com/sirthias/parboiled2/issues/106) устранить в следующих версиях.
 - Эта проблема относится больше не к parboiled2, а к scalac. Дело в том что ему сносит крышу, если внутри лямбды,
   захватывающей значения с Value Stack определить типы. Пример ниже не скомпилируется:


    def MyRule = rule { oneOrMore(Visible) ~> (s: String => "[" + s + "]") }
   а вот этот, скомпилируется:

    def MyRule = rule { oneOrMore(Visible) ~> (s: String => "[" + s + "]") }
   пожалуйста, имейте это в виду.
 - Многие IDE еще не научились поддерживтаь Parboiled2. Поэтому Верить подчеркиваниям вашей среды разработки не стоит.
   Сам, забыв об этом, потратил целый день на поиск ошибки в простом подпарсере.
 - Отсутствие механизма восстанолвения при не удачном разборе. Для проектирущих предметно-ориентированные языки, или же
   тех, кто хочет использовать Parboiled2 в качество фронт-энда к своему компилятору это сильно разочарует. Смею вас
   заверить, над этим [работают](https://github.com/sirthias/parboiled2/issues/42).
 - Я думаю, что многим разработчикам своих небольших IDE, текстовых редакторов хотелось бы видеть более гибкие сообщения
   об ошибках, чем те, что предоставляются сейчас. На данный [момент](https://github.com/sirthias/parboiled2/issues/96)
   существует всего два способа повлиять на сообщения об ошибках:
    - Именованное правило
    - Именованное вложенное правило
 - Существуют проблемы с разбором символов, не попадающих в UTF-16. Очень много недовольных китайцев ломится на
   issue tracker.
 - Parboiled не доработан для поддержки лево-рекурсивных грамматик


# Миграция
Данный раздел посвящен миграции. Процесс не сложный, но время занимает. Поэтмоу я постарался хотя бы чуточку сэкономить
ваше время, и описал основные тезисы в разделе:

## Стратегия
Для того чтобы избежать конфликтов с первой версией. Parboiled2 использует другой classpath `org.parboiled2`, тогда
как classpath для первой версии `org.parboiled`. Maven group-id однако остался `org.parboiled`
Благодаря этому можно иметь обе зависимости в одном проекте, и осуществлять постепенный переход на новую версию.

## Parser теперь не trait, а абстрактный класс.
Traits - удобнейшее средство для композиции програмных компонентов. И многие разр


# Что Parboiled (PEG) не может/не умеет
Большинство статей про комбинаторы парсеров начинается с изматывающих объяснений того что такое PEG, с чем его есть и
почему его надо бояться. Для того чтобы парсить конфиги, знать и разбираться в этом не обязательно. Однако знать что
знать ограничения данного типа грамматик стоит. Итак, в виду того что Parboiled является PEG он не умеет:

 - Разбирать грамматики на отступах (Indentation-based grammars), например Python или YAML. Не получается это сделать
   из-за того что сгенерированный парсер является однопроходным (лексер отсутствует). Разбор отступов же выполняется на
   этапе лексического анализа. Однако, для решения данной проблемы существует решение: установка витруальных символов
   (маркеров) до (INDENT) и после (DEDENT) выхода в отступ. Для Parboiled1 сущестует
   [стандарное решение](https://github.com/sirthias/parboiled/wiki/Indentation-Based-Grammars), для Parboiled2 подобную
   процедуру пока придется выполнять самостоятельно
 - Использовать потоковый ввод (Streaming input). PEG используют поиск с возвратом, он же
   [бэктрекинг](https://en.wikipedia.org/wiki/Backtracking). Потенциальное обходное решение буферизация потока,
   выделение чанка в буффер с последующим разбором. После того как чанк будет разобран можно передвигаться далее по
   потоку. Данное решение, возможно, и окажется кому-то полезным.


# Parboiled1 workarounds
## Ограничение на 7 правил.
Данная проблема решается использованием Кортежа, вместо одного Resulting rule.
Вот пример:




# Наиболее часто встречающиеся проблемы

## Квотированные строки
TODO

## (Indentation based grammars)

## Парсер на сайдэффектах
TODO


# Best practices
В этом разделе я раскажу о прописных истинах работающих для любого парсер комбинатора, а так же нюансах, специфичных для
Parboiled.

## Пишите модульные тесты
Банальный совет, которым многие пренебрегают. Парсер не так сложно протестировать, как, скажем IO: Вам не нужны
Mock-объекты и другие ухищрения для этой рутинной, но очень ценной работы. У нас была целая инфраструктура парсеров. И
поверьте, первое что я делал при поиске ошибок - садился и писал тесты, в случае их отсутствия.

## Делайте парсеры маленькими (по возможности)
Разделяйте ваши парсеры, на подпарсеры. Каждый компонент должен делать что-то вполне определенное. Например если вы
парсите LogEvent, у которого опредено поле Timestamp (особенно если этот Timestamp соответствует какому-нибудь Rfc).
Не поленитесь и вынесите его отдельно.

  - Во-первых это уменьшит код вашего основного прасера, и сделает его нагляднее
  - Во-вторых это заметно облегчит тестирование. Вы покроете модульными тестами ваш сабпарсер. А после этого приступите
    к разработке главного парсера

## Делайте правила маленькими
Правила должны быть максимально компактными, но не компактней. Чем меньше ваши правила, тем легче найти ошибку в
грамматике. Это спасало.

## Отправляйте case objects вместо строк в Value stack
Данный совет можно отнести и к оптимизациям, Потому что заставляет парсер работать быстрее.
Отправляйте в Value stack значимые объекты, а не строки. Это сделает ваш парсер быстрее а код нагляднее.

Плохо:

    def logLevel = rule {
      capture("info" | "warning" | "error") ~ ':’
    }

Хорошо:

    def logLevel = rule {
        “info:” ~ push(LogLevel.Info)
      | “warning" ~ push(LogLevel.Warning)
      | “error" ~ push(LogLevel.Error)
    }

## Используйте упрощенный синтаксис для сборки объекта
Этот красивый способ появился еще в Parboiled1. Никакой магии, просот конструктор case classа вызывается неявно.
Главное, чтобы количество и тип аргументов помещаемых на Value Stack совпадали с сигнаторой конструктора case classа.

Плохо:

    def charsAST: Rule1[AST] = rule { capture(Characters) ~> ((s: String) => AText(s)) }

Хорошо:

    def charsAST = rule { capture(Characters) ~> AText }

## Именнованные правила (named rules)
We make pretty extensive use of that
in order to get the parse errors right.
TODO два примера даже три у мнея есть

TODO Ксати рассказать о том что это доступно в Parboiled1

I've considered that, but that wont provide for sharing the same label for multiple rules.
For example, here:
https://github.com/neo4j/neo4j/blob/2.1.2/community/
cypher/cypher-compiler-2.1/src/main/scala/org/neo4j/cypher/internal/compiler/v2_1/parser/Expressions.scala#L34




# Оптимизации
Главное при выполнении оптимизаций - своевременность. Это то с чем не стоит спешить.

## Развертка n.times для n <= 4
Вы можете выиграть в производительности если вместо правила "times" для маленьких n

    rule { 4 times CharPredicate.Digit }

будете использовать правило последовательности "sequence" (оно же тильда)

    import CharPredicate.Digit
    rule { Digit ~ Digit ~ Digit ~ Digit }

Это связано в существующей на момент публикации данной статьи
[проблемой](https://github.com/sirthias/parboiled2/issues/101).


## Ускорение операций со стеком для nTimes
Использование подобной оптимизации при извлечении чисел со стека тоже позволит вам выжать немножко производительности

    def digit4 = rule {
      Digit ~ Digit ~ Digit ~ Digit ~ push(#(charAt(-4))*1000 + #(charAt(-3))*100 + #(charAt(-2))*10 + #(lastChar))
    }


## Используйте ANY, там где хотите видеть CharPredicate.All
TODO:

## CharPredicate работает очень медленно для больших диапазонов
TODO:

## Не пересоздавайте CharPredicate каждый раз
TODO
I currently don’t have the capacity to dig into this deeper but here is a work-around:
Instead of recreating the `CharPredicate` for every rule application by using

   CharPredicate.from(_.isUpper)

directly in the rule you should move it into a `val` of your parser class or, even better, into the companion object.
Alternatively you can use the `test` semantic predicate:

   def JavaUpperCase = rule { oneOrMore(test(currentChar.isUpper) ~ ANY) }



## Используйте инвертирующий предикат
TODO:


# Заключение
TODO:

Использованные источники
========================
 - [Юзергруппа](https://groups.google.com/forum/#!topic/parboiled-user/Ygb_M6XU5P8) посвященная Parboiled2
   Здесь вы можете задать все интересующие вас вопросы. Вам помогут.
 - Презентация Александра Мыльцева [Видео](http://www.youtube.com/watch?v=qZg4D62K4aQ)
 - [Слайды](http://myltsev.name/ScalaDays2014/#/) к презентации Александра [ENG]
 - Официальная [документация](https://github.com/sirthias/parboiled2/blob/master/README.rst)
 - [Примеры](https://github.com/sirthias/parboiled2/tree/master/examples/src/main/scala/org/parboiled2/examples)
   исходного кода
