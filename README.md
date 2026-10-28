# Genie A2A Bridge

This project is a Mule 4 application scaffold for the Genie A2A Bridge described in the build brief. It follows the architecture and contract in the brief while keeping all Genie-specific settings in the `genie.*` namespace and all A2A server wiring in a reusable subflow model.

## Purpose

The application exposes a Databricks Genie space as an A2A 1.0 agent behind a gateway. Its responsibilities are:

- authenticate at the gateway layer and trust the bearer token passed down
- resolve or start a Genie conversation for an A2A context
- submit one user question as a Genie message
- poll until a terminal state is reached
- convert Genie attachments into A2A artifact parts
- persist task/context mappings and task metadata
- emit a single terminal result per task

## Project layout

- `src/main/mule/global.xml` — HTTP, A2A, Object Store and Databricks configuration
- `src/main/mule/a2a-server.xml` — task authorizer, task listener and terminal error mapping
- `src/main/mule/genie-ask.xml` — orchestration for topic submission, polling and artifact creation
- `src/main/mule/genie-client.xml` — Databricks Genie HTTP client fragments and retry logic
- `src/main/mule/health.xml` — `/health` and `/ready`
- `src/main/resources/dw/*` — transformation modules for statuses, error mappings and attachments
- `src/test/munit/*` — MUnit coverage for new context, existing context, timeout and expiration cases

## Important implementation notes from the brief

### Connector and task metadata contract

The exact A2A connector DSL for task status and artifact updates varies by connector version. The application skeleton assumes the connector exposes the same operational shapes described in the brief:

- `Update Task Status` uses a status update object with `taskId`, `contextId`, `status.state` and `status.message`
- `Update Task Artifact` uses an artifact update object with `artifactId`, `name` and `parts`
- `Set Task Metadata` is used after each task and each poll, so calling clients can resolve the Genie IDs without a side channel

These details must be validated against the connector reference in the target Exchange version before production deployment.

### Context ID decision

The brief asks whether the connector allows the server flow to set the A2A `contextId` when the client supplies none. If it does, the preferred design is to reuse the Genie `conversation_id` as the A2A `contextId` and keep only the `taskId` mapping in Object Store. The skeleton leaves this behaviour configurable so the team can validate the actual connector capability in the target environment.

### Multi-Genie packaging

The brief prefers a single Genie deployment per artifact, but connotes a future multi-Genie packaging path. This project keeps all Genie-specific values under the `genie.*` namespace and isolates the interaction logic in a reusable subflow so a later version can register multiple A2A server configs that call the same subflow with different variable sets.

The connector’s tenant support should be reviewed if a single A2A server config is intended to serve several Genies with tenant-scoped routing. The skeleton does not assume tenant routing for this first build.

### Authorization and `ListTasks`

The app requires a bearer header on `SendMessage`, `GetTask`, `SubscribeToTask` and other operation types, and it scopes `ListTasks` by a caller identity header when the gateway forwards it. If the gateway does not add a caller identifier, `ListTasks` is intentionally denied with an empty result set.

### Runtime Fabric Object Store

The connector’s task repository should be validated against Runtime Fabric for persistence across pod restarts. The bridge also keeps its own context mapping store with TTL, and the skeleton uses a persistent object store configuration so this stays aligned with the platform guidance.

## Configuration summary

All keys are in the `genie.*` namespace and are declared in the property files under `src/main/resources`.

- `genie.agent.slug`
- `genie.workspace.host`
- `genie.space.id`
- `genie.auth.mode`
- `genie.auth.token`
- `genie.caller.header`
- `genie.poll.intervalMs`
- `genie.poll.maxAttempts`
- `genie.retry.maxAttempts`
- `genie.retry.initialMs`
- `genie.http.timeoutMs`
- `genie.table.max_rows`
- `genie.table.card_rows`
- `genie.include_reasoning`
- `genie.clarification.detect`
- `genie.context.ttl`
- `genie.log.content`
- `genie.card.*`
- `a2a.listener.port`
- `a2a.agent.path`

## Local commands

```bash
mvn clean package
```

For local development, use the `genie-dev.properties` profile with static auth and a fake host.

## Manual contract checks

The README in the original brief calls out the following checks to perform against a real Genie space:

- `message:send` blocking call
- `message:stream` showing progress updates then artifact
- `tasks/get` after a streamed completion
- a follow-up message with the returned `contextId`
- `tasks/cancel` unsupported result

These should be executed against the exact A2A connector artifact used in your environment.

## Known follow-up actions before production

1. Validate the exact A2A connector DSL and supported fields for status updates, task metadata and agent card generation in the installed connector version.
2. Confirm the server flow can set `contextId` during task creation and whether headers are exposed in the task authorizer flow or only there.
3. Verify the Runtime Fabric Object Store and persistent task repository behaviour for restarted pods.
4. Replace the placeholder connector versions in `pom.xml` with the connector versions pinned in your enterprise Exchange environment.

## License

This project is a scaffold created from the supplied design brief and is ready for enterprise adaptation and connector-specific tuning.
