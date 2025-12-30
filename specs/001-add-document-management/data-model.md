# Data Model: Document Upload and Management

**Date**: 2025-12-30  
**Feature**: Document Upload and Management  
**Plan**: specs/001-add-document-management/plan.md

## Entities

### Document

Core entity representing an uploaded document with its metadata.

**Fields:**
- `DocumentId` (int, PK) - Auto-incrementing primary key
- `Title` (string, required, 200 chars) - User-provided document title
- `Description` (string, optional, 1000 chars) - User-provided description
- `Category` (string, required, 50 chars) - Predefined category: "Project Documents", "Team Resources", "Personal Files", "Reports", "Presentations", "Other"
- `FileName` (string, required, 255 chars) - Original filename from upload
- `FilePath` (string, required, 500 chars) - Secure storage path (GUID-based)
- `FileSize` (long, required) - File size in bytes
- `FileType` (string, required, 255 chars) - MIME type (e.g., "application/pdf")
- `UploadDate` (DateTime, required) - When document was uploaded
- `UploaderId` (string, required, FK to User) - User who uploaded the document
- `ProjectId` (int, optional, FK to Project) - Associated project if applicable
- `Tags` (string, optional, 500 chars) - Comma-separated tags for search

**Relationships:**
- Belongs to User (Uploader) - Many-to-One
- Belongs to Project (optional) - Many-to-One
- Has many DocumentShare - One-to-Many

**Validation Rules:**
- Title: Required, 1-200 characters
- Description: Optional, max 1000 characters
- Category: Must be one of predefined values
- FileSize: Max 25MB (26,214,400 bytes)
- FileType: Must be in whitelist (PDF, Office docs, text, images)
- FilePath: Must follow secure pattern `{userId}/{projectId}/{guid}.{ext}`

### DocumentShare

Entity representing document sharing relationships between users.

**Fields:**
- `DocumentShareId` (int, PK) - Auto-incrementing primary key
- `DocumentId` (int, required, FK to Document) - Shared document
- `SharedById` (string, required, FK to User) - User who shared the document
- `SharedWithId` (string, required, FK to User) - User receiving the share
- `SharedDate` (DateTime, required) - When document was shared
- `Permissions` (string, required, 50 chars) - Permission level: "View", "Download"

**Relationships:**
- Belongs to Document - Many-to-One
- Belongs to User (SharedBy) - Many-to-One
- Belongs to User (SharedWith) - Many-to-One

**Validation Rules:**
- Cannot share with self
- Document owner can always share
- Project members can share project documents

### DocumentCategory

Enumeration/class for predefined document categories.

**Values:**
- "Project Documents" - Documents related to specific projects
- "Team Resources" - Shared team resources and templates
- "Personal Files" - User's personal work documents
- "Reports" - Business reports and analytics
- "Presentations" - Slides and presentation materials
- "Other" - Miscellaneous documents

## Business Rules

### Document Ownership
- Only uploader can edit/delete their documents
- Project managers can delete documents in their projects
- Administrators have full access to all documents

### Sharing Rules
- Document owners can share with any user
- Shared documents grant view/download permissions
- Recipients receive in-app notifications
- Shared documents appear in "Shared with Me" section

### Security Rules
- Files stored outside wwwroot with secure paths
- Access controlled by user permissions
- IDOR protection via user context validation
- File type and size validation on upload

## State Transitions

### Document Lifecycle
1. **Created**: Document uploaded, metadata saved, file stored
2. **Shared**: Document shared with other users (creates DocumentShare records)
3. **Updated**: Metadata edited or file replaced
4. **Deleted**: Document and file permanently removed

### Share Lifecycle
1. **Created**: Share record created, notification sent
2. **Accessed**: Recipient views/downloads shared document
3. **Revoked**: Owner removes share (deletes DocumentShare record)

## Data Integrity

### Constraints
- Foreign key constraints on all relationships
- Unique file paths to prevent conflicts
- Category values restricted to predefined list
- File size limits enforced at application level

### Indexing
- UploadDate for sorting recent documents
- UploaderId for user's document lists
- ProjectId for project document views
- Category for filtering
- Full-text index on Title, Description, Tags for search

## Migration Considerations

### Existing Schema Impact
- No changes to User or Project tables
- New Document and DocumentShare tables
- ApplicationDbContext updated with new DbSets

### Data Migration
- No existing data to migrate
- Fresh tables for new feature
- File storage directory created on first use</content>
<parameter name="filePath">c:\Projects\ContosoDashboard\specs\001-add-document-management\data-model.md