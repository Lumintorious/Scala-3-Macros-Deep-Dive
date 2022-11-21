# Annotation map example using Scala3 Macros

## Intention
We want to design a typeclass that is synthesized automatically at compile-time and gives us runtime access to a class' annotations.

### Example of desired functionality:
```scala

final case class speciesName(name: String) extends StaticAnnotation

@speciesName("Felis Silvestris Catus")
final case class Cat(name: String, age: Int)

@speciesName("Canis Lupus Familiaris")
final case class Dog(name: String, age: Int)

def summaryOf[T](obj: T)(using M: AnnotationMap[T]): String =
  val annotationOpt: Option[speciesName] = M.get[speciesName]
  val speciesOfT: String = speciesAnnotation.map(_.name).getOrElse("Ordinary Object")

  s"${speciesOfT}: ${obj}"
  
@main def run =
  println(summaryOf(Cat("Tom", 7)) // Felis Silvestris Catus: Cat(Tom, 7)
  println(summaryOf(Dog("Spike", 11)) // Canis Lupus Familiaris: Dog(Spike, 1)
  println(summaryOf(List(1, 2, 3)) // Ordinary Object: List(1, 2, 3)
```

```scala
import scala.quoted.{ Quotes, Type, Expr }

inline def 
```
