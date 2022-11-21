# Annotation map example using Scala3 Macros

## Intention
We want to design a typeclass that is synthesized automatically at compile-time and gives us runtime access to a class' annotations.

# Example of desired functionality:
```scala

final case class speciesName(name: String)
final case class tableName(name: String)

@tableName("pet")
@speciesName("Felis Silvestris Catus")
final case class Cat(name: String, age: Int)

@tableName("pet")
@speciesName("Canis Lupus Familiaris")
final case class Dog(name: String, age: Int)

def summaryOf[T](obj: T)(using M: AnnotationMap[T]): String =
  s"""
    - ${M.get[speciesName].map(_.name).getOrElse("Ordinary Object")}
    ${M.get[tableName].map(table => s"- Found in table '${table.name}'").getOrElse("")}
  
  """

```

```scala
import scala.quoted.{ Quotes, Type, Expr }

inline def 
```
