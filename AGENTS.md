# AGENTS

This file is the single source of operational knowledge for agentic helpers working in MudMCP. Use it before you start coding or issuing commands so that your workflows align with the repository conventions.

## Goals

- Provide runnable build, test, and runtime commands so you can reproduce the feedback loop an operator expects.
- Capture the code style, naming, and error-handling patterns that every tool, service, and test already follows.
- Point to repo-specific agent guidance so you can reuse the existing Copilot/AI instructions without duplication.

## Build / Lint / Test Commands

- `dotnet build` from the repo root (`MudBlazor.Mcp.slnx`) restores packages, runs analyzers, and compiles every project in one shot.
- `dotnet test --no-build` runs the entire xUnit suite without rebuilding (use this after a successful build to save time).
- Target the test project directly with `dotnet test tests/MudBlazor.Mcp.Tests` when you only need the unit suites under `tests/`.
- Watch mode is available via `dotnet watch test --project tests/MudBlazor.Mcp.Tests` for rapid iteration.
- Run a single test with `dotnet test --filter "FullyQualifiedName~ComponentListToolsTests"` or supply `FullyQualifiedName~Namespace.Class.Method` to isolate just that method.
- Collect coverage via `dotnet test --collect:"XPlat Code Coverage"` when you need report data for PRs.
- There is no dedicated lint command; rely on `dotnet build` for compiler diagnostics and apply-formatting as needed.

### Running the server

- HTTP transport: `dotnet run --project src/MudBlazor.Mcp/MudBlazor.Mcp.csproj -- --version X.Y.Z` (replace X.Y.Z with the MudBlazor version referenced in your `.csproj`).
- stdio transport (Cursor/Claude): `dotnet run --project src/MudBlazor.Mcp/MudBlazor.Mcp.csproj -- --stdio --version X.Y.Z`.
- The `README.md` and `.github/copilot-instructions.md` emphasize that `--version` is required; the host validates it before startup.
- Aspire dashboard host: `dotnet run --project src/MudBlazor.Mcp.AppHost/MudBlazor.Mcp.AppHost.csproj` to spin up Aspire 13.1 with health checks, telemetry, and service discovery.

### Container & Docker commands

- `docker compose up --build -d` builds the image and starts the container (MudBlazor clone lives in a named volume).
- `docker compose logs -f` follows the startup log stream; useful during the first clone (> 500 MB).
- `docker compose down` stops containers while keeping the cached MudBlazor clone.
- `docker compose down -v` removes the cached volume so the next run reclones everything.
- Use `curl http://localhost:5180/health` (and `/health/ready`, `/health/live`) to verify the HTTP server in Docker or when running locally.

## Code Style Guidelines

### Imports & Usings

- Keep the copyright/license banner at the top, followed by grouped `using` statements (System → Microsoft → ModelContextProtocol → MudBlazor).
- Use file-scoped `namespace` declarations (`namespace MudBlazor.Mcp;`) to keep files concise and match the existing layout.

### Formatting & Layout

- Preserve a single blank line between the license banner, `using` block, namespace, and type declarations.
- Public APIs, MCP tools, and service methods require XML docs (`/// <summary>`) and `[Description]` attributes on user-facing parameters.
- Prefer expression-bodied helpers when the logic fits on one line; expand braces for multi-line control flow blocks, especially inside tool implementations.
- When constructing Markdown responses, use `StringBuilder` with `AppendLine` and keep the generated text around 120 columns for readability.
- Keep helper methods (e.g., truncation, validation helpers) near the top or in dedicated partial files so readers don’t have to scroll past dozens of tools.

### Types & Naming

- PascalCase everything public (classes, records, methods, properties); append `Async` to asynchronous methods returning `Task` or `ValueTask` (e.g., `BuildIndexAsync`).
- Parameters and locals use camelCase, and private readonly fields adopt the `_camelCase` pattern for clarity (`_logger`, `_indexLock`).
- Domain models like `ComponentInfo`, `ComponentParameter`, etc., remain immutable `record`s; use `with` expressions when you need to derive a variant.

### Error Handling & Validation

- Validate incoming tool arguments with `ToolValidation` helpers (e.g., `RequireNonEmpty`, `RequireInRange`) before hitting business logic and throw `McpException` with helpful recovery hints.
- Tools that look up domain data should call `ToolValidation.ThrowComponentNotFound` or `ThrowCategoryNotFound` when the indexer returns null—this keeps clients inside the protocol.
- Guard long-running work with `try/catch` filters and log at the appropriate level; use `catch (Exception ex) when (ex is not OperationCanceledException)` to preserve cancellation tokens.

### Logging & Observability

- Inject `ILogger<T>` across services/tools and log structured templates (`_logger.LogInformation("Indexed {Count} components", count);`).
- Use `LogDebug/LogTrace` for diagnostics, `LogInformation` for lifecycle events, `LogWarning` for recoverable hiccups, `LogError` before rethrowing, and send all console logs to stderr in stdio mode (the host already configures `LogToStandardErrorThreshold`).

### Async & Cancellation

- Every async service method accepts `CancellationToken cancellationToken = default`; check `cancellationToken.ThrowIfCancellationRequested()` before starting I/O loops.
- When calling async APIs in libraries/services, append `.ConfigureAwait(false)` unless you are deliberately running on the ASP.NET Core synchronization context (Program.cs usually omits it for simplicity).
- Protect shared resources with `SemaphoreSlim`, `ConcurrentDictionary`, or `async` locks as shown in `GitRepositoryService` and `ComponentIndexer`.

### Testing & Tool Development

- Tests live under `tests/MudBlazor.Mcp.Tests/` and mirror the `src` tree (Tools, Services, Parsing).
- Use xUnit + Moq with `NullLoggerFactory.Instance.CreateLogger<T>()`, keep tests focused (one assertion per behavior when feasible), and name them `Method_Scenario_ExpectedResult`.
- Favor `[Theory]` with `[InlineData]`/`[MemberData]` over duplicate `[Fact]` tests when working with multiple inputs.
- Each new tool stays static, is decorated with `[McpServerTool]`, annotates user parameters with `[Description]`, and has a paired test covering both success and `McpException` paths (see `.github/copilot-instructions.md`).

## MCP Tool Guidelines

- Tool classes are marked with `[McpServerToolType]`, and each method specifies `[McpServerTool(Name = "...")]` so the protocol can discover it automatically.
- Implement each tool as a stateless static method that receives services (indexer, loggers, parsers) via parameters rather than constructors.
- Format responses as Markdown (headings, bullet lists, code fences) instead of raw JSON so AI clients get readable answers.
- Log the incoming request context (`componentName`, filters, include flags) at `LogDebug` or `LogTrace` level before hitting the indexer for easier debugging.
- Reference `VersionContext.Version` in output to confirm the tool served the expected MudBlazor release.

## Style Resources

- `docs/04-best-practices.md` codifies the error handling, logging, performance, and resource-management expectations this repo enforces.
- `docs/07-testing.md` lists the xUnit/Moq patterns, mocking helpers, builder utilities, and command-line filters (single test, coverage).
- `README.md` and `docs/01-overview.md` outline the quick-start workflow, the requirement to match MudBlazor versions, and the contribution steps.

## Agent References

- Read `.github/copilot-instructions.md` for extra architectural notes, tool anatomy, aspirational commands, and the required GPL header.
- There are no `.cursor` or `.cursorrules` files today; if they appear under `.cursor/rules/`, summarize them here so future agents obey them.

## When Editing

- Preserve the existing license header in every `.cs` file and mimic it when adding new helpers or tools.
- Keep Razor/Markdown docs tidy with four-space indents and blank lines between major sections to match the docs under `docs/`.
- Never leak absolute paths to MudBlazor sources in MCP responses; stick to component names, relative metadata, or friendly guidance instead.
- Add or update tests whenever you change parsing, indexing, or tool logic; new tools get a test class under `tests/.../Tools`.

## Debugging Tips

- Run the server with `--stdio` in one terminal and curl against `/mcp` from another to simulate Cursor/Claude traffic; the `docs/09-ide-integration.md` file demonstrates this.
- Review `.github/agents/mudblazor-expert.agent.md` for examples of how agents chain searches and tool calls when answering real questions.
- When the index build fails, check the logs from `VersionCacheManager` (clone/evict details) and `ComponentIndexer` (directory parsing errors) to pinpoint the issue.
- Do `docker compose logs -f` before `docker compose down` if you need to capture the cached clone status.

## Next Steps for Agents

- After making code changes, run `dotnet build` plus the relevant `dotnet test --filter "FullyQualifiedName~..."` command so you can cite the exact happy path that succeeded.
- Update the docs under `docs/` when commands, configuration, or agent guidance changes; add the entry to the Table of Contents in `docs/01-overview.md` when necessary.
- Mention any new third-party dependency in your PR description so reviewers can double-check GPL-2.0 compatibility.

## Verification

- Ensure `dotnet test --no-build` passes with no failing assertions; rerun the filtered command you used during development to confirm the fix.
- Document any new environment variables or configuration overrides required by your change inside this AGENTS file or the relevant `docs/` page.
- When tests touch cached clones, explain how to reset `./data` (delete the directory and let the server reclone) in the PR description.

## Contact

- Use GitHub issues or pull requests (per README) to request reviews before pushing service-default or Aspire instrumentation changes.
- Mention `MudBlazor.Mcp.AppHost` whenever you alter the Aspire host so the AppHost maintainers can review telemetry/health-check tweaks.
- Tag new MCP tools in issue descriptions so reviewers can route them to the tooling/agent maintainers.
- Keep examples in this AGENTS file up-to-date; revise it whenever you change commands, style rules, or agent operating assumptions.
- This document targets agentic helpers; treat it as the first reference before executing code in this repo.

## Global Configuration

- `global.json` pins the SDK to .NET 10; install that exact preview SDK locally or in CI to avoid restore/build mismatches.
- `Directory.Build.props` and `Directory.Packages.props` centralize package versions; avoid duplicating versions in individual project files.
- Application settings for repository cloning, caching, and parsing live in `src/MudBlazor.Mcp/appsettings*.json` and are shared between HTTP and stdio hosts.

## Documentation Upkeep

- `docs/08-mcp-inspector.md` describes integration testing with MCP Inspector—refer there when you add feature-level tests.
- `docs/05-tools-reference.md` mirrors every tool signature and description; keep it aligned whenever you add or rename a tool.

## Troubleshooting

- Health checks report component counts via `Program.WriteHealthCheckResponse`; inspect the JSON payload before filing issues about `/health` endpoints.
- `IndexerHealthCheck` logs structured data (`status`, `componentCount`, `lastIndexed`); use that output when diagnosing indexing problems.
- If version-specific clones misbehave, inspect `data/versions.json` and delete the `./data` folder so the next `dotnet run -- --version ...` starts from a clean cache.

- End of AGENTS file (update line count if you add sections).
