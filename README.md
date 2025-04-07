# SOFI – Project Implementation Document

## 🎯 Project Overview
**SOFI** is a data-driven, AI-powered SaaS application designed to help pension and insurance professionals make smarter, faster investment decisions through a clean, interactive dashboard interface.

### 🧱 Technologies Used
- **Backend**: ASP.NET Core Web API
  - Clean Architecture pattern
  - RESTful API design
  - Entity Framework Core (SQL Server)
  - FluentValidation
  - JWT Authentication & Role-Based Authorization
- **Frontend**: Blazor Server
  - Razor Components + CSS Isolation
  - HttpClient Services for API integration
  - AuthenticationStateProvider for user state
- **AI Integration**: ML.NET / External APIs (mocked in early versions)
- **Database**: SQL Server (initially with local dev DB)
- **Logging & Monitoring**: Serilog + Application Insights
- **CI/CD**: Azure DevOps Pipelines
- **Testing**: xUnit, Moq, bUnit (for component testing)

---


## ✅ Phase 1: Project Setup

### 1.1 Solution Structure
Create a solution named `SOFI.Solution` with these projects:
```
SOFI.Solution/
├── SOFI.Api/             --> Entry point (ASP.NET Core Web API)
├── SOFI.Application/     --> Business logic (CQRS, DTOs, Services)
├── SOFI.Domain/          --> Core domain models and interfaces
├── SOFI.Infrastructure/  --> External dependencies, third-party services
├── SOFI.Persistence/     --> Database context, EF Core repositories
├── SOFI.Shared/          --> Enums, constants, base classes
├── SOFI.Web/             --> Blazor Server frontend
├── tests/                --> Unit & integration tests
    ├── SOFI.UnitTests/
    └── SOFI.IntegrationTests/
```

### 1.2 Project References
- `SOFI.Api` → references `Application`, `Domain`, `Infrastructure`, `Persistence`, `Shared`
- `SOFI.Application` → references `Domain`, `Shared`
- `SOFI.Infrastructure` → references `Application`, `Domain`
- `SOFI.Persistence` → references `Application`, `Domain`, `Shared`
- `SOFI.Web` → references `Shared`

### 1.3 NuGet Dependencies
Install the following packages where needed:
- `Microsoft.EntityFrameworkCore.SqlServer`
- `Microsoft.AspNetCore.Authentication.JwtBearer`
- `AutoMapper.Extensions.Microsoft.DependencyInjection`
- `Swashbuckle.AspNetCore`
- `Serilog.AspNetCore`
- `FluentValidation.AspNetCore`
- `xUnit`, `bUnit`, `Moq`, `Microsoft.AspNetCore.Mvc.Testing`
- `BCrypt.Net-Next` for password hashing

---

## 🛠️ Full Implementation – JWT Authentication

### ✅ Program.cs for SOFI.Api
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// JWT Auth
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });
builder.Services.AddAuthorization();

// App DI setup
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IAppUserRepository, AppUserRepository>();

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddAutoMapper(typeof(Program));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

### ✅ AuthController.cs
```csharp
[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;

    public AuthController(IAuthService authService)
    {
        _authService = authService;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login(LoginRequestDto dto)
    {
        var result = await _authService.LoginAsync(dto);
        return Ok(result);
    }
}
```

---

### ✅ AppUser.cs (Domain)
```csharp
public class AppUser
{
    public Guid Id { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }
    public string Role { get; set; } // Admin, Analyst, Manager, Viewer
}
```

---

### ✅ DTOs (Application)
```csharp
public class LoginRequestDto
{
    public string Email { get; set; }
    public string Password { get; set; }
}

public class AuthResponseDto
{
    public string Token { get; set; }
    public string Role { get; set; }
}
```

---

### ✅ IAuthService.cs
```csharp
public interface IAuthService
{
    Task<AuthResponseDto> LoginAsync(LoginRequestDto loginDto);
}
```

---

### ✅ AuthService.cs (Application.Services)
```csharp
public class AuthService : IAuthService
{
    private readonly IAppUserRepository _userRepo;
    private readonly IConfiguration _config;

    public AuthService(IAppUserRepository userRepo, IConfiguration config)
    {
        _userRepo = userRepo;
        _config = config;
    }

    public async Task<AuthResponseDto> LoginAsync(LoginRequestDto dto)
    {
        var user = await _userRepo.GetByEmailAsync(dto.Email);
        if (user == null || !BCrypt.Net.BCrypt.Verify(dto.Password, user.PasswordHash))
            throw new UnauthorizedAccessException("Invalid credentials.");

        var token = GenerateJwtToken(user);

        return new AuthResponseDto { Token = token, Role = user.Role };
    }

    private string GenerateJwtToken(AppUser user)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role)
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(8),
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

### ✅ IAppUserRepository.cs
```csharp
public interface IAppUserRepository
{
    Task<AppUser> GetByEmailAsync(string email);
}
```

---

### ✅ AppUserRepository.cs (Persistence)
```csharp
public class AppUserRepository : IAppUserRepository
{
    private readonly ApplicationDbContext _db;
    public AppUserRepository(ApplicationDbContext db) => _db = db;

    public async Task<AppUser> GetByEmailAsync(string email)
        => await _db.AppUsers.FirstOrDefaultAsync(x => x.Email == email);
}
```

---

### ✅ appsettings.json
```json
"Jwt": {
  "Key": "your-super-secret-key-here",
  "Issuer": "sofi.api",
  "Audience": "sofi.web"
},
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost;Database=SOFI_Db;Trusted_Connection=True;MultipleActiveResultSets=true"
}
```

---



## 🚧 Phase 2: Core Features & Modules

### 🔐 2.1 Authentication & Authorization
- Implement JWT token generation in `AuthController`
- Configure token validation in `Startup.cs`
- Roles: Admin, Analyst, Manager, Viewer
- Blazor integration:
  - Use `AuthenticationStateProvider` for role-based UI rendering
  - Protect pages/components using `[Authorize]` attribute

### 📊 2.2 Dashboard Module
- Razor Page: `Dashboard.razor`
- Components:
  - `ChartTile.razor` (uses mock chart data)
  - `RiskBadge.razor` (color-coded risk score)
  - `RecommendationList.razor` (list of AI suggestions)
- Backend:
  - `DashboardController.cs`
    - `GET /api/dashboard/overview`
    - `GET /api/ai/recommendations`

### 💼 2.3 Fund Management
- Blazor Pages:
  - `FundList.razor`: List all funds with pagination & search
  - `FundDetail.razor`: Show fund overview + charts
  - `FundForm.razor`: Create/edit fund
- Backend API:
  - `FundController.cs`
    - `GET /api/funds`, `POST`, `PUT`, `DELETE`
- Frontend:
  - `FundService.cs` (HttpClient service layer)
- Validation:
  - Use FluentValidation for Create/Update Fund DTOs

### 📂 2.4 Portfolio Comparison
- Page: `PortfolioCompare.razor`
- Compare funds by performance/risk
- API Endpoint: `GET /api/portfolios/{id}/compare`

### 🧠 2.5 AI Insight Engine
- Backend: Mock service returning suggestions
- API Endpoints:
  - `/api/ai/recommendations`
  - `/api/risk/fund/{id}`
- Later Phase: Integrate ML.NET or external API

### 🧪 2.6 Risk Scoring
- Score computed per fund using mock data (Phase 1)
- Component: `RiskBadge.razor`
- API: `GET /api/risk/fund/{id}`

### 🧾 2.7 Reporting & Exports
- Component: `Export.razor`
- PDF/Excel export of reports
- API: `/api/reports/export`
- Later: Scheduled email delivery (Phase 3)

---

## 🧱 Phase 3: Infrastructure & Cross-Cutting Concerns

### 3.1 Database Configuration
- DbContext in `SOFI.Persistence`
- Use `Code-First Migrations`
- Seed users, funds, portfolios

### 3.2 Dependency Injection
- Register services in `Startup.cs` or `Program.cs`
- Use `IServiceCollection` extensions for organization

### 3.3 Logging & Monitoring
- Use `Serilog` for structured logs
- Integrate with Azure Application Insights

### 3.4 CI/CD Pipeline (Azure DevOps)
- YAML pipeline:
  - Restore, Build, Test, Publish
  - Deploy to Azure App Service

---

## 🧪 Phase 4: Testing

### 4.1 Unit Testing
- `xUnit` + `Moq` for services, controllers
- `bUnit` for component testing
  - Example: Test `FundCard.razor` renders risk level correctly

### 4.2 Integration Testing
- Use `WebApplicationFactory<T>` for API tests
- Test authentication, endpoint results, and error handling

---

## 🚀 Phase 5: Final Polish & Launch

### 5.1 UI/UX Enhancements
- CSS Isolation per component
- Responsive layouts using Tailwind or Bootstrap
- Add loading spinners and error messages

### 5.2 Security Review
- Secure endpoints with `[Authorize]`
- Use Azure Key Vault for secrets
- Validate all user input (backend & frontend)

### 5.3 Admin Panel
- Page: `UserManagement.razor`
- API: `/api/admin/users`
- Manage roles, invite users, audit logs

---

## 📘 Developer Training Resources
Each team member should:
1. Complete a mini course on:
   - Clean Architecture in ASP.NET Core
   - Blazor Component Fundamentals
   - Dependency Injection & HttpClient usage
2. Pair with a lead to:
   - Implement their first Razor Component (`FundCard`)
   - Add a new API endpoint with validation (`FundController`)
   - Write a test using `bUnit` or `xUnit`
3. Learn to use:
   - Git (feature branching + PR strategy)
   - Azure DevOps Pipelines
   - Swagger for API testing

Let’s begin building with **Authentication**, then move to **Dashboard + Fund Management**.

