# API-barber-biling

Abaixo está um projeto completo em C# (.NET 7) para a API de faturamento da barbearia, com:
- CRUD de faturamentos (Billings)
- Relatório semanal exportável em PDF (QuestPDF) e Excel (ClosedXML)
- Validações com FluentValidation
- AutoMapper para DTOs
- Entity Framework Core com SQLite (exemplo)
- Tratamento global de exceções
- Testes de integração com xUnit e WebApplicationFactory (SQLite in-memory)
- Instruções para rodar

Estrutura e arquivos estão apresentados sequencialmente. Crie a pasta do projeto (por exemplo BarberBilling) e adicione os arquivos conforme abaixo. Em seguida explico como construir e executar.

Observações sobre versões:
- .NET 7 SDK
- Pacotes NuGet usados (serão listados no .csproj):
  - Microsoft.EntityFrameworkCore.Sqlite
  - Microsoft.EntityFrameworkCore.InMemory (para testes)
  - FluentValidation.AspNetCore
  - AutoMapper.Extensions.Microsoft.DependencyInjection
  - QuestPDF
  - ClosedXML
  - Swashbuckle.AspNetCore (Swagger)
  - xunit, Microsoft.AspNetCore.Mvc.Testing, FluentAssertions para testes

1) BarberBilling.csproj
Create a project file for the API.

<File: BarberBilling/BarberBilling.csproj>
```
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="7.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="7.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="FluentValidation.AspNetCore" Version="11.5.0" />
    <PackageReference Include="AutoMapper.Extensions.Microsoft.DependencyInjection" Version="12.0.1" />
    <PackageReference Include="QuestPDF" Version="2024.4.0" />
    <PackageReference Include="ClosedXML" Version="0.105.0" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  </ItemGroup>

  <ItemGroup>
    <Folder Include="Migrations\" />
  </ItemGroup>

</Project>
```

2) Program.cs
Minimal API bootstrapping, DI, middlewares, Swagger, and DB migration on startup.

<File: BarberBilling/Program.cs>
```
using BarberBilling.Data;
using BarberBilling.Middleware;
using BarberBilling.Services;
using FluentValidation;
using FluentValidation.AspNetCore;
using Microsoft.EntityFrameworkCore;
using AutoMapper;
using BarberBilling.Profiles;
using BarberBilling.Validators;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers().AddJsonOptions(opts =>
{
    opts.JsonSerializerOptions.PropertyNamingPolicy = null;
});
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// EF Core with SQLite
var connectionString = builder.Configuration.GetValue<string>("ConnectionStrings:Default") ?? "Data Source=barberbilling.db";
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(connectionString));

// AutoMapper
builder.Services.AddAutoMapper(typeof(BillingProfile));

// FluentValidation
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<BillingCreateValidator>();

// Services
builder.Services.AddScoped<IReportService, ReportService>();

var app = builder.Build();

// Middleware
app.UseMiddleware<ExceptionMiddleware>();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

// Ensure DB created
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}

app.Run();
```

3) appsettings.json

<File: BarberBilling/appsettings.json>
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "Default": "Data Source=barberbilling.db"
  },
  "AllowedHosts": "*"
}
```

4) Data: AppDbContext and migrations will be created by EF.

<File: BarberBilling/Data/AppDbContext.cs>
```
using BarberBilling.Models;
using Microsoft.EntityFrameworkCore;

namespace BarberBilling.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> opts) : base(opts) { }

        public DbSet<Billing> Billings => Set<Billing>();

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Billing>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Amount).HasColumnType("decimal(18,2)");
                entity.Property(e => e.Date).IsRequired();
                entity.Property(e => e.ClientName).IsRequired().HasMaxLength(200);
            });
        }
    }
}
```

5) Models: Billing and PaymentMethod enum

<File: BarberBilling/Models/Billing.cs>
```
using System.ComponentModel.DataAnnotations;

namespace BarberBilling.Models
{
    public class Billing
    {
        public Guid Id { get; set; }
        public DateTime Date { get; set; }
        public string ClientName { get; set; } = null!;
        public string Service { get; set; } = null!;
        public decimal Amount { get; set; }
        public PaymentMethod PaymentMethod { get; set; }
        public string? Notes { get; set; }
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    }
}
```

<File: BarberBilling/Models/PaymentMethod.cs>
```
namespace BarberBilling.Models
{
    public enum PaymentMethod
    {
        Cash = 0,
        Card = 1,
        Pix = 2,
        Other = 99
    }
}
```

6) DTOs

<File: BarberBilling/Dtos/BillingCreateDto.cs>
```
using BarberBilling.Models;

namespace BarberBilling.Dtos
{
    public class BillingCreateDto
    {
        public DateTime Date { get; set; }
        public string ClientName { get; set; } = null!;
        public string Service { get; set; } = null!;
        public decimal Amount { get; set; }
        public PaymentMethod PaymentMethod { get; set; }
        public string? Notes { get; set; }
    }
}
```

<File: BarberBilling/Dtos/BillingUpdateDto.cs>
```
using BarberBilling.Models;

namespace BarberBilling.Dtos
{
    public class BillingUpdateDto
    {
        public DateTime Date { get; set; }
        public string ClientName { get; set; } = null!;
        public string Service { get; set; } = null!;
        public decimal Amount { get; set; }
        public PaymentMethod PaymentMethod { get; set; }
        public string? Notes { get; set; }
    }
}
```

<File: BarberBilling/Dtos/BillingReadDto.cs>
```
using BarberBilling.Models;

namespace BarberBilling.Dtos
{
    public class BillingReadDto
    {
        public Guid Id { get; set; }
        public DateTime Date { get; set; }
        public string ClientName { get; set; } = null!;
        public string Service { get; set; } = null!;
        public decimal Amount { get; set; }
        public PaymentMethod PaymentMethod { get; set; }
        public string? Notes { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
    }
}
```

7) AutoMapper profile

<File: BarberBilling/Profiles/BillingProfile.cs>
```
using AutoMapper;
using BarberBilling.Dtos;
using BarberBilling.Models;

namespace BarberBilling.Profiles
{
    public class BillingProfile : Profile
    {
        public BillingProfile()
        {
            CreateMap<BillingCreateDto, Billing>()
                .ForMember(dest => dest.Id, opt => opt.Ignore())
                .ForMember(dest => dest.CreatedAt, opt => opt.Ignore())
                .ForMember(dest => dest.UpdatedAt, opt => opt.Ignore());

            CreateMap<BillingUpdateDto, Billing>()
                .ForMember(dest => dest.Id, opt => opt.Ignore())
                .ForMember(dest => dest.CreatedAt, opt => opt.Ignore());

            CreateMap<Billing, BillingReadDto>();
        }
    }
}
```

8) Validators (FluentValidation)

<File: BarberBilling/Validators/BillingCreateValidator.cs>
```
using BarberBilling.Dtos;
using FluentValidation;

namespace BarberBilling.Validators
{
    public class BillingCreateValidator : AbstractValidator<BillingCreateDto>
    {
        public BillingCreateValidator()
        {
            RuleFor(x => x.ClientName).NotEmpty().MaximumLength(200);
            RuleFor(x => x.Service).NotEmpty().MaximumLength(200);
            RuleFor(x => x.Amount).GreaterThan(0);
            RuleFor(x => x.Date).NotEmpty();
        }
    }
}
```

<File: BarberBilling/Validators/BillingUpdateValidator.cs>
```
using BarberBilling.Dtos;
using FluentValidation;

namespace BarberBilling.Validators
{
    public class BillingUpdateValidator : AbstractValidator<BillingUpdateDto>
    {
        public BillingUpdateValidator()
        {
            RuleFor(x => x.ClientName).NotEmpty().MaximumLength(200);
            RuleFor(x => x.Service).NotEmpty().MaximumLength(200);
            RuleFor(x => x.Amount).GreaterThan(0);
            RuleFor(x => x.Date).NotEmpty();
        }
    }
}
```

9) Middleware: ExceptionHandler

<File: BarberBilling/Middleware/ExceptionMiddleware.cs>
```
using System.Net;
using System.Text.Json;

namespace BarberBilling.Middleware
{
    public class ExceptionMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<ExceptionMiddleware> _logger;

        public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
        {
            _next = next;
            _logger = logger;
        }

        public async Task InvokeAsync(HttpContext httpContext)
        {
            try
            {
                await _next(httpContext);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled exception");
                await HandleExceptionAsync(httpContext, ex);
            }
        }

        private static Task HandleExceptionAsync(HttpContext context, Exception exception)
        {
            var code = HttpStatusCode.InternalServerError;
            var result = JsonSerializer.Serialize(new { error = exception.Message });
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = (int)code;
            return context.Response.WriteAsync(result);
        }
    }
}
```

10) Controllers: BillingsController and ReportsController

<File: BarberBilling/Controllers/BillingsController.cs>
```
using AutoMapper;
using BarberBilling.Data;
using BarberBilling.Dtos;
using BarberBilling.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace BarberBilling.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class BillingsController : ControllerBase
    {
        private readonly AppDbContext _db;
        private readonly IMapper _mapper;

        public BillingsController(AppDbContext db, IMapper mapper)
        {
            _db = db;
            _mapper = mapper;
        }

        [HttpPost]
        public async Task<ActionResult<BillingReadDto>> Create([FromBody] BillingCreateDto dto)
        {
            var billing = _mapper.Map<Billing>(dto);
            billing.Id = Guid.NewGuid();
            billing.CreatedAt = DateTime.UtcNow;
            billing.UpdatedAt = DateTime.UtcNow;

            _db.Billings.Add(billing);
            await _db.SaveChangesAsync();

            var result = _mapper.Map<BillingReadDto>(billing);
            return CreatedAtAction(nameof(GetById), new { id = billing.Id }, result);
        }

        [HttpGet]
        public async Task<ActionResult<IEnumerable<BillingReadDto>>> GetAll([FromQuery] DateTime? from, [FromQuery] DateTime? to, [FromQuery] string? client, [FromQuery] int page = 1, [FromQuery] int pageSize = 50)
        {
            var query = _db.Billings.AsQueryable();

            if (from.HasValue)
                query = query.Where(b => b.Date >= from.Value.Date);
            if (to.HasValue)
                query = query.Where(b => b.Date <= to.Value.Date);
            if (!string.IsNullOrEmpty(client))
                query = query.Where(b => b.ClientName.Contains(client));

            var items = await query
                .OrderByDescending(b => b.Date)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync();

            return Ok(items.Select(b => _mapper.Map<BillingReadDto>(b)));
        }

        [HttpGet("{id:guid}")]
        public async Task<ActionResult<BillingReadDto>> GetById(Guid id)
        {
            var billing = await _db.Billings.FindAsync(id);
            if (billing == null) return NotFound();
            return Ok(_mapper.Map<BillingReadDto>(billing));
        }

        [HttpPut("{id:guid}")]
        public async Task<ActionResult<BillingReadDto>> Update(Guid id, [FromBody] BillingUpdateDto dto)
        {
            var billing = await _db.Billings.FindAsync(id);
            if (billing == null) return NotFound();

            _mapper.Map(dto, billing);
            billing.UpdatedAt = DateTime.UtcNow;
            await _db.SaveChangesAsync();

            return Ok(_mapper.Map<BillingReadDto>(billing));
        }

        [HttpDelete("{id:guid}")]
        public async Task<IActionResult> Delete(Guid id)
        {
            var billing = await _db.Billings.FindAsync(id);
            if (billing == null) return NotFound();

            _db.Billings.Remove(billing);
            await _db.SaveChangesAsync();
            return NoContent();
        }
    }
}
```

<File: BarberBilling/Controllers/ReportsController.cs>
```
using BarberBilling.Data;
using BarberBilling.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace BarberBilling.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ReportsController : ControllerBase
    {
        private readonly AppDbContext _db;
        private readonly IReportService _reportService;

        public ReportsController(AppDbContext db, IReportService reportService)
        {
            _db = db;
            _reportService = reportService;
        }

        // GET api/reports/week?start=YYYY-MM-DD&format=pdf|excel
        [HttpGet("week")]
        public async Task<IActionResult> Week([FromQuery] DateTime start, [FromQuery] string format = "pdf")
        {
            // Determine week range (start is day 1 of week)
            var weekStart = start.Date;
            var weekEnd = weekStart.AddDays(6).Date.AddDays(1).AddTicks(-1);

            var items = await _db.Billings
                .Where(b => b.Date >= weekStart && b.Date <= weekEnd)
                .OrderBy(b => b.Date)
                .ToListAsync();

            var total = items.Sum(i => i.Amount);

            if (format.ToLower() == "excel")
            {
                var bytes = _reportService.GenerateExcelReport(items, weekStart, weekEnd, total);
                return File(bytes, "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", $"week-report-{weekStart:yyyyMMdd}.xlsx");
            }
            else
            {
                var bytes = _reportService.GeneratePdfReport(items, weekStart, weekEnd, total);
                return File(bytes, "application/pdf", $"week-report-{weekStart:yyyyMMdd}.pdf");
            }
        }
    }
}
```

11) Services: ReportService using QuestPDF and ClosedXML

<File: BarberBilling/Services/IReportService.cs>
```
using BarberBilling.Models;

namespace BarberBilling.Services
{
    public interface IReportService
    {
        byte[] GeneratePdfReport(IEnumerable<Billing> items, DateTime weekStart, DateTime weekEnd, decimal total);
        byte[] GenerateExcelReport(IEnumerable<Billing> items, DateTime weekStart, DateTime weekEnd, decimal total);
    }
}
```

<File: BarberBilling/Services/ReportService.cs>
```
using BarberBilling.Models;
using ClosedXML.Excel;
using QuestPDF.Fluent;
using QuestPDF.Helpers;
using QuestPDF.Infrastructure;

namespace BarberBilling.Services
{
    public class ReportService : IReportService
    {
        public byte[] GeneratePdfReport(IEnumerable<Billing> items, DateTime weekStart, DateTime weekEnd, decimal total)
        {
            var doc = Document.Create(container =>
            {
                container.Page(page =>
                {
                    page.Margin(30);
                    page.Size(PageSizes.A4);
                    page.PageColor(Colors.White);
                    page.DefaultTextStyle(x => x.FontSize(12));

                    page.Header()
                        .Text($"Relatório Semanal - {weekStart:yyyy-MM-dd} até {weekEnd:yyyy-MM-dd}")
                        .SemiBold().FontSize(16).AlignCenter();

                    page.Content()
                        .Column(col =>
                        {
                            col.Spacing(5);
                            col.Item().Table(table =>
                            {
                                // Columns
                                table.ColumnsDefinition(columns =>
                                {
                                    columns.ConstantColumn(90); // Date
                                    columns.RelativeColumn();
                                    columns.RelativeColumn();
                                    columns.ConstantColumn(80); // Amount
                                    columns.ConstantColumn(80); // Payment
                                });

                                // Header
                                table.Header(header =>
                                {
                                    header.Cell().Element(CellStyle).Text("Data");
                                    header.Cell().Element(CellStyle).Text("Cliente");
                                    header.Cell().Element(CellStyle).Text("Serviço");
                                    header.Cell().Element(CellStyle).Text("Valor");
                                    header.Cell().Element(CellStyle).Text("Pagamento");
                                });

                                foreach (var item in items)
                                {
                                    table.Cell().Element(CellStyle).Text(item.Date.ToString("yyyy-MM-dd"));
                                    table.Cell().Element(CellStyle).Text(item.ClientName);
                                    table.Cell().Element(CellStyle).Text(item.Service);
                                    table.Cell().Element(CellStyle).Text(item.Amount.ToString("C"));
                                    table.Cell().Element(CellStyle).Text(item.PaymentMethod.ToString());
                                }

                                static IContainer CellStyle(IContainer c)
                                {
                                    return c.BorderBottom(1).BorderColor(Colors.Grey.Lighten2).PaddingVertical(5).PaddingHorizontal(3);
                                }
                            });

                            col.Item().AlignRight().Text($"Total da semana: {total:C}").Bold().FontSize(14);
                        });

                    page.Footer()
                        .AlignCenter()
                        .Text(txt => {
                            txt.Span("Gerado em ");
                            txt.Span(DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm")).SemiBold();
                        });
                });
            });

            using var ms = new MemoryStream();
            doc.GeneratePdf(ms);
            return ms.ToArray();
        }

        public byte[] GenerateExcelReport(IEnumerable<Billing> items, DateTime weekStart, DateTime weekEnd, decimal total)
        {
            using var wb = new XLWorkbook();
            var ws = wb.Worksheets.Add("Relatório Semanal");

            ws.Cell(1, 1).Value = $"Relatório Semanal: {weekStart:yyyy-MM-dd} até {weekEnd:yyyy-MM-dd}";
            ws.Range(1, 1, 1, 5).Merge().Style.Font.SetBold();

            // Headers
            ws.Cell(3, 1).Value = "Data";
            ws.Cell(3, 2).Value = "Cliente";
            ws.Cell(3, 3).Value = "Serviço";
            ws.Cell(3, 4).Value = "Valor";
            ws.Cell(3, 5).Value = "Pagamento";

            var row = 4;
            foreach (var it in items)
            {
                ws.Cell(row, 1).Value = it.Date;
                ws.Cell(row, 1).Style.DateFormat.Format = "yyyy-mm-dd";
                ws.Cell(row, 2).Value = it.ClientName;
                ws.Cell(row, 3).Value = it.Service;
                ws.Cell(row, 4).Value = it.Amount;
                ws.Cell(row, 4).Style.NumberFormat.Format = "#,##0.00";
                ws.Cell(row, 5).Value = it.PaymentMethod.ToString();
                row++;
            }

            ws.Cell(row, 3).Value = "Total:";
            ws.Cell(row, 4).Value = total;
            ws.Cell(row, 4).Style.NumberFormat.Format = "#,##0.00";
            ws.Cell(row, 3).Style.Font.SetBold(true);
            ws.Cell(row, 4).Style.Font.SetBold(true);

            ws.Columns().AdjustToContents();

            using var ms = new MemoryStream();
            wb.SaveAs(ms);
            return ms.ToArray();
        }
    }
}
```

12) Migrations
You can add migrations locally:
- dotnet ef migrations add InitialCreate
- dotnet ef database update

(Ensure dotnet-ef tool installed globally or use EF tools.)

13) Tests project
Create a test project BarberBilling.Tests.

<File: BarberBilling.Tests/BarberBilling.Tests.csproj>
```
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="7.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="7.0.0" />
    <PackageReference Include="xunit" Version="2.4.1" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.3">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="FluentAssertions" Version="6.10.0" />
    <PackageReference Include="coverlet.collector" Version="3.2.0">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\BarberBilling\BarberBilling.csproj" />
  </ItemGroup>

</Project>
```

14) Tests: integration tests using WebApplicationFactory

<File: BarberBilling.Tests/IntegrationTests/BillingControllerTests.cs>
```
using System.Net;
using System.Net.Http.Json;
using BarberBilling.Dtos;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;
using FluentAssertions;

namespace BarberBilling.Tests.IntegrationTests
{
    public class BillingControllerTests : IClassFixture<WebApplicationFactory<Program>>
    {
        private readonly WebApplicationFactory<Program> _factory;

        public BillingControllerTests(WebApplicationFactory<Program> factory)
        {
            _factory = factory;
        }

        [Fact]
        public async Task Create_Get_Update_Delete_Billing()
        {
            var client = _factory.CreateClient();

            var createDto = new BillingCreateDto
            {
                Date = DateTime.UtcNow.Date,
                ClientName = "João",
                Service = "Corte Masculino",
                Amount = 30m,
                PaymentMethod = Models.PaymentMethod.Cash,
                Notes = "Cliente novo"
            };

            var createResp = await client.PostAsJsonAsync("/api/Billings", createDto);
            createResp.StatusCode.Should().Be(HttpStatusCode.Created);
            var created = await createResp.Content.ReadFromJsonAsync<BillingReadDto>();
            created.Should().NotBeNull();
            created!.ClientName.Should().Be("João");

            // Get by id
            var getResp = await client.GetAsync($"/api/Billings/{created.Id}");
            getResp.StatusCode.Should().Be(HttpStatusCode.OK);
            var found = await getResp.Content.ReadFromJsonAsync<BillingReadDto>();
            found.Should().NotBeNull();
            found!.Id.Should().Be(created.Id);

            // Update
            var updateDto = new BillingUpdateDto
            {
                Date = created.Date,
                ClientName = "João Silva",
                Service = created.Service,
                Amount = 40m,
                PaymentMethod = created.PaymentMethod,
                Notes = created.Notes
            };

            var putResp = await client.PutAsJsonAsync($"/api/Billings/{created.Id}", updateDto);
            putResp.StatusCode.Should().Be(HttpStatusCode.OK);
            var updated = await putResp.Content.ReadFromJsonAsync<BillingReadDto>();
            updated!.ClientName.Should().Be("João Silva");
            updated.Amount.Should().Be(40m);

            // Delete
            var delResp = await client.DeleteAsync($"/api/Billings/{created.Id}");
            delResp.StatusCode.Should().Be(HttpStatusCode.NoContent);

            var getAfterDel = await client.GetAsync($"/api/Billings/{created.Id}");
            getAfterDel.StatusCode.Should().Be(HttpStatusCode.NotFound);
        }
    }
}
```

<File: BarberBilling.Tests/IntegrationTests/ReportsControllerTests.cs>
```
using System.Net;
using System.Net.Http.Json;
using BarberBilling.Dtos;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;
using FluentAssertions;

namespace BarberBilling.Tests.IntegrationTests
{
    public class ReportsControllerTests : IClassFixture<WebApplicationFactory<Program>>
    {
        private readonly WebApplicationFactory<Program> _factory;

        public ReportsControllerTests(WebApplicationFactory<Program> factory)
        {
            _factory = factory;
        }

        [Fact]
        public async Task Generate_Week_Report_PDF_and_Excel()
        {
            var client = _factory.CreateClient();

            // Create sample entries within same week
            var date = DateTime.UtcNow.Date;
            var createDto = new BillingCreateDto
            {
                Date = date,
                ClientName = "Maria",
                Service = "Corte",
                Amount = 50m,
                PaymentMethod = Models.PaymentMethod.Card,
                Notes = null
            };

            var createResp = await client.PostAsJsonAsync("/api/Billings", createDto);
            createResp.EnsureSuccessStatusCode();
            var created = await createResp.Content.ReadFromJsonAsync<BillingReadDto>();

            // PDF
            var pdfResp = await client.GetAsync($"/api/Reports/week?start={date:yyyy-MM-dd}&format=pdf");
            pdfResp.StatusCode.Should().Be(HttpStatusCode.OK);
            pdfResp.Content.Headers.ContentType?.MediaType.Should().Be("application/pdf");
            var pdfBytes = await pdfResp.Content.ReadAsByteArrayAsync();
            pdfBytes.Length.Should().BeGreaterThan(100);

            // Excel
            var xlsResp = await client.GetAsync($"/api/Reports/week?start={date:yyyy-MM-dd}&format=excel");
            xlsResp.StatusCode.Should().Be(HttpStatusCode.OK);
            xlsResp.Content.Headers.ContentType?.MediaType.Should().Be("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            var xlsBytes = await xlsResp.Content.ReadAsByteArrayAsync();
            xlsBytes.Length.Should().BeGreaterThan(100);
        }
    }
}
```

15) Startup of tests: Program class must be public for WebApplicationFactory to find it. Modify Program.cs top to include:

At top of Program.cs, after using statements add:

```
public partial class Program { }
```

This allows tests to reference Program.

16) README / Run instructions

- Prerequisites: .NET 7 SDK
- Restore packages:
  - dotnet restore
- Add EF tools (optional):
  - dotnet tool install --global dotnet-ef
- Create migrations and DB:
  - dotnet ef migrations add InitialCreate -p BarberBilling -s BarberBilling
  - dotnet ef database update -p BarberBilling -s BarberBilling
  (Alternatively, run the API — it runs db.Database.Migrate() at startup.)
- Run API:
  - dotnet run --project BarberBilling
- Run tests:
  - dotnet test

17) Notes, improvements and production considerations
- Authentication/Authorization (JWT) can be added later.
- Use PostgreSQL in production; change EF provider and connection string.
- Add paging metadata for GET /billings responses.
- Improve logging and structured error responses.
- Add rate limiting, request size limits if needed.
- Add OpenAPI documentation details for DTOs.
