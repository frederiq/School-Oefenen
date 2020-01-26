# Cheatsheet Dev 5

Model code needed for One - One:

``` csharp
//In this version, an Actor can only play in one movie.
//Yeah, not exactly ideal but it'll do as explanation.
public class Movie
{
public int Id { get; set; }
public string Title { get; set; }
public int ActorId { get; set; }
}

public class Actor
{
public int Id { get; set; }
public string Name { get; set; }
}

//No Fluent API is needed here.
//This can however still be done (optionally -> not in the exam) to set requirements:
modelBuilder.Entity<OfficeAssignment>()
    .HasKey(t => t.InstructorID);

modelBuilder.Entity<Instructor>()
    .HasRequired(t => t.OfficeAssignment)
    .WithRequiredPrincipal(t => t.Instructor);
```

All classes in Model should be implemented in the DbContext, so it'll be properly set up.
Like so:

```csharp
public class MovieContext : DbContext
{
    DbSet<Movie> Movie {get; set}
    DbSet<Actor> Actor {get; set;}

    public override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //Here all Fluent API code will be set up.
    }
}
```

If you add any other classes in the Model, these should also be added in the MovieContext.

One to Many relationship:

```csharp
//The Movie class has been edited here, to contain a list of Actors.
//In this version, a movie can contain Many Actors, but an Actor may only play in One Movie.
//Actor contains MovieId as Foreign Key to its One Movie.
public class Movie
{
    public int Id { get; set; }
    public string Title { get; set; }
    public List<Actor> Actors { get; set; }
}

public class Actor
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int MovieId { get; set; }
}

//No Fluent API needed here, as in One-to-One.
```

Many-Many relationship:

``` csharp
//These classes need to be modified a bit, as to each contain a list of the join table.
public class Movie
 {
    public int Id { get; set; }
    public string Title { get; set; }
    public virtual List<MovieActor> Actors { get; set; }
}

public class Actor
{
    public int Id { get; set; }
    public string Name { get; set; }
    public virtual List<MovieActor> Movies { get; set; }
}

//To complete the relationship, we need a Join table that contains keys to both classes.
public class MovieActor
{
    public int MovieId { get; set; }
    public Movie Movie { get; set; }
    public int ActorId { get; set; }
    public Actor Actor { get; set; }
}

//This requires the following Fluent API code.
//This code is to be added to the MovieContext:
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<MovieActor>()
        .HasKey(t => new { t.ActorId, t.MovieId });

    modelBuilder.Entity<MovieActor>()
        .HasOne(ma => ma.Movie)
        .WithMany(m => m.Actors)
        .HasForeignKey(ma => ma.MovieId);

    modelBuilder.Entity<MovieActor>()
        .HasOne(ma => ma.Actor)
        .WithMany(m => m.Movies)
        .HasForeignKey(ma => ma.ActorId);
}
```

Bij elke verandering van het Model moeten migrations gemaakt worden.

``` csharp
dotnet ef migrations add <name>
dotnet ef database update
```

With this setup, I want to explain simplified CRUD functions.\
Later on, I'd like to give examples of LINQ with more intricate queries.

I'm still trying to properly find out where these functions are to be used, following the Dev 5 lessons and tests.

To insert data into the database (**Create**) (**in Program.cs**):

``` csharp
using (var db = new MovieContext())
{
    Movie m = new Movie
    {
        Title = "No country for old men",
        Actors = new System.Collections.Generic.List<Actor>
        {
            new Actor {Name = "Tommy Lee"},
            new Actor {Name = "Xavier Berdem"}
        }
    }
    db.Movies.Add(m);
    db.SaveChanges();
}
```

To modify (**Update**) data from the database (**in Program.cs**):

``` csharp
using (var db = new MovieContext())
{
    Movie foundMovie = db.Movies.Find(1);
    Console.WriteLine("Found movie with title: " + foundMovie.Title);
    foundMovie.Title = "White cats, Black cats...";
    db.SaveChanges();
    Console.WriteLine("Title changed");
}
```
