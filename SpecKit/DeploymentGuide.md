# Deployment Guide - ContosoDashboard

## Overview

This guide provides instructions for deploying ContosoDashboard to various environments. The application is designed for easy deployment with minimal configuration requirements.

## Prerequisites

### System Requirements
- **Operating System:** Windows 10/11, Windows Server 2019/2022, or Linux
- **Web Server:** IIS, Nginx, or Apache
- **Database:** SQL Server 2019+ or Azure SQL Database
- **Runtime:** .NET 8.0 Runtime or SDK
- **Memory:** 512 MB minimum, 1 GB recommended
- **Storage:** 100 MB for application, additional for database

### Required Software
- .NET 8.0 SDK (for building) or Runtime (for running)
- SQL Server LocalDB (development) or full SQL Server (production)
- Web server (IIS, Kestrel, etc.)
- Git (for source control)

## Development Environment Setup

### 1. Clone Repository
```bash
git clone https://github.com/contoso/ContosoDashboard.git
cd ContosoDashboard
```

### 2. Restore Dependencies
```bash
dotnet restore
```

### 3. Database Setup
The application automatically creates and seeds the database on first run. No manual database setup is required for development.

### 4. Run Application
```bash
dotnet run
```

### 5. Access Application
- Open browser to `https://localhost:5001` (HTTPS) or `http://localhost:5000` (HTTP)
- Login with any user from the dropdown (no password required)

## Production Deployment

### Option 1: IIS Deployment (Windows)

#### 1. Install IIS
```powershell
# Enable IIS Windows feature
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole, IIS-WebServer, IIS-DefaultDocument, IIS-DirectoryBrowsing, IIS-HttpErrors, IIS-StaticContent, IIS-HttpLogging, IIS-RequestFiltering, IIS-ISAPIExtensions, IIS-ISAPIFilter, IIS-NetFxExtensibility48, IIS-ASPNET48
```

#### 2. Install .NET Core Hosting Bundle
Download and install the .NET Core Hosting Bundle from:
https://dotnet.microsoft.com/download/dotnet/8.0

#### 3. Publish Application
```bash
dotnet publish -c Release -o ./publish
```

#### 4. Create IIS Site
```powershell
# Create application pool
New-WebAppPool -Name "ContosoDashboard" -Force

# Create website
New-Website -Name "ContosoDashboard" -Port 80 -PhysicalPath "C:\inetpub\ContosoDashboard\publish" -ApplicationPool "ContosoDashboard" -Force
```

#### 5. Configure SSL (Recommended)
```powershell
# Bind SSL certificate
New-WebBinding -Name "ContosoDashboard" -IP "*" -Port 443 -Protocol https
```

### Option 2: Docker Deployment

#### 1. Create Dockerfile
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["ContosoDashboard.csproj", "."]
RUN dotnet restore "./ContosoDashboard.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "ContosoDashboard.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "ContosoDashboard.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ContosoDashboard.dll"]
```

#### 2. Build Docker Image
```bash
docker build -t contosodashboard .
```

#### 3. Run Container
```bash
docker run -d -p 8080:80 -p 8443:443 --name contosodashboard contosodashboard
```

### Option 3: Azure App Service

#### 1. Create App Service
```azurecli
az group create --name ContosoDashboardRG --location eastus
az appservice plan create --name ContosoDashboardPlan --resource-group ContosoDashboardRG --sku B1
az webapp create --name contosodashboard --resource-group ContosoDashboardRG --plan ContosoDashboardPlan --runtime "DOTNET|8.0"
```

#### 2. Deploy via Git
```bash
# Configure local Git deployment
az webapp deployment source config-local-git --name contosodashboard --resource-group ContosoDashboardRG

# Get deployment URL
az webapp deployment source show --name contosodashboard --resource-group ContosoDashboardRG
```

#### 3. Push Code
```bash
git remote add azure <deployment-url>
git push azure main
```

## Database Configuration

### Development (LocalDB)
Default configuration uses SQL Server LocalDB:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ContosoDashboard;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}
```

### Production (SQL Server)
Update `appsettings.json`:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:your-server.database.windows.net,1433;Initial Catalog=ContosoDashboard;Persist Security Info=False;User ID=your-username;Password=your-password;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}
```

### Database Migration
```bash
# Apply migrations
dotnet ef database update
```

## Environment Configuration

### appsettings.json Configuration
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "your-connection-string"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ApplicationSettings": {
    "MaxFileSize": 26214400,  // 25MB in bytes
    "AllowedFileTypes": "pdf,doc,docx,xls,xlsx,ppt,pptx,txt,jpg,jpeg,png"
  }
}
```

### Environment Variables
```bash
# Database connection
ConnectionStrings__DefaultConnection="Server=...;Database=...;..."

# Application settings
ApplicationSettings__MaxFileSize="26214400"
ApplicationSettings__AllowedFileTypes="pdf,doc,docx,xls,xlsx,ppt,pptx,txt,jpg,jpeg,png"

# ASP.NET Core settings
ASPNETCORE_ENVIRONMENT="Production"
ASPNETCORE_URLS="http://+:80;https://+:443"
```

## Security Configuration

### SSL/TLS Setup
1. Obtain SSL certificate from trusted CA
2. Configure HTTPS binding in web server
3. Redirect HTTP to HTTPS
4. Enable HSTS headers

### Firewall Configuration
- **Inbound:** Allow TCP 80 (HTTP), 443 (HTTPS)
- **Outbound:** Allow database connections, NTP, DNS
- **Internal:** Restrict to necessary services only

### Authentication Configuration (Production)
For production, replace mock authentication with real identity provider:

```csharp
// Program.cs
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration);
```

## Monitoring and Logging

### Application Insights
```csharp
// Add to Program.cs
builder.Services.AddApplicationInsightsTelemetry();
```

### Health Checks
```csharp
// Add health check endpoint
app.MapHealthChecks("/health");
```

### Log Configuration
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}
```

## Backup and Recovery

### Database Backup
```sql
-- Full backup
BACKUP DATABASE ContosoDashboard
TO DISK = 'C:\Backup\ContosoDashboard_Full.bak'
WITH FORMAT, INIT;

-- Transaction log backup
BACKUP LOG ContosoDashboard
TO DISK = 'C:\Backup\ContosoDashboard_Log.trn';
```

### Application Backup
- **Source Code:** Git repository
- **Configuration:** Backup `appsettings.json` and environment variables
- **SSL Certificates:** Export certificates with private keys
- **User Files:** Backup uploaded files directory

### Recovery Procedure
1. Restore database from backup
2. Deploy application code
3. Restore configuration files
4. Update DNS records if necessary
5. Test application functionality

## Performance Optimization

### IIS Optimization
```xml
<!-- web.config -->
<configuration>
  <system.webServer>
    <security>
      <requestFiltering>
        <requestLimits maxAllowedContentLength="26214400" />
      </requestFiltering>
    </security>
    <staticContent>
      <mimeMap fileExtension=".webp" mimeType="image/webp" />
    </staticContent>
  </system.webServer>
</configuration>
```

### Database Optimization
```sql
-- Create indexes for performance
CREATE INDEX IX_Tasks_AssignedUser_Status ON TaskItems (AssignedUserId, Status);
CREATE INDEX IX_Tasks_DueDate ON TaskItems (DueDate) WHERE DueDate IS NOT NULL;
CREATE INDEX IX_Projects_Manager_Status ON Projects (ProjectManagerId, Status);
```

### Caching Strategy
```csharp
// Add response caching
builder.Services.AddResponseCaching();
app.UseResponseCaching();
```

## Troubleshooting

### Common Issues

#### Database Connection Failed
- Verify connection string
- Check SQL Server service status
- Ensure firewall allows connections
- Validate credentials

#### Application Won't Start
- Check .NET runtime installation
- Verify port availability
- Review application logs
- Check file permissions

#### Slow Performance
- Monitor database query performance
- Check memory and CPU usage
- Review application logs for errors
- Optimize database indexes

### Log Locations
- **Windows/IIS:** `C:\inetpub\logs\LogFiles\`
- **Linux:** `/var/log/`
- **Application Logs:** Configured in `appsettings.json`

### Support Contacts
- **Development Team:** dev@contoso.com
- **IT Support:** it-support@contoso.com
- **Emergency:** 1-800-HELP-DESK

## Maintenance

### Regular Tasks
- **Weekly:** Review application logs
- **Monthly:** Update dependencies and security patches
- **Quarterly:** Performance optimization review
- **Annually:** Security audit and penetration testing

### Update Procedure
1. Backup current system
2. Deploy updates to staging environment
3. Test functionality
4. Deploy to production
5. Monitor for issues
6. Rollback if necessary</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\SpecKit\DeploymentGuide.md