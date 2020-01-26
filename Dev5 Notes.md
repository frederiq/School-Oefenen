# Cheatsheet Dev 5

## Entities and Relationships (Model)

### Model code needed for One - One

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

    publid override void OnConfiguring (
        DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseNpgsql("User ID=postgres;
            Password=;Host=localhost;Port=5432;
            Database=MovieDB;Pooling=true")
            //Dit is een voorbeeld om met een Postgres database te werken.
        }

    public override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //Here all Fluent API code will be set up.
    }
}
```

If you add any other classes in the Model, these should also be added in the MovieContext.

### One to Many relationship

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

### Many-Many relationship

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
Anders is de code niet gelijk aan de database die gebruikt wordt.
Om migrations goed te laten verlopen, moet je ook de connectie met de database goed hebben lopen, met een correcte connectiestring in de DbContext.

``` csharp
dotnet ef migrations add <name>
dotnet ef database update
```

## CRUD functions and queries (Controller)

Later on, I'd like to give examples of LINQ with more intricate queries.

I'm still trying to properly find out where these functions are to be used, following the Dev 5 lessons and tests.

TO properly give examples of queries, we'll need to use a more detailed version of the Model:

```csharp
public class Movie {
    public int Id { get; set; }
    public string Title { get; set; }
    public DateTime Release { get; set; }
    public List<Actor> Actors { get; set; }
}

public class Actor {
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime Birth { get; set; }
    public string Gender { get; set; }
    public int MovieId { get; set; }
    public Movie Movie { get; set; }
}
```

To insert data into the database (**Create**):

``` csharp
static void DataInsertion()
{
    using (var db = new MovieContext()){
        Movie m = new Movie{
            Title = "Divorce Italian Style",
            Release = DateTime.Now,
            Actors = new System.Collections.Generic.List<Actor> {
                new Actor{ Name = "Marcello Mastroianni",
                            Birth = new DateTime(1988, 8, 29),
                            Gender=  "Male",
                            },
                new Actor{ Name = "Daniela Rocca",
                            Birth = new DateTime(1986, 5, 1),
                            Gender=  "Female",
                            }
                    }
            };

        //Add more movies here
        ...

        db.Add(m);
        db.SaveChanges();
        ...
    }
}
```

### To **Read** from the database (and Get the needed results)

An example of a simple query (Projection):

```csharp
var projected_movies = from m in db.Movies select new {m.Title, m.Release};
```

This simply selects the Title and Release from the Movie entity.

To filter (Projection and filtering) in this query a bit more:

```csharp
var projected_movies =  from m in db.Movies
                        where m.Release > new DateTime (2000, 1, 1)
                        select m;
```

To set an order (Projection with ordering) of the results:

```csharp
var projected_movies = from m in db.Movies
                       where m.Release > new DateTime (2000, 1, 1)
                       orderby m.Release descending
                       select m;
```

To order the results into groups (Grouping and aggregation):

```csharp
var projected_movies = from a in db.Actors
                       group a by a.Gender into genderGroup
                       select genderGroup;
```

These groups can also be used in combination with count():

```csharp
var result = from actor in db.Actors
              group actor by actor.Gender into GenderGrp
              select Tuple.Create (
                     GenderGrp.Key,
                     GenderGrp.Count()
              );
```

To select several tables, you'll need to use a subquery.
In LINQ this can be done by using a let variable within the query:

```csharp
var projected_movies =  from movie in db.Movies
                        let actors_of_movie = (
                            from actor in db.Actors
                            where actor.MovieId == movie.Id
                            select actor)
                        where actors_of_movie.Count() < 3
                        select new {
                                Title = movie.Title,
                                ActorsCount = actors_of_movie.Count()
                                };
```

The let can then be used within the 'select new' where a combination of the results can be set up.

### To modify (**Update**) data from the database

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
