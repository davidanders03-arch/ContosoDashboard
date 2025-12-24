<!--
Version change: N/A → 1.0.0
List of modified principles: All principles added (new constitution)
Added sections: Technical Standards, Development Process
Removed sections: None
Templates requiring updates: plan-template.md (constitution check updated)
Follow-up TODOs: RATIFICATION_DATE needs historical research
-->
# ContosoDashboard Constitution

## Core Principles

### I. Spec-Driven Development (NON-NEGOTIABLE)
All features must be fully specified before implementation begins. Specifications drive development, not vice versa. Every feature requires complete user stories, acceptance criteria, and technical design before coding starts. Rationale: Ensures quality, prevents scope creep, and enables parallel development.

### II. Clean Architecture
Application follows layered architecture with clear separation of concerns. Business logic isolated from UI and data layers. Dependency injection used throughout. Services are interface-based for testability and flexibility. Rationale: Maintainable, testable, and adaptable codebase.

### III. Security First
Security considerations integrated into every feature from design phase. IDOR protection, input validation, and authorization checks mandatory. No feature deployed without security review. Rationale: Protects user data and prevents vulnerabilities in production.

### IV. Test Coverage
All business logic must have unit tests. Integration tests required for critical paths. UI components tested for accessibility and responsiveness. Minimum 80% code coverage enforced. Rationale: Ensures reliability and prevents regressions.

### V. User-Centric Design
Features designed around user needs and workflows. Responsive design for all devices. Accessibility standards (WCAG 2.1) followed. User feedback incorporated into iterations. Rationale: Delivers value and ensures adoption.

## Technical Standards

Technology stack standardized on ASP.NET Core with Blazor Server. Entity Framework Core for data access. Bootstrap for UI consistency. All code follows C# coding standards with nullable reference types enabled. Database migrations version-controlled.

## Development Process

Features follow: Spec → Plan → Tasks → Implement → Test → Review → Deploy. Code reviews mandatory for all changes. CI/CD pipeline enforces quality gates. Documentation updated with each feature.

## Governance

Constitution supersedes all other practices. Amendments require consensus from development team and documentation of rationale. Compliance verified in code reviews and automated checks. Version follows semantic versioning.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): Original adoption date unknown | **Last Amended**: 2025-12-24
