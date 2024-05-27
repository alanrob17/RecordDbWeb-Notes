# Adding the Record Controller and making the Repository more generic

## Record Domain Model

```bash
    public class Record
    {
        #region " Properties "

        public int RecordId { get; set; } // identity field

        public int ArtistId { get; set; } // relate to the artist entity

        public string Name { get; set; }

        public string Field { get; set; }

        public int Recorded { get; set; }

        public string Label { get; set; }

        public string Pressing { get; set; }

        public string Rating { get; set; }

        public int Discs { get; set; }

        public string Media { get; set; }

        public DateTime? Bought { get; set; }

        public decimal? Cost { get; set; }

        public string? CoverName { get; set; }

        public string? Review { get; set; }

        #endregion
    }
```

## Add Data Transfer Objects (DTO's)

### RecordDto

```bash
    public class RecordDto
    {
        public int RecordId { get; set; } 
        public int ArtistId { get; set; } 
        public string Name { get; set; }
        public string Field { get; set; }
        public int Recorded { get; set; }
        public string Label { get; set; }
        public string Pressing { get; set; }
        public string Rating { get; set; }
        public int Discs { get; set; }
        public string Media { get; set; }
        public DateTime? Bought { get; set; }
        public decimal? Cost { get; set; }
        public string? CoverName { get; set; }
        public string? Review { get; set; }
    }
```

### AddRecordDto

```bash
    public class addRecordDto
    {
        public int ArtistId { get; set; }
        public string Name { get; set; }
        public string Field { get; set; }
        public int Recorded { get; set; }
        public string Label { get; set; }
        public string Pressing { get; set; }
        public string Rating { get; set; }
        public int Discs { get; set; }
        public string Media { get; set; }
        public DateTime? Bought { get; set; }
        public decimal? Cost { get; set; }
        public string? CoverName { get; set; }
        public string? Review { get; set; }
    }
```

### UpdateRecordDto

```bash
    public int ArtistId { get; set; }
    public string Name { get; set; }
    public string Field { get; set; }
    public int Recorded { get; set; }
    public string Label { get; set; }
    public string Pressing { get; set; }
    public string Rating { get; set; }
    public int Discs { get; set; }
    public string Media { get; set; }
    public DateTime? Bought { get; set; }
    public decimal? Cost { get; set; }
    public string? CoverName { get; set; }
    public string? Review { get; set; }
```

## Add Record Mappings to AutoMapperProfiles.cs

```bash
    public class AutoMapperProfiles: Profile
    {
        public AutoMapperProfiles()
        {
            CreateMap<Artist, ArtistDto>().ReverseMap();
            CreateMap<AddArtistDto, Artist>().ReverseMap();
            CreateMap<UpdateArtistDto, Artist>().ReverseMap();
            CreateMap<Record, RecordDto>().ReverseMap();
            CreateMap<AddRecordDto, Record>().ReverseMap();
            CreateMap<UpdateRecordDto, Record>().ReverseMap();
        }
    }
```

## Add Records Controller

Create the RecordsController from the API Controller template and add a constructor with the AutoMapper ``IMapper`` interface injected into it.

Build the ``Create()`` method and add the *DTO to Domain Model* mapping.

```bash
    [Route("api/[controller]")]
    [ApiController]
    public class RecordsController : ControllerBase
    {
        private readonly Mapper mapper;

        public RecordsController(RecordDbContext dbContext, Mapper mapper)
        {
            this.mapper = mapper;
        }

        // POST: https://localhost:1234/api/records
        [HttpPost]
        public async Task<IActionResult> Create([FromBody] AddRecordDto addRecordDto)
        {
            // Map DTO to Domain Model
            var record = mapper.Map<Record>(addRecordDto);
        }        
    }
```

We are now at the stage where we need to create the ``IRecordRepository`` and the ``RecordRepository``.

## Create the Record Repository

### IRecordRepository.cs

```bash
    public interface IRecordRepository
    {
        Task<List<Record>> GetAllAsync();

        Task<Record?> GetByIdAsync(int id);

        Task<Record> CreateAsync(Record record);

        Task<Record?> UpdateAsync(int id, Record record);

        Task<Record?> DeleteAsync(int id);
    }
```

### RecordRepository.cs

Create the constructor and inject the RecordDbContext into it. Also build the ``CreateAsync()`` method.

```bash
    private readonly RecordDbContext dbContext;

    public RecordRepository(RecordDbContext dbContext) 
    {
        this.dbContext = dbContext;
    }
    
    public async Task<Record> CreateAsync(Record record)
    {
        await dbContext.Record.AddAsync(record);
        await dbContext.SaveChangesAsync();
    
        return record;
    }
```

Modify the constructor by injecting the ``RecordDbContext`` into it.

```bash
    private readonly IRecordRepository recordRepository;
    private readonly IMapper mapper;
    
    public RecordsController(IRecordRepository recordRepository, IMapper mapper)
    {
        this.recordRepository = recordRepository;
        this.mapper = mapper;
    }
```

Now complete the ``Create()`` method in the ``RecordsController``.

```bash
	// POST: https://localhost:1234/api/records
	[HttpPost]
	public async Task<IActionResult> Create([FromBody] AddRecordDto addRecordDto)
	{
		// Map DTO to Domain Model
		var record = mapper.Map<Record>(addRecordDto);
	
		// Use Domain Model to create Artist
		record = await recordRepository.CreateAsync(record);
	
		// Map Domain model back to DTO
		var recordDto = mapper.Map<RecordDto>(record);
	
		// return CreatedAtAction(nameof(GetById), new { id = recordDto.ArtistId }, recordDto);
		return Ok(recordDto);
	}
```

We don't have a ``GetById()`` method so for the moment we will just return the ``RecordDto``.

Before we can test this method we will have to register ``IRecordRepository`` into ``Program.cs``.

```bash
    builder.Services.AddScoped<IRecordRepository, RecordRepository>();
```

Now we are ready for testing.

Once we have tested that the Create method is working correctly we can build our ``GetAll()`` method in the Records Controller.

```bash
    // GET: https://localhost:1234/api/records
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        // GET data from the database - Domain Model
        var records = await recordRepository.GetAllAsync()  
    
        // Return the DTO back to the client
        return Ok(mapper.Map<List<RecordDto>>(records));
    }
```

We now have to build the ``RecordRepository.GetAllAsync()`` method in the Record Repository.

```bash
    public async Task<List<Record>> GetAllAsync()
    {
        return await dbContext.Record.ToListAsync();
    }
```

Create the ``GetById()`` method in ``RecordsController``.

```bash
    // GET: https://localhost:1234/api/artists/114
    [HttpGet]
    [Route("{id:int}")]
    public async Task<IActionResult> GetById([FromRoute] int id)
    {
        // GET Artist Domain mode from database
        var record = await recordRepository.GetByIdAsync(id);

        if (record == null)
        {
            return NotFound($"An Record with Id: {id} wasn't found!");
        }

        // Return the DTO back to the client 
        return Ok(mapper.Map<RecordDto>(record));
    }
```

Now build the ``recordRepository.GetByIdAsync(id)`` method in the Record Repository.

```bash
    public async Task<Record?> GetByIdAsync(int id)
    {
        return await dbContext.Record.FirstOrDefaultAsync(r => r.RecordId == id);
    }
```

Once we have tested this we can go back to the ``Create()`` method in ``RecordsController`` and finish the ``return`` statement.

```bash
    // POST: https://localhost:1234/api/records
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] AddRecordDto addRecordDto)
    {
        // Map DTO to Domain Model
        var record = mapper.Map<Record>(addRecordDto);

        // Use Domain Model to create Artist
        record = await recordRepository.CreateAsync(record);

        // Map Domain model back to DTO
        var recordDto = mapper.Map<RecordDto>(record);

        return CreatedAtAction(nameof(GetById), new { id = recordDto.ArtistId }, recordDto);
    }
```

### Record Controller Update() method

```bash
    // PUT: https://localhost:1234/api/artists/114
    [HttpPut]
    [Route("{id:int}")]
    public async Task<IActionResult> Update([FromRoute] int id, [FromBody] UpdateRecordDto updateRecordDto)
    {
        // Map DTO to Domain Model
        var record = mapper.Map<Record>(updateRecordDto);

        record = await recordRepository.UpdateAsync(id, record);

        if (record == null)
        {
            return NotFound($"Record with Id: {id} wasn't found!");
        }

        // Return the DTO back to the client 
        return Ok(mapper.Map<RecordDto>(record));
    }
```

### Record Repository UpdateAsync() method

```bash
    public async Task<Record?> UpdateAsync(int id, Record record)
    {
        var existingRecord = await dbContext.Record.FirstOrDefaultAsync(b => b.RecordId == id);

        if (existingRecord == null)
        {
            return null;
        }

        existingRecord.ArtistId = record.ArtistId;
        existingRecord.Name = record.Name;
        existingRecord.Field = record.Field;
        existingRecord.Recorded = record.Recorded;
        existingRecord.Label = record.Label;
        existingRecord.Pressing = record.Pressing;
        existingRecord.Rating = record.Rating;
        existingRecord.Discs = record.Discs;
        existingRecord.Media = record.Media;
        existingRecord.Bought = record.Bought;
        existingRecord.Cost = record.Cost;
        existingRecord.CoverName = record.CoverName;
        existingRecord.Review = record.Review;

        await dbContext.SaveChangesAsync();

        return existingRecord;
    }
```

### Record Controller Delete() method

```bash
    // DELETE: https://localhost:1234/api/artists/114
    [HttpDelete]
    [Route("{id:int}")]
    public async Task<IActionResult> Delete([FromRoute] int id)
    {
        var record = await recordRepository.DeleteAsync(id);

        if (record == null)
        {
            return NotFound($"Record with Id: {id} not found!");
        }

        // Return the DTO back to the client 
        return Ok(mapper.Map<RecordDto>(record));
    }
```

### Record Repository DeleteAsync() method

```bash
    public async Task<Record?> DeleteAsync(int id)
    {
        var record = await dbContext.Record.FindAsync(id);

        if (record == null)
        {
            return null;
        }

        dbContext.Record.Remove(record);
        await dbContext.SaveChangesAsync();

        return record;
    }
```

## Navigation Properties In Entity Framework Core
