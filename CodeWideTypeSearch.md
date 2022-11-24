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
//       myProject.frontend.department.employee.EmployeeOverviewPage,
//       myProject.frontend.department.employee.EmployeeDetailsPage,
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
