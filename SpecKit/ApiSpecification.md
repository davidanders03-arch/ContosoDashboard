# API Specification - ContosoDashboard

## Overview

ContosoDashboard uses a service-oriented architecture with clear interfaces for all business operations. This document specifies the API contracts for the core services.

## Service Interfaces

### IUserService

#### GetUserByIdAsync
```csharp
Task<User?> GetUserByIdAsync(int userId)
```
**Parameters:**
- `userId`: The unique identifier of the user

**Returns:** User object or null if not found

**Authorization:** Any authenticated user

#### GetUserByEmailAsync
```csharp
Task<User?> GetUserByEmailAsync(string email)
```
**Parameters:**
- `email`: The email address of the user

**Returns:** User object or null if not found

**Authorization:** Any authenticated user

#### GetUsersByDepartmentAsync
```csharp
Task<IEnumerable<User>> GetUsersByDepartmentAsync(string department)
```
**Parameters:**
- `department`: The department name to filter by

**Returns:** Collection of users in the specified department

**Authorization:** Any authenticated user

#### UpdateUserAsync
```csharp
Task UpdateUserAsync(User user)
```
**Parameters:**
- `user`: The user object with updated information

**Returns:** Task (void)

**Authorization:** User can only update their own profile, or administrator

#### UpdateUserAvailabilityAsync
```csharp
Task UpdateUserAvailabilityAsync(int userId, AvailabilityStatus status)
```
**Parameters:**
- `userId`: The user ID to update
- `status`: The new availability status

**Returns:** Task (void)

**Authorization:** User can only update their own status

### ITaskService

#### GetTaskByIdAsync
```csharp
Task<TaskItem?> GetTaskByIdAsync(int taskId)
```
**Parameters:**
- `taskId`: The unique identifier of the task

**Returns:** TaskItem object or null if not found or not authorized

**Authorization:** Task assignee, creator, or project member

#### GetTasksForUserAsync
```csharp
Task<IEnumerable<TaskItem>> GetTasksForUserAsync(int userId)
```
**Parameters:**
- `userId`: The user ID to get tasks for

**Returns:** Collection of tasks assigned to the user

**Authorization:** The specified user or their manager

#### GetTasksForProjectAsync
```csharp
Task<IEnumerable<TaskItem>> GetTasksForProjectAsync(int projectId)
```
**Parameters:**
- `projectId`: The project ID to get tasks for

**Returns:** Collection of tasks in the project

**Authorization:** Project member

#### CreateTaskAsync
```csharp
Task CreateTaskAsync(TaskItem task)
```
**Parameters:**
- `task`: The task object to create

**Returns:** Task (void)

**Authorization:** Team lead or higher, or administrator

#### UpdateTaskAsync
```csharp
Task UpdateTaskAsync(TaskItem task)
```
**Parameters:**
- `task`: The task object with updated information

**Returns:** Task (void)

**Authorization:** Task assignee or authorized manager

#### UpdateTaskStatusAsync
```csharp
Task UpdateTaskStatusAsync(int taskId, TaskStatus status)
```
**Parameters:**
- `taskId`: The task ID to update
- `status`: The new status

**Returns:** Task (void)

**Authorization:** Task assignee

#### DeleteTaskAsync
```csharp
Task DeleteTaskAsync(int taskId)
```
**Parameters:**
- `taskId`: The task ID to delete

**Returns:** Task (void)

**Authorization:** Task creator or administrator

### IProjectService

#### GetProjectByIdAsync
```csharp
Task<Project?> GetProjectByIdAsync(int projectId)
```
**Parameters:**
- `projectId`: The unique identifier of the project

**Returns:** Project object or null if not found or not authorized

**Authorization:** Project member

#### GetProjectsForUserAsync
```csharp
Task<IEnumerable<Project>> GetProjectsForUserAsync(int userId)
```
**Parameters:**
- `userId`: The user ID to get projects for

**Returns:** Collection of projects the user is a member of

**Authorization:** The specified user or their manager

#### GetAllProjectsAsync
```csharp
Task<IEnumerable<Project>> GetAllProjectsAsync()
```
**Parameters:** None

**Returns:** Collection of all projects

**Authorization:** Administrator only

#### CreateProjectAsync
```csharp
Task CreateProjectAsync(Project project)
```
**Parameters:**
- `project`: The project object to create

**Returns:** Task (void)

**Authorization:** Project manager or administrator

#### UpdateProjectAsync
```csharp
Task UpdateProjectAsync(Project project)
```
**Parameters:**
- `project`: The project object with updated information

**Returns:** Task (void)

**Authorization:** Project manager or administrator

#### AddProjectMemberAsync
```csharp
Task AddProjectMemberAsync(int projectId, int userId)
```
**Parameters:**
- `projectId`: The project ID
- `userId`: The user ID to add

**Returns:** Task (void)

**Authorization:** Project manager

#### RemoveProjectMemberAsync
```csharp
Task RemoveProjectMemberAsync(int projectId, int userId)
```
**Parameters:**
- `projectId`: The project ID
- `userId`: The user ID to remove

**Returns:** Task (void)

**Authorization:** Project manager

### INotificationService

#### GetNotificationsForUserAsync
```csharp
Task<IEnumerable<Notification>> GetNotificationsForUserAsync(int userId)
```
**Parameters:**
- `userId`: The user ID to get notifications for

**Returns:** Collection of notifications for the user

**Authorization:** The specified user only

#### GetNotificationByIdAsync
```csharp
Task<Notification?> GetNotificationByIdAsync(int notificationId)
```
**Parameters:**
- `notificationId`: The notification ID

**Returns:** Notification object or null

**Authorization:** Notification recipient

#### CreateNotificationAsync
```csharp
Task CreateNotificationAsync(Notification notification)
```
**Parameters:**
- `notification`: The notification object to create

**Returns:** Task (void)

**Authorization:** System or administrator

#### MarkAsReadAsync
```csharp
Task MarkAsReadAsync(int notificationId)
```
**Parameters:**
- `notificationId`: The notification ID to mark as read

**Returns:** Task (void)

**Authorization:** Notification recipient

#### GetUnreadCountAsync
```csharp
Task<int> GetUnreadCountAsync(int userId)
```
**Parameters:**
- `userId`: The user ID

**Returns:** Count of unread notifications

**Authorization:** The specified user only

### IDashboardService

#### GetDashboardSummaryAsync
```csharp
Task<DashboardSummary> GetDashboardSummaryAsync(int userId)
```
**Parameters:**
- `userId`: The user ID

**Returns:** DashboardSummary object with counts and metrics

**Authorization:** The specified user only

**DashboardSummary Structure:**
```csharp
public class DashboardSummary
{
    public int TotalActiveTasks { get; set; }
    public int TasksDueToday { get; set; }
    public int ActiveProjects { get; set; }
    public int UnreadNotifications { get; set; }
}
```

#### GetRecentNotificationsAsync
```csharp
Task<IEnumerable<Notification>> GetRecentNotificationsAsync(int userId, int count = 5)
```
**Parameters:**
- `userId`: The user ID
- `count`: Number of recent notifications to return

**Returns:** Collection of recent notifications

**Authorization:** The specified user only

#### GetActiveAnnouncementsAsync
```csharp
Task<IEnumerable<Announcement>> GetActiveAnnouncementsAsync(int count = 3)
```
**Parameters:**
- `count`: Number of announcements to return

**Returns:** Collection of active announcements

**Authorization:** Any authenticated user

### IAnnouncementService

#### GetAllAnnouncementsAsync
```csharp
Task<IEnumerable<Announcement>> GetAllAnnouncementsAsync()
```
**Parameters:** None

**Returns:** Collection of all announcements

**Authorization:** Administrator only

#### CreateAnnouncementAsync
```csharp
Task CreateAnnouncementAsync(Announcement announcement)
```
**Parameters:**
- `announcement`: The announcement object to create

**Returns:** Task (void)

**Authorization:** Administrator only

#### UpdateAnnouncementAsync
```csharp
Task UpdateAnnouncementAsync(Announcement announcement)
```
**Parameters:**
- `announcement`: The announcement object with updates

**Returns:** Task (void)

**Authorization:** Administrator only

## Data Models

### User
```csharp
public class User
{
    public int UserId { get; set; }
    public string Email { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public string? Department { get; set; }
    public string? JobTitle { get; set; }
    public UserRole Role { get; set; }
    public string? ProfilePhotoUrl { get; set; }
    public AvailabilityStatus AvailabilityStatus { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime? LastLoginDate { get; set; }
    public string? PhoneNumber { get; set; }
    public bool EmailNotificationsEnabled { get; set; }
    public bool InAppNotificationsEnabled { get; set; }
}
```

### TaskItem
```csharp
public class TaskItem
{
    public int TaskId { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public TaskPriority Priority { get; set; }
    public TaskStatus Status { get; set; }
    public DateTime? DueDate { get; set; }
    public int AssignedUserId { get; set; }
    public int CreatedByUserId { get; set; }
    public int? ProjectId { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime UpdatedDate { get; set; }
}
```

### Project
```csharp
public class Project
{
    public int ProjectId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public int ProjectManagerId { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime? TargetCompletionDate { get; set; }
    public ProjectStatus Status { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime UpdatedDate { get; set; }
}
```

### Notification
```csharp
public class Notification
{
    public int NotificationId { get; set; }
    public int UserId { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public NotificationType Type { get; set; }
    public bool IsRead { get; set; }
    public DateTime CreatedDate { get; set; }
}
```

### Announcement
```csharp
public class Announcement
{
    public int AnnouncementId { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public int CreatedByUserId { get; set; }
    public DateTime PublishDate { get; set; }
    public bool IsActive { get; set; }
}
```

## Error Handling

All service methods should throw appropriate exceptions:

- `ArgumentException`: Invalid parameters
- `UnauthorizedAccessException`: Insufficient permissions
- `KeyNotFoundException`: Entity not found
- `InvalidOperationException`: Business rule violation

## Asynchronous Operations

All service methods are asynchronous and return `Task` or `Task<T>`. Calling code should use `await` to ensure proper execution flow.</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\SpecKit\ApiSpecification.md