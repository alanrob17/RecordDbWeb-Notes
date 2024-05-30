# Validation in ASP.NET Core Web API

## Model Validation

We are at the stage where our API is built and anyone can consume it with any language. These applications are speaking to our API endpoints. Now that we are letting clients create or update data we have to validate the models that they are sending in and this is what Model Validation is all about. We need to have Model Validation on the endpoints we expose that receive data from the client. For example, we have our Create and Update methods so we need to check that the data in these properties is valid. If it is valid we can then proceed. If it isn't we need to send back a ``BadRequest()`` or **400** response. By sending a ``BadRequest()`` to the client we are asking them to correct the data before they can submit again.

## Adding Model Validation to our Endpoints

We need to add Model Validation to the endpoints that are exposed to the client which is through the controllers and that is where we will add the validation.

We will start with the ``ArtistsController``. In here we want to validate the endpoints that are accepting data. This will be ``Create()`` and ``Update()`` endpoints.

We do Model Validation using data annotations on the ``AddArtistDto`` and the ``UpdateArtistDto`` models.

```bash
    public class AddArtistDto
    {
        [MaxLength(50, ErrorMessage = "Artist FirstName can't be larger than 50 characters!")]
        public string? FirstName { get; set; }

        [Required]
        [MaxLength(50, ErrorMessage ="Artist LastName can't be larger than 50 characters!")]
        public string LastName { get; set; } // not null

        [Required]
        [MaxLength(50, ErrorMessage ="Artist LastName can't be larger than 50 characters!")]
        public string Name { get; set; }

        public string? Biography { get; set; }
    }
```

The ``UpdateArtistDto`` class is identical to this.

We also add data annotations on the ``AddRecordDto`` and the ``UpdateRecordDto`` models.

```bash
    public class AddRecordDto
    {
        [Required]
        public int ArtistId { get; set; } // relate to the artist entity

        [Required]
        [MaxLength(80, ErrorMessage = "Record Name can't be larger than 80 characters!")]
        public string Name { get; set; }

        [Required]
        [MaxLength(50, ErrorMessage = "Field can't be larger than 50 characters!")]
        public string Field { get; set; }

        [Required]
        [Range(1900, 2024, ErrorMessage = "Recorded must be between 1900 and 2024!")]
        public int Recorded { get; set; }

        [Required]
        [MaxLength(50, ErrorMessage = "Label can't be larger than 50 characters!")]
        public string Label { get; set; }

        [Required]
        [MaxLength(50, ErrorMessage = "Pressing can't be larger than 50 characters!")]
        public string Pressing { get; set; }

        [Required]
        [MinLength(1, ErrorMessage = "Rating can't be less than 1 character!")]
        [MaxLength(4, ErrorMessage = "Rating can't be larger than 4 characters!")]
        public string Rating { get; set; }

        [Required]
        [Range(1, 10, ErrorMessage = "Discs must be between 1 and 10!")]
        public int Discs { get; set; }

        [Required]
        [MaxLength(50, ErrorMessage = "Media can't be larger than 50 characters!")]
        public string Media { get; set; }

        public DateTime? Bought { get; set; }

        public decimal? Cost { get; set; }

        public string? CoverName { get; set; }

        public string? Review { get; set; }
    }
```

The ``UpdateRecordDto`` class is identical to this.

Now that we have defined our Model Validation we can work on the ``ArtistsController``.

### ArtistsController - Create() method

```bash
    // POST: https://localhost:1234/api/artists
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] AddArtistDto addArtistDto)
    {
        if (ModelState.IsValid)
        {
            // Map DTO to Domain Model
            var artist = mapper.Map<Artist>(addArtistDto);
    
            // Use Domain Model to create Artist
            artist = await artistRepository.CreateAsync(artist);
    
            // Map Domain model back to DTO
            var artistDto = mapper.Map<ArtistDto>(artist);
            return CreatedAtAction(nameof(GetById), new { id = artistDto.ArtistId }, artistDto);
        }
    
        return BadRequest(ModelState);
    }
```

## Custom Action Filter

There is a better way of doing validation and that is using a custom action filter. This will allow us to remove the manual validation checking that we have to add to each ``Create()`` and ``Update()`` method.

Create a new Folder, ``CustomActionFilters`` with a class, ``ValidateModelAttribute``.

We inherit from ``ActionFilterAttribute`` and override the ``OnActionExecuting()`` method, 

Add the ``BadRequest()`` result.

```bash
    public class ValidateModelAttribute: ActionFilterAttribute
    {
        public override void OnActionExecuting(ActionExecutingContext context)
        {
            context.Result = new BadRequestResult();
        }
    }
```

Now we add the ``[ValidateModel]`` attribute to all of our ``Create()`` and ``Update()`` controller methods.

Example.

```bash
    // POST: https://localhost:1234/api/artists
    [HttpPost]
    [ValidateModel]
    public async Task<IActionResult> Create([FromBody] AddArtistDto addArtistDto)
    {
        ...
```

If we enter.

```bash
{
  "artistId": 860,
  "name": "Bass To The Max!.......................................................................................................................................................................................................................................................................................................................................",
  "field": "Rock",
  "recorded": 2026,
  "label": "Rubber Dubber",
  "pressing": "Am",
  "rating": "*****",
  "discs": 16,
  "media": "CD",
  "bought": "2024-05-30",
  "cost": -19.99,
  "coverName": null,
  "review": "This is James first album."
}
```

It returns the following ``BadRequest()`` error message.

```bash
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Name": [
      "Record Name can't be larger than 80 characters!"
    ],
    "Discs": [
      "Discs must be between 1 and 10!"
    ],
    "Rating": [
      "Rating can't be larger than 4 characters!"
    ],
    "Recorded": [
      "Recorded must be between 1900 and 2024!"
    ]
  },
  "traceId": "00-a4d59d2549a9ae21cb21b3d50704747d-dea060317dcaebfb-00"
}
```
