# ITSM Backend API

ASP.NET Core 8 / EF Core (SQL Server) / Clean Architecture / JWT + Refresh Token /
Dynamic Permission Authorization / SignalR Ticket Chat / FluentValidation / AutoMapper / Serilog.

## Projects
- `ITSM.Domain` — entities, enums
- `ITSM.Application` — DTOs, interfaces, validators, AutoMapper profile, constants/exceptions
- `ITSM.Infrastructure` — DbContext, repositories, services (Auth, Jwt, Password, Audit, User, Department, Ticket, Chat), seed data
- `ITSM.API` — controllers, SignalR hub, JWT + dynamic permission authorization, middleware, Program.cs

## Setup

1. Update `src/ITSM.API/appsettings.json`:
   - `ConnectionStrings:DefaultConnection` — your SQL Server instance
   - `Jwt:Key` — a long random secret (32+ chars), do not commit a real secret

2. Restore and build:
   ```
   dotnet restore
   dotnet build
   ```

3. Create the initial migration (this was NOT pre-generated since it requires the
   .NET SDK + EF Core tools to run against your target SQL Server):
   ```
   dotnet tool install --global dotnet-ef
   cd src/ITSM.API
   dotnet ef migrations add InitialCreate --project ../ITSM.Infrastructure --startup-project .
   ```

4. Run the API (migrations apply automatically via `DbSeeder.SeedAsync` on startup,
   which calls `Database.MigrateAsync()`):
   ```
   dotnet run --project src/ITSM.API
   ```

5. Open Swagger at `https://localhost:7001/swagger`.

## Default seeded data
- Roles: Admin, IT Manager, Technician, Employee
- Permissions: all 17 from spec, assigned per role (Admin = all; IT Manager = user/department/ticket/chat/asset/report/approval management; Technician = ticket handling + chat + asset; Employee = create/view own tickets + chat)
- Default Department: "IT Department"
- Default Admin user: `admin@itsm.local` / `Admin@12345` — **change this password immediately via `/api/auth/reset-password` or `/api/auth/change-password` in production.**

## Known spec gaps (intentionally NOT implemented per "no invented entities" rule)
The spec defines `asset.create`, `asset.update`, `report.view`, and `approval.approve`
permissions but never defines the Asset, Report, or Approval entity fields. Controllers
(`AssetsController`, `ReportsController`, `ApprovalsController`) exist with correct
permission enforcement wired up, but return HTTP 501 since there is no entity schema to
persist against. Provide field definitions for these three entities to complete them.

## Phase 2 — Core Business Modules
Added on top of Phase 1 (Auth/Users/Roles/Departments/Permissions/Audit — unchanged):

- **Tickets**: full lifecycle (New → InProgress → Resolved → Closed, plus PendingUser/PendingVendor/Cancelled), ticket numbers, categories, queue visibility (creator sees own; IT/Admin see all; unassigned visible to all IT users), take-ownership flow.
- **Two chat channels**: `TicketConversationMessage` (creator + assigned technician only) and `TicketInternalMessage` (IT department users only), each with REST endpoints + a dedicated SignalR hub (`/hubs/ticket-conversation`, `/hubs/ticket-internal-chat`), group name = TicketId (or `internal-{TicketId}` for the internal channel).
- **SLA**: policy table seeded per spec (Critical 15m/4h, High 30m/8h, Medium 2h/24h, Low 8h/72h), response/resolution due dates calculated on ticket creation, resolution SLA pauses while `PendingUser`/`PendingVendor`, response SLA never pauses, breach flags only (never auto-closes).
- **Auto-close**: `TicketLifecycleBackgroundService` polls every minute; closes tickets 5 hours after `ResolvedAt` with no creator reply/reopen, and fires SLA breach evaluation for all open tickets.
- **Notifications**: in-app `Notification` entity + `INotificationService`, fired for Created/Assigned/Updated/Resolved/Closed/SlaBreached. New-ticket creation also emails all IT department users via `IEmailService` (SMTP; logs only if `Smtp:Host` is unconfigured, so it runs without a mail server in dev).
- **Attachments**: `TicketAttachment` entity, disk storage under `Attachments:StorageRoot` (default `<app>/attachments`), enforces PDF/DOCX/XLSX/JPG/PNG/ZIP + 25MB limit.

### Phase 2 spec gaps / assumptions (flagged per "stop and ask" rule)
1. **CQRS**: the Phase 2 brief asks for Commands/Queries but also says "follow existing architecture exactly," and Phase 1 has no CQRS/MediatR — it's a plain service-layer Clean Architecture. I kept the existing service pattern rather than introducing a second pattern. Say the word if you actually want MediatR added.
2. **Ticket.Priority**: not in the literal Phase 2 "Ticket Fields" list, but required because the SLA Policy table is keyed by Priority. Added as the minimum field needed to satisfy the stated SLA rules.
3. **"IT Department Users"**: the existing `Department` entity has no `IsITDepartment` flag, so "IT user" is defined as Role ∈ {Admin, IT Manager, Technician} via `IItUserDirectory`. If you have an actual IT department row you want used instead, say so and I'll switch the check to DepartmentId.
4. **Category/SLA management permissions**: spec's permission list has no `category.create/update`; those endpoints reuse `ticket.update`. SLA update endpoint uses the explicit `sla.manage` permission as given.
5. **`ticket.chat.attach` and `ticket.reopen`** existed in Phase 1's permission set but are absent from the Phase 2 list — removed. Reopening is handled via `POST /api/tickets/{id}/status` with `status=InProgress`. Attachments are gated by `ticket.update` since no dedicated permission was listed for them.

### New EF Migration
Same flow as Phase 1 — generate a fresh migration capturing all Phase 2 tables/columns:
```
dotnet ef migrations add Phase2_TicketsSlaChatNotificationsAttachments --project src/ITSM.Infrastructure --startup-project src/ITSM.API
```
Applied automatically on startup via `DbSeeder.SeedAsync` → `Database.MigrateAsync()`.


## Production Hardening Pass
See the hardening report delivered alongside this codebase for the full file-by-file
breakdown. New migration needed for: `Ticket.DepartmentId`, `Ticket.RowVersion`,
`Department.IsDeleted`, `AuditLog.EntityType`/`EntityId`, plus the new indexes on
`Tickets(Status, AssignedToId, CreatedById, DepartmentId, Priority, CreatedAt)`:
```
dotnet ef migrations add Hardening_DeptIsolation_Concurrency_SoftDelete_Audit --project src/ITSM.Infrastructure --startup-project src/ITSM.API
```
Applied automatically on startup via the existing `DbSeeder.SeedAsync` → `Database.MigrateAsync()` call.

**One-time data backfill needed**: existing Ticket rows (if any) predate the
`DepartmentId` column and will have `Guid.Empty` until backfilled — run an UPDATE setting
each ticket's `DepartmentId` from its `CreatedBy.DepartmentId` after migrating, or they will
be invisible to Technician/IT Manager department-scoped queries until backfilled.
