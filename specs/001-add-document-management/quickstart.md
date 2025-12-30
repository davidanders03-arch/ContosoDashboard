# Quickstart: Document Upload and Management

**Date**: 2025-12-30  
**Feature**: Document Upload and Management  
**Plan**: specs/001-add-document-management/plan.md

## Overview

This guide provides step-by-step instructions for implementing the document upload and management feature in the ContosoDashboard application.

## Prerequisites

- .NET 10.0 SDK installed
- Visual Studio 2022 or VS Code
- SQL Server LocalDB (included with Visual Studio)
- Basic understanding of ASP.NET Core and Blazor Server

## Implementation Steps

### 1. Database Setup

Add the new entities to your `ApplicationDbContext`:

```csharp
// In ApplicationDbContext.cs
public DbSet<Document> Documents { get; set; }
public DbSet<DocumentShare> DocumentShares { get; set; }

// Add to OnModelCreating if needed
modelBuilder.Entity<Document>(entity =>
{
    entity.HasKey(e => e.DocumentId);
    entity.Property(e => e.Title).IsRequired().HasMaxLength(200);
    entity.Property(e => e.Category).IsRequired().HasMaxLength(50);
    entity.Property(e => e.FileName).IsRequired().HasMaxLength(255);
    entity.Property(e => e.FilePath).IsRequired().HasMaxLength(500);
    entity.Property(e => e.FileType).IsRequired().HasMaxLength(255);
    entity.HasOne(d => d.Uploader).WithMany().HasForeignKey(d => d.UploaderId);
    entity.HasOne(d => d.Project).WithMany().HasForeignKey(d => d.ProjectId);
});

modelBuilder.Entity<DocumentShare>(entity =>
{
    entity.HasKey(e => e.DocumentShareId);
    entity.HasOne(ds => ds.Document).WithMany().HasForeignKey(ds => ds.DocumentId);
    entity.HasOne(ds => ds.SharedBy).WithMany().HasForeignKey(ds => ds.SharedById);
    entity.HasOne(ds => ds.SharedWith).WithMany().HasForeignKey(ds => ds.SharedWithId);
});
```

Run the migration:

```bash
dotnet ef migrations add AddDocumentManagement
dotnet ef database update
```

### 2. File Storage Service

Create the storage abstraction:

```csharp
// Services/IFileStorageService.cs
public interface IFileStorageService
{
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType, string userId, int? projectId = null);
    Task DeleteAsync(string filePath);
    Task<Stream> DownloadAsync(string filePath);
    Task<bool> ExistsAsync(string filePath);
}

// Services/LocalFileStorageService.cs
public class LocalFileStorageService : IFileStorageService
{
    private readonly string _basePath;

    public LocalFileStorageService(IConfiguration configuration)
    {
        _basePath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
            "ContosoDashboard",
            "uploads"
        );
        Directory.CreateDirectory(_basePath);
    }

    public async Task<string> UploadAsync(Stream fileStream, string fileName, string contentType, string userId, int? projectId = null)
    {
        var extension = Path.GetExtension(fileName);
        var guid = Guid.NewGuid().ToString();
        var folder = projectId.HasValue ? $"{userId}/{projectId}" : $"{userId}/personal";
        var relativePath = $"{folder}/{guid}{extension}";
        var fullPath = Path.Combine(_basePath, relativePath);

        Directory.CreateDirectory(Path.GetDirectoryName(fullPath));

        using (var fileStream = File.Create(fullPath))
        {
            await fileStream.CopyToAsync(fileStream);
        }

        return relativePath;
    }

    public Task DeleteAsync(string filePath)
    {
        var fullPath = Path.Combine(_basePath, filePath);
        if (File.Exists(fullPath))
        {
            File.Delete(fullPath);
        }
        return Task.CompletedTask;
    }

    public Task<Stream> DownloadAsync(string filePath)
    {
        var fullPath = Path.Combine(_basePath, filePath);
        return Task.FromResult<Stream>(File.OpenRead(fullPath));
    }

    public Task<bool> ExistsAsync(string filePath)
    {
        var fullPath = Path.Combine(_basePath, filePath);
        return Task.FromResult(File.Exists(fullPath));
    }
}
```

Register in `Program.cs`:

```csharp
builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();
```

### 3. Document Service

Create the business logic service:

```csharp
// Services/IDocumentService.cs
public interface IDocumentService
{
    Task<Document> UploadDocumentAsync(DocumentUploadRequest request, string userId);
    Task<IEnumerable<Document>> GetUserDocumentsAsync(string userId, DocumentFilter filter);
    Task<Document> GetDocumentAsync(int documentId, string userId);
    Task UpdateDocumentAsync(int documentId, DocumentUpdateRequest request, string userId);
    Task DeleteDocumentAsync(int documentId, string userId);
    Task ShareDocumentAsync(int documentId, string shareWithUserId, string permissions, string sharedById);
    Task<IEnumerable<SharedDocument>> GetSharedDocumentsAsync(string userId);
}

// Services/DocumentService.cs
public class DocumentService : IDocumentService
{
    private readonly ApplicationDbContext _context;
    private readonly IFileStorageService _fileStorage;
    private readonly INotificationService _notifications;

    public DocumentService(
        ApplicationDbContext context,
        IFileStorageService fileStorage,
        INotificationService notifications)
    {
        _context = context;
        _fileStorage = fileStorage;
        _notifications = notifications;
    }

    public async Task<Document> UploadDocumentAsync(DocumentUploadRequest request, string userId)
    {
        // Validate file
        if (request.File.Length > 25 * 1024 * 1024) // 25MB
            throw new ArgumentException("File too large");

        var allowedTypes = new[] { "application/pdf", "application/msword", /* etc */ };
        if (!allowedTypes.Contains(request.ContentType))
            throw new ArgumentException("Unsupported file type");

        // Upload file first
        var filePath = await _fileStorage.UploadAsync(
            request.File.OpenReadStream(),
            request.File.FileName,
            request.ContentType,
            userId,
            request.ProjectId
        );

        // Create document record
        var document = new Document
        {
            Title = request.Title,
            Description = request.Description,
            Category = request.Category,
            FileName = request.File.FileName,
            FilePath = filePath,
            FileSize = request.File.Length,
            FileType = request.ContentType,
            UploadDate = DateTime.UtcNow,
            UploaderId = userId,
            ProjectId = request.ProjectId,
            Tags = request.Tags
        };

        _context.Documents.Add(document);
        await _context.SaveChangesAsync();

        // Notify project members if applicable
        if (request.ProjectId.HasValue)
        {
            await NotifyProjectMembersAsync(document);
        }

        return document;
    }

    // Implement other methods...
}
```

### 4. Blazor Components

Create the main documents page:

```csharp
@page "/documents"
@inject IDocumentService DocumentService
@inject NavigationManager Navigation

<PageTitle>Documents</PageTitle>

<div class="container-fluid">
    <div class="row">
        <div class="col-12">
            <h1>My Documents</h1>

            <div class="mb-3">
                <button class="btn btn-primary" @onclick="ShowUploadModal">
                    <i class="bi bi-upload"></i> Upload Document
                </button>
            </div>

            @if (documents == null)
            {
                <div class="text-center">
                    <div class="spinner-border" role="status">
                        <span class="sr-only">Loading...</span>
                    </div>
                </div>
            }
            else if (!documents.Any())
            {
                <div class="alert alert-info">
                    No documents uploaded yet. Click "Upload Document" to get started.
                </div>
            }
            else
            {
                <div class="table-responsive">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>Title</th>
                                <th>Category</th>
                                <th>Size</th>
                                <th>Upload Date</th>
                                <th>Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach (var doc in documents)
                            {
                                <tr>
                                    <td>@doc.Title</td>
                                    <td><span class="badge bg-secondary">@doc.Category</span></td>
                                    <td>@FormatFileSize(doc.FileSize)</td>
                                    <td>@doc.UploadDate.ToShortDateString()</td>
                                    <td>
                                        <button class="btn btn-sm btn-outline-primary" @onclick="() => DownloadDocument(doc)">
                                            <i class="bi bi-download"></i>
                                        </button>
                                        <button class="btn btn-sm btn-outline-danger" @onclick="() => DeleteDocument(doc)">
                                            <i class="bi bi-trash"></i>
                                        </button>
                                    </td>
                                </tr>
                            }
                        </tbody>
                    </table>
                </div>
            }
        </div>
    </div>
</div>

@code {
    private List<Document> documents;
    private bool showUploadModal;

    protected override async Task OnInitializedAsync()
    {
        await LoadDocumentsAsync();
    }

    private async Task LoadDocumentsAsync()
    {
        // Get current user ID from authentication state
        var authState = await AuthenticationStateProvider.GetAuthenticationStateAsync();
        var userId = authState.User.FindFirst("sub")?.Value;

        if (!string.IsNullOrEmpty(userId))
        {
            documents = (await DocumentService.GetUserDocumentsAsync(userId, new DocumentFilter()))
                .ToList();
        }
    }

    private void ShowUploadModal()
    {
        showUploadModal = true;
    }

    private async Task DownloadDocument(Document doc)
    {
        Navigation.NavigateTo($"/api/documents/{doc.DocumentId}/download");
    }

    private async Task DeleteDocument(Document doc)
    {
        if (await JSRuntime.InvokeAsync<bool>("confirm", $"Delete {doc.Title}?"))
        {
            await DocumentService.DeleteDocumentAsync(doc.DocumentId, userId);
            await LoadDocumentsAsync();
        }
    }

    private string FormatFileSize(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB" };
        int order = 0;
        double size = bytes;
        while (size >= 1024 && order < sizes.Length - 1)
        {
            order++;
            size /= 1024;
        }
        return $"{size:0.##} {sizes[order]}";
    }
}
```

### 5. API Controllers

Create controllers for file operations:

```csharp
// Controllers/DocumentsController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class DocumentsController : ControllerBase
{
    private readonly IDocumentService _documentService;

    public DocumentsController(IDocumentService documentService)
    {
        _documentService = documentService;
    }

    [HttpPost]
    public async Task<IActionResult> UploadDocument([FromForm] DocumentUploadRequest request)
    {
        var userId = User.FindFirst("sub")?.Value;
        if (string.IsNullOrEmpty(userId)) return Unauthorized();

        try
        {
            var document = await _documentService.UploadDocumentAsync(request, userId);
            return CreatedAtAction(nameof(GetDocument), new { id = document.DocumentId }, document);
        }
        catch (ArgumentException ex)
        {
            return BadRequest(ex.Message);
        }
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetDocument(int id)
    {
        var userId = User.FindFirst("sub")?.Value;
        if (string.IsNullOrEmpty(userId)) return Unauthorized();

        var document = await _documentService.GetDocumentAsync(id, userId);
        if (document == null) return NotFound();

        return Ok(document);
    }

    [HttpGet("{id}/download")]
    public async Task<IActionResult> DownloadDocument(int id)
    {
        var userId = User.FindFirst("sub")?.Value;
        if (string.IsNullOrEmpty(userId)) return Unauthorized();

        var document = await _documentService.GetDocumentAsync(id, userId);
        if (document == null) return NotFound();

        var stream = await _fileStorage.DownloadAsync(document.FilePath);
        return File(stream, document.FileType, document.FileName);
    }

    // Implement other endpoints...
}
```

### 6. Testing

Create unit tests:

```csharp
// Tests/DocumentServiceTests.cs
public class DocumentServiceTests
{
    private readonly ApplicationDbContext _context;
    private readonly Mock<IFileStorageService> _fileStorageMock;
    private readonly DocumentService _service;

    public DocumentServiceTests()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase("TestDb")
            .Options;

        _context = new ApplicationDbContext(options);
        _fileStorageMock = new Mock<IFileStorageService>();
        _service = new DocumentService(_context, _fileStorageMock.Object, null);
    }

    [Fact]
    public async Task UploadDocumentAsync_ValidFile_CreatesDocument()
    {
        // Arrange
        var request = new DocumentUploadRequest
        {
            Title = "Test Document",
            Category = "Personal Files",
            File = new Mock<IFormFile>().Object
        };

        _fileStorageMock.Setup(x => x.UploadAsync(It.IsAny<Stream>(), It.IsAny<string>(),
            It.IsAny<string>(), It.IsAny<string>(), It.IsAny<int?>()))
            .ReturnsAsync("test/path/file.pdf");

        // Act
        var result = await _service.UploadDocumentAsync(request, "user1");

        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test Document", result.Title);
        Assert.Equal("test/path/file.pdf", result.FilePath);
    }
}
```

## Configuration

Add to `appsettings.json`:

```json
{
  "FileStorage": {
    "MaxFileSize": 26214400,
    "AllowedTypes": ["application/pdf", "application/msword", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "text/plain", "image/jpeg", "image/png"]
  }
}
```

## Deployment Checklist

- [ ] Database migration applied
- [ ] File storage directory created with proper permissions
- [ ] Antivirus exclusions configured for upload directory
- [ ] SSL configured for secure file transfers
- [ ] Backup strategy implemented for files and database
- [ ] Monitoring alerts configured for storage usage

## Troubleshooting

**Common Issues:**

1. **File upload fails**: Check file size limits and allowed types
2. **Database errors**: Ensure migrations are applied
3. **Permission denied**: Verify user authentication and authorization
4. **File not found**: Check file storage path configuration

**Debug Commands:**

```bash
# Check database
dotnet ef database update

# Clean rebuild
dotnet clean
dotnet build

# Run tests
dotnet test
```

## Next Steps

1. Implement the remaining service methods
2. Create the upload modal component
3. Add search and filtering functionality
4. Implement sharing features
5. Add comprehensive error handling
6. Create integration tests
7. Update navigation menu
8. Add dashboard widgets</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\specs\001-add-document-management\quickstart.md