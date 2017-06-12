Исключения в Scala
==================

### Об исключениях
TODO:

Checked exceptions Scala (их нет)


Exceptions:
http://stackoverflow.com/questions/13012149/is-the-use-of-exceptions-a-bad-practice-in-scala
http://stackoverflow.com/questions/12886285/throwing-exceptions-in-scala-what-is-the-official-rule
Add more information about the Validation monad


TODO: exceptions
Failure handling:
  Try Either and Validation
  todo:

    val first = Try(Console.readLine("enter a number"))
    val second = Try(Console.readLIne("another number"))

    val sum: Try[Int] =
       for { f <- first; s <- second }
         yield f.toInt + s.toInt


todo: add validation monad

## Структурные типы
Просто не используйте их. И если ваши руки к ним тянутся, вам скорее всего они не нужны. Во-первых структурные типы работают через рефлексию, и это достаточно дорого с точки зрения производительности.
(мысли на этот счет)[http://www.draconianoverlord.com/2011/10/04/why-no-one-uses-scala-structural-typing.html]


TODO:
Рассказать про салат
https://coderwall.com/p/_dxhza/scala-salad-anti-pattern


General:
https://www.quora.com/What-are-some-bad-practices-in-functional-programming
http://stackoverflow.com/questions/15848856/are-there-any-documented-anti-patterns-for-functional-programming


