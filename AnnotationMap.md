# Annotation map example using Scala3 Macros

## Intention
We want to design a typeclass that is synthesized automatically at compile-time and gives us runtime access to a class' annotations.  
Why? Because it seemed like a nice use case for macros and because there are dozens of questions out there on how to get a type's annotation easily in Scala

### Example of desired functionality:
```scala

final case class speciesName(name: String) extends StaticAnnotation

@speciesName("Felis Silvestris Catus")
final case class Cat(name: String, age: Int)

@speciesName("Canis Lupus Familiaris")
final case class Dog(name: String, age: Int)

def summaryOf[T](obj: T)(using M: AnnotationMap[T]): String = {
  val annotationOpt: Option[speciesName] = M.get[speciesName]
  val speciesOfT: String = speciesAnnotation.map(_.name).getOrElse("Ordinary Object")

  s"${speciesOfT}: ${obj}"
}
  
@main def run = {
  println(summaryOf(Cat("Tom", 7)) // Felis Silvestris Catus: Cat(Tom, 7)
  println(summaryOf(Dog("Spike", 11)) // Canis Lupus Familiaris: Dog(Spike, 1)
  println(summaryOf(List(1, 2, 3)) // Ordinary Object: List(1, 2, 3)
}
```

The annotation `speciesName` is present on `Cat` and `Dog` with their specified scientific names, using a generic function requiring the typeclass `AnnotationMap[T]` we can find out if the type parameter `T` has the annotations we require

## Implementation

### In a file `AnnotationMap.scala`:
```scala
import scala.reflect.{ ClassTag, classTag }

final class AnnotationMap[T](rawMap: Map[Class[?], ?]) {
  inline def get[T : ClassTag]: Option[T] =
    // We will only insert matching key-value pairs so casting is ok
    rawMap.get(classTag[T].runtimeClass).asInstanceOf[Option[T]]
}
```

### In a file `AnnotationMapMacros.scala`:
Here is our short macro, for explanations, scroll down:
```scala
import scala.quoted.{ Quotes, Type, Expr }

object AnnotationMapMacros {
  inline def annotationsOf[T]: Map[Class[?], ?] =
    ${ annotationsOfMacro[T] }

  def annotationsOfMacro[T : Type](using quotes: Quotes): Expr[Map[Class[?], ?]] = {
    import quotes.reflect.*
    
    val repr: TypeRepr[T] = TypeRepr.of[T]
    val symbol: Symbol = repr.typeSymbol
    val annotations: List[Term] = symbol.annotations

    val tupleExprsInList: List[Expr[(Class[?], Any)]] = annotations
      .filter(_.tpe <:< TypeRepr.of[StaticAnnotation])
      .map { annTerm =>
        '{
          val annotation = ${annTerm.asExpr}
          annotation.getClass() -> annotation
        }
      }

    val tuplesInList: Expr[List[(Class[?], Any)] =
      Expr.ofList(tupleExprsInList)
      
    '{ ${tuplesInList}.toMap }
  }
}
```

Here is a commented version of our macro, explaining each step:
```scala
import scala.quoted.{ Quotes, Type, Expr }

object AnnotationMapMacros {
  // This is our entry function, this is what is callable from the client code
  inline def annotationsOf[T]: Map[Class[?], ?] =
    ${ annotationsOfMacro[T] }

  // The macro implementation uses the Type typeclass for T and the Quotes context parameter, which
  // contains information about the place of macro injection and all of our compliation unit, really
  def annotationsOfMacro[T : Type](using quotes: Quotes): Expr[Map[Class[?], ?]] = {
    // The Quotes has a reflect module that implements abstract types to help us integrate with the program's Abstract Syntax Tree
    import quotes.reflect.*

    // Obtain the compile-time type representation of our type T
    // This can tell us whether the type is an alias, a refinement, a union, a intersection etc.
    val repr: TypeRepr[T] = TypeRepr.of[T]

    // Get the type symbol of our type representation.
    // This finds the declaration of the type T, it holds information about it's
    // fields, methods, name, parents and, most importantly, annotations
    val symbol: Symbol = repr.typeSymbol

    // This allows us to get the annotations of this type as Terms
    // Terms are representations of expressions (anything that has a result: literals, variables, if blocks etc)
    // In this particular case, all of our terms will be constructor calls
    val annotations: List[Term] = symbol.annotations

    val tupleExprsInList: List[Expr[(Class[?], Any)]] = annotations
      .filter(_.tpe <:< TypeRepr.of[StaticAnnotation]) // Make sure all annotations' types extend StaticAnnotation
      .map { annTerm =>
        // Build an expression, a block that will be injected into whenever we use our macro
        '{
          // When in a '{ ... } block, you have to use ${ ... }
          // To bring your values down from the macro-building scope to the injected code scope
          val annotation = ${annTerm.asExpr}

          annotation.getClass() -> annotation
        }
      }

    // Turn our list of expressions into an expression of the list
    val tuplesInList: Expr[List[(Class[?], Any)] =
      Expr.ofList(tupleExprsInList)

    // The transition to Map will be done at runtime (everything inside a '{ ... } happens at runtime
    // Everything in a macro method or a ${ ... } happens at compiletime
    '{ ${tuplesInList}.toMap }
  }
}
```

**Important** Macro implementations should be in seperate files from where they are called

And now, to provide our typeclass, we go back to `AnnotationMap.scala`
```scala
object AnnotationMap {
  // This given must be inline to make sure macros can reach the real type T, not just an anonymous one
  inline given synthesizedAnnotationMap[T]: AnnotationMap[T] =
    val rawMap = AnnotationMapMacros.annotationsOf[T]
    AnnotationMap[T](rawMap)
}
```

## Conclusion:
That's it, we have made our utility typeclass AnnotationMap that can be required as a context bound `[T : AnnotationMap]` or as a given with `(using M: AnnotationMap[T])`.

Now not only can we access a type's annotations at runtime without hassle in client code, but we don't even have to make all of our functions inline, because the typeclass is synthesised the first time it is required, just like `ClassTag`, `Mirro` or the `=:=`/`<:<` evidence typeclasses in the standard library. Try it out yourself :) 
