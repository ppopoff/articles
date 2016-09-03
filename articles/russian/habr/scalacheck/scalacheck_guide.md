Про Scalacheck
==============

Scalacheck - это [комбинАторная](combinator-library) библиотека значительно облегчающая написание модульных тестов
на Scala. Эта библиотека использует подход свойство-ориентированного тестирования (property-based testing) впервые
реализованного в библиотеке [Quickcheck](quickcheck-intro) для языка Haskell.
Существует множество реализаций Quickcheck: есть реализации для [Java](quickcheck-java), [C](quickcheck-c), а так
же множества [других](quickcheck-wiki) языков и платформ.


В этой статье я расскажу вам о подходе, Property-based testing ().
Использование данного подхода позволяет значительно сократить время на разработку тестов.

В этой статье мы научимся смотреть на мир через свойства, подробно рассмотрим Scalacheck и даже напишем несколько
тестов. Также я расскажу о методах интеграции этой замечательной библиотеки в ваш проект. И главное, стоит ли овчинка
выделки. Если интересно, прошу под кат.

[combinator-library]: https://en.wikipedia.org/wiki/Combinator_library
[quickcheck-wiki]: https://en.wikipedia.org/wiki/QuickCheck
[quickcheck-intro]: http://www.stuartgunter.org/intro-to-quickcheck/
[quickcheck-c]: https://github.com/silentbicycle/theft
[quickcheck-java]: https://github.com/pholser/junit-quickcheck/

<cut text="Читать про Scalacheck →">


# Введение:
Модульное тестирование является одним из важнейших подходов в разработке программного обеспечения. И даже если ваша
программа проходит различные проверки типов, это еще не означает что в ней отсутствуют логические ошибки. Стоимость
тестирования является высокой - помимо потраченных человеко-часов, требуется тратить нечеловеко-yсилия на выполнение
рутинного написания модульных тестов. Именно поэтому многие заказчики экономят на модульном тестировании, а многие
программисты пользуются этим с превеликой разницей. Модульные тесты писать скучно, но нужно.

Использование функциональной парадигмы значительно облегчает модульное тестирование - отсутствие побочных эффектов
позволяет рассматривать меньше краеугольных случаев и забыть о тестировании состояния. Объем тестирующего кода это
значительно не сокращает, и скорее всего найдется найдется случай, который вы упустили из виду. Однако подход даст
вам определенные гарантии и некоторую уверенность в вашем коде.

Даже функциональный и типобезопасный код нужно тестировать. Но согласитесь, покрывать огромное количество случаев,
и писать однотипные синтаксические конструкции изо дня в день - удовольствие сомнительное.
Свойство-ориентированное тестирование позволяет значительно сократить объем тестирующего кода, тем самым повысив
его ремонтопригодность.


# Cвойство-ориентированное тестирование



# Подготовительные работы




# Руководство:

общий план:
 - показать примеры
 - показать тесты
 - показать генераторы
     - объекты
     - функции
 - генераторы для пользовательских типов
 - ошибки
 - pretty pringing


0) начать с функции reverse
1) бесконечные структуры?е



----------------------

Давайте протестируем функцию take()
Инвариант будет выглядеть так:

∀ s.length(take5 s) = 5

\s -> length (take5 s) == 5

*A> quickCheck (\s -> length (take5 s) == 5)
Falsifiable, after 0 tests:
""

Давайте изменим условие.

∀ s.length(take5 s) <= 5


# Интеграция

## Самостоятельная


## С использованием scalatest


# Мониторинг тестовых данных





# Достоинства
 - Лаконичность
 - Больше думаем над набором входных данных

# Недостатки
 - Чувствительность к медленному коду
 - нечувствительность к граничным условиям
    - Видно ботлнеки
    - Можно подождать
    -
 - Равномерное распределения для случайных результатов. (Пример с большими числами)

  False sense of serurity

  Many properties of unification apply to the case when unification Succeeds. They can be stated conveniently in
  the form:

  prop_Unify t1 t2 = s /= Nothing => ..
    where s = unifier t1 t2

  Thus the case where one tyerm is a variable is heavily overrepresented
  among the test cases that satisfy the precondition -- we found that over 95% of
  test cases had this property. Although QuickCheck succeeded in 'verifying' every
  property, we can hardly consider that they were thoroughly tested.

  The solution was to use a custom test data generator

  prop_Unify =
    forAll probablyUnifiablePair $ \(t1,t2) ->
      s /= Nothiing =>
    where s = unifier t1 t2

  This usually genereates unifiable terms althgough may fail to when variables are used inconsistently
  in the two terms. With this modification, the proportion of trivial cases fell to a reasonable 20-25%.
  This exprience underlines the importance of investigating the distribution of actual test cases, whenever conditional
  properties are used.


# Заключение
Это не [серебряная пуля](no-silver-bullet), но жизнь облегчает заметно.

[no-silver-bullet]: bit.ly/1TxQkJc




# Список литературы:
  - [QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs](ClaessenAndHughes) - оригинальная и весьма
  познавательная статья, описывающая библиотеку quickcheck.

# Для дальнейшего чтения
  - [An introduction to Quickcheck testing by Pedre Vasconcelos](Pedro-Vasconcelos)


[ClaessenAndHughes]: http://www.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf
[PedroVasconcelos]: https://www.schoolofhaskell.com/user/pbv/an-introduction-to-quickcheck-testing

