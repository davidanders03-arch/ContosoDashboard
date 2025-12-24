# Database Design - ContosoDashboard

## Overview

ContosoDashboard uses SQL Server LocalDB with Entity Framework Core for data persistence. The database follows a relational design with proper normalization and indexing for performance.

## Schema Diagram

```
Users (1) -----> (Many) TaskItems (Many) -----> (1) Projects
  |                    |                        |
  |                    |                        |
  +---- (1) Projects   +---- (Many) TaskComments |
       (Manager)                                |
                                               |
Notifications (Many) <---- (1) Users          |
                                               |
Announcements (Many) <---- (1) Users          |
                                               |
ProjectMembers (Many) <-----> (Many) Users    |
                              |               |
                              +---- (1) Projects
```

## Tables

### Users
**Purpose:** Stores user account information and profile data

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| UserId | INT | PRIMARY KEY IDENTITY | Unique user identifier |
| Email | NVARCHAR(255) | NOT NULL UNIQUE | User email address |
| DisplayName | NVARCHAR(255) | NOT NULL | User's display name |
| Department | NVARCHAR(100) | NULL | User's department |
| JobTitle | NVARCHAR(100) | NULL | User's job title |
| Role | INT | NOT NULL | User role (0=Employee, 1=TeamLead, 2=ProjectManager, 3=Administrator) |
| ProfilePhotoUrl | NVARCHAR(500) | NULL | URL to profile photo |
| AvailabilityStatus | INT | NOT NULL DEFAULT 0 | Availability status (0=Available, 1=Busy, 2=InMeeting, 3=OutOfOffice) |
| CreatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Account creation date |
| LastLoginDate | DATETIME2 | NULL | Last login timestamp |
| PhoneNumber | NVARCHAR(20) | NULL | User's phone number |
| EmailNotificationsEnabled | BIT | NOT NULL DEFAULT 1 | Email notification preference |
| InAppNotificationsEnabled | BIT | NOT NULL DEFAULT 1 | In-app notification preference |

**Indexes:**
- PRIMARY KEY on UserId
- UNIQUE INDEX on Email
- INDEX on Department
- INDEX on Role

### Projects
**Purpose:** Stores project information and metadata

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| ProjectId | INT | PRIMARY KEY IDENTITY | Unique project identifier |
| Name | NVARCHAR(255) | NOT NULL | Project name |
| Description | NVARCHAR(2000) | NULL | Project description |
| ProjectManagerId | INT | NOT NULL | Foreign key to Users table |
| StartDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Project start date |
| TargetCompletionDate | DATETIME2 | NULL | Target completion date |
| Status | INT | NOT NULL DEFAULT 0 | Project status (0=Planning, 1=Active, 2=OnHold, 3=Completed, 4=Cancelled) |
| CreatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Project creation date |
| UpdatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Last update timestamp |

**Indexes:**
- PRIMARY KEY on ProjectId
- FOREIGN KEY on ProjectManagerId -> Users.UserId
- INDEX on ProjectManagerId
- INDEX on Status
- INDEX on CreatedDate

### TaskItems
**Purpose:** Stores individual task information

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| TaskId | INT | PRIMARY KEY IDENTITY | Unique task identifier |
| Title | NVARCHAR(255) | NOT NULL | Task title |
| Description | NVARCHAR(2000) | NULL | Task description |
| Priority | INT | NOT NULL DEFAULT 1 | Task priority (0=Low, 1=Medium, 2=High, 3=Critical) |
| Status | INT | NOT NULL DEFAULT 0 | Task status (0=NotStarted, 1=InProgress, 2=Completed) |
| DueDate | DATETIME2 | NULL | Task due date |
| AssignedUserId | INT | NOT NULL | Foreign key to assigned user |
| CreatedByUserId | INT | NOT NULL | Foreign key to task creator |
| ProjectId | INT | NULL | Foreign key to associated project |
| CreatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Task creation date |
| UpdatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Last update timestamp |

**Indexes:**
- PRIMARY KEY on TaskId
- FOREIGN KEY on AssignedUserId -> Users.UserId
- FOREIGN KEY on CreatedByUserId -> Users.UserId
- FOREIGN KEY on ProjectId -> Projects.ProjectId
- INDEX on AssignedUserId
- INDEX on CreatedByUserId
- INDEX on ProjectId
- INDEX on Status
- INDEX on DueDate
- INDEX on Priority

### ProjectMembers
**Purpose:** Junction table for project membership (many-to-many relationship)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| ProjectMemberId | INT | PRIMARY KEY IDENTITY | Unique membership identifier |
| ProjectId | INT | NOT NULL | Foreign key to project |
| UserId | INT | NOT NULL | Foreign key to user |
| JoinedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Membership start date |

**Indexes:**
- PRIMARY KEY on ProjectMemberId
- FOREIGN KEY on ProjectId -> Projects.ProjectId
- FOREIGN KEY on UserId -> Users.UserId
- UNIQUE INDEX on (ProjectId, UserId) - prevents duplicate memberships
- INDEX on ProjectId
- INDEX on UserId

### Notifications
**Purpose:** Stores user notifications

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| NotificationId | INT | PRIMARY KEY IDENTITY | Unique notification identifier |
| UserId | INT | NOT NULL | Foreign key to recipient user |
| Title | NVARCHAR(255) | NOT NULL | Notification title |
| Message | NVARCHAR(2000) | NOT NULL | Notification message |
| Type | INT | NOT NULL | Notification type (0=Info, 1=Warning, 2=Error, 3=Success) |
| IsRead | BIT | NOT NULL DEFAULT 0 | Read status |
| CreatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Creation timestamp |

**Indexes:**
- PRIMARY KEY on NotificationId
- FOREIGN KEY on UserId -> Users.UserId
- INDEX on UserId
- INDEX on IsRead
- INDEX on CreatedDate

### Announcements
**Purpose:** Stores company-wide announcements

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| AnnouncementId | INT | PRIMARY KEY IDENTITY | Unique announcement identifier |
| Title | NVARCHAR(255) | NOT NULL | Announcement title |
| Content | NVARCHAR(MAX) | NOT NULL | Announcement content |
| CreatedByUserId | INT | NOT NULL | Foreign key to creator |
| PublishDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Publication date |
| IsActive | BIT | NOT NULL DEFAULT 1 | Active status |

**Indexes:**
- PRIMARY KEY on AnnouncementId
- FOREIGN KEY on CreatedByUserId -> Users.UserId
- INDEX on PublishDate
- INDEX on IsActive

### TaskComments
**Purpose:** Stores comments on tasks

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| TaskCommentId | INT | PRIMARY KEY IDENTITY | Unique comment identifier |
| TaskId | INT | NOT NULL | Foreign key to task |
| UserId | INT | NOT NULL | Foreign key to comment author |
| Comment | NVARCHAR(2000) | NOT NULL | Comment text |
| CreatedDate | DATETIME2 | NOT NULL DEFAULT GETUTCDATE() | Comment timestamp |

**Indexes:**
- PRIMARY KEY on TaskCommentId
- FOREIGN KEY on TaskId -> TaskItems.TaskId
- FOREIGN KEY on UserId -> Users.UserId
- INDEX on TaskId
- INDEX on CreatedDate

## Seed Data

### Default Users
```sql
INSERT INTO Users (Email, DisplayName, Department, JobTitle, Role) VALUES
('admin@contoso.com', 'System Administrator', 'IT', 'System Administrator', 3),
('camille.nicole@contoso.com', 'Camille Nicole', 'Engineering', 'Project Manager', 2),
('floris.kregel@contoso.com', 'Floris Kregel', 'Engineering', 'Team Lead', 1),
('ni.kang@contoso.com', 'Ni Kang', 'Engineering', 'Employee', 0);
```

### Sample Projects
```sql
INSERT INTO Projects (Name, Description, ProjectManagerId, Status) VALUES
('Dashboard Redesign', 'Modernize the employee dashboard interface', 2, 1),
('Task Management System', 'Implement advanced task tracking features', 2, 1),
('Team Collaboration Tools', 'Add real-time collaboration capabilities', 3, 0);
```

### Sample Tasks
```sql
INSERT INTO TaskItems (Title, Description, Priority, Status, AssignedUserId, CreatedByUserId, ProjectId, DueDate) VALUES
('Design new dashboard layout', 'Create wireframes for the updated dashboard', 2, 1, 4, 2, 1, DATEADD(day, 7, GETUTCDATE())),
('Implement task filtering', 'Add advanced filtering options to task list', 1, 0, 4, 3, 2, DATEADD(day, 14, GETUTCDATE())),
('Setup project repository', 'Initialize Git repository for new project', 0, 2, 4, 2, 3, DATEADD(day, -1, GETUTCDATE()));
```

## Data Integrity Rules

### Business Rules Enforced by Database
1. **User Email Uniqueness:** Each email address can only be associated with one user account
2. **Project Membership Uniqueness:** A user cannot be added to the same project multiple times
3. **Task Assignment Validity:** Tasks can only be assigned to existing users
4. **Project Association:** Tasks can optionally be associated with projects
5. **Status Transitions:** Status values are constrained to valid enum values

### Application-Level Validation
1. **IDOR Protection:** Users can only access their own data or data they have permission to view
2. **Role-Based Permissions:** Operations are restricted based on user roles
3. **Data Consistency:** Foreign key relationships prevent orphaned records
4. **Audit Trail:** Created/Updated dates track data changes

## Performance Considerations

### Query Optimization
- **Indexes:** Strategic indexes on frequently queried columns
- **Pagination:** Large result sets are paginated
- **Eager Loading:** Related data is loaded efficiently using Include/ThenInclude
- **Compiled Queries:** Frequently used queries are cached

### Connection Management
- **Connection Pooling:** EF Core manages connection pooling automatically
- **Async Operations:** All database operations are asynchronous
- **Transaction Scope:** Related operations are wrapped in transactions

## Backup and Recovery

### Backup Strategy
- **Full Backup:** Weekly full database backups
- **Transaction Log Backup:** Daily transaction log backups
- **Retention:** 30 days of backup retention

### Recovery Procedures
- **Point-in-Time Recovery:** Restore to specific point using transaction logs
- **Data Corruption:** Restore from last known good backup
- **Failover:** Automatic failover to standby server (future implementation)

## Migration Strategy

### Database Migrations
- **EF Core Migrations:** Version-controlled schema changes
- **Downward Compatibility:** Migrations support rollback
- **Data Migration:** Custom migration scripts for data transformation

### Deployment Process
1. **Backup:** Create full backup before migration
2. **Test Migration:** Apply migrations to staging environment
3. **Deploy:** Apply migrations to production
4. **Verify:** Validate data integrity post-migration
5. **Rollback Plan:** Prepared rollback procedures</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\SpecKit\DatabaseDesign.md