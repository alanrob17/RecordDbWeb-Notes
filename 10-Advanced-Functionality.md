# 10-Advanced Functionality in ASP.NET Core Web APIs

## Logging

Is used to capture and store information about the applications behavior and operation during runtime.

Logging allows the developer to see how their application is functioning and provides valuable insights into how issues may arise.

We will use a 3rd party application named Serilog to carry out logging.

### Packages

* Serilog
* Serilog.AspNetCore
* Serilog.Sinks.Console
* Serilog.Sinks.File

### Logging to the Console window

Once again you have to inject it into your application.

In ``Program.cs`` add logging above the controller section.

```bash
    var logger = new LoggerConfiguration()
        .WriteTo.Console()
        .MinimumLevel.Warning()
        .CreateLogger();    
    
    builder.Logging.ClearProviders();
    builder.Logging.AddSerilog(logger);
```

In our configuration we are logging all **Warning** issues. We have a number of other options available including Information, Error and Fatal.

The ``ClearProviders()`` method will clear any other logging methods that we used in the project.

The ``AddSerilog(logger)`` method adds logging to our project using the ``logger`` configuration.

Now that we have injected Serilog into our services we can use it by injecting it into our Controllers constructor.

To inject the logger it must have a type an that should be the controllers name. In our case we will add it to the ``ArtistsController``.

```bash
    private readonly ILogger<ArtistsController> logger; 

    public ArtistsController(IArtistRepository artistRepository, IMapper mapper, ILogger<ArtistsController> logger)
    {
        this.artistRepository = artistRepository;
        this.mapper = mapper;
        this.logger = logger;
    }
```

We will use logging in the ``GetAll()`` method.

```bash
    logger.LogInformation("In the Artists GetAll() controller method...");
```

This will show the following Console messages

> [12:08:07 WRN] No store type was specified for the decimal property 'Cost' on entity type 'Record'. This will cause values to be silently truncated if they do not fit in the default precision and scale. Explicitly specify the SQL server column type that can accommodate all the values in 'OnModelCreating' using 'HasColumnType', specify precision and scale using 'HasPrecision', or configure a value converter using 'HasConversion'.      
> ...

You can see that it is only showing Warning messages so change ``Program.cs`` to log Information messages.

Now you see this.

> [12:13:48 INF] In the Artists GetAll() controller method...       
> [12:13:48 WRN] No store type was specified for the decimal property 'Cost' on entity type 'Record'. This will cause values to be silently     
> ...

We can use ``JsonSerializer`` (``System.Text.Json``) to output the results. Add the following to ``GetAll()``.

```bash
	// GET data from the database - Domain Model
	var artists = await artistRepository.GetAllAsync(filterOn, filterQuery, sortBy, isAscending ?? true, pageNumber, pageSize);
	
	logger.LogInformation($"Finished GetAll()\n {JsonSerializer.Serialize(artists)}");
```

Returns.

> [12:21:56 INF] In the Artists GetAll() controller method...		
> [12:21:57 INF] Finished GetAll()		
> [{"ArtistId":1,"FirstName":"William","LastName":"Ackerman","Name":"William Ackerman","Biography":"\u003Cp\u003EWill Ackerman...

ASP.Net Core logs all database requests so I have removed these from the output.

The interesting part of the output to the console is that at this stage I can see the Biographies. These don't make it to the web page.

### Throw an exception

```bash
    try
    {
        throw new Exception("Throw exception...");

        // GET data from the database - Domain Model
        var artists = await artistRepository.GetAllAsync(filterOn, filterQuery, sortBy, isAscending ?? true, pageNumber,    pageSize);

        logger.LogInformation($"Finished GetAll()\n {JsonSerializer.Serialize(artists)}");

        // Return the DTO back to the client
        return Ok(mapper.Map<List<ArtistQueryDto>>(artists));
    }
    catch (Exception ex)
    {
        logger.LogError(ex, ex.Message);
        throw;
    }
```

Returns.

> [16:47:20 ERR] In catch!:     
> System.Exception: Throw exception...
   at RecordDb.API.Controllers.ArtistsController.GetAll(String filterOn, String filterQuery, String sortBy, Nullable`1 isAscending, Int32 pageNumber, Int32 pageSize) in D:\Projects\RecordDbWeb\RecordDb.API\Controllers\ArtistsController.cs:line 37

### Logging to a text file

#### Package

* Serilog.Sinks.File

Create the **Logs** folder in the API.

change ``Program.cs``.

```bash
    var logger = new LoggerConfiguration()
        .WriteTo.Console()
        .WriteTo.File("Logs/recorddbLog.txt", rollingInterval: RollingInterval.Day)
        .MinimumLevel.Warning()
        .CreateLogger();
```

We have added the ``.WriteTo.File()`` method. Try ``GetAll()`` and this is written to the log file.

> 2024-06-02 17:18:01.378 +10:00 [ERR] Hmmmmn...!       
> System.Exception: Hmmmmn...!      
   at RecordDb.API.Controllers.ArtistsController.GetAll(String filterOn, String filterQuery, String sortBy, Nullable`1 isAscending, Int32 pageNumber, Int32 pageSize) in D:\Projects\RecordDbWeb\RecordDb.API\Controllers\ArtistsController.cs:line 38      
> 2024-06-02 17:18:01.440 +10:00 [ERR] An unhandled exception has occurred while executing the request.
> System.Exception: Hmmmmn...!      
   at RecordDb.API.Controllers.ArtistsController.GetAll(String filterOn, String filterQuery, String sortBy, Nullable`1

## Global Exception Handling

* Handles any unexpected exceptions that occur during the processing of a request
* By centralising your exceptions you can create a consistent error response format
* This will avoid having to add exception handling code in your methods

Normally we would handle exceptions by adding ``Try-Catch`` blocks into our code. This gets repetitive if we have to add this to all our endpoints.

It is a much better idea to add global error handling. We are going to use this as a middleware class.

Create a folder named, ``Middleware`` and create a new class named, ``ExceptionHandler``.

Create a constructor and inject the following.

```bash
    public class ExceptionHandler
    {
        public ILogger<ExceptionHandler> Logger { get; }
        public RequestDelegate Next { get; }

        public ExceptionHandler(ILogger<ExceptionHandler> logger, RequestDelegate next)
        {
            Logger = logger;
            Next = next;
        }
    }
```

``RequestDelegate`` returns a **task** that represents the completion of a request processing. So this completes the Http Request.

we are going to create an ``InvokeAsync()`` method so that we can use this request delegate to pass the context.

```bash
    public async Task InvokeAsync(HttpContext httpContext)
    {
        try
        {
            await next(httpContext);
        }
        catch (Exception ex)
        {
            var errorId = Guid.NewGuid();
            logger.LogError(ex, $"{errorId} : {ex.Message}");

            // return a custom error response
            httpContext.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
            httpContext.Response.ContentType = "application/json";

            var error = new
            {
                Id = errorId,
                ErrorMessage = "Something went wrong! We are looking into resolving this."
            };

            await httpContext.Response.WriteAsJsonAsync(error);
        }
    }
```

Now we are ready to inject this piece of middleware into the middleware request pipeline in ``Program.cs``.

After the ``Swagger()`` class in the request pipeline add this.

```bash
    app.UseMiddleware<ExceptionHandler>();
```

So now we have added this middleware into our request pipeline so it is ready to be used.

Go to the ``ArtistsController`` and add an exception.

```bash
    throw new Exception("Heck, something has broken!");
```

Now run the application and see the results.

> [18:41:55 ERR] 57a80c5d-bb8f-436a-a13a-2cc4b47f8cbf : Heck, something has broken!     
> System.Exception: Heck, something has broken!
   at RecordDb.API.Controllers.ArtistsController.GetAll(String filterOn, String filterQuery, String sortBy, Nullable`1 isAscending, Int32 pageNumber, Int32 pageSize) in D:\Projects\RecordDbWeb\RecordDb.API\Controllers\ArtistsController.cs:line 36

We are now able to catch any exceptions that happen in our controllers, repositories or other classes.
