# Consuming our Web API

## Add an MVC Core project

Choose the ASP.Net Core Web App (Model-View-Controller) template.

Name the project ``RecordDb.UI``.

We have created two projects in our solution we need to run both of them at the same time.

Right-click on the Solution and select properties.

Select ``Multiple Startup`` projects.

Change both actions to **Start**.

![Multiple startup projects](assets/images/startup.jpg "Multiple startup projects")

You should see the following view to start multiple projects.

![Multiple startup projects](assets/images/multiple-startup.jpg "Multiple startup projects")

Click ``Start`` to run both projects. This should show you the MVC and API projects in your browser.

## GET - Artists

Create a new ``ArtistsController.cs`` in our UI project.

This is the template.

```bash
    public class ArtistsController : Controller
    {
        public IActionResult Index()
        {
            return View();
        }
    }
```

Right-click on ``View()``

Select **Add View** and select the empty Razor view named ``Index.cshtml``.

This creates a view in the structure ``Views-->Artists-->Index.cshtml``.

In the ``ArtistsController`` we want to get all ``Artists`` from the API and pass that data to the ``View()``.

The ``HttpClient`` class is used to send HTTP requests and receive HTTP responses from a resource identified by a URI.

[HttpClient class web page.](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient?view=net-8.0)

To use the ``HttpClient`` we have to inject the ``HttpClient`` factory inside ``Program.cs`` so that it can create the ``HttpClient`` for us.

Add the ``HttpClient`` into the Services section.

```bash
    // Add services to the container.
    builder.Services.AddControllersWithViews();

    builder.Services.AddHttpClient();

    var app = builder.Build();
```

Now in the ``ArtistsController`` add a constructor to inject the ``HttpClient``.

```bash
    private readonly IHttpClientFactory httpClientFactory;

    public ArtistsController(IHttpClientFactory httpClientFactory)
    {
        this.httpClientFactory = httpClientFactory;
    }
```

In the API go to Properties-->LaunchSettings to get the URL, in our case, <https://localhost:7262>.

```bash
    [HttpGet]
    public async Task<IActionResult> Index()
    {
        var client = httpClientFactory.CreateClient();

        var response = await client.GetAsync("https://localhost:7262/api/artists");
        
        response.EnsureSuccessStatusCode();

        return View();
    }
```

**Note:** we shouldn't hardcode the URL in our code. This should be returned from ``appsettings.cs``.

``client.GetAsync()`` returns a response message so we capture this with a variable (``response``). It should also return a Success status code so ASP.Net has a method to get this result (``response.EnsureSuccessStatusCode()``).

THis could cause an exception of no data is sent back so we need to wrap this code in a ``try-catch`` block. We can do this with a ``Ctrl K-S`` keystroke and then we select the try-catch option.

```bash
    [HttpGet]
    public async Task<IActionResult> Index()
    {
        try
        {
            var client = httpClientFactory.CreateClient();

            var response = await client.GetAsync("https://localhost:7262/api/artists");

            response.EnsureSuccessStatusCode();

        }
        catch (Exception ex)
        {
            // log the exception
        }

        return View();
    }
```

If we reach the ``response.EnsureSuccessStatusCode();`` statement we know that we have data so we can capture it with.

```bash
    var body = await response.Content.ReadAsStringAsync();
    
    ViewBag.Response = body;
```

### Index.cshtml

```bash
    <h3 class="mt-3">Artists</h3>

    @if (ViewBag.Response is not null)
    {
        <p>@ViewBag.Response</p>    
    }
```

This returns a string that we can print to the browser. 

Use this link to see your data.

> <https://localhost:7102/artists>

In the ``_Layout.cshtml`` layout page change the Privacy link to. This saves you having to put the link into the browser.

```bash
<li class="nav-item">
    <a class="nav-link text-dark" asp-area="" asp-controller="Artists" asp-action="Index">Artists</a>
</li>
```

Returns the first 15 artists.

![String results](assets/images/string-results.jpg "String results")

This isn't the way we should view our output. We should use the ``Artist`` object to store the data and then return it in a table output.

In the ``Models`` folder create a ``DTO`` folder and add an ``ArtistDto`` class with the fields from the ``ArtistQueryDto`` class. Remember that we created a cut down version of the class in the API that removed the ``Biography`` field.

Change the ``ArtistsController-Index()`` method.

```bash
    [HttpGet]
    public async Task<IActionResult> Index()
    {
        List<ArtistDto> response = new List<ArtistDto>();

        try
        {
            var client = httpClientFactory.CreateClient();

            var responseMessage = await client.GetAsync("https://localhost:7262/api/artists");

            responseMessage.EnsureSuccessStatusCode();

            response.AddRange(await responseMessage.Content.ReadFromJsonAsync<IEnumerable<ArtistDto>>());
        }
        catch (Exception ex)
        {
            // log the exception
        }

        return View(response);
    }
```

Now change the ``View()`` page.

```bash
@model IEnumerable<RecordDb.UI.Models.DTO.ArtistDto>

<h3 class="mt-3">Artists</h3>

<table class="table table-bordered mt-3">
    <thead>
        <tr>
            <th>Id</th>
            <th>First Name</th>
            <th>Last Name</th>
            <th>Name</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var artist in Model)
        {
            <tr>
                <td>@artist.ArtistId</td>
                <td>@artist.FirstName</td>
                <td>@artist.LastName</td>
                <td>@artist.Name</td>
                <td>
                    <a asp-controller="Artists" asp-action="Edit" asp-route-id="@artist.ArtistId"
                       class="btn btn-light">Edit</a>
                </td>
            </tr>
        }
    </tbody>
</table>
```

Returns.

![Artist table view](assets/images/artist-table-view.jpg "Artist table view")

## Add an Artist

In the Index page we will add a link to add a new ``Artist`` record.

```bash
<h3 class="mt-3">Artists</h3>

<div class="d-flex justify-content-end">
    <a class="btn btn-secondary" asp-controller="Artists" asp-action="Add">Add Artist</a>
</div>
```

Add a new ``Add()`` method to the ArtistsController. This will be another ``HttpGet`` method that is only used to view the ``Add`` form.

```bash
    [HttpGet]
    public IActionResult Add()
    {
        return View();
    }
```

Once we fill in the fields as required we can create an ``HttpPost`` Add() method to save the data.

```bash
[HttpPost]
public async Task<IActionResult> Add(AddArtistViewModel model)
{
    var client = httpClientFactory.CreateClient();

    var httpRequestMessage = new HttpRequestMessage() 
    { 
        Method = HttpMethod.Post,
        RequestUri = new Uri("https://localhost:7262/api/artists"),
        Content = new StringContent(JsonSerializer.Serialize(model), Encoding.UTF8, "application/json")
    };

    var httpResponseMessage = await client.SendAsync(httpRequestMessage);

    httpResponseMessage.EnsureSuccessStatusCode();

    var response = await httpResponseMessage.Content.ReadFromJsonAsync<ArtistDto>();

    if (response is not null)
    {
        return RedirectToAction("Index", "Artists");
    }

    return View(response);
}
```
