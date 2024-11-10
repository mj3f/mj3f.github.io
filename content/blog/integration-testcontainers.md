+++
title = "Writing Integration tests with Testcontainers"
date = "2024-11-10T20:02:08Z"
description = "Tutorial for setting up a PostgreSQL Testcontainer for ASP.NET Core API Integration tests"

tags = []
+++

This tutorial will demonstrate how to setup a Testcontainer running PostgreSQL for running integration tests on a ASP.NET Core Web API.
For database access, both Entity Framework and Dapper are covered in this tutorial.

## Prerequisites

- A Web API project setup with a database, and at least one endpoint to test again.
- Docker installed on your machine.

## Setup

Create a new xUnit project, and within it install any dependencies you require (e.g. fluentAssertions, Bogus for fake data, etc.). You also need to add a reference to the Web API project in the test projects csproj file.

Once the project is created, Install the `Testcontainers.PostgreSql` package to be able to run a postgres database in a container.
You'll also need to install the `Microsoft.AspNetCore.Mvc.Testing` package from nuget. This package includes the `WebApplicationFactory` for creating a 'mocked' version of our API, so that we don't need to manually run our API each time we want to run our tests.

You may also need to create an https certificate for your API in you haven't done so already.
To do this, open a terminal, navigate to your API project, and simply run the following command:
`dotnet dev-certs https -ep cert.pfx -p Testy123!`

### WebApplicationFactory

Once the `Microsoft.AspNetCore.Mvc.Testing` package is installed, we can create a class that inherits the WebApplicationFactory. This is where we will setup our Testcontainer, as well as our http client to make the calls to our API.
The class will look like this:

```C#
public class ApiFactory : WebApplicationFactory<IApiMarker>, IAsyncLifetime
{
	public Task InitializeAsync() => Task.CompletedTask;
	public Task DisposeAsync() => Task.CompletedTask;
}
```

Here, the WebApplicationFactory is giving us a bootstrapped API to run in memory for our tests. We inherit from it to override certain behaviours, such as registering our database container and other services required for our tests.

Notice the generic type in the WebApplicationFactory declaration 'IApiMarker'. This lives in the API project, and as the name suggests, it's just a empty interface marker to your API, which enables the WebApplicationFactory to easily find the API it is bootstrapping. You _can_ use the 'Program.cs' file in the API, but it is generally considered a better practice to use a marker interface.

The factory should implement the IAsyncLifetime interface, which implements two methods, `InitializeAsync` and `DisposeAsync`. These async methods are where we are going to initialise and teardown our testcontainer(s) and other resources.

### Creating a Testcontainer

Within the ApiFactory class created in the previous section, create a property for the Postgres container for our database like so:

```C#
private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
    .WithDatabase("<your-db-name>")
    .WithUsername("postgres")
    .WithPassword("postgres")
    .WithPortBinding(5432, 5432)
    .WithWaitStrategy(Wait.ForUnixContainer()
        .UntilPortIsAvailable(5432))
    .Build();
```

Since this container will be running for the lifetime of our tests only, and then spun down when no longer needed, and as they're running locally on our machine, the username and password can be anything, but make sure to set the database name.
The `.WithPortBinding` is not strictly necessary unless you find that your container does not start on the expected port by default. With it, you can explicitly set the port binding, where the first parameter is the host port, and the second being the internal container port that the service is running on.
The `WithWaitStrategy` is just there as a precaution in case a local database is already running on that port, you can leave it out if you wish, but essentially it will wait until the port is available before starting.

### Adding an HTTP client

We also need to setup an HTTP client in order to run our integration tests against our API. I like to set it up within the ApiFactory class, but you can create them in the test file(s) if you prefer.
The WebApplicationFactory provides a method called `CreateClient()` that returns an instance of `HttpClient`. We just need to define a public property in the factory (with a private setter so it can only be assigned by the factory).
`public HttpClient HttpClient { get; private set; }`
The assignment of this property must be done within the `InitializeAsync` method, but more on this in the next section.

### Starting and Stopping our factory resources

For any Testcontainers declared in our factory, to start them up they must be called from the `InitializeAsync` method. In this example, we're using a PostgreSQL container, which includes a `StartAsync` method. For our HttpClient that we defined earlier, this must be created within the `InitializeAsync` method, but **AFTER** the database container is started. The ordering here matters, because if you create the http client **before** starting the db container, your integration tests will fail as they will make the API call before the database is fully up and running, thus causing a database connection failure.

```C#
public async Task InitializeAsync()
{
    await _dbContainer.StartAsync();
    HttpClient = CreateClient(); // This must come AFTER the db is started.
}

public async new Task DisposeAsync()
{
    await _dbContainer.StopAsync();
}
```

Call `StopAsync` on the Testcontainer within the `DisposeAsync` method to ensure the container is torn down when all integration tests have completed. (the 'new' keyword is required in the `DisposeAsync` method to prevent hiding the ValueTask override.)

### Setting up the ORM

#### EF Core - configuring the Testcontainer to work with your DbContext

If your project is using EntityFramework core, follow the steps below:

The Testcontainer needs to be configured with EF Core in order for it to be used for your tests. The WebApplicationFactory allows us to override the `ConfigureTestServices` method, in here we can override registrations to our applications service collection for dependency injection.
In this case we need to remove all registrations of our existing DbContext and register a new one that uses the connection string of our Testcontainer, rather than the APIs one:

```C#
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureTestServices(async services =>
    {
        services.RemoveAll(typeof(AppDbContext));
                services.AddDbContext<AppDbContext>(opt =>
        {
            opt.UseNpgsql(_dbContainer.GetConnectionString(), npgsqlOptions =>
            {
            });
        });
    });
}
```

In my case, I'm using a defined context called 'AppDbContext', with a single DbSet of users. Replace this with your own defined context as required.

```C#
public class AppDbContext : DbContext {
	public DbSet<User> Users { get; set; }
}
```

In the example above, we are removing all registrations of the AppDbcontext that will have occurred in our application code, and registered a new one using the connection string of our created Testcontainer.

#### Dapper

If you're using dapper as your ORM, follow the steps below to get your application to use the Testcontainer:

Create an interface called IDbConnectionFactory, with a method that returns an `IDbConnection`.

```C#
public interface IDbConnectionFactory
{
  Task<IDbConnection> CreateConnectionAsync(CancellationToken cancellationToken = default);
}
```

And then an implementation that uses the db of your choice, in this case it's PostgreSQL:

```C#
public sealed class PostgresDbConnectionFactory(string connectionString) : IDbConnectionFactory
{
    public async Task<IDbConnection> CreateConnectionAsync(CancellationToken cancellationToken = default)
    {        var connection = new NpgsqlConnection(connectionString);
        await connection.OpenAsync(cancellationToken);
        return connection;
    }}
```

Now we just have to remove the APIs DI registration for the IDbConnectionFactory, and then add a new one that points to the test container.
The WebApplicationFactory allows us to override the `ConfigureTestServices` method, in here we can override registrations to our applications service collection for dependency injection.

```C#
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureLogging(logging =>
    {
        logging.ClearProviders();
    });
    builder.ConfigureTestServices(services =>
    {
        services.RemoveAll(typeof(IDbConnectionFactory));
        services.AddSingleton<IDbConnectionFactory>(_ =>
            new PostgresDbConnectionFactory(_dbContainer.GetConnectionString()));
    });}
```

At this point, our ApiFactory should be all setup and ready to go. Now we move on to writing some integration tests!

## Creating an integration test

The following example is testing an API endpoint that takes in user input to create a new user and save it to the database. The `Bogus` package is used to generate some fake but 'realistic' looking data that passes validation. The HttpClient is then used to call the POST create user endpoint in our API, passing in the request data, and waiting for the expected response.
(This test assumes that the inputs provided are valid).

This test class implements the `IClassFixture` interface. This interface comes from xUnit, and it allows for the API factory instance created to be shared among any and all tests that are defined in this class (instead of spinning up a new instance of the API in memory for each test).
The api factory can then be injected into the constructor of the test class, and assigned to a variable for use.

```C#
public class CreateUserEndpointTests : IClassFixture<ApiFactory>
{
    private readonly ApiFactory _apiFactory;
    private readonly Faker<CreateUserRequest> _createUserRequestGenerator;
    private readonly List<Guid> _createdUserIds = [];

    public CreateUserEndpointTests(ApiFactory apiFactory)
    {
	    _apiFactory = apiFactory;
        _createUserRequestGenerator = new Faker<CreateUserRequest>()
            .RuleFor(x => x.Email, f => f.Person.Email)
            .RuleFor(x => x.Password, f => "AbcdefghijklMNOP12345")
            .RuleFor(x => x.Username, f => f.Person.UserName)
            .RuleFor(x => x.DateOfBirth,
	            f => DateOnly.FromDateTime(f.Person.DateOfBirth));
    }

    [Fact]
    public async Task GIVEN_Create_WHEN_DataIsValid_THEN_CreateUser()
    {

	    var request = _createUserRequestGenerator.Generate();

		var response = await _apiFactory.HttpClient.PostAsync(
           "/api/v1/users",
           new StringContent(
           JsonSerializer.Serialize(request),
           Encoding.UTF8,
           "application/json"));

       var stringResponse = await response.Content.ReadAsStringAsync();

	   response.StatusCode.Should().Be(HttpStatusCode.Created);
       stringResponse = stringResponse.Trim('"');
       var createdUserId = Guid.Parse(stringResponse);
       stringResponse.Should().NotBeNullOrEmpty();
    }
}
```

By following all the steps in this tutorial, you should now have a working integration test for your Postgresql database running in a Testcontainer!
