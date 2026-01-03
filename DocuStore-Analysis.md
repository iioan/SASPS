# DocuStore Architectural Comparison

High-level review of the dual DocuStore implementations that compare **Active Record** with **Repository + Unit of Work**. Both target document CRUD, versioning, tagging, and search on .NET 10 + PostgreSQL behind minimal APIs and a Docker Compose setup.

## 1. Project Structure & Layering
- **Modules:** Document, Versioning, Tagging, MetadataIndexing (+ Shared utilities) in both `docustore-activerecord/` and `docustore-repoUow/`.
- **Layers per module:**
  - **API:** Minimal API endpoints grouped by resource (e.g., `/api/documents`, `/api/versions`, `/api/tags`, `/api/search`). Swagger is enabled via `DocuStore.Gateway`.
  - **Application:** MediatR commands/queries + validators. Coordinates domain actions and integrates cross-module events.
  - **Domain:** 
    - *Active Record:* Entities inherit `ActiveRecordBase`, hold persistence logic, and pull dependencies via a static `ServiceLocator` (`Document.Domain/Entities/DocumentEntity.cs`).
    - *Repository+UoW:* Domain entities are persistence-ignorant POCOs; persistence contracts live in `Document.Application/Interfaces`.
  - **Infrastructure:** EF Core DbContexts, migrations, and per-module wiring. Repository+UoW registers repositories + units of work; Active Record mostly wires DbContexts and initializes service locators (`DocuStore.Gateway/Program.cs` lines 13-23).

## 2. Endpoints & API Design
- **Documents:** `POST /api/documents`, `GET /api/documents`, `GET /api/documents/{id}`, `PUT /api/documents/{id}`, `DELETE /api/documents/{id}`, `GET /api/documents/{id}/download`.
- **Versions:** `POST /api/versions`, `GET /api/versions/document/{docId}`, `PUT /api/versions/document/{docId}/set-current`, `GET /api/versions/document/{docId}/version/{n}/download`.
- **Tagging:** `POST /api/tags`, `GET /api/tags`, `POST /api/tags/documents/{docId}/tags`, `DELETE /api/tags/documents/{docId}/tags/{tagId}`, `GET /api/tags/documents/{docId}/tags`, `GET /api/tags/{tagId}/documents`, `GET /api/tags/documents?tagIds=a,b`.
- **Search:** `GET /api/search/documents` (filters, pagination, sorting), `POST /api/search/reindex`.
- **REST design:** Resource-oriented URIs, HTTP verbs, and error semantics are consistent across both implementations. Document listing lacks built-in pagination, which may limit scalability. Controllers are expressed as minimal APIs; Repository+UoW keeps composition strictly in the host builder, whereas Active Record additionally boots service locators.

## 3. Patterns in Use
- **Common:** DTO mapping between API and domain, MediatR for request handling, FluentValidation validators on commands/queries, and a layered modular monolith layout per bounded context.
- **Active Record specifics:**
  - Domain entities own persistence and querying (e.g., `Save`, `Find`, `All`) and directly publish events after DB writes.
  - Dependencies are pulled via a static `ServiceLocator`, coupling domain logic to infrastructure and static global state.
  - Example (`docustore-activerecord/src/Document/Document.Domain/Entities/DocumentEntity.cs`):
    ```csharp
    public async Task UploadAndSave(byte[] fileContent, CancellationToken ct = default)
    {
        FilePathOnDisk = await GetService<IFileStorageService>()
            .CreateDocumentFolderAsync(this.Id, this.FileName, ct);
        await Save(ct); // writes via DbContext from ServiceLocator
        await GetService<IEventPublisher>().PublishAsync(new DocumentCreatedEvent(...), ct);
    }
    ```
- **Repository + Unit of Work specifics:**
  - Persistence contracts (`IDocumentRepository`, `IUnitOfWork`) and transaction helpers are injected; domain stays persistence-ignorant.
  - Application handlers orchestrate repositories and event publication:
    ```csharp
    public async Task<DocumentDto> Handle(CreateDocumentCommand request, CancellationToken ct)
    {
        var document = DocumentEntity.Create(...);
        var path = await _fileStorageService.CreateDocumentFolderAsync(document.Id, document.FileName, ct);
        document.SetFileInfo(path, request.FileContent.Length);
        await _unitOfWork.Documents.AddAsync(document, ct);
        await _unitOfWork.SaveChangesAsync(ct);
        await _eventPublisher.PublishAsync(new DocumentCreatedEvent(...), ct);
        return new DocumentDto(...);
    }
    ```
  - Db access is centralized in repositories; UnitOfWork exposes optional transaction boundaries.

## 4. Anti-Patterns & Code Smells
- **Active Record:**
  - Static `ServiceLocator` + `DocumentDbContextProvider` in domain code tightly couple business logic to EF and global state, hindering testability and dependency tracing.
  - Domain methods intermix persistence, orchestration, and validation; events are published outside an explicit transaction boundary.
  - Queries like `All/Where/Count` run directly from entities, encouraging logic spread across the domain instead of through application services/repositories.
  - Tests require a live PostgreSQL instance; failures occur when Docker DB is unavailable (see test run notes below).
- **Repository + UoW:**
  - Extra abstraction adds boilerplate and learning overhead.
  - UnitOfWork exposes transaction methods, but most handlers only call `SaveChangesAsync`, so cross-aggregate workflows may still lack explicit transactional coordination.

## 5. Architectural Choices
**Good:**
- Clear module boundaries for Document, Versioning, Tagging, and Search.
- Consistent API surface with Swagger, CORS, and MediatR-driven handlers.
- Event-driven integration (e.g., `DocumentCreatedEvent` feeding versioning/indexing) promotes cross-module decoupling.
- Repository+UoW variant cleanly separates concerns and is highly testable (pure domain tests, mocked infrastructure).

**Questionable:**
- Active Record’s service locator and static DbContext access introduce hidden dependencies and make unit testing difficult; persistence and domain rules are entwined.
- Lack of pagination on document listing endpoints could impact performance at scale.
- Event publication is not wrapped in a transaction in either variant; failures after DB writes could leave downstream modules inconsistent.
- Package version warnings in Repository infrastructure (EF vs Npgsql rc) could become a maintenance hazard if left unresolved.

## 6. Testing Strategy
- **Unit/Domain:** Repository+UoW has fast, isolated domain tests (e.g., `Document.Domain.Tests/Entities/DocumentEntityTests.cs`) and application handler tests with mocked repositories/events.
- **Active Record:** Domain tests are integration-style against PostgreSQL (`docustore-activerecord/tests/...`) and need Docker + applied migrations; current local run failed because Postgres was unreachable.
- **Performance:** k6 suites in `performance-tests/` with generated summaries (`run-all-tests.sh` / `analyze-results.js`). Scenarios cover smoke, load, stress, scalability (data/users), pagination, soak, and concurrent writes.
- **Test runs here:** Repository+UoW domain/application/infrastructure tests **pass**. Active Record domain tests **failed** to start due to `Connection refused` on `127.0.0.1:5432` (database not running).

## 7. Comparative Metrics (from `performance-tests/reports/comparison-report-2026-01-03T19-48-10.md`)
- **Smoke (baseline CRUD):** Avg 5.18 ms (AR) vs 4.95 ms (Repo); throughput 3.8 req/s; 0% errors (Repo slightly faster).
- **Load, 50 users:** Avg 6.16 ms vs 6.45 ms; P95 19.49 ms vs 28.96 ms; throughput 2.65 vs 2.50 req/s. Error rate 0.80% (AR) vs 0.20% (Repo) — AR faster but less reliable.
- **Stress, ramp to 200 users:** Avg 10.29 ms vs 8.42 ms; throughput 3.01 vs 3.69 req/s (Repo win). Error rate 0.91% (AR) vs 1.37% (Repo).
- **Data volume (100→10k docs):** Avg 3.70 ms vs 3.88 ms; throughput 4.67 vs 4.75 req/s; 0% errors (near parity).
- **User concurrency (5→200 users):** Avg 8.08 ms vs 8.40 ms; throughput 2.36 vs 2.30 req/s. Error rate 0.37% (AR) vs 1.13% (Repo).
- **Takeaway:** Active Record edges some latency numbers under load, but Repository+UoW shows steadier error rates in critical scenarios (smoke/load).

## 8. Trade-offs Summary
- **Active Record**
  - ✅ Simple flow (entities handle their own persistence), fewer layers to grasp, slightly lower latency in some runs.
  - ❌ Tight coupling to EF and service locator, harder to unit test, event consistency relies on implicit DbContext state, scaling changes risk cross-cutting edits.
- **Repository + Unit of Work**
  - ✅ Separation of concerns, high testability, clearer transaction boundaries, repositories centralize queries. Good fit for evolving requirements.
  - ❌ More boilerplate (interfaces, UoW, DI wiring) and a steeper learning curve; requires discipline to use transactions across handlers.

## 9. Actionable Insights
- Prefer the **Repository + UoW** variant for maintainability and testability; keep using MediatR + validators and event handlers per module.
- In the Active Record variant, consider replacing the service locator with DI, and isolate persistence from domain logic to improve test coverage.
- Add pagination/filtering to `GET /api/documents` to align with existing search and pagination tests.
- Wrap event publication with transaction boundaries (or outbox) in both variants to avoid inconsistent downstream states after DB writes.
