# Database Migration Agent

An AI-assisted workflow that reads database migration acceptance criteria from
Jira, generates FluentMigrator C# and SQL artifacts, builds the migration
solution, creates an Azure DevOps branch and pull request, and applies repository
labels/title conventions.

The agent can design migrations for:

- `AddFileTransferRule`
- `UpdateFileTransferRule`
- `DeleteFileTransferRule`

Deactivating a FileTransfer rule is handled through `UpdateFileTransferRule` with
`NewIsActiveFlag: false` or `NewActiveFlag: false`.

## Safety Boundary

The agent may read database values through the approved read-only SQL MI
snapshot lookup. It must never execute generated SQL or change database state.

- SQL MI access is SELECT-only.
- The snapshot lookup is parameterized.
- Database values are not sent to the language model.
- The agent generates and validates migration artifacts only.
- Generated SQL is a review/deployment artifact; this project does not execute
  it or apply migrations.

## Prerequisites

- Python 3.13
- `uv`
- Azure CLI authenticated with `az login`
- Microsoft ODBC Driver 18 for SQL Server
- .NET SDK/build environment used by `DatabaseMigration` (Fluent migrator repo)
- Access to the configured Jira site
- Azure DevOps PAT with Code Read and Write permissions
- A clean local clone of `DatabaseMigration`
- A read-only SQL MI Entra identity for FileTransfer update/delete snapshots

Install and run project commands with `uv`:

```powershell
cd C:\Code\FluentMigratorAI
uv sync --frozen
```

## Environment Configuration

Create a local `.env` file in the project root. Keep it out of Git.

```text
OPENAI_API_KEY=your-openai-key
OPENAI_MODEL=gpt-4.1-mini

JIRA_BASE_URL=https://your-site.atlassian.net
JIRA_EMAIL=you@example.com
JIRA_API_TOKEN=your-jira-api-token

SQLMI_SERVER=your-server.database.windows.net,3342
SQLMI_DATABASE=DbConfig

AZURE_DEVOPS_PAT=your-azure-devops-pat
FM-DATABASE_REPOSITORY_PATH=your-local-fluent-migrator-repo-path 

ADO_ORG_URL=https://dev.azure.com/your-azure-organization
ADO_PROJECT_NAME=Your-Project-Name
ADO_REPO_NAME=Your-Fluent-Migrator-Repo
```

The workflow accepts either `AZURE_DEVOPS_PAT` or the existing `ADO_PAT` name.
Do not put credentials in Git remotes or source files.

## Get a Jira Issue

Print Jira details and acceptance criteria:

```powershell
uv run --frozen jira-issue DE-8559
```

Print the parsed migration payload:

```powershell
uv run --frozen jira-issue DE-8559 --migration-json
```

Jira's Acceptance Criteria field is discovered by its field name. The underlying
custom-field ID does not need to be hardcoded.

## Generate a Migration Without a PR

Read a Jira card and ask the agent to format its migration payload:

```powershell
uv run --frozen databasemigrationagent --jira-card DE-8559
```

This generation path does not create a branch or pull request.

## Run the Full Jira-to-PR Workflow

Replace `DE-XXXX` with the Jira issue key:

```powershell
cd C:\Code\FluentMigratorAI
uv run --frozen databasemigrationagent --workflow --jira-card DE-XXXX --repo-path FM-DATABASE_REPOSITORY_PATH
```

Example:

```powershell
uv run --frozen databasemigrationagent --workflow --jira-card DE-8559 --repo-path FM-DATABASE_REPOSITORY_PATH
```

The workflow:

1. Reads the Jira card and structured Acceptance Criteria.
2. Resolves the exact Jira Sprint and migration folder.
3. For Update/Delete cards, reads only the required previous values through
   the read-only SQL MI snapshot.
4. Generates the FileTransfer migration class, Up/Down scripts, and validation SQL
   locally from the parsed Jira payload and any read-only snapshot.
5. Fails Add migrations when the business key already exists, so rollback
   cannot delete a row the migration skipped.
6. Validates required names, paths, script headers, and PR fields.
7. Builds `DE.DatabaseMigrator.sln`.
8. Creates and pushes a feature branch.
9. Creates an Azure DevOps pull request.
10. Adds `AI_Agent` to the PR label and title.

The workflow does not merge or deploy the migration. It stops instead of
overwriting an existing remote feature branch or duplicate PR.

The workflow does not execute generated SQL or apply migrations. Database
script execution and deployment are outside this agent workflow.

## FileTransfer Acceptance Criteria

### Add rule

```yaml
MigrationType: AddFileTransferRule
RuleName: Daily Zip File
Action: Create
DatasourceName: File_Source1
Pattern: Daily_db_202601_*.zip
SourceLocation: sftpLocal
SourceLocationType: sftp
SourcePath: /Drive2/incoming/DE/file_source1/IN
DestinationLocation: blobLocal
DestinationLocationType: blob
DestinationPath: /Drive2/incoming/DE/file_source1/IN
Frequency: Daily
Schedule: "0600"
RollbackStrategy: DeleteInsertedRule
```

### Update or deactivate rule

Optional current rule fields may be omitted when they are not needed to
identify the matching rows:

```yaml
MigrationType: UpdateFileTransferRule
RuleName: Weekly Quality File
Action: Update
DatasourceName: File_Source2
SourcePath: /Drive2/incoming/DE/file_source2/IN
NewPattern: quality_Report.xlsx
RollbackStrategy: UpdateOldRule
```

To deactivate matching rules:

```yaml
MigrationType: UpdateFileTransferRule
RuleName: Monthly Quality File
Action: Update
DatasourceName: File_Source3
NewIsActiveFlag: false
RollbackStrategy: UpdateOldRule
```

### Delete rule

Use `DatasourceName` for one datasource or comma-separated `DatasourceNames`
for multiple datasources:

```yaml
MigrationType: DeleteFileTransferRule
RuleName: Delete monthly rules
Action: Delete
DatasourceNames: File_Source1, File_Source2, File_Source3
SourcePath: /Drive2/incoming/DE/IN/ManualFileTransfer
DestinationLocation: blobLocal
RollbackStrategy: RestoreDeletedRule
```

Delete migrations:

- Insert matching active-rule rows into
  `DbConfig.FileTransferRulesHistory` with `IsActiveFlag = 0`.
- Make the history insert idempotent with `NOT EXISTS`.
- Delete the matching rows from `DbConfig.FileTransferRules`.
- Make the rollback restore idempotent.
- Add no explicit transaction statements; FluentMigrator/migration repository
  configuration owns the migration transaction.
- Add no existence/count or `@@ROWCOUNT` guards.

## SQL MI Update/Delete Configuration

Set:

```text
SQLMI_SERVER=your-server.database.windows.net,3342
SQLMI_DATABASE=DbConfig
```

Authenticate before running Update/Delete workflows:

```powershell
az login
```

The SQL MI identity must have SELECT-only access to:

- `DbConfig.FileTransferRules`
- `DbConfig.FileTransferRulesHistory`
- `DbConfig.FileLocations`
- `DWNexus.dbo.DataSources`

Update/Delete snapshots may match multiple rows. A snapshot stops generation
when rollback values differ across matched rows or a selected location cannot
be resolved consistently.

## Repository Layout

- `src/databasemigrationagent/jira/`: Jira API client, issue formatting, and CLI
- `src/databasemigrationagent/migration_types/`: migration-type parser registry
- `src/databasemigrationagent/migration_types/FileTransfer/`: FileTransfer Add, Update,
  and Delete acceptance-criteria parsers
- `src/databasemigrationagent/agent.py`: OpenAI workflow and system guidance
- `src/databasemigrationagent/db_connection.py`: read-only SQL MI snapshot lookup
- `src/databasemigrationagent/FileTransfer_update.py`: update script materialization
- `src/databasemigrationagent/FileTransfer_delete.py`: delete/history script materialization
- `src/databasemigrationagent/workflow.py`: Git, build, Azure DevOps, and PR workflow
- `src/databasemigrationagent/migration_knowledge/`: FileTransfer and migration standards

Migration output is written to the configured migration repository under:

```text
src/DE.DatabaseMigrator/Migrations/<SprintFolder>/
src/DE.DatabaseMigrator/Scripts/<SprintFolder>/Up/
src/DE.DatabaseMigrator/Scripts/<SprintFolder>/Down/
```

## Pull Request Conventions

- PR titles include `AI_Agent -`.
- PRs receive the `AI_Agent` label.
- PR descriptions include the Jira link and manual validation SQL.
- Manual validation SQL uses fully qualified table names such as
  `[DbConfig].[config].[FileTransferRules]`.
- Version deployment validation uses `dbo.VersionInfo`.

## Ask the agent a question:
-Run this command
```
PS C:\Code\FluentMigratorAI> uv run python main.py "Explain FileTransfer migration standards"
```

## Tests

Run the complete Python test suite:

```powershell
cd C:\Code\FluentMigratorAI
uv run --frozen python -m unittest discover -p "test_*.py"
```

Build the migration solution separately:

```powershell
dotnet build C:\Code\DatabaseMigration\src\DE.DatabaseMigrator.sln
```
