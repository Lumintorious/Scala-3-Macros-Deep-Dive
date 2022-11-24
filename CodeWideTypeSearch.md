# Code-wide Type Search
### Intention
Have you ever been in a situation where you have a main file or a central control file where you add entries to a list, entries that you define all over the codebase?   Wouldn't it be great if we could generate a macro that scans the whole code for static values of a certain type to return to our main/control module in a list?  

These are just a couple of entities you would want gathered from all over your packages:
- Endpoints for HTTP servers
- Page components for React router
- Tests for a testing framework to run
- Stories for Storybook
- Articles for Documentation

### Example
Here is an example of what we want to accomplish, code-wise:

##### FrontendPage.scala:
```scala
trait FrontendPage {
  def render: Component
}
```
##### myProject/frontend/employee/overview/EmployeeOverviewPage.scala
```scala
@exposed object EmployeeOverviewPage extends FrontendPage {
  override def render = ??? // display a table of all employees
}
```
##### myProject/frontend/employee/details/EmployeeDetailsPage.scala
```scala
@exposed object EmployeeDetailsPage extends FrontendPage {
  override def render = ... // display the details of employee based on url params
}
```
##### myProject/frontend/department/overview/DepartmentOverviewPage.scala
```scala
@exposed object DepartmentOverviewPage extends FrontendPage {
  override def render = ??? // display a table of deparments with expandable lists of employees, whatever
}
```

#### MainFrontend.scala
```scala
@main def run = {
  val allPages: List[FrontendPage] = PackageSearch.searchByType[FrontendPage]("myProject.frontend")
  
  println(allPages)
// ==> List(
//       myProject.frontend.employee.overview.EmployeeOverviewPage,
//       myProject.frontend.employee.details.EmployeeDetailsPage,
//       myProject.frontend.department.overview.DepartmentOverviewPage,
// )

  val allPagesAsComponents = allPages.map(_.render)
  println(allPagesAsComponents)
// ==> List(
//       <table>...<table/>,
//       <form class="...">...<form/>
//       <table>...<table/>
// )
}
```

### Implementation
Here is how the macro code would look like, for explanations, scroll down:
##### PackageSearch:
```scala
final class exposed extends StaticAnnotation

object PackageSearch {
  transparent inline def findByType[T](inline path: String): List[T] =
    ${ findByTypeMacro[T]('path) }

  def findByTypeMacro[T : Type](pathExpr: Expr[String])(using q: Quotes): Expr[List[T]] =
    import q.reflect.*

    val packageName = pathExpr.value match {
      case Some(str) => str
      case None => report.errorAndAbort("Package name is not visible at compiletime")
    }
    
    def isEligible(valDef: ValDef): Boolean =
      valDef.tpt.tpe <:< TypeRepr.of[T] &&
      valDef.symbol.hasAnnotation(TypeRepr.of[exposed].typeSymbol)

    val symbol = Symbol.requiredPackage(packageName)
    if(symbol == Symbol.noSymbol) {
      report.errorAndAbort(s"Package by the name of ${packageName} could not be found")
    }

    val exprs = searchSymbolForDeclaration(symbol, isEligible)
      .map { valDef =>
        Ref(valDef.symbol)
      }
      .distinctBy(_.show)
      .to(List)
      .map(_.asExprOf[T])

    Expr.ofList(exprs)

  def searchSymbolForDeclaration(using q: Quotes)(symbol: q.reflect.Symbol, test: q.reflect.ValDef => Boolean): Seq[q.reflect.ValDef] =
    import q.reflect.*
    
    try {
      symbol.declarations.collect {
        case obj if obj.fullName.contains("#") =>
          Seq.empty
        case pkg if pkg.isPackageDef =>
          searchSymbolForDeclaration(pkg, test)
        case obj if obj.isValDef &&
          symbol.isValDef &&
          !symbol.isPackageDef &&
          obj.tree.asInstanceOf[ValDef].tpt.tpe =:= symbol.tree.asInstanceOf[ValDef].tpt.tpe =>
            Seq.empty
        case obj if obj.isValDef && test(obj.tree.asInstanceOf[ValDef]) =>
          Seq(obj.tree.asInstanceOf[ValDef])
        case obj if obj.isValDef =>
          searchSymbolForDeclaration(obj, test)
        case _ =>
          Seq.empty
      }.flatten
    } catch e => {
      Seq.empty
    }
}
```

Here is a commented version of our macro, explaining each step:
```scala
// A marker annotation so we can have some control over what gets found
final class exposed extends StaticAnnotation

object PackageSearch {
  // the parameter path must be inline so it's value is visible at compiletime
  transparent inline def findByType[T](inline path: String): List[T] =
    // When passing an inline value to a macro, use '{...} to turn it into an Expr
    ${ findByTypeMacro[T]('{ path }) }

  def findByTypeMacro[T : Type](pathExpr: Expr[String])(using q: Quotes): Expr[List[T]] =
    import q.reflect.*

    // Get package name from the string parameter
    // Expressions which represent literals ("hello", 12, true) can be seen by macros
    // But expressions coming from parameters or user input cannot be seen
    // pathExpr.value will return Some(String) if the parameter is a string literal, otherwise None
    val packageName = pathExpr.value match {
      case Some(str) => str
      // This is how you report errors and stop compilation in macros
      // Nothing can be compiled without your explicit blessing
      case None => report.errorAndAbort("Package name is not visible at compiletime")
    }
    
    // Predicate to filter out values
    // A ValDef is the Definition type of when you do `val x = 2`
    // Where x is the name, Int is the tpt (Type Tree) and 2 is the rhs (Right-hand side)
    // Singleton objects are also ValDefs, but they have their own singleton type `object App extends Main {...}`
    def isEligible(valDef: ValDef): Boolean =
      valDef.tpt.tpe <:< TypeRepr.of[T] &&
      valDef.symbol.hasAnnotation(TypeRepr.of[exposed].typeSymbol)

    // We can use this neat method from the Symbol package to find the symbol of the 
    val symbol = Symbol.requiredPackage(packageName)
    // It will return Symbol.noSymbol if the package does not exist, we must log and abort if so
    if(symbol == Symbol.noSymbol) {
      report.errorAndAbort(s"Package by the name of ${packageName} could not be found")
    }

    // We use our utility method below to find ValDefs that match the predicate recursively
    val exprs = searchSymbolForDeclaration(symbol, isEligible)
      .map { valDef =>
        // Ref is a type of Term,
        // To reference our value definition as a term, we reference it's symbol
        // If valDef == `val x = 2`, the definition, then Ref(valDef.symbol) == `x`, the variable
        Ref(valDef.symbol)
      }
      // Make sure to not return the same thing twice if referenced in multiple places
      .distinctBy(_.show)
      .to(List)
      // Turn our terms into Expr[T]s that can be injected into code
      .map(_.asExprOf[T])

    // Turn our list of expressions in an expression of a list
    // This can be injected into the code and the result will be `List(..., ..., ...)`, all values found
    Expr.ofList(exprs)

  // Utility recursive method, could probably be tail recursive but eh
  // Notice that the using clause comes before the actual parameter list
  // This is so we can reference the types on q.reflect in the signature
  def searchSymbolForDeclaration(using q: Quotes)(symbol: q.reflect.Symbol, test: q.reflect.ValDef => Boolean): Seq[q.reflect.ValDef] =
    import q.reflect.*
    
    try {
      // Get the declarations inside our symbol (that could be anything at this point)
      // And check each of them with the collect method on List
      symbol.declarations.collect {
        case obj if obj.fullName.contains("#") =>
          // Some synthetic objects to be disregarded, can't explain it, it's magic
          Seq.empty
        case pkg if pkg.isPackageDef =>
          // If the child symbol is a package, recurse over it's definitions
          searchSymbolForDeclaration(pkg, test)
        case obj if obj.isValDef &&
          symbol.isValDef &&
          !symbol.isPackageDef &&
          obj.tree.asInstanceOf[ValDef].tpt.tpe =:= symbol.tree.asInstanceOf[ValDef].tpt.tpe =>
            // If the child symbol is the same as the parent symbol, stop, we don't want stack overflows
            Seq.empty
        case obj if obj.isValDef && test(obj.tree.asInstanceOf[ValDef]) =>
          // BASE CASE, if our symbol is a ValDef and it satisfies the test predicate, return it's tree
          // as a ValDef
          Seq(obj.tree.asInstanceOf[ValDef])
        case obj if obj.isValDef =>
          // If our symbol is a ValDef of another kind, recurse over it's definitions
          // This means our macro will look inside objects inside objects inside vals inside objects etc...
          searchSymbolForDeclaration(obj, test)
      }.flatten
    } catch e => {
      // The scala compiler throws some really unhelpful and unreadable exceptions sometimes, I suggest a try catch
      Seq.empty
    }
}
```

I understand some of the distinctions between Tree, ValDef, Term, Expr, Symbol, Ref are foggy after reading this, they were for me too, hopefully this helps:
- **Tree** is the abstract base type of language entities
- **Exprs** are a Quotes-agnostic datatype that can be created with '{ some code } and injected using ${ ... }, they represent anything that can be returned from a method
- **Terms** are the Quotes-specific equivalent of Exprs, but they do not take type parameters (yet they know their type parameters at compiletime), they are a type of Tree
- **Symbols** are the visible declarations of packages, classes, methods, variables, they are what you would see if you used `myVar.` and tried tab-completion in an IDE
- **ValDefs** are a type of Tree that represent `val x = 2`-like statements, lazy vals are also here
- **Ref** is a type of Term that references symbols, so if you have hold of a Symbol but you need to return it as a value from your macro, try `Ref(symbol)
