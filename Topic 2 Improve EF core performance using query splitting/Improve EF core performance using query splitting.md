### Improve EF Core performance using Query Splitting

**Read time**: 5 minutes.
##



Hello everyone, today we will discuss the **Query Splitting** feature introduced in EF Core 5. 

Query splitting enables you to instruct Entity Framework to split your SQL query into multiple queries rather than one large query. Let's explore how to leverage query splitting to our advantage.

We'll start with the **SchoolController**. This controller contains a single endpoint named **GetById**, which accepts a School ID from the route and returns the corresponding School object.

```c#
using BlogApp.DBContext;
using BlogApp.Entities;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace BlogApp.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class SchoolController : ControllerBase
    {
        private readonly DataContext _context;

        public SchoolController(DataContext context)
        {
            _context = context;
        }

        [HttpGet(Name = "GetSchoolById")]
        public ActionResult<School> Get(Guid idSchool)
        {
            var result = _context.Schools
                .Where(school => school.Id == idSchool)
                .Include(_ => _.Teachers).Include(_ => _.Students).ToList();

            if (result is null)
            {
                return NotFound($"The school with the id '{idSchool}' was not found.");
            }

            return Ok(result);
             
        }
    }
}
```


As evident here, the **SchoolController** relies on a single dependency, namely the **DataContext**. Within the **GetById** implementation, we access the **Schools** DB Set, including both the **Teachers** and **Students** while retrieving the School by its ID. In case the School is not found, we return a failure result, specifically a 404 not found error. In case of a happy path, we return a new instance of School.

This SQL query showcases the generation by Entity Framework when invoking the **GetById** endpoint.

```sql
SELECT [t].[Id], [t].[Name], [t0].[Id], [t0].[Email], [t0].[FirstName], [t0].[LastName], [t0].[SchoolId], [s0].[Id], [s0].[Email], [s0].[FirstName], [s0].[LastName], [s0].[SchoolId]
FROM (
    SELECT TOP(1) [s].[Id], [s].[Name]
    FROM [Schools] AS [s]
    WHERE [s].[Id] = @__idSchool_0
) AS [t]
LEFT JOIN [Teachers] AS [t0] ON [t].[Id] = [t0].[SchoolId]
LEFT JOIN [Students] AS [s0] ON [t].[Id] = [s0].[SchoolId]
ORDER BY [t].[Id], [t0].[Id]
```
As you can observe, the query retrieves School data based on its ID from the **Schools** table. It executes a join on the **Teachers** table to gather teachers details and further joins the **Students** table to collect information about the students.

EF Core generates a single extensive query encompassing multiple join statements, one for each Include statement. Handling a large amount of data from this query may lead to a phenomenon termed Cartesian Explosion. This could significantly elongate SQL query execution times and occasionally result in timeouts. I recently encountered and resolved this issue, and I'll demonstrate the resolution.

In the EF Core-generated query, each Include statement triggers EF Core to generate a corresponding Join statement. To tackle this, we can use **Query Splitting**. This instructs EF Core to dispatch separate SQL queries for each Include statement, as shown below:


```c#
        [HttpGet(Name = "GetSchoolById")]
        public ActionResult<School> GetById(Guid idSchool)
        {
            var result = _context.Schools
                .AsSplitQuery() // Query Splitting
                .Include(_ => _.Teachers)
                .Include(_ => _.Students)
                .FirstOrDefault(school => school.Id == idSchool);

            if (result is null)
            {
                return NotFound($"The school with the id '{idSchool}' was not found.");
            }

            return Ok(result);
             
        }
```

let's see the generated SQL query by Entity Framework after the use of ***Query Splitting***

```sql
SELECT TOP(1) [s].[Id], [s].[Name]
FROM [Schools] AS [s]
WHERE [s].[Id] = @__idSchool_0
ORDER BY [s].[Id]

SELECT [t0].[Id], [t0].[Email], [t0].[FirstName], [t0].[LastName], [t0].[SchoolId], [t].[Id]
FROM (
    SELECT TOP(1) [s].[Id]
    FROM [Schools] AS [s]
    WHERE [s].[Id] = @__idSchool_0
) AS [t]
INNER JOIN [Teachers] AS [t0] ON [t].[Id] = [t0].[SchoolId]
ORDER BY [t].[Id]


SELECT [s0].[Id], [s0].[Email], [s0].[FirstName], [s0].[LastName], [s0].[SchoolId], [t].[Id]
FROM (
    SELECT TOP(1) [s].[Id]
    FROM [Schools] AS [s]
    WHERE [s].[Id] = @__idSchool_0
) AS [t]
INNER JOIN [Students] AS [s0] ON [t].[Id] = [s0].[SchoolId]
ORDER BY [t].[Id]

```

So, rather than creating a single large query, EF Core generates multiple individual queries, each sent to the database separately.

The initial query retrieves the school object. Once this query completes, the school object is loaded into memory.

Next, Entity Framework executes a query to retrieve data from the Teachers table.

Lastly, EF initiates a third query to fetch information from the Students table.

What are the benefits of this approach?

When dealing with numerous join statements, using multiple individual queries often yields better performance compared to executing all the joins together. I recommend testing both implementations in your applications to determine the most efficient approach.

Another aspect to consider is that EF Core does not dispatch these SQL statements concurrently. This could pose issues if there are concurrent updates to the **Students** or **Teachers** tables, potentially leading to inconsistent results in memory due to concurrency conflicts.

Additionally, each SQL statement sent to the database incurs an extra round-trip over the network. This becomes significant, especially if your database server is distant from your application server. Implementing **query splitting** may affect performance due to these additional round trips.

Now, I'll conduct a simple test by attempting to retrieve a School using an ID that doesn't exist in my database. This test will demonstrate the SQL statement generated by EF Core.

```sql
SELECT TOP(1) [s].[Id], [s].[Name]
FROM [Schools] AS [s]
WHERE [s].[Id] = @__idSchool_0
ORDER BY [s].[Id]
```

**Query Splitting**, as demonstrated here, generates a single query instead of multiple queries. This particular query fetches the **Schools** table. EF Core identifies the null response and optimizes by not sending additional queries.


If you learned something useful, I'd appreciate a like on my LinkedIn post to help it reach more people.











