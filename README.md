## Investigacion

### **Arquitectura de la Solución**

1. **SQL Server**:
   - Usado para almacenar los datos, y manejar las transacciones.
   - Se insertan los registros y se mantienen los datos actualizados.

2. **Elasticsearch**:
   - Utilizado para las búsquedas rápidas, optimizadas y la paginación eficiente de los resultados.
   - Se utiliza para realizar búsquedas de texto completo y paginación de los registros.

### **Pasos del flujo de la solución**

1. **Generación de los datos**: Insertamos 1 millón de registros en **SQL Server**.
2. **Elasticsearch**: Al insertar o actualizar registros en **SQL Server**, los mismos registros se indexan en **Elasticsearch** para habilitar la búsqueda rápida.
3. **Paginación y búsqueda**: **Elasticsearch** maneja las búsquedas y paginación de los registros. Solo devuelve los IDs de los registros coincidentes, y luego consultamos **SQL Server** para obtener los detalles completos de esos registros.

### **Estructura de Clases**

1. **Part Entity**: Definimos la clase `Part` que representa la estructura de datos.
2. **PartDto**: DTO para enviar datos optimizados al frontend.
3. **ElasticsearchService**: Servicio para interactuar con **Elasticsearch**.
4. **PartService**: Lógica de negocio para manejar la paginación y los filtros.
5. **PartRepository**: Repositorio para interactuar con **SQL Server**.
6. **PartController**: Controlador que maneja las solicitudes HTTP y delega las acciones al servicio.

---

### **1. `Part` Entity (Core/Entities/Part.cs)**

```csharp
namespace PaginationApp.Core.Entities
{
    public class Part
    {
        public int Id { get; set; }
        public DateTime Date { get; set; }
        public DateTime Timestamp { get; set; }
        public int IntegerValue { get; set; }
        public decimal DecimalValue { get; set; }
        public string ShortString1 { get; set; }
        public string ShortString2 { get; set; }
        public string Description { get; set; }
    }
}
```

**Documentación**: La entidad `Part` representa los datos que se almacenan en SQL Server y se indexan en Elasticsearch. Tiene columnas para la fecha, la hora, valores numéricos y cadenas, lo que permite hacer búsquedas por descripción y otros filtros.

---

### **2. `PartDto` (Models/DTOs/PartDto.cs)**

```csharp
namespace PaginationApp.Models.DTOs
{
    public class PartDto
    {
        public int Id { get; set; }
        public DateTime Date { get; set; }
        public DateTime Timestamp { get; set; }
        public int IntegerValue { get; set; }
        public decimal DecimalValue { get; set; }
        public string ShortString1 { get; set; }
        public string ShortString2 { get; set; }
    }
}
```

**Documentación**: `PartDto` es el objeto de transferencia de datos utilizado para enviar solo la información relevante a la interfaz de usuario. Se omiten detalles complejos como relaciones, para optimizar el rendimiento en el frontend.

---

### **3. `ElasticsearchService` (Services/ElasticsearchService.cs)**

```csharp
using Nest;
using PaginationApp.Core.Entities;

namespace PaginationApp.Services
{
    public class ElasticsearchService
    {
        private readonly ElasticClient _client;

        public ElasticsearchService()
        {
            var settings = new ConnectionSettings(new Uri("http://localhost:9200"))
                .DefaultIndex("parts");

            _client = new ElasticClient(settings);
        }

        // Indexar datos en Elasticsearch
        public async Task IndexPartAsync(Part part)
        {
            await _client.IndexDocumentAsync(part);
        }

        // Buscar partes en Elasticsearch con paginación y filtros
        public async Task<SearchResponse<Part>> SearchPartsAsync(string filter, int pageNumber, int pageSize)
        {
            var response = await _client.SearchAsync<Part>(s => s
                .Query(q => q
                    .Bool(b => b
                        .Should(
                            sh => sh.Match(m => m.Field(f => f.Description).Query(filter)),
                            sh => sh.Range(r => r.Field(f => f.Date).GreaterThanOrEquals("2022-01-01"))
                        )
                    )
                )
                .From((pageNumber - 1) * pageSize)
                .Size(pageSize)
            );

            return response;
        }
    }
}
```

**Documentación**: Este servicio se encarga de interactuar con **Elasticsearch**. Permite indexar registros y realizar búsquedas por filtros como `Description` o `Date`. La paginación se realiza utilizando los parámetros `From` y `Size`.

---

### **4. `PartService` (Services/PartService.cs)**

```csharp
using PaginationApp.Core.Entities;
using PaginationApp.Models.DTOs;
using PaginationApp.Services;

namespace PaginationApp.Services
{
    public class PartService
    {
        private readonly ElasticsearchService _elasticsearchService;

        public PartService(ElasticsearchService elasticsearchService)
        {
            _elasticsearchService = elasticsearchService;
        }

        // Obtener las partes paginadas desde Elasticsearch
        public async Task<List<PartDto>> GetPaginatedPartsAsync(int pageNumber, int pageSize, string? filter = null)
        {
            var elasticSearchResponse = await _elasticsearchService.SearchPartsAsync(filter, pageNumber, pageSize);

            // Mapear los resultados de Elasticsearch a DTOs
            var partDtos = elasticSearchResponse.Documents.Select(p => new PartDto
            {
                Id = p.Id,
                Date = p.Date,
                Timestamp = p.Timestamp,
                IntegerValue = p.IntegerValue,
                DecimalValue = p.DecimalValue,
                ShortString1 = p.ShortString1,
                ShortString2 = p.ShortString2
            }).ToList();

            return partDtos;
        }
    }
}
```

**Documentación**: Esta clase es responsable de la **lógica de negocio**. Obtiene las partes de **Elasticsearch**, mapea los datos a DTOs y los devuelve al controlador para ser enviados al frontend.

---

### **5. `PartRepository` (Repositories/PartRepository.cs)**

```csharp
using PaginationApp.Core.Entities;

namespace PaginationApp.Repositories
{
    public class PartRepository
    {
        private readonly ApplicationDbContext _context;

        public PartRepository(ApplicationDbContext context)
        {
            _context = context;
        }

        // Método para agregar o actualizar una parte en SQL Server y en Elasticsearch
        public async Task AddOrUpdatePartAsync(Part part)
        {
            // Agregar o actualizar el registro en SQL Server
            _context.Parts.Update(part);
            await _context.SaveChangesAsync();

            // Sincronizar el registro con Elasticsearch
            var elasticsearchService = new ElasticsearchService();
            await elasticsearchService.IndexPartAsync(part);
        }
    }
}
```

**Documentación**: Esta clase es responsable de las operaciones de **CRUD** en **SQL Server**. Después de cada operación, también sincroniza los datos con **Elasticsearch** para asegurar que las búsquedas estén actualizadas.

---

### **6. `PartsController` (Controllers/PartsController.cs)**

```csharp
using Microsoft.AspNetCore.Mvc;
using PaginationApp.Models.DTOs;
using PaginationApp.Services;

namespace PaginationApp.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class PartsController : ControllerBase
    {
        private readonly PartService _partService;

        public PartsController(PartService partService)
        {
            _partService = partService;
        }

        // Obtener partes con paginación
        [HttpGet]
        public async Task<IActionResult> GetPaginatedParts(int pageNumber = 1, int pageSize = 10, string? filter = null)
        {
            var parts = await _partService.GetPaginatedPartsAsync(pageNumber, pageSize, filter);
            return Ok(parts);
        }
    }
}
```

**Documentación**: Este controlador expone el endpoint **GET** para que los usuarios soliciten las partes con paginación. El controlador llama al servicio `PartService` para obtener las partes y devolverlas al cliente.

---

### **7. Configuración de `Program.cs`**

```csharp
using Microsoft.EntityFrameworkCore;
using PaginationApp.Services;
using PaginationApp.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Configurar servicios
builder.Services.AddSingleton<ElasticsearchService>(); // Servicio de Elasticsearch
builder.Services.AddScoped<PartService>(); // Lógica de negocio para las partes
builder.Services.AddScoped<PartRepository>(); // Repositorio de las partes

// Configuración de DbContext (SQL Server)
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// Configuración de rutas y middlewares
app.MapControllers();

app.Run();
```

**Documentación**: Aquí configuramos los servicios necesarios, como el **ElasticsearchService**, **PartService** y el **repositorio**. También configuramos **SQL Server** con la cadena de conexión.

---

### **8. Configuración de Base de Datos en `appsettings.json`**

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=PartsCatalogDb;Trusted_Connection=True;"
  },
  "Elasticsearch": {
    "Url": "http://localhost:9200",
    "DefaultIndex": "parts"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

**Documentación**: Este archivo contiene la configuración de las conexiones a **SQL Server** y **Elasticsearch**.

---


