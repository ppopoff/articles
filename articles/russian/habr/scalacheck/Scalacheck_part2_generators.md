**Часть 2. Генераторы**

В вводной статье серии вы уже познакомились с генераторами. В этой статье --
мы закрепим полученные знания, а также научимся писать собственные в том числе
рекурсивные генераторы. Хоть эта статья и посвящена генераторам, свойства мы
тоже не обойдем: мы будем их использовать, демонстрируя всю мощь механизма
генераторов, также не обойдем вниманием механизм предусловий (preconditions),
Если интересно, прошу под кат:

<cut text="Читать про генераторы в ScalaCheck →">

**Структура цикла**

 - Введение
 - Генераторы
 - Свойства
 - Минимизация
 - Интеграция и настройки


# Генераторы
Метод `forAll` из объекта `Prop` использует данные, сгенерированные произвольным
образом. Это могут быть как простые типы данных, так и контейнерные (*compound*).
В ScalaCheck, любой генератор является наследником [Gen](http://bit.ly/2jKPK0La):

    sealed abstract class Gen[+T] {
      def apply(prms: Gen.Params): Option[T]
      ...
    }

Класс является параметрическим по типу генерируемого значения. Соответственно,
для строк мы будем иметь `Gen[String]`, а для пресловутых `Person` --
`Gen[Person]`. Также вы можете использовать метод `arbitrary` из объекта
`Arbitrary` при условии наличия в области видимости необходимых имплисита.

## Добавляем поддержку implicitов к нашим генераторам
Раз уж мы заговорили об имплиситах, давайте разберемся, как сделать `arbitrary`
для вашего типа. Для начала нам нужен сам тип:

    sealed trait LED
    case object Red extends LED
    case object Green extends LED
    case object Blue extends LED

Теперь создадим генератор, для наших диодов:

    val ledGenerator: Gen[LED] = Gen.oneOf(Red, Green, Blue)

А теперь создадим `implicit` экземпляр `Arbitrary` из имеющегося у нас
генератора:

    implicit val arbitraryLed: Arbitrary[LED] = Arbitrary(ledGenerator)

    Arbitrary.arbitrary[LED].sample
    // Some(Green)

Теперь, имея светодиоды в нашей области видимости, мы можем проверить некоторые
свойства:

    val ledProp = forAll { diode: LED =>
      // тестируем несчастные диоды
    }

Для ряда примитивных типов, а так же контейнеров из стандартной библиотеки
неявные преобразования уже предопределены внутри объекта `Arbitrary`:

    import org.scalacheck.Arbitrary.arbitrary

    val arbitraryString  = arbitrary[String]
    val arbitraryInteger = arbitrary[Int]

`Gen.resultOf` может вам значительно помочь в этом деле. Эта функция принимает
case класс, для параметров конструктора которого уже определены `arbitrary`
значения:

    case class Coord(x: Double, y: Double)

    // создаем генератор при помощи resultOf
    val genCoord = Gen.resultOf(Coord)

    // помещаем его в Arbitrary
    implicit val arbitraryCoord = Arbitrary(genCoord)

    // А теперь проверяем все что нам придет в голову
    val prop = Prop.forAll { coord: Coord =>
      //...
    }

## Sample
К сожалению для экземпляров класса `Gen`, не определен метод toString, но есть
метод который лучше и удобнее. Функция, которую вам придется очень часто
использовать при написании генераторов именуется `sample`. Она возвращает пример
сгенерированного значения внутри `Option`:

    // ScalaCheck может возвращать очень длинные строки поэтому отрежем
    // первые 10 символов
    Arbitrary.arbitrary[String].sample map (_ take 10)

    // А вот и конечный резуольтат
    // Some(Ｑ冋执龭轙⮻拍箘㖕)

    Arbitrary.arbitrary[Double].sample
    // Some(-5.180668081211655E245)


## Трансформации
Как уже было рассказано в предыдущей части, к генераторам так же можно
применять метод `map`. Проиллюстрируем это следующим примером:

    val octDigitStr = choose(0, 7) map (_.toString)

    octDigitStr.sample
    // Some(3)

    octDigitStr.sample
    // Some(5)

## Фильтрация
Для генераторов также существует метод `filter`, хотя на самом деле это лишь
псевдоним для вызова метода `suchThat`:

    val evenOct = octDigit.filter(_ % 2 == 0)

    evenOct.sample
    // None

    // Не повезло, попробуем еще раз:
    evenOct.sample
    // None

    // Вот досада, а может быть сейчас?
    evenOct.sample
    // Some(2)

Пример выше демонстрирует, почему пользоваться `filter` или `suchThat` не стоит.
Новый генератор просто отвергает значения, которые не проходят фильтр. Это
сильно замедляет процесс тестирования.

> Предусловия действуют аналогично фильтрам

## Предусловия
Представляемые в логике операцией импликации, в Scalacheck предусловия это методы
используемые внутри свойств, позволяющий ограничить входные данные.
Представляется оператором `==>`. Для того чтобы воспользоваться предусловиями,
вам необходимо добавить в область видимости `Prop.propBoolean`. Итак, давайте
проверим утверждение, о том, что для любого четного числа, последняя цифра также
будет являться четным числом:

    import org.scalacheck.Prop.{forAll, propBoolean}

    def isEven(number: Int) = number % 2 == 0

    // Получаем последнюю цифру числа
    def lastChar(number: Int) =
      number.toString.last.toInt

    // фильтрация
    val p2 = forAll(arbitrary[Int] suchThat isEven) { number =>
      isEven(lastChar(number))
    }

    // использование предусловия
    val p1 = forAll(arbitrary[Int]) { number =>
        isEven(number) ==> isEven(lastChar(number))
    }

Чтобы запустить свойство, мы можем воспользоваться функцией `check`

    // проверим наше первое свойство
    p1.check
    // + OK, passed 100 tests.

    // и второе
    p2.check
    // + OK, passed 100 tests.

Чем жестче фильтр, тем больше значений будет отфильтровано это критично для
малого набора значений, а так же для значений находящися очень далеко друг от
друга на заданном диапазоне (например простых чисел на диапазоне Int).
ScalaCheck легко может устать от этой затеи и сдаться вам в плен:

    scala> p1.check
    ! Gave up after only 1 passed tests. 100 tests were discarded

На примере с четными числами вы этого не увидите, а вот на примере с простыми,
легко. Также следует знать, что:

> Фильтр для четных чисел можно реализовать по-разному. Например можно брать
> любое целое число и просто умножать его на 2. В случае с простыми числами
> намного проще будет посчитать их заранее, воспользовавшись, например,
> [решетом Эратосфена](http://bit.ly/2jI81b3), а затем просунуть список
> вычисленных значений внутрь `Gen.oneOf`, который мы рассмотрим далее в этой
> статье.

Если вы комбинируете генераторы, использующие `suchThat`, у вас также могут
возникнуть проблемы. В таких случаях вам следует воспользоваться методом
`retryUtil`. В отличии от `suchThat` он старается сильнее. Однако, его усердие
может закончиться бесконечным циклом. Вызывается `retrtyUtil` аналогично
`fiter` или `suchThat`:

    arbitrary[Int] retryUntil isEven

Вы также можете заметить, что методы `collect`, `flatten` также как `fromOption`
объявлены `Deprecated`, это сделано для того, чтобы у вас было меньше соблазна
плодить пустые генераторы.

## Создаем свой генератор
Как уже было замечено ранее вы можете создавать свои собственные генераторы,
комбинируя уже существующие, используя for comprehension

    val coordGen =
        for { i <- arbitrary[Int]; j <- arbitrary[Int] } yield (i, j)

    coordGen.sample
    // Option[(Int, Int)] = Some((45,60))

    coordGen.sample
    // Option[(Int, Int)] = Some((29, 37))

Вы можете объявить генераторы для любой структуры данных, простой случай
использования ADT:

    sealed trait LightColor
    case object Red     extends LightColor
    case object Orange  extends LightColor
    case object Green   extends LightColor
    case object FlashingGreen extends LightColor
    case object Extinct extends LightColor

    val colorGen = Gen.oneOf(Red, Orange, Green, Extinct)

Если усложнять себе жизнь, так усложнять:

    // Если речь здесь будет идти о цвете, то только о цвете света
    type Color = LightColor

    object PedestrianTrafficLight {
      sealed class State(_1: Color, _2: Color)
      case object Stop extends State(Red, Extinct)
      case object Move extends State(Extinct, Green)
      case object FinishMove extends State(Extinct, FlashingGreen)
      def states: Seq[State] = Seq(Move, FinishMove, Stop)
    }

    object VehicularTrafficLight {
      sealed class State(_1: Color, _2: Color, _3: Color)
      case object Stop extends State(Red, Extinct, Extinct)
      case object Ready extends State(Red, Orange, Extinct)
      case object Move extends State(Extinct, Extinct, Green)
      case object Warn extends State(Red, Orange, Extinct)
      case object FinishMove extends State(Extinct, Extinct, FlashingGreen)
      def states: Seq[State] = Seq(Stop, Ready, Move, Warn, FinishMove)
    }

    sealed trait TrafficLight

    case class PedestrianTrafficLight(state: PedestrianTrafficLight.State)
    extends TrafficLight

    case class VehicularTrafficLight(state: VehicularTrafficLight.State)
    extends TrafficLight

А теперь давайте объявим наши генераторы:

    import Gen._

    // Набор состояний для пешеходного светофора
    val pedestrianTrafficLightStateGen =
      Gen.oneOf(PedestrianTrafficLight.states)

    // Набор состояний для транспортного светофора
    val vehicularTrafficLightStateGen =
      Gen.oneOf(VehicularTrafficLight.states)

А теперь, давайте сгенерируем кортеж, из пешеходного и транспортного светофоров.
Есть только одно но, состояния светофоров не должны быть взаимоисключающими.
Для начала, давайте разберемся, когда пешеход может совершить маневр. Так как мы
создаем генератор путем композиции, мы используем `retryUntil`, вместо
`suchThat`:

    val pedestrianCanGo = pedestrianTrafficLightStateGen.retryUntil { state =>
      state != PedestrianTrafficLight.Stop
    }

Для начала сгенерируем состояние автомобильного светофора, и уже отталкиваясь
от него сгенерируем допустимое состояние для пешеходного, и объединим их
в список. Полученный нами генератор будет иметь следующий тип:
`Gen[(VehicularTrafficLight.State, PedestrianTrafficLight.State)]`.

    val states = vehicularTrafficLightStateGen flatMap {
      case vehState @ VehicularTrafficLight.Stop =>
        pedestrianCanGo map (pedState => (vehState, pedState))
      case pedestrianCanNotGo =>
        (pedestrianCanNotGo, PedestrianTrafficLight.Stop)
    }

Теперь отобразим состояния светофоров в сами объекты:

    val trafficLightPair = states map { case (v, p) =>
      (VehicularTrafficLight(v), PedestrianTrafficLight(p))
    }

Ну что, проверим результаты? Я получил:

    (VehicularTrafficLight(FinishMove),PedestrianTrafficLight(Stop))

и:

    (VehicularTrafficLight(Move),PedestrianTrafficLight(Stop))

что вполне себе похоже на правду. Может быть где-то и ошибся. Главным было
продемонстрировать сам механизм.

> ScalaCheck также позволяет вам генерировать рекурсивные структуры данных,
> однако легким и тривиальным этот процесс назвать сложно. Далее в этой статье
> мы рассмотрим рекурсивные генераторы, и проблемы с которыми вы можете
> столкнуться при их использовании

## Используем генераторы в свойствах
В большинстве использованных ранее примеров ScalaCheck самостоятельно подбирает
подходящий экземпляр генератора и использует его для вычисления свойства.
Однако, не для того мы сами писали генераторы чтобы ScalaCheck забирал из
области видимости что попало. Помните наши "совместимые" светофоры? Давайте
уже куда-нибудь их приткнем:

    import Prop.{forAll, propBoolean}

    def isRed(trafficLight: VehicularTrafficLight) =
      trafficLight.state == VehicularTrafficLight.Stop

    def notRed(trafficLight: PedestrianTrafficLight) =
      trafficLight.state != PedestrianTrafficLight.Stop

    // Генератор передается в качестве первого параметра для Prop.forAll
    val canNotBothBeRed =
      forAll(trafficLightPair) { case (vehicular, pedestrian) =>
        isRed(vehicular) ==> notRed(pedestrian)
      }

Проверим?:

    canNotBothBeRed.check

И убедимся, что все хорошо:

    + OK, passed 100 tests.

Зачастую, одного генератора бывает недостаточно, и вы можете использовать
несколько:

    val p = forAll(Gen.posNum[Int], Gen.negNum[Int]) { (pos, neg) =>
      neg < pos
    }

Свойства можно вкладывать друг в друга, и, генераторы, соответственно тоже,
поэтому пример выше можно переписать так:

    val p = forAll(Gen.posNum[Int]) { pos =>
      forAll(Gen.negNum[Int]) { neg =>
        neg < pos
      }
    }

Вместо вложенных вызовов `forAll`, вы также можете собрать кортеж из генераторов,
которые хотите использовать. И вызвать forAll единожды.

## Метки на генераторах
Использование меток является хорошей практикой. Это позволяет улучшить отчеты
об ошибках, а также повысить читаемость кода, использующего генераторы. Для
того чтобы добавить метку, вы можете использовать операторы `|:` если добавляете
метку до, и `:|`, если добавляете метку после генератора. Метка может являться
как строкой, так и символом. К каждому генератору может быть добавлено несколько
меток. Возьмем для иллюстрации предыдущий пример (только изменим знак, сделав
свойство абсурдным):

    val p = forAll('positive |: Gen.posNum[Int]) { pos =>
      forAll(Gen.negNum[Int] :| "negative numbers") { neg =>
        neg > pos
      }
    }

Согласитесь, стало лучше?

    ! Falsified after 0 passed tests.
    > positive: 0
    > positive_ORIGINAL: 1
    > negative numbers: 0


# Генераторы примитивов

## Константы
Сложно придумать что-то проще: `Gen.const` -- принимает любое значение и
заботливо упаковывает его в генератор. Любой объект находящийся в удачном
месте (и в нужное время) будет неявным образом приведен к Gen. Поэтому, скорее
всего в явном виде вызывать этот метод вам не придетяс.

## Умирашка
Внутри ScalaCheck используется метод `Gen.fail`, возвращающий нежизнеспособный
генератор а именно `None`. Скорее всего, на вашем пути он не встретится, а если
и встретится, считайте что вы с ним знакомы:

    Gen.fail.sample
    // Option[Nothing] = None

## Числа
Вы можете использовать `Gen.posNum` и `Gen.negNum` для генерации положительных
и отрицательных чисел соответственно:

    import org.scalacheck.Gen
    import org.scalacheck.Prop.forAll

    val negative = Gen.negNum[Int]
    val positive = Gen.posNum[Int]

    val propCube = forAll (negative) { x =>
      x * x * x < 0
    }

Интересным функционалом обладает `Gen.chooseNum`: Этот генератор порождает
числа в заданном диапазоне (включительно),с большим весом для нуля (если он
входит в заданный диапазон), минимального и максимального значения, а так же для
представленного списка значений:

    // значения specials являются vararg
    Gen.chooseNum(minT = 2, maxT = 10, specials = 9, 5)


`Gen.choose` можно использовать для генерации числовых значений в рамках
заданного диапазона. Более того, это тот редкий тип генератора, который никогда
не  проваливается. Его также можно использовать применительно к символам. Кстати
о символьных генераторах

## Символы
коих в ScalaCheck существует великое множество. Вам предлагается выбрать из
следующих:

  - `Gen.alphaUpperChar`
  - `Gen.alphaLowerChar`
  - `Gen.alphaChar`
  - `Gen.numChar`
  - `Gen.alphaNumChar`

Давайте, теперь попробуем сгенерировать координаты для игры в "Морской бой":

    val coord = for {
      letter <- Gen.alphaUpperChar
      number <- Gen.numChar
    } yield s"$letter$number"

    coord.sample
    // Some(L1)

Благодаря for comprehension, вы можете генерировать любые агрегаты: списки,
кортежи, строки. Кстати о строках...

## Строки
Вы также можете их легко генерировать. `Gen.alphaStr` генерирует строки только
из алфавитных символов. `Gen.numStr` генерирует числовые строки, а
`Gen.idenifier` дает вам непустую строку которая всегда начинается с алфавитного
символа в низком регистре, за которым следуют алфавитно-числовые символы. Для
некоторых грамматик вполне себе идентификатор.

    import org.scalacheck.Gen

    val stringsGen = for {
      key   <- Gen.idenifier
      value <- Gen.numStr
    } yield Bucket(key take 8, value take 2)

    stringsGen.sample

    // Some(Bucket(kinioQqg,60))

## Функции
В Scalacheck можно генерировать и функции (экземпляры `Function0`: `() => T`).
Для этого нужно Передать генератор, отвечающий за возвращаемое значение в
`Gen.function0`:

    val f = Gen.function0(Arbitrary.arbitrary[Int]).sample
    // toString любит функции, однако лямбду мы можем лицезреть
    // Some(org.scalacheck.Gen$$$Lambda$40/1418621776@4f7d0008)


# Генераторы контейнеров
Вы можете генерировать экземпляры стандартных коллекций используя `arbitrary`:

    import Arbitrary.arbitrary

    val listgen = arbitrary[List[Int]]
    val optgen  = arbitrary[Option[Int]]

Но этого зачастую бывает недостаточно, поэтому ScalaCheck предоставляет
расширенные инструменты для работы с коллекциями, а также их генерации. Далее
мы будем опускать `Option`ы которые возвращает метод `sample` и будем показывать
вам сразу значения. Для того чтобы не в водить в лишнее заблуждение.

### Генерируем Option
`Option` они как опционы: опцион то есть, а прибыль может быть, а может не быть.
Однако, если вы используете `Gen.some`, то "доход" вам гарантирован:

    // Тот редкий случай счастливый случай:
    Gen.some("денежка")
    // Some(денежка)

хуже когда у денежки появляются дополнительные степени свободы:

    // Потерялась
    Gen.option("денежка")
    // None

### Генерируем списки и словари
`Gen.listOf` порождает списки произвольных длин, `Gen.listOfN` порождает списки
заданной длинны.

    // Константа 3 будет неявно приведена к Gen[Int]
    Gen.listOf(3) map (_ take 5)
    // List(3, 3, 3, 3, 3)

    // Для получения пятиэлиментного списка
    Gen.listOfN(5, Gen.posNum[Double]) map (_ take 5)

Генераторы `Gen.listOf` может вернуть пустой список, если вам гарантированно
нужен непустой список, вы можете воспользоваться `Gen.nonEmptyListOf`.
Для генерации словарей, или отображений (Map) существуют аналогичные методы:
`Gen.mapOf`, `Gen.mapOfN`, а также `Gen.nonEmptyMapOf`. В отличии от методов
для списка, генераторы для словарей, требуют чтобы им передали генератор кортежа
из двух элементов. Продемонстрируем это следующим примером:

    import Arbitrary._

    val tupleGen = for {
      i <- arbitrary[Short]
      j <- arbitrary[Short]
    } yield (i, j)

    Gen.mapOfN(3, tupleGen) map (_ take 3)
    // Map(10410 -> -7991, -19269 -> -18509, 0 -> -18730)


## Порождаем список элементов заданного множества
Вернет вам коллекцию `ArrayBuffer` из некоторого набора элементов, входящих
в заданное вами множество, например:

    import org.scalacheck.Gen.someOf

    val numbers = someOf(List(1, 2, 3, 4, 5)).sample
    // Some(ArrayBuffer(5, 4, 3))

    val leds = someOf(List(Red, Green, Blue)).sample
    // Some(ArrayBuffer(Blue, Green))

`Gen.pick` работает аналогично `Gen.someOf`. Отличие лишь в том, что `Gen.pick`
позволяет вам указать необходимое количество элементов:

    import org.scalacheck.Gen.pick

    val lettersGen =
      Gen.pick(2, List('a', 'e', 'i', 'o', 't', 'n', 'm')).sample

    // Some(ArrayBuffer(m, n))

### Порождаем последовательность
`Gen.sequence` создает `ArrayList` основе принятой последовательности значений.
Если хотябы один из предоставленных генераторов упадет, вся последовательность
упадет.

    import Gen.{const, choose}

    val ArrayListGen =
      Gen.sequence(List(const('a'), const('b'), choose('c', 'd')))

    // [a, b, d]

### Генерируем бесконечный поток
ScalaCheck поддерживает возможность генерации бесконечных потоков:

    val stream = Gen.infiniteStream(Gen.choose(0,1))

    // возьмем первые 8 элементов
    stream.sample map (stream => stream.take(8).toList)
    // Some(List(1, 0, 0, 0, 1, 1, 1, 0))

## Генератор контейнеров
Перечисленные выше методы для словарей и списков, на самом деле, являются
частными случаями использования `Gen.containerOf` и его товарищей:
`containerOfN` и `nonEmptyContainerOf`. Это наиболее обобщенная форма, которая
позволяет сгенерировать большинство известных вам коллекций. Давайте рассмотрим
следующие примеры:

    import Arbitrary._

    // создадим коллекцию "коротких" целых чисел из трех элементов
    val prettyShortSeq =
      Gen.containerOfN[Seq, Short](3, arbitrary[Short])

    // Vector(0, -24114, 32767)

    // задача посложнее соорудим мапу
    val genNonEmptyList =
      Gen.nonEmptyContainerOf[Map[K, V], (K, V)] (oneOf("foo", "bar"))

Генераторы, созданные с помощью `containerOf` абстрагируются над типами, но не
над [родами](https://en.wikipedia.org/wiki/Kind_(type_theory)) (kinds). Давайте
посмотрим на сигнатуру метода `containerOf`:

    def containerOf[C[_],T](g: Gen[T])(implicit
      evb: Buildable[T,C[T]], evt: C[T] => Traversable[T]
    ): Gen[C[T]] =

и на его реализацию:

    buildableOf[C[T],T](g)

Так что, если вам нужно сгенерировать список, или что-то что может быть
представленно, как * ⟶ *, то `containerOf` будет вам полезен. А если вам нужно
сгенерировать, что-то вроде * ⟶ * ⟶ * (примером будет являться `Map`)?

Придется использовать еще более абстрактны метод, который мы можем подсмотреть
в реализации `mapOf`:

    def mapOf[T,U](g: => Gen[(T,U)]) = buildableOf[Map[T,U],(T,U)](g)

Аналогично методам `container`, существуют методы `buildableOf`, `buildableOfN`
и `nonEmptyBuildableOf`. Обычно, Scalacheck сам решает как построить переданную
ему в качестве типа коллекцию, для этого он использует иеявный экземпляр типа
`org.scalacheck.util.Buildable`. Такие экземпляры объявлены в ScalaCheck для
всех стандарных типов  коллекций. Достаточно легко реализовать свою  коллекцию и
получить поддержку `containerOf`, так же как иэкземпляры `Arbitrary`.

## Генераторы высшего порядка
Я думаю вам, уважаемый читатель, известно что представляют из себя
[функции высшего порядка](https://en.wikipedia.org/wiki/Higher-order_function).
Генераторам тоже свойственно подобное поведение: один генератор может принимать
один и более других генераторов чтобы порождать другие генераторы.
Рассматриваемые нами ранее генераторы конейтеров, за редким исключением, также
являются генераторами высшего порядка, за редким исключением.

## Один из множества
`Gen.oneOf` способен принимать от двух до `N` генераторов, и возвращать новый.

    import Gen.{oneOf, const, choose}

    // В данном случае мы получаем Gen[T]
    val dogies = oneOf("Lucy", "Max", "Daisy", "Barney")

    // А здесь мы передаем генераторы явным образом
    val windows = oneOf(choose(1, 8), const(10))

В `oneOf` можно передать любой наследник `scala.collection.Seq`, например
список.

## Частотный генератор
`Gen.frequency` Принимает на вход спосок кортежей (пар) Каждая пара состоит из
генератора, и вероятностного веса этого генератора. Предположим что
генераторов, а затем возвращает новый генератор, который будет использовать
переданные ему целочисленные значения как вероятностные веса:

    val russianLettersInText = Gen.frequency (
      (9, 'о'),
      (9, 'а'),
      (8, 'е'),
      (7, 'и'),
      (6, 'н')
      //.. оставшиеся буквы
    )

## Ленивый генератор
Использование `Gen.lzy` позволит вам отложить генерацию пока она действительно
не будет нужна. Очень удобен в композитных генераторах, а также при генерации
рекурсивных структур данных, например AST. В случае использования `Gen.lzy`
вычисления производятся один раз. `Gen.delay` также откладывает вычисления,
однако выполняться они будут каждый раз.

# Размер генераторов
`Gen.size` принимает своим единственным параметром лямбду, которая принимает
параметром целое число. Это число является размером который ScalaCheck указывает
по время выполнения генератора. `Gen.resize` позволяет установить размер для
генератора там, где это необходимо. Проиллюстрируем это следующим примером:


    def genNonEmptySeq[T](genElem: Gen[T]): Gen[Seq[T]] = Gen.sized { size =>
      for {
        listSize <- Gen.choose(1, size)
        list <- Gen.containerOfN[Seq, T](listSize, genElem)
      } yield list
    }

    val intList = genNonEmptySeq(Arbitrary.arbitrary[Int])

    val p = Prop.forAll(Gen.resize(5, intList)) { list =>
      list.length <= 5
    }

Проверим:

    p.check
    // + OK, passed 100 tests.

`Gen.resize` создает новый генератор на базе существующего, но с измененным
размером. Это полезно при создании рекурсивных генераторов, а так же при создании
сложных генераторов из простых при помощи композиции.


# Рекурсивные генераторы
Мне достаточно часто приходится генерировать рекурсивыные структуры данных, а
именно AST. Рекурсивные генераторы используют себя внутри себя. Как-то так.
Использовать себя они могут как явно, так неявно (прибегая к помощи других
генераторов). В качестве примера, давайте рассмотрим бинарное дерево, с
целочисленными вершинами:

    trait IntTree
    case class Leaf (value: Int) extends IntTree
    case class Node (children: Seq[IntTree]) extends IntTree

Так как деревья являются рекурсивными структурами данных, сгенерируем их
рекурсивно:

    import org.scalacheck.Gen._

    def treeGen: Gen[IntTree] =
      oneOf(leafGen, nodeGen)

    def leafGen: Gen[Leaf] =
      arbitrary[Int] map (value => Leaf(value))

    def nodeGen: Gen[Node] =
      listOf(treeGen) map (child => Node(child))

Оно скомпилируется, и будет работать до первой попытки вызвать генератор:

    treeGen.sample

    Exception in thread "main" java.lang.StackOverflowError
        at org.scalacheck.ArbitraryLowPriority$$anon$1.arbitrary(Arbitrary.scala:70)
        at org.scalacheck.ArbitraryLowPriority.arbitrary(Arbitrary.scala:74)
        at org.scalacheck.ArbitraryLowPriority.arbitrary$(Arbitrary.scala:74)
        at org.scalacheck.Arbitrary$.arbitrary(Arbitrary.scala:61)

Внутри `treeGen` мы одновременно запускаем генераторы для обоих веток. Это
приводит к быстрому заполнению стека. Хм, `infiniteStream`-то ScalaCheck как-то
генерирует. Давайте попробуем сделать наше дерево ленивым. Для этого перепишем
`treeGen` так:

    def treeGen: Gen[IntTree] =
      lzy(oneOf(leafGen, nodeGen))

Запустим:

    treeGen.sample
    // Some(Leaf(857998833))
    // Some(Leaf(2147483647))
    // Some(Leaf(489549235))

[тут будет картинка: http://risovach.ru/kartinka/10629542].
Снова!

    Exception in thread "main" java.lang.StackOverflowError
        at org.scalacheck.rng.Seed.next(Seed.scala:17)
        at org.scalacheck.rng.Seed.long(Seed.scala:39)
        at org.scalacheck.Gen$Choose$.org$scalacheck$Gen$Choose$$chLng(Gen.scala:309)
        at org.scalacheck.Gen$Choose$$anon$5.$anonfun$choose$1(Gen.scala:358)
        at org.scalacheck.Gen$$anon$3.doApply(Gen.scala:254)
        at org.scalacheck.Gen.$anonfun$map$1(Gen.scala:75)
        at org.scalacheck.Gen$$anon$3.doApply(Gen.scala:254)
        at org.scalacheck.Gen.$anonfun$flatMap$2(Gen.scala:80)
        at org.scalacheck.Gen$R.flatMap(Gen.scala:242)
        at org.scalacheck.Gen$R.flatMap$(Gen.scala:239)

Да, мы не геренируем несколько нод сразу, но если генерируем, то с бесконечной
вложенностью, которую следует сделать конечной. Использование `Gen.sized` и
`Gen.resize` в тандеме сможет нам помочь:

    def nodeGen: Gen[Node] = sized { size =>
      choose(0, size) flatMap { currSize =>
        val nGen = resize(size / (currSize + 1), treeGen)
        listOfN(currSize, nGen) map (child => Node(child))
      }
    }

В этой строчке:

    val nGen = resize(size / (currSize + 1), treeGen)

Мы создаем новы генератор меньшего размера: размер уменьшается на уровень
вложенности. Для определенного N будет создан пустой генератор размера 0, и мы
не получим переполнения стека. Размер генератора следует уменьшать каждый раз,
так как генератор, не знает на каком уровне вложености он находится.

Вот мы и разобрались с рекурсивными генераторами.
[Пример](https://etorreborre.blogspot.ru/2011/02/scalacheck-generator-for-json.html)
генерации AST для JSON вы можете найти в блоге Eric Torreborre
(создателя specs2).

На этом мы законичим с генераторами и передем к свойствам. Подробнее о них будет
рассказано в следующей статье серии. Не переключайтесь :)

