**Часть 3. Свойства**

В предыдущих частях мы уже успели познакомиться со свойствами и опробовать их
в связке с генераторами. В этом туториале мы рассмотрим свойства подробнее.
Статья состоит из двух частей: первая — техническая, в ней будет рассказано
про комбинаторы свойств, а также другие возможности библиотеки ScalaCheck. Эта
часть будет посвящена различным техникам тестирования.

<cut text="Читать про свойства в ScalaCheck →">

**Структура цикла**

 - [Введение](https://habrahabr.ru/post/319456/)
 - [Генераторы](https://habrahabr.ru/post/320104/)
 - [Свойства](#)

# Комбинаторы свойств
## Константные свойства
В Scalacheck существуют *постоянные* свойства — свойства которые всегда
возвращают один и тот же результат. Примерами таких свойств являются:

 - `Prop.undecided`
 - `Prop.falsified`
 - `Prop.proved`
 - `Prop.passed`
 - `Prop.exception(e: Throwable)`

С методами `Prop.passed` и `Prop.falsified` мы уже знакомы: `Prop.passed` соответствует
удачному прохождению свойством теста для комбинатора `forAll`, а `Prop.falsified`
соответствует неудачному прохождению хотя бы одного теста для свойства с
комбинатором `forAll`. В дополнение к ним:

 * `Prop.exception` возвращается, если внутри вашего свойства ~что-то рвануло~
    выстрелило исключение;
 * `Prop.proved` используется совместно с `Prop.throws` и `Prop.exist`:
   наличие хотя бы одного результата по определению, все-таки ближе к доказательству;
 * `Prop.undecided`, говорит о том что свойство нельзя было ни опровергнуть, ни доказать.

## Комбинации свойств
ScalaCheck позволяет вам вкладывать свойства `forAll`, `throws` и `exists`
произвольным образом. Продемонстрируем это на примере с `forAll`:

    import org.scalacheck.Prop.forAll

    // Угадайте результирующий тип целочисленного сложения.
    val intsum = forAll { x: Int =>
      forAll { y: Int =>
        (x + y).isInstanceOf[Int]
      }
    }

## Prop.throws
Логический метод, возвращающий истину только в случае, если во время
выполнения выражения будет выброшено вполне ожидаемое исключение. Вы
можете использовать свойства следующим образом:

    import org.scalacheck.Prop

    // Простейший случай:
    val p0 = Prop.throws(classOf[ArithmeticException])(3 / 0)

    p0.check
    // + OK, proved property.

Однако, особого смысла тестировать константы нет. Проверим Prop.throws при
делении на 0 произвольного целого числа:

    val p = Prop.forAll { x: Int =>
      Prop.throws(classOf[ArithmeticException]) (x / 0)
    }

    p.check
    // + OK, passed 100 tests.

## Prop.forAll
Называемый в логике *универсальным квантификатором*, является также наиболее
часто используемым нами свойством. Условие, переданное в `forAll`, должно быть
или `Boolean`, или же являться экземпляром класса `Prop`.

> Следует понимать, что при тестировании заданного свойства, библиотека не
> может проверить на истинность все допустимые значения. Поэтому, зачастую,
> она удовлетворяется некоторым описанным в настройках числом.
> По-умолчанию это число равняется 100. Вы можете его изменить, вручную
> сконфигурировав свойство. Подробнее о конфигурации вы узнаете в следующих
> следующих статьях серии.

## Prop.exists
Ведет себя точь-в-точь как *квантор существования*. Поведение во многом схоже
с `forAll`, за исключением того, что в случае с данным комбинатором, свойство
засчитывается, если хотя бы один элемент множества входных данных удовлетворяет
заданному условию. На практике использование `Prop.exist` является
проблематичным в виду того, что может быть достаточно сложно найти случай, удовлетворяющий
заданному условию:

    import org.scalacheck.Prop

    val p1 = Prop.exists { x: Int =>
      (x % 2 == 0) && (x > 0)
    }

При вызове `p1.check` ScalaCheck выведет нам следующее:

    scala> p1.check
    + OK, proved property.
    > ARG_0: 73115928

А теперь попробуем попросить у ScalaCheck невозможного:

    val p2 = Prop.exists(posNum[Int]) { x: Int =>
      (x % 2 == 0) && (x < 0)
    }

Как только ScalaCheck найдет первый устраивающий нас элемент, он сообщит о том, что
свойство *доказано* (proved), а не *протестированно* (passed).

    scala> p2.check
    ! Gave up after only 0 passed tests. 501 tests were discarded.

Наличие `Prop.exists` определенно решает чьи-то проблемы. В моей практике
это свойство использовать не приходилось.


## Именование свойств
Именование является хорошей практикой как для генераторов, так и для
свойств. При именовании свойств используются те же операторы, что и для
генераторов: в качестве имени свойства может использоваться строка либо
символ. Используются операторы `:|` и `|:`.

  // Оператор |: используется, если имя идет до свойства.
  'linked |: isLinkedProp

  // Оператор :| используется, если имя идет после свойства.
  isComplete :| "is complete property


# Логические операторы
Свойства представляют собой логические выражения. В ScalaCheck вы можете
использовать логические операторы применительно к свойствам. Внутри `Prop`
объявлены операторы `&&` и `||`, поведение которых в точности совпадает с
одноименными операторами класса `Boolean`. В дополнение к названным выше
операторам, существуют синонимы с символьными именами: `Prop.all` и
`Prop.atLeastOne`.

Использование логических операторов позволяет собирать сложные свойства из
более простых. Более того, вы также можете объединять экземпляры `Prop` и
переменные логического типа в  одном выражении: для этого вам необходимо явным
образом добавить в область видимости `Prop.propBoolean`, так как это один из
тех случаев, когда компилятор Scala не может автоматически выполнить
приведение типов. Если же вы хотите выполнить преобразование явно, вы можете
поступить следующим образом:

    // January has April showers and...
    val prop = Prop.propBoolean(2 + 2 == 5)

Итак, рассмотрим пример, для списка и метода `reversed`:

    // Для начала давайте определимся, что значит reversed.
    def elementsAreReversed(list: List[Int], reversed: List[Int]): Boolean =
      // А означает это то, что для непустого списка...
      if (list.isEmpty) true else {
        val lastIdx = list.size - 1

        // ... на равноотстоящих позициях с разных сторон списка
		// находятся одинаковые элементы.
        list.zipWithIndex.forall { case (element, index) =>
          element == reversed(lastIdx - index)
        }
      }

Этот метод замечательно описывает основное свойство метода `reversed` и его
вполне достаточно. Однако, нашей задачей сейчас является не четкая
формулировка свойства, а демонстрация возможностей ScalaCheck. Поэтому
притянем за уши еще пару свойств, которые неявным образом выражены в
`elementsAreReversed`:

    val hasSameSize    = reversed.size == list.size
    val hasAllElements = list.forall(reversed.contains)

Эти свойства являются булевыми значениями. Добавление метки (при наличии
`propBoolean` в области видимости) автоматически сконвертирует наши переменные
к типу `Prop`. Теперь давайте опишем наше первое составное свойство
*и заодно воспользуемся метками*:

    val propReversed = forAll { list: List[Int] =>
      val reversed = list.reverse

      if (list.isEmpty)
      // Неявным образом, конвертируем к Prop.propBoolean добавляя метку
        (list == reversed) :| "Пустые списки для заданного типа равны"
      else {
        val hasSameSize    = reversed.size == list.size
        val hasAllElements = list.forall(reversed.contains)

        hasSameSize :| "имеют одинаковый размер" &&
        hasAllElements :| "содержат все элементы друг друга" &&
        ("В эту сторону тоже можно" |: elementsAreReversed(list, reversed))
      }
    }


# Когда получили не то что хотели
Хотели бы вы в случае ошибки видеть какое из значений мы имеем, а какое
ожидали? ScalaCheck дает вам такую возможность: всего-лишь следует заменить
тривиальное равенство `==` на операторы `?=` или `=?`. Как только вы это
сделаете, ScalaCheck запомнит обе части выражения при выполнении данного
свойства, и в случае, если свойство окажется неверным, вам будут
представлены оба значения:

    ! Falsified after 0 passed tests.
    > Labels of failing property:
    Expected 4 but got 5
    > ARG_0: "

Для того чтобы воспользоваться операторами `?=` и `=?`, вам необходимо
добавить `Prop.AnyOperators` в область видимости:

    import org.scalacheck.Prop.{AnyOperators, forAll}

    val propConcat = forAll { s: String =>
      2 + 2 =? 5
    }

*Актуальным*, будет значение располагающееся ближе к знаку `?`, *Ожидаемым*,
будет значение, ближайшее к знаку равенства.

Вы также можете проинтегрироваться cо ScalaTest и использовать прилагающиеся
к нему матчеры для того чтобы получать читаемые сообщения об ошибках.
Подробнее об этом будет рассказано в разделе «Интеграция и настройки».

# Собираем статистику
## classify
Даже если ваши тесты вполне себе успешны и привлекательны, статистика по
сгенерированным входным данным может все-равно представлять интерес.

Даже если все ваши тесты выполняются успешно и все хорошо, возможно,
вы захотите получить информацию, которая была использована при тестировании.
Например, если у вас есть нетривиальные предусловия для метода, и вы
точно хотите знать насколько жестко ScalaCheck выбирает входные данные.
Так что если вам нужна статистика, `Prop.classify` к вашим услугам:

    import org.scalacheck.Prop.{forAll, classify}

    val classifiedProperty = forAll { n: Double =>
      // classify принимает одно или два именования,
      // по которым и выполняет классификацию.
      classify(n < 0, "negative", "positive") {
        classify(n % 2 == 0, "even", "odd") {
          n == n
        }
      }
    }

Вы можете добавить столько классификаторов, сколько сочтете нужным, ScalaCheck
сольет их воедино и представит в виде распределения:

    + OK, passed 100 tests.
    > Collected test data:
    33% odd, negative
    31% even, negative
    18% odd, positive
    18% even, positive


## collect

В дополнение к `classify` существует более обобщенный метод для сбора и
статистики: метод `Prop.collect` собирает любую интересную вам статистику и
группирует ее под наиболее удобным для вас именем:

    collect(label)(boolean || prop)

Кстати, имя может быть любого типа: `toString` будет вызван автоматически.
Рассмотрим простейший пример:

    val moreLessAndZero = Prop.forAll { n: Int =>
      val label = {
        if (n == 0) 0 // Я же говорил, что вызовется toString.
        else if (n > 0) "> 0"
        else "< 0"
      }
      collect(label)(true) // А тут сработает имплисит.
    }

После вызова метода `check` имеем:

    + OK, passed 100 tests.
    > Collected test data:
    46% < 0
    45% > 0
    9% 0


# Эталонная реализация
Итак, представьте, вы пишете очередную реализацию списка. Ваша реализация
определенно лучше других (по-крайней мере, автор на это рассчитывает). И вот
пришла пора ваш список тестировать.

> Может существовать множество причин, способных побудить на реализацию того,
> что уже имеется в стандартной библиотеке, и это вовсе не обязательно жажда познания
> или саморазвития.

Было бы очень здорово каким-либо образом протестировать ваш список
используя, например, уже реализованный в JDK класс `ConcurrentHashMap`. И вы можете
это сделать: вместо того чтобы создавать спецификацию, содержащую набор жестких
условий и контрактов, вы можете задать спецификацию неявно, используя уже
известную работающую реализацию ([эталонную](reference-implementation-ru)).
Данный подход широко используется в тестировании. В англоязычных источниках,
вы можете найти его под названием
[*reference implementation*](reference-implementation-en).

    import org.scalacheck.Prop.AnyOperators
    import org.scalacheck.Properties

    // Предположим, у нас уже есть генератор для нашего
    // замечательного списка и эталонной реализации.
    def listsGen: Gen[(List, MyList)] = ???

    object MyListSpec extends Propertes("My Awesome List") {
      property("size") = Prop.forAll(listsGen) { case (list, myList) =>
        list.size =? myList.size
      }

      property("is empty") = Prop.forAll(listsGen) { case (list, myList) =>
        list.isEmpty =? myList.isEmpty
      }
    }

[reference-implementation-en]: https://en.wikipedia.org/wiki/Reference_implementation
[reference-implementation-ru]: https://ru.wikipedia.org/wiki/%D0%AD%D1%82%D0%B0%D0%BB%D0%BE%D0%BD%D0%BD%D0%B0%D1%8F_%D1%80%D0%B5%D0%B0%D0%BB%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F


# Симметричные свойства
Также именуемые *round-trip properties*. Проще не придумать: мы берем некую
*обратимую* функцию и применяем ее два раза, тем самым тестируя ее на
обратимость:

    def neg(a: Long) = -a

    val negatedNegation = forAll { n: Long =>
      neg(neg(n)) == n
    }

Данное свойство не описывает метод `neg` полностью, как и не говорит о
его функциональности. Однако, оно говорит о его обратимости.

Возможно, данный пример, как и пресловутый `List.reverse`, который вы можете
найти ни в одном десятке туториалов, покажется вам примитивным. Однако,
существуют более сложные системы, для которых данный подход применим: парсеры
и кодировщики всех сортов и мастей. Например, при использовании симметричных
свойств для тестирования парсеров, вы можете найти ошибки в весьма
труднодоступных местах.

Метод, представленный ниже, разбирает текст и создает на его основе абстрактное
синтаксическое дерево (AST), а метод `prettyPrint` конвертирует это дерево
обратно в текст:

    // Определенно, где-то в глубине нашего кода,
    // AST обязательно будет определено так.
    sealed trait AST = ...

    // Метод выполняющий синтаксический разбор.
    def parse(s: String): AST = ...

    // Генератор, порождающий синтаксические деревья.
    val astGen: Gen[AST] = ...

    // Метод, транслирующий AST в строку. Можно воспользоваться
	// вариативностью в генерации пробелов и табуляций:
	// ScalaCheck вам в этом поможет.
    def pretty(ast: AST): String = ...

    // И наконец, самый простой элемент системы, позволяющий
    // протестировать наш парсер.
    val prop = forAll(astGen) { ast =>
      parse(pretty(ast)) == ast
    }


# В заключение
В большинстве туториалов по ScalaCheck вас сразу же познакомят со свойствами
и их надуманной классификацией, а потом будут рассказывать о сложности
вычленения свойств из уже написанного кода. Отчасти это верно: без должной
тренировки достаточно непросто выделить свойства, которые можно протестировать.

Однако, по своему скромному опыту использования ScalaCheck, самым сложным,
все-таки является составление генераторов. Этот процесс требует больше
усилий, нежели написание свойств: для написания одного свойства может
потребоваться написание десятка-другого генераторов. Именно поэтому я начал
рассказ с генераторов, что многим, возможно могло показаться странным.
