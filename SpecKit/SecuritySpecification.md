# Security Specification - ContosoDashboard

## Overview

ContosoDashboard implements a comprehensive security framework designed for training purposes with production-ready patterns. The application follows defense-in-depth principles with multiple security layers.

## Authentication

### Authentication Mechanism
- **Type:** Cookie-based authentication with claims
- **Framework:** ASP.NET Core Identity (mock implementation)
- **Session Management:** Sliding expiration (8 hours)
- **Cookie Security:** HttpOnly, Secure, SameSite

### Login Process
1. User selects identity from predefined dropdown
2. System creates authentication cookie with user claims
3. Cookie contains: UserId, Email, DisplayName, Department, Role
4. Automatic redirect to dashboard or originally requested page

### Session Security
```csharp
// Cookie configuration
options.Cookie.HttpOnly = true;        // Prevents JavaScript access
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // HTTPS only
options.Cookie.SameSite = SameSiteMode.Lax;  // CSRF protection
options.SlidingExpiration = true;      // Extends session on activity
options.ExpireTimeSpan = TimeSpan.FromHours(8);  // Session timeout
```

## Authorization

### Role-Based Access Control (RBAC)
- **Roles:** Employee, TeamLead, ProjectManager, Administrator
- **Hierarchical Permissions:** Higher roles inherit lower role permissions
- **Page-Level Authorization:** `[Authorize]` attributes on protected pages
- **Service-Level Authorization:** Permission checks in business logic

### Permission Matrix

| Feature | Employee | Team Lead | Project Manager | Administrator |
|---------|----------|-----------|-----------------|---------------|
| View Own Tasks | ✓ | ✓ | ✓ | ✓ |
| Update Own Tasks | ✓ | ✓ | ✓ | ✓ |
| View Team Tasks | ✗ | ✓ | ✓ | ✓ |
| Create Tasks | ✗ | ✓ | ✓ | ✓ |
| View Projects | Own Projects | Team Projects | All Projects | All Projects |
| Manage Projects | ✗ | ✗ | Own Projects | All Projects |
| View All Users | ✗ | ✗ | ✗ | ✓ |
| Create Announcements | ✗ | ✗ | ✗ | ✓ |
| System Administration | ✗ | ✗ | ✗ | ✓ |

### Authorization Implementation
```csharp
// Page-level authorization
[Authorize]
public class Tasks : ComponentBase

// Role-based authorization
[Authorize(Roles = "ProjectManager,Administrator")]
public class ProjectManagement : ComponentBase

// Policy-based authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ProjectManagerOnly", policy =>
        policy.RequireRole("ProjectManager", "Administrator"));
});
```

## Data Protection

### IDOR (Insecure Direct Object Reference) Protection
- **Principle:** Users can only access resources they own or have permission to access
- **Implementation:** Service methods validate user context before data access
- **Example:**
```csharp
public async Task<TaskItem?> GetTaskByIdAsync(int taskId, int currentUserId)
{
    var task = await _context.TaskItems.FindAsync(taskId);
    if (task == null) return null;

    // Check ownership or permissions
    if (task.AssignedUserId != currentUserId &&
        task.CreatedByUserId != currentUserId &&
        !await HasProjectAccessAsync(task.ProjectId, currentUserId))
    {
        return null; // IDOR protection
    }

    return task;
}
```

### Input Validation
- **Client-Side:** JavaScript validation for user experience
- **Server-Side:** Model validation with DataAnnotations
- **Sanitization:** Automatic HTML encoding in Razor components
- **SQL Injection Protection:** EF Core parameterized queries

### Data Validation Rules
```csharp
public class TaskItem
{
    [Required(ErrorMessage = "Title is required")]
    [MaxLength(255, ErrorMessage = "Title cannot exceed 255 characters")]
    public string Title { get; set; } = string.Empty;

    [MaxLength(2000, ErrorMessage = "Description cannot exceed 2000 characters")]
    public string? Description { get; set; }

    [Required]
    public TaskPriority Priority { get; set; }

    [Required]
    public TaskStatus Status { get; set; }
}
```

## Security Headers

### HTTP Security Headers
```csharp
// Program.cs - Security headers middleware
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["X-XSS-Protection"] = "1; mode=block";
    context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    context.Response.Headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'";
    await next();
});
```

### Content Security Policy (CSP)
- **Default Source:** 'self' (only allow resources from same origin)
- **Script Source:** 'self' 'unsafe-inline' (allow inline scripts for Blazor)
- **Style Source:** 'self' 'unsafe-inline' (allow inline styles for Bootstrap)
- **Image Source:** 'self' data: https: (allow data URIs and HTTPS images)

## Secure Development Practices

### Code Security
- **Input Validation:** All user inputs validated and sanitized
- **Output Encoding:** Automatic HTML encoding prevents XSS
- **Error Handling:** Generic error pages prevent information disclosure
- **Logging:** Security events logged for audit trail

### Dependency Security
- **Package Updates:** Regular updates of NuGet packages
- **Vulnerability Scanning:** Automated security scanning in CI/CD
- **License Compliance:** Open source license validation

## Network Security

### HTTPS Enforcement
- **SSL/TLS:** All communications encrypted in production
- **HSTS:** HTTP Strict Transport Security enabled
- **Certificate:** Valid SSL certificate required

### CORS (Cross-Origin Resource Sharing)
- **Policy:** Same-origin only (default secure configuration)
- **Headers:** CORS headers not set to prevent cross-origin requests

## Audit and Compliance

### Audit Logging
- **Authentication Events:** Login, logout, session timeout
- **Authorization Events:** Access denied attempts
- **Data Changes:** Create, update, delete operations
- **Security Incidents:** Suspicious activity detection

### Compliance Requirements
- **Data Privacy:** User data protection and consent
- **Access Control:** Principle of least privilege
- **Incident Response:** Security breach procedures
- **Regular Reviews:** Security assessment and updates

## Security Testing

### Testing Strategy
- **Unit Tests:** Security logic validation
- **Integration Tests:** Authentication and authorization flows
- **Penetration Testing:** External security assessment
- **Code Reviews:** Security-focused code reviews

### Security Test Cases
1. **Authentication Bypass:** Attempt login without credentials
2. **IDOR Exploitation:** Try accessing other users' data
3. **XSS Injection:** Test script injection in input fields
4. **CSRF Attacks:** Attempt cross-site request forgery
5. **SQL Injection:** Test parameterized query protection
6. **Session Hijacking:** Attempt cookie theft scenarios
7. **Privilege Escalation:** Try unauthorized role changes

## Incident Response

### Security Incident Procedure
1. **Detection:** Monitor for security events and anomalies
2. **Assessment:** Evaluate impact and scope of incident
3. **Containment:** Isolate affected systems
4. **Recovery:** Restore systems from clean backups
5. **Lessons Learned:** Document and implement improvements

### Contact Information
- **Security Team:** security@contoso.com
- **Emergency:** 1-800-SECURITY
- **Response Time:** Critical incidents within 1 hour

## Future Security Enhancements

### Planned Improvements
- **Multi-Factor Authentication (MFA):** Additional authentication factor
- **OAuth 2.0 Integration:** External identity provider support
- **API Security:** JWT tokens for API authentication
- **Rate Limiting:** DDoS protection and abuse prevention
- **Security Monitoring:** Real-time threat detection
- **Encryption at Rest:** Database encryption for sensitive data

### Production Deployment Security
- **Web Application Firewall (WAF):** Azure Application Gateway or similar
- **DDoS Protection:** Azure DDoS Protection service
- **Security Information and Event Management (SIEM):** Centralized logging
- **Regular Penetration Testing:** Quarterly security assessments
- **Compliance Audits:** SOC 2, ISO 27001 certification</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\SpecKit\SecuritySpecification.md