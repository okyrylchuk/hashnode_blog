## Entity Framework Core 6 features - Part 1

### 1. Unicode Attribute

A new Unicode attribute in EF Core 6.0 allows you to map a string property to a non-Unicode column without specifying the database type directly. The Unicode attribute is ignored when the database system supports only Unicode types.

```cs
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }

    [Unicode(false)]
    [MaxLength(22)]
    public string Isbn { get; set; }
}
``` 
The migration:
```cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Books",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
            Title = table.Column<string>(type: "nvarchar(max)", nullable: true),
            Isbn = table.Column<string>(type: "varchar(22)", unicode: false, maxLength: 22, nullable: true)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Books", x => x.Id);
        });
}
```

### 2. Precision Attribute

Before EF Core 6.0 you could configure precision and scale with Fluent API. Now you can do it also using Data Annotations with a new attribute Precision.

```cs
public class Product
{
    public int Id { get; set; }

    [Precision(precision: 10, scale: 2)]
    public decimal Price { get; set; }
}
```
The migration:
```cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Products",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                .Annotation("SqlServer:Identity", "1, 1"),
            Price = table.Column<decimal>(type: "decimal(10,2)", precision: 10, scale: 2, nullable: false)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Products", x => x.Id);
        });
}
```

### 3. EntityTypeConfiguration Attribute

Starting with EF Core 6.0, you can place a new EntityTypeConfiguration attribute on the entity type, so EF Core can find and use the appropriate configuration. Before that, the class configuration had to be instantiated and called from the *OnModelCreating* method.

```cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.Property(p => p.Name).HasMaxLength(250);
        builder.Property(p => p.Price).HasPrecision(10, 2);
    }
}
[EntityTypeConfiguration(typeof(ProductConfiguration))]
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }
    public string Name { get; set; }
} 
```

### 4. Column Attribute

When you use inheritance in models, you can be not satisfied with the default EF Core columns order in the created tables. In EF Core 6.0, you can specify columns order with ColumnAttribute.

Also, you can do it using a new Fluent API - *HasColumnOrder()*.

```cs
public class EntityBase
{
    [Column(Order = 1)]
    public int Id { get; set; }
    [Column(Order = 99)]
    public DateTime UpdatedOn { get; set; }
    [Column(Order = 98)]
    public DateTime CreatedOn { get; set; }
}
public class Person : EntityBase
{
    [Column(Order = 2)]
    public string FirstName { get; set; }
    [Column(Order = 3)]
    public string LastName { get; set; }
    public ContactInfo ContactInfo { get; set; }
}
public class Employee : Person
{
    [Column(Order = 4)]
    public string Position { get; set; }
    [Column(Order = 5)]
    public string Department { get; set; }
}
[Owned]
public class ContactInfo
{
    [Column(Order = 10)]
    public string Email { get; set; }
    [Column(Order = 11)]
    public string Phone { get; set; }
}
```
The migration:
```cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Employees",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                .Annotation("SqlServer:Identity", "1, 1"),
            FirstName = table.Column<string>(type: "nvarchar(max)", nullable: true),
            LastName = table.Column<string>(type: "nvarchar(max)", nullable: true),
            Position = table.Column<string>(type: "nvarchar(max)", nullable: true),
            Department = table.Column<string>(type: "nvarchar(max)", nullable: true),
            ContactInfo_Email = table.Column<string>(type: "nvarchar(max)", nullable: true),
            ContactInfo_Phone = table.Column<string>(type: "nvarchar(max)", nullable: true),
            CreatedOn = table.Column<DateTime>(type: "datetime2", nullable: false),
            UpdatedOn = table.Column<DateTime>(type: "datetime2", nullable: false)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_Employees", x => x.Id);
        });
}
```

### 5. Temporal Tables
EF Core 6.0 supports SQL Server temporal tables. A table can be configured as a temporal table with the SQL Server defaults for the timestamps and history table.

```cs
public class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder
            .Entity<Person>()
            .ToTable("People", b => b.IsTemporal());
    }
    protected override void OnConfiguring(DbContextOptionsBuilder options)
            => options.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=TemporalTables;Trusted_Connection=True;");
}
public class Person 
{ 
    public int Id { get; set; }
    public string Name { get; set; }    
}
```
The migration:
```cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "People",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                .Annotation("SqlServer:Identity", "1, 1"),
            Name = table.Column<string>(type: "nvarchar(max)", nullable: true),
            PeriodEnd = table.Column<DateTime>(type: "datetime2", nullable: false)
                .Annotation("SqlServer:IsTemporal", true)
                .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
                .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart"),
            PeriodStart = table.Column<DateTime>(type: "datetime2", nullable: false)
                .Annotation("SqlServer:IsTemporal", true)
                .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
                .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart")
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_People", x => x.Id);
        })
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "PersonHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");
}
```
You can query or retrieve historical data with methods:

- TemporalAsOf
- TemporalAll
- TemporalFromTo
- TemporalBetween
- TemporalContainedIn

Using temporal tables:
```cs
using ExampleContext context = new();
context.People.Add(new() { Name = "Oleg" });
context.People.Add(new() { Name = "Steve" });
context.People.Add(new() { Name = "John" });
await context.SaveChangesAsync();

var people = await context.People.ToListAsync();
foreach (var person in people)
{
    var personEntry = context.Entry(person);
    var validFrom = personEntry.Property<DateTime>("PeriodStart").CurrentValue;
    var validTo = personEntry.Property<DateTime>("PeriodEnd").CurrentValue;
    Console.WriteLine($"Person {person.Name} valid from {validFrom} to {validTo}");
}
// Output:
// Person Oleg valid from 06-Nov-21 17:50:39 PM to 31-Dec-99 23:59:59 PM
// Person Steve valid from 06-Nov-21 17:50:39 PM to 31-Dec-99 23:59:59 PM
// Person John valid from 06-Nov-21 17:50:39 PM to 31-Dec-99 23:59:59 PM
```
Querying historical data:
```cs
var oleg = await context.People.FirstAsync(x => x.Name == "Oleg");
context.People.Remove(oleg);
await context.SaveChangesAsync();
var history = context
                .People
                .TemporalAll()
                .Where(e => e.Name == "Oleg")
                .OrderBy(e => EF.Property<DateTime>(e, "PeriodStart"))
                .Select(
                    p => new
                    {
                        Person = p,
                        PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                        PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                    })
                .ToList();
foreach (var pointInTime in history)
{
    Console.WriteLine(
        $"Person {pointInTime.Person.Name} existed from {pointInTime.PeriodStart} to {pointInTime.PeriodEnd}");
}

// Output:
// Person Oleg existed from 06-Nov-21 17:50:39 PM to 06-Nov-21 18:11:29 PM
```
Retrieving historical data:
```cs
var removedOleg = await context
    .People
    .TemporalAsOf(history.First().PeriodStart)
    .SingleAsync(e => e.Name == "Oleg");

Console.WriteLine($"Id = {removedOleg.Id}; Name = {removedOleg.Name}");
// Output:
// Id = 1; Name = Oleg
```
More about [temporal tables](https://devblogs.microsoft.com/dotnet/prime-your-flux-capacitor-sql-server-temporal-tables-in-ef-core-6-0/).

### 6. Sparse Columns
EF Core 6.0 supports SQL Server sparse columns. It can be useful when using TPH (table-per-hierarchy) inheritance mapping.

```cs
public class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    public DbSet<Employee> Employees { get; set; }
    public DbSet<User> Users { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder
            .Entity<User>()
            .Property(e => e.Login)
            .IsSparse();

        modelBuilder
            .Entity<Employee>()
            .Property(e => e.Position)
            .IsSparse();
    }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
            => options.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=SparseColumns;Trusted_Connection=True;");
}

public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}
public class User : Person
{
    public string Login { get; set; }
}
public class Employee : Person
{
    public string Position { get; set; }
}
```
The migration:
```cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "People",
        columns: table => new
        {
            Id = table.Column<int>(type: "int", nullable: false)
                .Annotation("SqlServer:Identity", "1, 1"),
            Name = table.Column<string>(type: "nvarchar(max)", nullable: false),
            Discriminator = table.Column<string>(type: "nvarchar(max)", nullable: false),
            Position = table.Column<string>(type: "nvarchar(max)", nullable: true)
                .Annotation("SqlServer:Sparse", true),
            Login = table.Column<string>(type: "nvarchar(max)", nullable: true)
                .Annotation("SqlServer:Sparse", true)
        },
        constraints: table =>
        {
            table.PrimaryKey("PK_People", x => x.Id);
        });
}
```

Sparse columns have limitations. Link to the [documentation](https://docs.microsoft.com/en-us/sql/relational-databases/tables/use-sparse-columns?view=sql-server-ver15).

### 7. Minimal API in EF Core
EF Core 6.0 gets its own minimal APIs. New extension methods register a DbContext type and supply a database provider's configuration in a single line.

```cs
const string AccountKey = "[CosmosKey]";

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSqlServer<MyDbContext>(@"Server = (localdb)\mssqllocaldb; Database = MyDatabase");

// OR
builder.Services.AddSqlite<MyDbContext>("Data Source=mydatabase.db");

// OR
builder.Services.AddCosmos<MyDbContext>($"AccountEndpoint=https://localhost:8081/;AccountKey={AccountKey}", "MyDatabase");

var app = builder.Build();
app.Run();

class MyDbContext : DbContext
{ }
```

### 8. Migration Bundles
A new DevOps-friendly feature in EF Core 6.0 - Migration Bundles. It allows creating a small executable containing migrations. You can use it in CD. It doesn’t require you to copy source code or install the .NET SDK (only the runtime).

CLI:
```
dotnet ef migrations bundle --project MigrationBundles
```

Package Manager Console:
```
Bundle-Migration
```

![tweet.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1642345446277/W5FtP2-iZ.png)

More about [Migration Bundles](https://devblogs.microsoft.com/dotnet/introducing-devops-friendly-ef-core-migration-bundles/).

### 9. Pre-convention Model Configuration
EF Core 6.0 introduces a pre-convention model configuration. It allows you to specify mapping configuration once for a given type. It could be helpful, for instance, when working with value objects.

```cs
public class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }
    public DbSet<Product> Products { get; set; }

    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        configurationBuilder
            .Properties<string>()
            .HaveMaxLength(500);

        configurationBuilder
            .Properties<DateTime>()
            .HaveConversion<long>();

        configurationBuilder
            .Properties<decimal>()
            .HavePrecision(12, 2);

        configurationBuilder
            .Properties<Address>()
            .HaveConversion<AddressConverter>();
    }
}
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }
}
public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
    public Address Address { get; set; }
}
public class Address
{
    public string Country { get; set; }
    public string Street { get; set; }
    public string ZipCode { get; set; }
}
public class AddressConverter : ValueConverter<Address, string>
{
    public AddressConverter()
        : base(
            v => JsonSerializer.Serialize(v, (JsonSerializerOptions)null),
            v => JsonSerializer.Deserialize<Address>(v, (JsonSerializerOptions)null))
    {
    }
}
```

### 10. Compiled Models
In EF Core 6.0, you can generate compiled models. The feature makes sense when you have a large model and your EF Core startup is slow. You can do it using CLI or Package Manager Console.

```cs
public class ExampleContext : DbContext
{
    public DbSet<Person> People { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    { 
        options.UseModel(CompiledModelsExample.ExampleContextModel.Instance)
        options.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=SparseColumns;Trusted_Connection=True;"); 
    }
}
public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

CLI:
```
dotnet ef dbcontext optimize -c ExampleContext -o CompliledModels -n CompiledModelsExample
```
Package Manager Console:
```
Optimize-DbContext -Context ExampleContext -OutputDir CompiledModels -Namespace CompiledModelsExample
```

More about [Compiled Models](https://devblogs.microsoft.com/dotnet/announcing-entity-framework-core-6-0-preview-5-compiled-models/) and [their limitations](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-6.0/whatsnew#limitations).

### Wrapping Up

All code samples you can find on my [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/EF%20Core%206#miscellaneous-enhancements).