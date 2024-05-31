# Filtering, Sorting, Pagination in ASP.NET Core Web API

## Filtering

Filtering is common in ASP.Net Core. It allows us to return a subset of records rather than the complete set of records.

For example, we want to sort on the ``Field`` property in the ``Record`` entity. We need to add two query parameters to our ``IRecordRepository``, ``RecordRepository`` and ``RecordController`` classes by changing the ``getAll()`` methods.

### RecordsController

```bash
    // GET: https://localhost:1234/api/records
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] string? filterOn, [FromQuery] string? filterQuery)
    {
        // GET data from the database - Domain Model
        var records = await recordRepository.GetAllAsync(filterOn, filterQuery);
    
        // Return the DTO back to the client
        return Ok(mapper.Map<List<RecordDto>>(records));
    }
```

We are retrieving the parameters from the Query string so require the ``[FromQuery]`` attribute. Both parameters can be nullable strings so we can add a ``?`` to the end of the types.

### IRecordRepository

```bash
    Task<List<Record>> GetAllAsync(string? filterOn = null, string? filterQuery = null);
```

### RecordRepository

```bash
    public async Task<List<Record>> GetAllAsync(string? filterOn = null, string? filterQuery = null)
    {
        var records = dbContext.Record.Include("Artist").AsQueryable(); 
    
        // Filtering
        if (string.IsNullOrWhiteSpace(filterOn) == false && string.IsNullOrWhiteSpace(filterQuery) == false)
        {
            if (filterOn.Equals("Field", StringComparison.OrdinalIgnoreCase))
            {
                records = records.Where(r => r.Field.Contains(filterQuery));
            }
        }   
    
        return await records.ToListAsync();
    }
```

**Note:** The ``dbContext`` statement retrieves all ``Artist`` and ``Record`` entities. While this is relatively fast the process of sending all records as JSON to the webpage is very slow. I should be able to mitigate this by filtering, then using pagination to limit the records being sent to the web page.

Note also in ``GetAllAsync()`` that I only have one option to filter and there are a number of other fields that I should be able to filter on.

I can improve on this by moving the filtering to another method so that I don't clutter the ``GetAllAsync()`` method.

Now I can filter on all the properties that require it.

### GetAllAsync()

```bash
    public async Task<List<Record>> GetAllAsync(string? filterOn = null, string? filterQuery = null)
    {
        var records = dbContext.Record.Include("Artist").AsQueryable();

        // Filtering
        if (string.IsNullOrWhiteSpace(filterOn) == false && string.IsNullOrWhiteSpace(filterQuery) == false)
        {
            records = FilterRecords(records, filterOn, filterQuery);
        }
        else 
        {
            records = RemoveTextFields(records);
        }

        return await records.ToListAsync();
    }
```

### FilterRecords()

```bash
    private IQueryable<Record> FilterRecords(IQueryable<Record> records, string filterOn, string filterQuery)
    {
        if (filterOn.Equals("Field", StringComparison.OrdinalIgnoreCase))
        {
            records = records.Where(r => r.Field.Contains(filterQuery));
        }
        else if (filterOn.Equals("Media", StringComparison.OrdinalIgnoreCase))
        {
            records = records.Where(r => r.Media.Contains(filterQuery));
        }
        else if (filterOn.Equals("Review", StringComparison.OrdinalIgnoreCase))
        {
            string pattern = "%" + filterQuery + "%";
            records = records.Where(r => EF.Functions.Like(r.Review, pattern));
        }
        else if (filterOn.Equals("Name", StringComparison.OrdinalIgnoreCase))
        {
            string pattern = "%" + filterQuery + "%";
            records = records.Where(r => EF.Functions.Like(r.Name, pattern));
        }
        else if (filterOn.Equals("ArtistName", StringComparison.OrdinalIgnoreCase))
        {
            string pattern = "%" + filterQuery + "%";
            records = records.Where(r => EF.Functions.Like(r.Artist.Name, pattern));
        }
        else if (filterOn.Equals("Recorded", StringComparison.OrdinalIgnoreCase))
        {
            records = records.Where(r => r.Recorded.Equals(int.Parse(filterQuery)));
        }        
        records = RemoveTextFields(records) 
        return records;
    }
```

### RemoveTextFields()

```bash
    private IQueryable<Record> RemoveTextFields(IQueryable<Record> records)
    {
        foreach (Record record in records) 
        { 
            record.Review = null;
            record.Artist.Biography = null;
        }

        return records;
    }
```

## Sorting

We will sort on the ``Recorded`` field in ``RecordRepository()``.

We will add two more query parameters, ``sortBy`` for the field name and ``isAscending``, a boolean for the sort direction. with ``true`` for ascending.

Once again we need add these query parameters to our ``IRecordRepository``, ``RecordRepository`` and ``RecordController`` classes by changing the ``getAll()`` methods.

### RecordsController

```bash
// GET: https://localhost:1234/api/records?filterOn=field&filterQuery=recorded&sortOn=name&isAscending
[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] string? filterOn, [FromQuery] string? filterQuery, 
    [FromQuery] string?  sortBy, [FromQuery] bool isAscending)
{
    // GET data from the database - Domain Model
    var records = await recordRepository.GetAllAsync(filterOn, filterQuery, sortBy, isAscending);

    // Return the DTO back to the client
    return Ok(mapper.Map<List<RecordDto>>(records));
}
```

### IRecordRepository

```bash
    Task<List<Record>> GetAllAsync(string? filterOn = null, string? filterQuery = null, 
        string? sortBy = null, bool isAscending = true);
```

### RecordRepository

```bash
public async Task<List<Record>> GetAllAsync(string? filterOn = null, string? filterQuery = null, string?
    sortBy = null, bool isAscending = true)
{
    var records = dbContext.Record.Include("Artist").AsQueryable();

    // Filtering
    if (string.IsNullOrWhiteSpace(filterOn) == false && string.IsNullOrWhiteSpace(filterQuery) == false)
    {
        records = FilterRecords(records, filterOn, filterQuery);
    }
    else
    {
        records = RemoveTextFields(records);
    }

    // Sorting
    if (string.IsNullOrWhiteSpace(sortBy) == false)
    {
        if (sortBy.Equals("Recorded", StringComparison.OrdinalIgnoreCase))
        {
            records = isAscending ? records.OrderBy(r => r.Recorded) : records.OrderByDescending(r => r.Recorded);
        }
    }

    return await records.ToListAsync();
}
```

## Pagination
