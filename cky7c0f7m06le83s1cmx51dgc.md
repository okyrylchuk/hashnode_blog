## LINQ Enhancements in Entity Framework Core 6

In this post, I focus on LINQ query enhancements in Entity Framework Core 6.

### Better Support for GroupBy Queries
EF Core 6.0 has better support for GroupBy queries.

- Translates GroupBy followed by FirstOrDefault over a group
- Expands navigations after the GroupBy
- Supports selecting the top N results from a group

```cs
using var context = new ExampleContext();
var query = context.People
    .GroupBy(p => p.FirstName)
    .Select(g => g.OrderBy(e => e.FirstName)
                  .ThenBy(e => e.LastName)
                  .FirstOrDefault())
    .ToQueryString();
Console.WriteLine(query);

class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public int LastName { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6GroupBy");
}
``` 
Translated SQL:
```sql
SELECT[t0].[Id], [t0].[FirstName], [t0].[LastName]
FROM (
SELECT[p].[FirstName]
   FROM [People] AS [p]
   GROUP BY [p].[FirstName]
) AS[t]
LEFT JOIN(
   SELECT[t1].[Id], [t1].[FirstName], [t1].[LastName]
   FROM (
       SELECT[p0].[Id], [p0].[FirstName], [p0].[LastName],
       ROW_NUMBER() OVER(PARTITION BY [p0].[FirstName]
       ORDER BY [p0].[FirstName], [p0].[LastName]) AS[row]
       FROM[People] AS[p0]
   ) AS[t1]
   WHERE[t1].[row] <= 1
) AS[t0] ON[t].[FirstName] = [t0].[FirstName]
```

### String.Concat Translation with 3 and 4 Arguments

Previously EF Core translated *string.Concat* only with two arguments. EF Core 6.0 translates *string.Concat* with 3 and 4 arguments.

```cs
using var context = new ExampleContext();
string fullName = "SamuelLanghorneClemens";
var query = context.Blogs
    .Where(b => string.Concat(b.FirstName, b.MiddleName, b.LastName) == fullName)
    .ToQueryString();
Console.WriteLine(query);

class Blog
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string MiddleName { get; set; }
    public string LastName { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6StringConcat");
}
```
Translated SQL:

```sql
DECLARE @__fullName_0 nvarchar(4000) = N'SamuelLanghorneClemens';

SELECT[b].[Id], [b].[FirstName], [b].[LastName], [b].[MiddleName]
FROM[Blogs] AS[b]
WHERE(COALESCE([b].[FirstName], N'') + (COALESCE([b].[MiddleName], N'') +COALESCE([b].[LastName], N ''))) = @__fullName_0
```

### EF.Functions.FreeText Supports Binary Columns

Previously, you couldn't use *EF.Functions.FreeText* method on binary columns despite the SQL FreeText function supporting them. EF Core 6.0 fixes that.

```cs
using var context = new ExampleContext();
var query = context.Posts
    .Where(p => EF.Functions.FreeText(EF.Property<string>(p, "Content"), "Searching text"))
    .ToQueryString();
Console.WriteLine(query);

class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public byte[] Content { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Post> Posts { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Post>()
            .Property(x => x.Content)
            .HasColumnType("varbinary(max)");
    }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6FlexibleTextSearch");
}
```
Translated SQL:
```sql
SELECT[p].[Id], [p].[Content], [p].[Title]
FROM[Posts] AS[p]
WHERE FREETEXT([p].[Content], N'Searching text')
```

### Translate ToString on SQLite

Starting with EF Core 5.0, *ToString* translation for SQL Server has been added. EF Core 6.0 also translates *ToString* for SQLite database provider. It can be helpful for text searches on non-string columns.

```cs
using var context = new ExampleContext();
var query = context.People
    .Where(u => EF.Functions.Like(u.PhoneNumber.ToString(), "%368%"))
    .ToQueryString();
Console.WriteLine(query);

class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public long PhoneNumber { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=:memory:");
}
```
Translated SQL:
```sql
SELECT "p"."Id", "p"."Name", "p"."PhoneNumber"
FROM "People" AS "p"
WHERE CAST("p"."PhoneNumber" AS TEXT) LIKE '%368%'
```

### EF.Functions.Random

EF Core 6.0 introduces a new *EF.Functions.Random* method. It maps SQL function *RAND()*. Translations have been implemented for SQL Server, SQLite, and Cosmos.

```cs
using var context = new ExampleContext();
var query = context.Posts
    .Where(p => p.Rating == (int)(EF.Functions.Random() * 5.0) + 1)
    .ToQueryString();
Console.WriteLine(query);

class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Rating { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Post> Posts { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6Random");
}
```
Translated SQL:
```sql
SELECT[p].[Id], [p].[Rating], [p].[Title]
FROM[Posts] AS[p]
WHERE[p].[Rating] = (CAST((RAND() * 5.0E0) AS int) + 1)
```
### Improved SQL Server Translation for IsNullOrWhitespace

Previously, EF Core translated *string.IsNullOrWhiteSpace* into a trimming of value before checking if it's empty. EF Core 6.0 doesn't do trimming anymore.

```cs
using var context = new ExampleContext();
var query = context.Entities
                    .Where(e => string.IsNullOrWhiteSpace(e.Property))
                    .ToQueryString();
Console.WriteLine(query);

class Entity
{
    public int Id { get; set; }
    public string Property { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Entity> Entities { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6IsNullOrWhiteSpace");
}
```
Previously translated SQL:
```sql
SELECT [e].[Id], [e].[Property]
FROM [Entities] AS[e]
WHERE [e].[Property] IS NULL OR (LTRIM(RTRIM([e].[Property])) = N'')
```
Translated SQL now:
```sql
SELECT [e].[Id], [e].[Property]
FROM [Entities] AS[e]
WHERE [e].[Property] IS NULL OR ([e].[Property] = N'')
```

### Defining Query for In-memory Provider

In EF Core 6.0, you can define a query against the in-memory database for a given type with a new method *ToInMemoryQuery*. This is most useful for creating the equivalent of views on the memory databases.

```cs
using var context = new ExampleContext();
var blogEn = new Blog
{
    Title = "All about .NET",
    Language = "English",
    Posts = new List<Post>
        {
            new Post { Title = "Post one", Content = "Some content" },
            new Post { Title = "Post two", Content = "Some content" }
        }
};
var blogPl = new Blog
{
    Title = "Wszystko o .NET",
    Language = "Polish",
    Posts = new List<Post>
        {
            new Post { Title = "Pierwszy post", Content = "Treść" }
        }
};
context.Blogs.Add(blogEn);
context.Blogs.Add(blogPl);
await context.SaveChangesAsync();

var postsByLanguages = context.PostsByLanguages.ToList();
postsByLanguages
    .ForEach(p => Console.WriteLine($"{p.PostCount} posts in {p.Language}"));
// Output:
// 2 posts in English
// 1 posts in Polish

class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
}
class Blog
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Language { get; set; }
    public ICollection<Post> Posts { get; set; }
}
class PostsByLanguage
{
    public string Language { get; set; }
    public int PostCount { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Post> Posts { get; set; }
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<PostsByLanguage> PostsByLanguages { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder
                .Entity<PostsByLanguage>()
                .HasNoKey()
                .ToInMemoryQuery(
                    () => Blogs
                        .GroupBy(c => c.Language)
                        .Select(
                            g =>
                                new PostsByLanguage
                                {
                                    Language = g.Key,
                                    PostCount = g.Sum(b => b.Posts.Count)
                                }));
    }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseInMemoryDatabase("ToInMemoryQuery");
}
```

### Substring Translation with a Single Parameter

Previously EF Core translated only *string.Substring* overload with two parameters.
EF Core 6.0 translates *string.Substring* with a single parameter.

```cs
using var context = new ExampleContext();
context.People.Add(new Person { Name = "John" });
context.People.Add(new Person { Name = "Bred" });
context.People.Add(new Person { Name = "Ron" });
await context.SaveChangesAsync();

var result = await context.People
    .Select(a => new { Name = a.Name.Substring(1) })
    .ToListAsync();
result.ForEach(p => Console.WriteLine(p.Name));
// Output:
// ohn
// red
// on

class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6Substring");
}
```
Translated SQL:
```sql
SELECT SUBSTRING([p].[Name], 1 + 1, LEN([p].[Name])) AS [Name]
FROM [People] AS [p]
```

### Split-queries for Non-navigation Collections
EF Core supports splitting a single LINQ query into multiple SQL queries. EF Core 6.0 can split a LINQ query where non-navigation collections are contained in the query projection.

```cs
using var context = new ExampleContext();
var blog = new Blog { Name = ".NET Blog"};
blog.Posts.Add(new Post { Title = "First .NET post" });
blog.Posts.Add(new Post { Title = "Second Java post" });
blog.Posts.Add(new Post { Title = "Third .NET post" });
context.Blogs.Add(blog);
await context.SaveChangesAsync();

var blogsWithDotnetPosts = await context.Blogs
    .Select(b => new
    {
        b,
        Posts = b.Posts.Where(p => p.Title.Contains(".NET")),
    })
    .AsSplitQuery()
    .ToListAsync();

class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}
class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public Blog Blog { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options
        .UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6SplitQueries");
}
```
Single SQL query:
```sql
SELECT [b].[Id], [b].[Name], [t].[BlogId], [t].[Title]
FROM [Blogs] AS [b]
LEFT JOIN (
     SELECT [p].[Id], [p].[BlogId], [p].[Title]
     FROM [Posts] AS [p]
     WHERE [p].[Title] LIKE N'%.NET%'
) AS [t] ON [b].[Id] = [t].[BlogId]
ORDER BY [b].[Id]
```
Multiple SQL queries:
```sql
SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
ORDER BY [b].[Id]

SELECT [t].[Id], [t].[BlogId], [t].[Title], [b].[Id]
FROM [Blogs] AS [b]
INNER JOIN (
     SELECT [p].[Id], [p].[BlogId], [p].[Title]
     FROM [Posts] AS [p]
     WHERE [p].[Title] LIKE N'%.NET%'
) AS [t] ON [b].[Id] = [t].[BlogId]
ORDER BY [b].[Id]
```

### Remove Last ORDER BY Clause

When joining related entities, EF Core adds ORDER BY clauses to ensure all related entities for a given entity are grouped together. However, the last clause is not necessary and can have an impact on performance. EF Core 6.0 removes it.

```cs
using var context = new ExampleContext();
var query = context.Blogs
    .Include(b => b.Posts.Where(p => p.Rating > 3))
    .ToQueryString();
Console.WriteLine(query);

class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Post> Posts { get; set; }
}
class Post 
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Rating { get; set; }
    public Blog Blog { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6RemoveLastOrderByClause");
}
```

Translated SQL by EF Core 5.0:
```sql
SELECT [b].[Id], [b].[Name], [t].[Id], [t].[BlogId], [t].[Rating], [t].[Title]
FROM [Blogs] AS [b]
LEFT JOIN (
    SELECT [p].[Id], [p].[BlogId], [p].[Rating], [p].[Title]
    FROM [Posts] AS [p]
    WHERE [p].[Rating] > 3
) AS [t] ON [b].[Id] = [t].[BlogId]
ORDER BY [b].[Id], [t].[Id]
```
Translated SQL by EF Core 6.0:
```sql
SELECT [b].[Id], [b].[Name], [t].[Id], [t].[BlogId], [t].[Rating], [t].[Title]
FROM [Blogs] AS [b]
LEFT JOIN (
    SELECT [p].[Id], [p].[BlogId], [p].[Rating], [p].[Title]
    FROM [Posts] AS [p]
    WHERE [p].[Rating] > 3
) AS [t] ON [b].[Id] = [t].[BlogId]
ORDER BY [b].[Id]
```

### Tag Queries with the File Name and Line Number
Starting with EF Core 2.2, you could add a tag to your query for better debug purposes. EF Core 6.0 went further, and now you can tag queries with the filename and line number of the LINQ code.

```cs
using var context = new ExampleContext();
var query = context.Blogs
    .TagWithCallSite()
    .OrderBy(b => b.CreationDate)
    .Take(10)
    .ToQueryString();
Console.WriteLine(query);

class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime CreationDate { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6TagWithCallSite");
}
```
Translated SQL:
```sql
DECLARE @__p_0 int = 10;

--File: D:\EFCore6\TagWithCallSite\TagWithCallSite\Program.cs:6

SELECT TOP(@__p_0) [b].[Id], [b].[CreationDate], [b].[Name]
FROM[Blogs] AS[b]
ORDER BY[b].[CreationDate] 
```

### Changes to Owned Optional Dependent Handling
EF Core 6.0 introduces some changes for owned optional dependent handling. When a model has owned optional dependent, EF Core will warn you when you save it with all missing properties.

```cs
using var context = new ExampleContext();
var person = new Person
{
    FirstName = "Oleg",
    LastName = "Kyrylchuk",
    Address = new Address()
};
context.People.Add(person);
await context.SaveChangesAsync();

class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public Address Address { get; set; }
}
class Address
{
    public string City { get; set; }
    public string Street { get; set; }
    public string PostalCode { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder
            .Entity<Person>()
            .OwnsOne(p => p.Address);
    }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options
        .EnableSensitiveDataLogging()
        .LogTo(Console.WriteLine)
        .UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6OwnedDependentHandling");
}
```
Warning in logs:
![warning log.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1641736235673/-yeBMlffD.png)

When you have nested owned optional dependents, EF Core will not allow creating the model at all.
```cs
using var context = new ExampleContext();
var person = new Person
{
   FirstName = "Oleg",
   LastName = "Kyrylchuk",
   ContactInfo = new ContactInfo()
};
context.People.Add(person);
await context.SaveChangesAsync();

class Person
{
   public int Id { get; set; }
   public string FirstName { get; set; }
   public string LastName { get; set; }
   public ContactInfo ContactInfo { get; set; }
}
class ContactInfo
{
   public string Phone { get; set; }
   public Address Address { get; set; }
}
class Address
{
   public string City { get; set; }
   public string Street { get; set; }
   public string PostalCode { get; set; }
}
class ExampleContext : DbContext
{
   public DbSet<Person> People { get; set; }
   protected override void OnModelCreating(ModelBuilder modelBuilder)
   {
       modelBuilder
           .Entity<Person>()
           .OwnsOne(p => p.ContactInfo)
           .OwnsOne(p => p.Address);
   }
   protected override void OnConfiguring(DbContextOptionsBuilder options)
       => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6OwnedDependentHandling");
}
```
An exception is thrown when the model is created:
![error log.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1641736381115/3siMj7Tz1.png)

The changes force you to avoid such cases.
You can fix them by:
- making dependent required,
- making at least one required property in the dependent,
- creating own tables for optional dependents, instead of sharing them with the principal.

### Wrapping Up

All code samples you can find on my  [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/EF%20Core%206#linq-query-enhancements).

