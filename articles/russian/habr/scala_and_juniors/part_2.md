**Часть 2. Обо всем и ни о чем**

Сегодня мы обсудим ряд скалических идиом, которые не поместились в [первую часть](https://habrahabr.ru/post/323706/) статьи. Мы рассмотрим вопросы интероперации языка с Java и, главное, неправильное использование объектно-ориентированных особенностей Scala.

<cut text="Ну что, поехали? →">


**Структура цикла**

 - [Часть 1. Функциональная](https://habrahabr.ru/post/323706/)
 - [Часть 2. Обо всем и ни о чем](#)


## Длина выражений
В Scala практически все является выражением, и даже если что-то возвращает `Unit`, вы всегда можете получить на выходе ваш `()`. После длительного программирования на языках, где превалируют операторы (statements), у многих из нас (и я не являюсь исключением) возникает желание запихнуть все вычисления в одно выражение, составив из них длинный-предлинный паровозик. Следующий пример я нагло утащил из Effective Scala. Допустим, у нас есть последовательность кортежей:

    val votes = Seq(("scala", 1),
                    ("java", 4),
                    ("scala", 10),
                    ("scala", 1),
                    ("python", 10))

Мы можем лихо её обработать (разбить на группы, просуммировать внутри групп, упорядочить по убыванию) единственным выражением:

    val orderedVotes = votes
      .groupBy(_._1)
      .map { case (which, counts) =>
        (which, counts.foldLeft(0)(_ + _._2))
      }.toSeq
      .sortBy(_._2)
      .reverse

Этот код прост, понятен, выразителен? Возможно — для съевшего собаку скалиста. Однако, если мы разобьём выражение  на именованные составляющие, легче станет всем:

    val votesByLang =
      votes groupBy { case (lang, _) => lang }

    val sumByLang =
      votesByLang map { case (lang, counts) =>
        val countsOnly = counts map { case (_, count) => count }
        (lang, countsOnly.sum)
      }

    val orderedVotes = sumByLang.toSeq
      .sortBy { case (_, count) => count }
      .reverse

> Наверное, этот пример недостаточно нагляден — чего уж, я даже поленился его сам придумать. Но поверьте, мне попадались очень длинные конструкции, которые их авторы даже не удосуживались переносить на несколько строк.

Очень часто в Scala приходят через Spark, а уж используя Spark так и хочется сцепить побольше «вагончиков»-преобразований в длинный и выразительный «поезд». Читать такие выражения сложно, их нить повествования теряется достаточно быстро.

**Сверхдлинные выражения и operator notation**

Надеюсь, всем известно, что `2 + 2` в Scala является синтаксическим сахаром для выражения `2.+(2)`. Этот вид записи именуется операторной нотацией (operator notation). Благодаря ей в языке нет операторов как таковых, а есть лишь методы, пусть и с небуквенными именами, а сама она -- мощный инструмент, позволяющий создавать выразительные DSL (собственно, для этого символьная нотация и была добавлена в язык). Вы можете сколь угодно долго записывать вызовы методов без точек и скобочек: `object fun arg fun1 arg1`. Это безумно круто, если вы хотите сделать читаемый DSL:

    myList should have length 10

Но, в большинстве случаев, операторная нотация в сочетании с длинными выражениями приносит сплошные неудобства: да, операции над коллекциями без скобок выглядят круче, вот только *понять* их можно только тогда, когда они разбиты на именованные составляющие.

**«Поезда» и postfix notation**

Постфиксные операторы, при определенных условиях, способны вскружить голову несчастному парсеру, поэтому в последних версиях Scala эти выражения нужно импортировать явно:

    import language.postfixOps

Старайтесь не использовать эту возможность языка и проектировать ваши DSL так, чтобы и вашим пользователям не приходилось ее использовать. Это довольно просто сделать.


## Неинициализируемые значения
Scala поддерживает неинициализированные значения. Например, это может вам пригодиться при создании beans. Давайте посмотрим на следующий Java-класс:

    class MyClass {
        // По-умолчанию, любой наследник Object инциализируется в null.
        // Примитивные типы инициализируются значениями по-умолчанию.
        String uninitialized;
    }

Такого же поведения мы можем добиться и от Scala:

    class MyClass {
      // Синтаксис с нижним подчеркиванием говорит Scala, что
      // данное поле не будет инциализировано.
      var uninitialized: String = _
    }

Пожалуйста, *не делайте этого бездумно*. Инициализируйте значения везде, где можете. Используйте эту языковую конструкцию только если используемый вами фреймворк или библиотека яростно на этом настаивают. При неаккуратном использовании вы можете получить тонны `NullPointerException`. Однако знать об этой функции следует: однажды подобное знание сэкономит время. Если вы хотите отложить инициализацию, используйте ключевое слово `lazy`.

## Никогда не используйте null

  - Всегда инициализируйте значения.
  - Оборачивайте `Nullable`, которые могут прийти извне в `Option`.
  - Не возвращайте `null`: используйте `Option`, `Either`, `Try` и др.
  - Видите предпосылки для появления null -- быстрее исправляйте, пока ваши коллеги на радостях не завезли в проект  *специально предназначенный для борьбы с NPE язык*.

Иногда встречаются ситуации, когда null-значения являются частью модели. Возможно, эта ситуация возникла ещё задолго до вашего прихода в команду, а уж тем более задолго до внедрения Scala. Как говорится: если пьянку нельзя предотвратить, ее следует возглавить. И в этом вам поможет паттерн, именуемый Null Object. Зачастую это всего-лишь еще один case-класс в ADT:

    sealed trait User
    case class Admin extends User
    case class SuperUser extends User
    case class NullUser extends User

Что мы получаем? Null, пользователя и типобезопасность.


## О перегрузках

**Методы**

В Scala существует возможность перегрузки конструкторов классов. И это — не лучший способ решить проблему. Скажу больше, это — *не идиоматичный* способ решения проблемы. Если говорить о практике, эта функция полезна, если вы используете Java-reflection и ваш Scala-код вызывается из Java или вам необходимо такое поведение (а почему бы в таком случае не сделать Builder)? В остальных случаях лучшая стратегия — создание объекта-компаньона и определение в нем нескольких методов `apply`.

Наиболее примечательны случаи перегрузки конструкторов из-за незнания о [параметрах по-умолчанию](http://docs.scala-lang.org/tutorials/tour/default-parameter-values.html) (default parameters).

Совсем недавно, я стал свидетелем следующего безобразия:

    // Все включено!
    case class Monster (pos: Position, health: Int, weapon: Weapon) {
      def this(pos: Position) = this(pos, 100, new Claws)
      def this(pos: Position, weapon: Weapon) = this(pos, 100, weapon)
    }

Ларчик открывается проще:

    case class Monster(
      pos: Position,
      health: Short = 100,
      weapon: Weapon = new Claws
    )

Хотите наградить вашего монстра базукой? Да, не проблема:

    val aMonster = Monster(Position(300, 300, 20), weapon = new Bazooka)

Мы сделали мир лучше, монстра — миролюбивее, а заодно перестали перегружать все, что движется. Миролюбивее? Определенно. Ведь базука — это еще и музыкальный инструмент (Википедия об этом умалчивать не станет).

Сказанное относится не только к конструкторам: люди часто перегружают и обычные методы (там где этого можно было избежать).

**Перегрузка операторов**

Считается достаточно противоречивой фичей Scala. Когда я только-только погрузился в язык, перегрузка операторов использовалась повсюду, всеми и везде, где только можно. Сейчас эта фича стала менее популярна. Изначально перегрузка операторов была сделана, в первую очередь, для того, чтобы иметь возможность составлять DSL, как в Parboiled, или роутинг для akka-http.

Не перегружайте операторы без надобности, и если считаете, что эта надобность у вас есть, то все-равно не перегружайте.

А если перегружаете (вам нужен DSL или ваша библиотека делает нечто математическое (или трудновыразимое словами)), обязательно дублируйте оператор функцией с нормальным именем. И думайте о последствиях. Так, Благодаря scalaz оператор `|@|` ([Applicative Builder](http://eed3si9n.com/learning-scalaz/Applicative+Builder.html)) получил имя Maculay Culkin. А вот и фотография "виновника":

![Шокированный Maculay Culkin](https://habrastorage.org/web/aac/c6b/771/aacc6b771744466d8f78d6592c6165ff.jpg)

Безусловно, после того, как вы многократно перегрузите конструкторы, вам для довершения картины захочется налепить геттеров и сеттеров.


## О геттерах и сеттерах
Scala предоставляет отличное взаимодействие с Java. Она также способна облегчить вам жизнь при дизайне так называемых Beans. Если вы не знакомы с Java или концепцией Beans, возможно, вам следует с ней [ознакомиться](https://en.wikipedia.org/wiki/JavaBeans).

Слышали ли вы о [Project Lombok](https://projectlombok.org/)? В стандартной библиотеке Scala имеется схожий механизм. Он именуется `BeanProperty`. Все, что вам нужно, — создать bean и добавить аннотацию `BeanProperty` к каждому полю, для которого хотите создать getter или setter.

> Для того чтобы получить имя вида `isProperty` для переменных булева типа, следует добавить `scala.beans.BooleanBeanProperty` в вашу область видимости.

Аннотацию `@BeanProperty` можно так же использовать и для полей класса:

    import scala.beans.{BooleanBeanProperty, BeanProperty}

    class MotherInLaw {
      // По закону, она может сменить имя:
      @BeanProperty var name = "Megaera"

      // А эти ребята имеют свойство плодиться.
      @BeanProperty var numberOfCatsSheHas = 0

      // Но некоторые вещи неизменны.
      @BooleanBeanProperty val jealous = true
    }

Для case class-ов тоже работает:

    import scala.beans.BeanProperty

    case class Dino(@BeanProperty name: String,
                    @BeanProperty var age: Int)

Поиграем с нашим динозавром:

    // Начнем с того, что он не так стар как вы думаете
    val barney = Dino("Barney", 29)

    barney.setAge(30)

    barney.getAge
    // res4: Int = 30

    barney.getName
    // res14: String = Barney

В виду того, что мы не сделали name переменной, при попытке использовать сеттер, мы получим следующее:

    barney.setName
    <console>:15: error: value setName is not a member of Dino
         barney.setName


## Кстати о case-классах
Появление case-классов — это прорыв для JVM-платформы. В чем же их основное преимущество? Правильно, в их неизменяемости (immutability), а также наличию готовых `equals`, `toString` и `hashCode`. Однако, зачастую и в них можно встретить подобное:

    // Внимание на var.
    case class Person(var name: String, var age: Int)

> Иногда case-классы приходится-таки делать изменяемыми: например, если вы, имитируете beans, как в примере выше.

Но зачастую подобное случается тогда, когда глубокий джуниор не понимает, что такое имутабельность. С разработчиками уровнем повыше бывает не менее интересно, ведь они прекрасно осознают что делают:

    case class Person (name: String, age: Int) {
      def updatedAge(newAge: Int) = Person(name, newAge)
      def updatedName(newName: String) = Person(newName, age)
    }

Однако про метод `copy` знают не все. Это норма. Подобное мне доводилось наблюдать не раз, чего уж там, в свое время я сам так хулиганил. Работает `copy` аналогично своему тезке, который определен для кортежей:

    // Обновили возраст, получили новый инстанс.
    person.copy(age = 32)


## О размерах case-классов
Иногда case-классы имеют свойство раздуваться до 15–20 полей. До появления Scala 2.11 этот процесс хоть как-то ограничивался 22 элементами. Но сейчас ваши руки развязаны:

    case class AlphabetStat (
      a: Int, b: Int,
      c: Int, d: Int,
      e: Int, f: Int,
      g: Int, h: Int,
      i: Int, j: Int,
      k: Int, l: Int,
      m: Int, n: Int,
      o: Int, p: Int,
      q: Int, r: Int,
      s: Int, t: Int,
      u: Int, v: Int,
      w: Int, x: Int,
      y: Int, z: Int
    )

> Хорошо, я вам наврал: руки, конечно, стали свободнее, однако ограничения JVM никто не отменял.

Большие case-классы это плохо. Это очень плохо. Бывают ситуации, когда этому есть оправдание: предметная область, в которой вы работаете, не допускает агрегации, и структура представляется плоской; вы работаете с API, спроектированным глубокими идиотами, которые сидят на сильнодействующих транквилизаторах.

И знаете, чаще всего приходится иметь дело со вторым вариантом. Хочется, чтобы case-классы легко и непринужденно укладывались на API. И я вас пойму, если так.

Но я перечислил только уважительные оправдания монструозности ваших case-классов. Есть и наиболее очевидная: для того, чтобы обновить поле, глубоко запрятанного вглубь вложенных классов, приходится очень сильно помучиться. Каждый case-класс надо старательно разобрать, подменить значение и собрать. И от этого недуга есть средство: вы можете использовать линзы (lenses).

Почему линзы называются линзами? Потому что они способны сфокусироваться на главном. Вы фокусируете линзу на определенную часть структуры, и получаете ее, вместе с возможностью ее (структуру) обновить. Для начала объявим наши case-классы:

    case class Address(street: String,
                       city: String,
                       postcode: String)

    case class Person(name: String, age: Int, address: Address)

А теперь заполним их данными:

    val person = Person("Joe Grey", 37,
                        Address("Southover Street",
                                "Brighton", "BN2 9UA"))

Создаем линзу для улицы (предположим что наш персонаж захотел переехать):

    import shapeless._

    val streetLens = lens[Person].address.street

Читаем поле (прошу заметить что строковый тип будет выведен автоматически):

    val street = streetLens.get(person)
    // "Southover Street"

Обновляем значение поля:

    val person1 = streetLens.set(person)("Montpelier Road")
    // person1.address.street == "Montpelier Road"

> Пример был нагло украден «из [отсюда](https://github.com/milessabin/shapeless/blob/master/examples/src/main/scala/shapeless/examples/optics.scala)»


Аналогичную операцию вы можете совершить и над адресом. Как видите, это достаточно просто. К сожалению, а может быть и к счастью, Scala не имеет встроенных линз. Поэтому вам придется использовать стороннюю библиотеку. Я бы рекомендовал вам использовать `shapeless`. Собственно, приведенный выше пример, был написан с помощью этой весьма доступной для начинающего скалиста библиотеки.

Существует множество других реализаций линз, если хотите, вы можете использовать [scalaz](http://eed3si9n.com/learning-scalaz/Lens.html), [monocle](https://github.com/julien-truffaut/Monocle). Последняя предоставляет более продвинутые механизмы использования оптики, и я бы рекомендовал ее к дальнейшему использованию.

К сожалению, для того, чтобы описать и объяснить механизм действия линз, может потребоваться отдельная статья, поэтому считаю, что вышеизложенной информации достаточно для того, чтобы начать собственное исследование оптических систем.


## Переизобретение enum'ов
Берем опытного Java-разработчика и заставляем его писать на Scala. Не проходит и пары дней, как он отчаянно начинает искать enum'ы. Не находит их и расстраивается: в Scala нет ключевого слова `enum` или, хотя бы, `enumeration`. Далее есть два варианта событий: или он нагуглит идиоматичное решение, или начнет изобретать свои перечисления. Часто лень побеждает, и в результате мы видем вот это:

    object Weekdays {
      val MONDAY = 0
      // догадайтесь что будет дальше...
    }

А дальше то что? А вот что:

    if (weekday == Weekdays.Friday) {
       stop(wearing, Tie)
    }

Что не так? В Scala есть идиоматичный способ создания перечислений, именуется он ADT (Algebraic Data Types), по-русски *алгебраические типы данных*. Используется, например в Haskell. Вот как он выглядит:

    sealed trait TrafficLight
    case object Green extends TrafficLight
    case object Yellow extends TrafficLight
    case object Red extends TrafficLight
    case object Broken extends TrafficLight

Многословно, самодельное перечисление, конечно, было короче. Зачем столько писать? Давайте объявим следующую функцию:

    def tellWhatTheLightIs(tl: TrafficLight): Unit = tl match {
      case Red => println("No cars go!")
      case Green => println("Don't stop me now!")
      case Yellow => println("Ooohhh you better stop!")
    }

И получим:

    warning: match may not be exhaustive.
    It would fail on the following input: Broken
           def tellWhatTheLightIs(tl: TrafficLight): Unit = tl match {
                                                            ^
    tellWhatTheLightIs: (tl: TrafficLight)Unit

Мы получаем перечисление, свободное от привязки к каким либо константам, а также проверку на полноту сопоставления с образцом. И да, если вы используете «enum для бедных», как их обозвал один мой небезызвестный коллега, используйте сопоставление с образцом. Это наиболее идиоматичный способ. Стоит заметить, об этом упоминается в начале книги [Programming in Scala](https://www.scala-lang.org/documentation/books.html). Не каждая птица долетит до середины Днепра, так же как и не каждый скалист прочтет *Magnum Opus*.

Неплохо про алгебраические типы данных рассказано, как ни странно, в [Википедии](https://en.wikipedia.org/wiki/Algebraic_data_type). Касательно Scala, есть достаточно доступный [пост](https://gleichmann.wordpress.com/2011/01/30/functional-scala-algebraic-datatypes-enumerated-types) и [презентация](http://tpolecat.github.io/presentations/algebraic_types.html), которая, возможно, покажется вам интересной.


## Избегайте булевых аргументов в сигнатурах функций
Признайтесь, вы писали методы, которые в качестве аргумента принимают Boolean? В случае с Java ситуация вообще катастрофичная:

    PrettyPrinter.print(text, 1, true)

Что может означать 1? Доверимся интуиции и предположим что это количество копий. А за что отвечает `true`? Это может быть что угодно. Ладно, сдаюсь, схожу в исходники и посмотрю, что это.

В Scala вы можете использовать ADT:

    def print(text: String, copies: Int, wrapWords: WordWrap)

Даже если вам достался в наследство код, требующий логических аргументов, вы можете использовать параметры по умолчанию.

    // К интам тоже применимо,
    // а вдруг это не количество копий, а отступы?
    PrettyPrinter.print(text, copies = 1, WordWrap.Enabled)


## Об итерации
### Рекурсия лучше
Хвостовая рекурсия работает быстрее, чем большинство циклов. Если она, конечно, хвостовая. Для уверенности используйте аннотацию `@tailrec`. Ситуации бывают разными, не всегда рекурсивное решение оказывается простым доступным и понятным, тогда используйте `while`. В этом нет ничего зазорного. Более того, вся библиотека коллекций написана на простых циклах с предусловиями.


### For comprehensions не для итерации (по индексам)
Главное, что вам следует знать про генераторы списков, или, как их еще Называют, «for comprehensions», — это то, что основное их предназначение — не в реализации циклов.

Более того, использование этой конструкции для итерации по индексам будет достаточно дорогостоящей процедурой. Цикл `while` или использование хвостовой рекурсии — намного дешевле. И нагляднее.

«For comprehension» представляет собой синтаксический сахар для методов `map`, `flatMap` и `withFilter`. Ключевое слово `yield` используется для последующей агрегации значений в результирующей структуре. Используя «for comprehension» вы, на самом деле, используете те же комбинаторы, просто в завуалированой форме. Используйте их напрямую:

    // хорошо
    1 to 10 foreach println

    // плохо
    for (i <- 1 to 10) println(i)

Помимо того, что вы вызвали тот же код, вы еще и добавили некую переменную `i`, которой совершенно здесь не место. Если вам нужна скорость, используйте цикл `while`.


**Об именах переменных**

Занимательная история имен переменных в формате вопрос-ответ:

**Вопрос**: Откуда вообще взялись `i`, `j`, `k` в качестве параметров циклов?
**Ответ**: Из математики. А в программирование они попали благодаря фортрану, в котором тип переменной определяется ее именем: если первая буква имени начинается с I, J, K, L, M, или N, это автоматически означает, что переменная принадлежит к целочисленному типу. В противном случае, переменная будет считаться вещественной (можно использовать директиву `IMPLICIT` для того, чтобы изменить тип, устанавливаемый по умолчанию).

И этот кошмар живет с нами вот уже почти 60 лет. Если вы не перемножаете матрицы, то использованию `i`, `j` и `k` даже в Java нет оправдания. Используйте `index`, `row`, `column` -- все что угодно. А вот если вы пишете на Scala, старайтесь вообще избегать итерации с переменными внутри `for`. От лукваого это.

Дополнением к этому разделу будет [видео](https://www.youtube.com/watch?v=WDaw2yXAa50), в котором подробно рассказывается все, что вы хотели знать про генераторы списков.


## О выражениях
### Не используйте return
В Scala почти все является выражением. Исключение составляет `return`, который не следует использовать ни при каких обстоятельствах. Это не опциональное слово, как думают многие. Это конструкция которая меняет семантику программы. Подробнее об этом можете прочитать [здесь](https://tpolecat.github.io/2014/05/09/return.html)

### Не используйте метки
Представьте себе, в Scala есть [метки](http://www.scala-lang.org/api/current/scala/util/control/Breaks$.html). И я понятия не имею, зачем они туда были добавлены. До выхода Scala 2.8 по данному адресу располагалась еще и метка `continue`, позже она была устранена.

> К счастью, метки не являются частью языка, а реализованы при помощи выбрасывания и отлова исключений (о том, что делать с исключениями, мы поговорим далее в этой статье).

В своей практике я не встречал еще ни единого случая, когда подобное поведение могло хоть как-то быть оправдано. Большая часть примеров, которые я нахожу в сети, притянуты за уши и высосаны из пальца. Нет, ну вы посмотрите:

Этот пример взят [отсюда](http://alvinalexander.com/scala/scala-break-continue-examples-breakable):

    breakable {
      for (i <- 1 to 10) {
        println(i)
        if (i > 4) break  // выскочить из цикла.
       }
     }


## Об исключительных ситуациях
Это исключительная тема заслуживает большой и отдельной статьи. Возможно, даже серии статей. Рассказывать об этом можно долго. Во-первых, потому, что Scala поддерживает несколько принципиально разных подходов к обработке исключений. В зависимости от ситуации, вы можете выбрать тот, который подходит лучше всего.

В Scala нет checked exceptions. Так что если где-то у нас исключения и могут возникнуть -- обязательно их обрабатывайте. И самое главное, не бросайтесь исключениями. Да, есть ситуации когда зеленые монстры вынуждают вас это делать. И более того, у зеленых монстров и их почитателей это вообще является нормой. Во всех остальных случаях -- не бросайтесь исключениями. То, что в свое время Джоэл Спольски [писал](https://www.joelonsoftware.com/2003/10/13/13/) применительно к C++ и Java, к Scala применимо даже в большей степени. И в первую очередь именно из-за ее функциональной природы. Метки и `goto` недопустимы в функциональном программировании. Исключения ведут себя схожим образом. Бросив исключение, вы прерываете flow. Но, как уже было сказано выше, ситуации бывают разными, и если ваш фреймворк этого требует -- Scala дает такую возможность.

Вместо того, чтобы возбуждать исключения, вы можете использовать `Validation` из scalaz, `scala.util.Try`, `Either`. Можно использовать и `Option`, если вам не жалко людей, которые будут поддерживать ваш код. И это будет все-равно лучше, чем бросаться исключениями.

## Структурные типы
Просто не используйте их. Даже если ваши руки к ним тянутся, вам, скорее всего, они не нужны. Во-первых, структурные типы работают через рефлексию, что достаточно дорого с точки зрения производительности, во вторых -- вы их не контролируете. Интересные мысли изложены на этот счет [здесь](http://www.draconianoverlord.com/2011/10/04/why-no-one-uses-scala-structural-typing.html).


## ? extends App
Знаете что не так с этим кодом:

    object Main extends App {
      Console.println("Hello World: " + (args mkString ", "))
    }

Конкретно с этим примером «все так». Все будет хорошо и прекрасно работать, пока вы не усложните код достаточно, чтобы встретиться с непредсказуемым поведением. И виной тому один трейт из стандартной библиотеки. Он называется `DelayedInit`. Прочитать о нем вы можете [здесь](http://www.scala-lang.org/api/current/scala/DelayedInit.html). Трейт `App`, который вам предлагается расширить в большинстве руководств, расширяет трейт `DelayedInit`. Подробнее об `App` в [документации](http://www.scala-lang.org/api/current/scala/App.html).

> It should be noted that this trait is implemented using the DelayedInit functionality, which means that fields of the object will not have been initialized before the main method has been executed.

По-русски:

> Следует учесть, что данный трейт реализован с использованием функциональности DelayedInit, что означает то, что поля объекта не будут проинициализированны до выполнения метода main.

В будущем это обещают исправить:

> Future versions of this trait will no longer extend DelayedInit.

Плохо ли использовать `App`? В сложных многопоточных приложениях я бы не стал этого делать. А если вы пишете «Hello world»? Почему бы нет. Я стараюсь лишний раз не связываться и использую традиционный метод `main`.


## Коллекции

Очень часто в коде можно увидеть эмуляцию функций стандартной библиотеки Scala. Приведу простой пример:

    tvs.filter(tv => tv.displaySize == 50.inches).headOption

Тоже самое, только короче:

    tvs.find(tv => tv.displaySize == 50.inches)

Подобные «антипаттерны» не редкость:

    list.size = 0  // плохо
    list.isEmpty   // ok

    !list.empty    // плохо
    list.nonEmpty  // ok

    tvs.filter(tv => !tv.supportsSkype)    // плохо
    tvs.filterNot(tv => tv.supportsSkype)  // ok

> Конечно, если вы используете IntelliJ IDEA, она вам обязательно подскажет наиболее эффективную комбинацию методов. Scala IDE, насколько мне известно, так не умеет.

О неэффективном использовании библиотеки коллекций Scala можно рассказывать сколь угодно долго. И это уже очень не плохо сделал Павел Фатин в своей статье [Scala collections Tips and Tricks](https://pavelfatin.com/scala-collections-tips-and-tricks), с которой я вам очень рекомендую ознакомиться. И да, старайтесь не вызывать элементы коллекций по индексам. Нехорошо это.


## Список литературы
В заключении этой статьи я бы хотел порекомендовать материалы, которые я нахожу полезными для изучения.

### Книги
[Книга](https://www.amazon.com/Programming-Scala-Updated-2-12/dp/0981531687), которую должен прочесть каждый Scala-разработчик. К сожалению, терпения хватает не всем, однако она стоит затраченных усилий.

### Официальная документация
[Руководство по стилю Scala](http://docs.scala-lang.org/style/)

### Статьи
 - [Effective Scala](http://twitter.github.io/effectivescala/), для которой существует и [перевод](http://twitter.github.io/effectivescala/index-ru.html) на русский, хотя я, безусловно, советую вам ознакомиться с оригиналом.
 - [Scala Collections Tips and Tricks Павла Фатина](https://pavelfatin.com/scala-collections-tips-and-tricks/).
 - О том, чего лучше не делать в Scala, очень доступно изложено [здесь](https://github.com/alexandru/scala-best-practices/blob/master/sections/2-language-rules.md).
 - [Scala Collections Tips and Tricks](https://pavelfatin.com/scala-collections-tips-and-tricks).

### Видео
 - [Scala with style](https://www.youtube.com/watch?v=kkTFx3-duc8) — доклад создателя языка о том, как идиоматично писать на Scala.
 - [Martin Odersky, Scala — the Simple Parts](https://www.youtube.com/watch?v=ecekSCX3B4Q)
 - [Daniel Spiewak, May Your Data Ever Be Coherent](https://youtu.be/gVXt1RG_yN0)
 - [For: What is it good for?](https://www.youtube.com/watch?v=WDaw2yXAa50) — выступление, посвященное подробному разбору «for comprehensions».

## Благодарности
Автор хотел бы выразить свою признательность

 - Владу Ледовских -- за вычитку,
 - Павлу Кретову (@firegurafiku) -- за безуспешные попытки привести этот текст к литературной норме, а также за помощь с разделом о `typedef` в первой части статьи,
 - Арсению Жижелеву (@primetalk) -- за внесение многочисленных уточнений в изначальный текст,
 - Семёну Попугаеву (@senia) -- за найденные неточности.

Отдельное спасибо EDU-отделу @DataArt и всем тем, кто, проходя наши курсы по Scala, подталкивал меня к написанию этой статьи. Спасибо вам, уважаемые читатели, за то, что дочитали (или хотя бы промотали) до конца.
