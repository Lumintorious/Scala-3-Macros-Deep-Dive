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
