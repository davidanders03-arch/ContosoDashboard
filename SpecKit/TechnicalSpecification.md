# Technical Specification - ContosoDashboard

## 1. System Architecture

### 1.1 Overview
ContosoDashboard is built using a layered architecture with clear separation of concerns:
- **Presentation Layer**: Blazor Server components
- **Application Layer**: Business services
- **Domain Layer**: Entity models and business logic
- **Infrastructure Layer**: Data access and external services

### 1.2 Technology Stack
- **Framework**: ASP.NET Core 8.0
- **UI Framework**: Blazor Server
- **Database**: SQL Server LocalDB / Entity Framework Core 8.0
- **Authentication**: Cookie-based authentication
- **Styling**: Bootstrap 5.3, Bootstrap Icons
- **Language**: C# 12.0

## 2. Database Design

### 2.1 Entity Relationships

#### Users Table
```sql
CREATE TABLE Users (
    UserId INT PRIMARY KEY IDENTITY,
    Email NVARCHAR(255) NOT NULL UNIQUE,
    DisplayName NVARCHAR(255) NOT NULL,
    Department NVARCHAR(100),
    JobTitle NVARCHAR(100),
    Role INT NOT NULL, -- Enum: 0=Employee, 1=TeamLead, 2=ProjectManager, 3=Administrator
    ProfilePhotoUrl NVARCHAR(500),
    AvailabilityStatus INT NOT NULL DEFAULT 0, -- Enum: 0=Available, 1=Busy, 2=InMeeting, 3=OutOfOffice
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    LastLoginDate DATETIME2,
    PhoneNumber NVARCHAR(20),
    EmailNotificationsEnabled BIT NOT NULL DEFAULT 1,
    InAppNotificationsEnabled BIT NOT NULL DEFAULT 1
);
```

#### Projects Table
```sql
CREATE TABLE Projects (
    ProjectId INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(255) NOT NULL,
    Description NVARCHAR(2000),
    ProjectManagerId INT NOT NULL,
    StartDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    TargetCompletionDate DATETIME2,
    Status INT NOT NULL DEFAULT 0, -- Enum: 0=Planning, 1=Active, 2=OnHold, 3=Completed, 4=Cancelled
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (ProjectManagerId) REFERENCES Users(UserId)
);
```

#### Tasks Table
```sql
CREATE TABLE TaskItems (
    TaskId INT PRIMARY KEY IDENTITY,
    Title NVARCHAR(255) NOT NULL,
    Description NVARCHAR(2000),
    Priority INT NOT NULL DEFAULT 1, -- Enum: 0=Low, 1=Medium, 2=High, 3=Critical
    Status INT NOT NULL DEFAULT 0, -- Enum: 0=NotStarted, 1=InProgress, 2=Completed
    DueDate DATETIME2,
    AssignedUserId INT NOT NULL,
    CreatedByUserId INT NOT NULL,
    ProjectId INT,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (AssignedUserId) REFERENCES Users(UserId),
    FOREIGN KEY (CreatedByUserId) REFERENCES Users(UserId),
    FOREIGN KEY (ProjectId) REFERENCES Projects(ProjectId)
);
```

#### Notifications Table
```sql
CREATE TABLE Notifications (
    NotificationId INT PRIMARY KEY IDENTITY,
    UserId INT NOT NULL,
    Title NVARCHAR(255) NOT NULL,
    Message NVARCHAR(2000) NOT NULL,
    Type INT NOT NULL, -- Enum: 0=Info, 1=Warning, 2=Error, 3=Success
    IsRead BIT NOT NULL DEFAULT 0,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (UserId) REFERENCES Users(UserId)
);
```

#### Announcements Table
```sql
CREATE TABLE Announcements (
    AnnouncementId INT PRIMARY KEY IDENTITY,
    Title NVARCHAR(255) NOT NULL,
    Content NVARCHAR(MAX) NOT NULL,
    CreatedByUserId INT NOT NULL,
    PublishDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive BIT NOT NULL DEFAULT 1,
    FOREIGN KEY (CreatedByUserId) REFERENCES Users(UserId)
);
```

### 2.2 Indexes
- Users: Email (unique), Department
- Projects: ProjectManagerId, Status, CreatedDate
- TaskItems: AssignedUserId, CreatedByUserId, ProjectId, Status, DueDate, Priority
- Notifications: UserId, IsRead, CreatedDate

## 3. API Design

### 3.1 Service Interfaces

#### IUserService
```csharp
public interface IUserService
{
    Task<User?> GetUserByIdAsync(int userId);
    Task<User?> GetUserByEmailAsync(string email);
    Task<IEnumerable<User>> GetUsersByDepartmentAsync(string department);
    Task<IEnumerable<User>> GetAllUsersAsync();
    Task UpdateUserAsync(User user);
    Task UpdateUserAvailabilityAsync(int userId, AvailabilityStatus status);
}
```

#### ITaskService
```csharp
public interface ITaskService
{
    Task<TaskItem?> GetTaskByIdAsync(int taskId);
    Task<IEnumerable<TaskItem>> GetTasksForUserAsync(int userId);
    Task<IEnumerable<TaskItem>> GetTasksForProjectAsync(int projectId);
    Task CreateTaskAsync(TaskItem task);
    Task UpdateTaskAsync(TaskItem task);
    Task UpdateTaskStatusAsync(int taskId, TaskStatus status);
    Task DeleteTaskAsync(int taskId);
}
```

#### IProjectService
```csharp
public interface IProjectService
{
    Task<Project?> GetProjectByIdAsync(int projectId);
    Task<IEnumerable<Project>> GetProjectsForUserAsync(int userId);
    Task<IEnumerable<Project>> GetAllProjectsAsync();
    Task CreateProjectAsync(Project project);
    Task UpdateProjectAsync(Project project);
    Task AddProjectMemberAsync(int projectId, int userId);
    Task RemoveProjectMemberAsync(int projectId, int userId);
}
```

#### INotificationService
```csharp
public interface INotificationService
{
    Task<IEnumerable<Notification>> GetNotificationsForUserAsync(int userId);
    Task<Notification?> GetNotificationByIdAsync(int notificationId);
    Task CreateNotificationAsync(Notification notification);
    Task MarkAsReadAsync(int notificationId);
    Task<int> GetUnreadCountAsync(int userId);
}
```

### 3.2 Authentication & Authorization

#### Custom Authentication State Provider
```csharp
public class CustomAuthenticationStateProvider : AuthenticationStateProvider
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IUserService _userService;

    public override async Task<AuthenticationState> GetAuthenticationStateAsync()
    {
        var httpContext = _httpContextAccessor.HttpContext;
        if (httpContext?.User?.Identity?.IsAuthenticated == true)
        {
            var userIdClaim = httpContext.User.FindFirst("UserId");
            if (int.TryParse(userIdClaim?.Value, out var userId))
            {
                var user = await _userService.GetUserByIdAsync(userId);
                if (user != null)
                {
                    var claims = new List<Claim>
                    {
                        new Claim(ClaimTypes.NameIdentifier, user.UserId.ToString()),
                        new Claim(ClaimTypes.Email, user.Email),
                        new Claim(ClaimTypes.Name, user.DisplayName),
                        new Claim("Department", user.Department ?? ""),
                        new Claim(ClaimTypes.Role, user.Role.ToString())
                    };
                    var identity = new ClaimsIdentity(claims, "ContosoAuth");
                    return new AuthenticationState(new ClaimsPrincipal(identity));
                }
            }
        }
        return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));
    }
}
```

## 4. User Interface Components

### 4.1 Page Structure
- **_Host.cshtml**: Main layout with Blazor initialization
- **MainLayout.razor**: Application shell with navigation
- **NavMenu.razor**: Sidebar navigation menu

### 4.2 Key Components
- **Dashboard (Index.razor)**: Summary cards, announcements, quick actions
- **Tasks (Tasks.razor)**: Task list with filtering and sorting
- **Projects (Projects.razor)**: Project overview with status indicators
- **ProjectDetails (ProjectDetails.razor)**: Detailed project view
- **Team (Team.razor)**: User directory with search
- **Notifications (Notifications.razor)**: Notification management
- **Profile (Profile.razor)**: User profile editing

### 4.3 Responsive Design
- Bootstrap grid system for layout
- Mobile-first responsive breakpoints
- Collapsible navigation for mobile devices

## 5. Security Implementation

### 5.1 Authentication Middleware
```csharp
// Program.cs
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/login";
        options.AccessDeniedPath = "/access-denied";
        options.SlidingExpiration = true;
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.HttpOnly = true;
    });
```

### 5.2 Authorization Policies
```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ProjectManagerOnly", policy =>
        policy.RequireRole("ProjectManager", "Administrator"));
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Administrator"));
});
```

### 5.3 IDOR Protection
All service methods include user context validation:
```csharp
public async Task<TaskItem?> GetTaskByIdAsync(int taskId, int currentUserId)
{
    var task = await _context.TaskItems
        .Include(t => t.AssignedUser)
        .Include(t => t.Project)
        .FirstOrDefaultAsync(t => t.TaskId == taskId);

    if (task == null) return null;

    // Check if user has permission to view this task
    if (task.AssignedUserId != currentUserId &&
        task.CreatedByUserId != currentUserId &&
        !await HasProjectAccessAsync(task.ProjectId, currentUserId))
    {
        return null;
    }

    return task;
}
```

## 6. Deployment & Configuration

### 6.1 appsettings.json
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ContosoDashboard;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### 6.2 launchSettings.json
```json
{
  "profiles": {
    "ContosoDashboard": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

### 6.3 Build Configuration
- Target framework: net8.0
- Nullable reference types: enabled
- Implicit usings: enabled
- Build warnings as errors: enabled in CI/CD</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\SpecKit\TechnicalSpecification.md