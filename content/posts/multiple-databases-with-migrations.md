---
title: "Multiple Databases With Migrations"
date: 2024-01-15T12:15:05+01:00
tags: [ ".net", "ef", "tutorial" ]
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/multiple-database-with-migrations).

I'm kind of having fun with these dotnet deep dives and I've had some ideas pop here and there so I want to keep the theme going. There were a lot of times where I needed to quickly test something and I didn't have the whole database set and the one standard migrations setup wasn't plausible. To further enhance my point, I think that with the way migrations are explained its hard to test new databases since migrations tie you to a database type. That got me interested in the topic of having multiple types of databases configured and use them at demand.

The whole idea is to have one [docker-compose](https://docs.docker.com/compose/) file which can spawn all the services and make use of environment variables to tell my API what setup to use. 

## What are .net migrations?

There is an [article](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli) by Microsoft themselves explaining what are migrations but I wanted to make a few additions. Given the quick pace code develops and the whole [DevSecOps](https://www.redhat.com/en/topics/devops/what-is-devsecops) there is a need to make everything present in source control. I quite like seeing neat repositories where there is everything setup for you. You have your documentation in `docs`. You have your scripts in `scripts` and so on... This is just anoter thing that is missing. Microsoft finds it easier to maintain long running projects by modifying data schemas incrementally. With which I agree. 

If I had a commit `a` and it has some code in it and if I align my database with all migrations present in commit `a` I expect that my API will conform with database and it should work.

## Alternatives

To be quite honest I don't know many alternatives to entity framework that work the same way. There are a number of [ORMs](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) that manage database and code links but none of them work the same way EF does. I've seen projects which try to use EF and [Dapper](https://github.com/DapperLib/Dapper) trying to gain performace from its beautiful reads but as the time flies they fail to migrate Dapper queries to align with code changes meaning in a lot of bugs. The real question is:
> Can EF ecosystem and other ORMs be combined together?

To that I still don't have the answer. I've used [NHibernate](https://nhibernate.info/) which is (you would guess it) inspired by java. It doesn't have any way to manage the schema and if the code changes one would have to track database schema in a different way. You could use [Database Projects](https://learn.microsoft.com/en-us/sql/ssdt/how-to-create-a-new-database-project?view=sql-server-ver16) to track those and it would be okay. The problem of being tied to Microsofts SQL Server then arises since they don't support any other database (who would've thought! 🤔).

## The problem

Put simple: 
> One cannot *easily* have multiple database migrations generated in the same project. But he can do it with multiple projects!

So lets see how to solve that!

## The setup

Lets first talk about the architecture. Generally I've gone for something simple since this is mostly an example. A lot of different setups could work. I wanted to develop a unitifed model without duplication and unified way to use the data and just change the database provider on the change of environment variables.

I wanted to keep track of members and projects they work on. To do that I have three clases:
```c#
public class Project
{
    public int Id { get; set; }

    public string Name { get; set; }

    public ICollection<Assignation> Assignation { get; set; }
}

public class Member
{
    public int Id { get; set; }

    public string FullName { get; set; }

    public string Email { get; set; }

    public ICollection<Assignation> Assignations { get; set; }
}

public class Assignation
{
    public int Id { get; set; }

    public int MemberId { get; set; }
    public Member Member { get; set; }

    public int ProjectId { get; set; }
    public Project Project { get; set; }

    public string Role { get; set; }
}
```
Now, since I said that I wanted to have everything in one place and really don't want to have database specific code if not needed I've chosen the following approach:
> I will have one GeneralContext that will have everything and all other contexts will really just wrap it.

Here it is:
```c#
public interface IDataProvider
{
    DbSet<Member> Members();

    DbSet<Project> Projects();

    DbSet<Assignation> Assignation();

    int SaveChanges();
}

public abstract class GeneralContext<T> : DbContext, IDataProvider where T: DbContext
{
    public static string ConnectionString => Environment.GetEnvironmentVariable("CONNECTION_STRING")!;

    public DbSet<Member> Members {get; set;}

    public DbSet<Project> Projects {get; set;}

    public DbSet<Assignation> Assignation {get; set;}

    DbSet<Member> IDataProvider.Members() => Members;

    DbSet<Project> IDataProvider.Projects() => Projects;

    DbSet<Assignation> IDataProvider.Assignation() => Assignation;

    public GeneralContext(DbContextOptions<T> options) : base(options)
    {
    }

    // Save changes is already implemented

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var members = new Member[] {
            new Member { Id = 1, FullName = "John Doe", Email = "john@doe.com" },
            new Member { Id = 2, FullName = "Simon Piece", Email = "simon@piece.com" },
        };

        var projects = new Project[] {
            new Project { Id = 1, Name = "R&D" },
            new Project { Id = 2, Name = "SDK" }
        };

        var assignations = new Assignation[] {
            new Assignation { Id = 1, Role = "Team Lead", MemberId = members[0].Id, ProjectId = projects[0].Id },
            new Assignation { Id = 2, Role = "Advisor", MemberId = members[0].Id, ProjectId = projects[1].Id },
            new Assignation { Id = 3, Role = "Developer", MemberId = members[1].Id, ProjectId = projects[1].Id },
        };

        modelBuilder.Entity<Member>().HasData(members);
        modelBuilder.Entity<Project>().HasData(projects);
        modelBuilder.Entity<Assignation>().HasData(assignations);

        base.OnModelCreating(modelBuilder);
    }
}
```

Now lets explain everything.
1. `IDataProvider`: this is how I will resolve my dependancies from the database. I wanted my code to be testable so having some kind of interface was needed. I didn't want to waste time with a [repository and unit of work](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application) patterns since they served no point in what I wanted to test out.
2. `GeneralContext<T>`: it needed to have the same constraint as DBContext has just so it can normally resolve from DI. I always like make everything as statically typed as possible (at least where I can) so I'd rather go for generics than to go for casting objects one into another.
3. `OnModelCreating`: is what I used to seed the data. As I said I didn't want to use any database specific code so creating an empty migrations with custom writen sql was not an option for me.
4. `ConnectionString`: as stated above, I want to choose my database via environment variable and then use a connection string for connecting to the source.

I will give example commands of how I tested the app later.

And picking an example concrete database provider to see how I coded it:
```c#
public class SqliteContext : GeneralContext<SqliteContext>
{
    public SqliteContext(DbContextOptions<SqliteContext> options) : base(options)
    {
    }
}

public class SqliteContextCreator : IDesignTimeDbContextFactory<SqliteContext>
{
    public SqliteContext CreateDbContext(string[] args)
    {
        var optionsBuilder = new DbContextOptionsBuilder<SqliteContext>();
        Apply(optionsBuilder);
        return new SqliteContext(optionsBuilder.Options);
    }

    public static void Apply(DbContextOptionsBuilder optionsBuilder) {
        optionsBuilder.UseSqlite(GeneralContext<SqliteContext>.ConnectionString);
    }
}
```

The only thing that maybe needs to be explained here is why do I have a static `Apply` func? Well to answer that we can see how the registration of the databases looks like in the flow of the API:
```c#
var dbOfChoice = Environment.GetEnvironmentVariable("DB_PROVIDER")!;
switch (dbOfChoice.ToLower())
{
    case "mssql": 
        builder.Services.AddDbContextPool<MssqlContext>(MssqlContextCreator.Apply);
        builder.Services.AddScoped<IDataProvider, MssqlContext>();
        builder.Services.AddHostedService(x => new Migrator<MssqlContext>(x.GetRequiredService<ILogger<Migrator<MssqlContext>>>(), x.GetRequiredService<IServiceScopeFactory>()));
        break;
    case "postgre":
        builder.Services.AddDbContextPool<PostgresContext>(PostgresContextCreator.Apply);
        builder.Services.AddScoped<IDataProvider, PostgresContext>();
        builder.Services.AddHostedService(x => new Migrator<PostgresContext>(x.GetRequiredService<ILogger<Migrator<PostgresContext>>>(), x.GetRequiredService<IServiceScopeFactory>()));
        break;
    case "sqlite":
        builder.Services.AddDbContextPool<SqliteContext>(SqliteContextCreator.Apply);
        builder.Services.AddScoped<IDataProvider, SqliteContext>();
        builder.Services.AddHostedService(x => new Migrator<SqliteContext>(x.GetRequiredService<ILogger<Migrator<SqliteContext>>>(), x.GetRequiredService<IServiceScopeFactory>()));
        break;
    default:
        throw new Exception("Unsupported option. Supported options include ['mssql', 'postgre', 'sqlite']");
}
```
As you can see each DB provider has its own setting up of options which I didn't want to duplicate. One more question arises and that is what is a `Migrator`? Looking back at my ['Using .NET EF Context concurrently'](/posts/concurrent-ef-context) example I really didn't like how the migration was handled so I decided to separate it and add some more logic here. Some of the stuff can be made even better but the sqlite package didn't support all the functionality since its a local file... Anyway on the repository you can see the concrete code for it since its of no real value here, basically it just auto migrates the code so I don't have to since I am lazy and I tend to forget stuff before going on different environments.

## End result

Given that you have an instance of Mssql and Postgres deployed somewhere you can use this code in the following manner:
1. sqlite:
```bash
DB_PROVIDER='sqlite' CONNECTION_STRING='Filename=db.sqlite' dotnet run --project WebApi/
```
2. mssql:
```bash
DB_PROVIDER='mssql' CONNECTION_STRING='Server=host,port;Database=db;User=sa;password=Strong_Password_123;TrustServerCertificate=True' dotnet run --project WebApi
```
3. postgre:
```bash
DB_PROVIDER='postgre' CONNECTION_STRING='User ID=postgres;Password=password;Server=localhost;Port=5432;Database=db' dotnet run --project WebApi
```

## Conclusion
I've achieved what I wanted and really this is a good way to test new databases against the same code. I would imagine some kind of this code in stress testing where one would see how different databases handle the same code and kind of pick their poison. As stated everywhere in my writing I don't think this is a silver bullet and needed for a lot of projects. If you kind of know your enemy you should prepare for him. I had the opportunity to work in R&D teams where we were tasked to *explore*. This kind of code would help us a lot really since we could easily test the same code against different sets of databases and determine what would be the best solution at the given moment. Furthermore flexibility to just drop a db and continue on the next one really is amazing. Of course, one would need custom scripts for migrating the data from one db to another but a bunch of those already exist.

Until next time, cheers!