## Entity Framework Core 6 features - Part 3

It's a continuation of blog series about EF Core 6 features. Link to [Part 2](https://blog.okyrylchuk.dev/entity-framework-core-6-features-part-2).

### 1. SQLite Supports DateOnly and TimeOnly

SQLite provider supports new *DateOnly* and *TimeOnly* types in EF Core 6.0. It stores them as *TEXT*.

```cs
using var context = new ExampleContext();

var query1 = context.People
    .Where(p => p.Birthday < new DateOnly(2000, 1, 1))
    .ToQueryString();

Console.WriteLine(query1);
// SELECT "p"."Id", "p"."Birthday", "p"."Name"
// FROM "People" AS "p"
// WHERE "p"."Birthday" < '2000-01-01'

var query2 = context.Notifications
    .Where(n => n.AllowedFrom >= new TimeOnly(8, 0) && n.AllowedTo <= new TimeOnly(16, 0))
    .ToQueryString();

Console.WriteLine(query2);
// SELECT "n"."Id", "n"."AllowedFrom", "n"."AllowedTo"
// FROM "Notifications" AS "n"
// WHERE("n"."AllowedFrom" >= '08:00:00') AND("n"."AllowedTo" <= '16:00:00')

class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateOnly Birthday { get; set; }
}
class Notification
{
    public int Id { get; set; }
    public TimeOnly AllowedFrom { get; set; }
    public TimeOnly AllowedTo { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    public DbSet<Notification> Notifications { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite(@"Data Source=Db\DateOnlyTimeOnly.db");
}```

### 2. SQLite Connections Are Pooled

SQLite database is a file. So creating a connection is fast most of the time. However, opening a connection to an encrypted database can be very slow. Thus SQLite connections are now pooled like in other database providers in EF Core 6.

```cs
class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=EncryptedDb.db;Mode=ReadWriteCreate;Password=password");
}
```

### 3. A Command Timeout in SQLite

In EF Core 6 for SQLite, a command timeout has been added to the connection string. SQLite treats *Default Timeout* as a synonym for *Command Timeout*, and it can be used instead if preferred.

```cs
class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }

    // 60 seconds as the default timeout for commands created by connection
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=Test.db;Command Timeout=60");
}
```

### 4. Savepoints in SQLite

In EF Core 6.0, SQLite supports [savepoints](https://docs.microsoft.com/en-us/ef/core/saving/transactions#savepoints). You can Save, Rollback, and Release savepoints.

```cs
var dbPath = Path.GetFullPath(Path.Combine(AppContext.BaseDirectory, "..\\..\\..\\Savepoints.db"));

using var connection = new SqliteConnection($"Data Source={dbPath}");
connection.Open();
using var transaction = connection.BeginTransaction();

// The insert is committed to the database
using (var command = connection.CreateCommand())
{
    command.CommandText = @"INSERT INTO People (Name) VALUES ('Oleg')";
    command.ExecuteNonQuery();
}

transaction.Save("MySavepoint");

// The update is not commited since savepoint is rolled back before commiting the transaction
using (var command = connection.CreateCommand())
{
    command.CommandText = @"UPDATE People SET Name = 'Not Oleg' WHERE Id = 1";
    command.ExecuteNonQuery();
}

transaction.Rollback("MySavepoint");
transaction.Commit();
```

### 5. In-memory Database Validates Required Properties

In EF Core 6.0, the in-memory database validates required properties. The exception will be thrown if you try to save an entity with null values for required properties. You can disable this validation if necessary.

```cs
using var context = new ExampleContext();

var blog = new Blog();
context.Blogs.Add(blog);

await context.SaveChangesAsync();
// Unhandled exception. Microsoft.EntityFrameworkCore.DbUpdateException:
// Required properties '{'Title'}' are missing for the instance of entity
// type 'Blog' with the key value '{Id: 1}'.

class Blog
{
    public int Id { get; set; }
    [Required]
    public string Title { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options
        .EnableSensitiveDataLogging()
        .LogTo(Console.WriteLine, new[] { InMemoryEventId.ChangesSaved })
        .UseInMemoryDatabase("ValidateRequiredProps");

    //  To disable the validation
    //  .UseInMemoryDatabase("ValidateRequiredProps", b => b.EnableNullChecks(false));
}
```

### 6. EF.Functions.Contains with value converters

You can use *EF.Functions.Contains* method with columns mapped using a value converter (also with binary columns) in EF Core 6.0.

```cs
using var context = new ExampleContext();

var query = context.People
    .Where(e => EF.Functions.Contains(e.FullName, "Oleg"))
    .ToQueryString();

Console.WriteLine(query);
// SELECT[p].[Id], [p].[FullName]
// FROM[People] AS[p]
// WHERE CONTAINS([p].[FullName], N'Oleg')

class Person
{
    public int Id { get; set; }
    public FullName FullName { get; set; }
}
public class FullName
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Person>()
            .Property(x => x.FullName)
            .HasConversion(
                    v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null),
                    v => JsonSerializer.Deserialize<FullName>(v, (JsonSerializerOptions)null));
    }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFCore6Contains");
}
```

### Wrapping Up

All code samples you can find on my [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/EF%20Core%206#miscellaneous-enhancements).

