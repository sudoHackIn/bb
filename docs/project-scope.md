# Project as the Primary Application Scope

Status: proposal

## Summary

Make a bb project the primary application scope. A project normally represents
one repository or independently operated service and owns the product objects
associated with that service: threads, tasks, OpenSpec documents, automations,
and plugin data.

The application also provides an explicit **All projects** scope for portfolio
views, cross-project search, and filtering. All projects is a view scope, not a
synthetic project.

## Motivation

A user may operate tens or hundreds of services. Each service has its own
repository, normal working directory, tasks, specifications, and agent work.
Presenting every thread and every plugin object in one implicit global space
makes it difficult to focus on one service and encourages plugins to invent
their own project concepts.

The current domain model already treats Project as a top-level container and
threads carry a project ID. However, the application does not consistently use
the selected project as the scope of every surface. In particular, a global
plugin panel cannot distinguish a selected project from an unscoped route, and
the Tasks plugin maintains a second project entity linked back to a bb project.

## Goals

- Make one selected bb project the normal context for navigating and creating
  work.
- Let users deliberately switch to an All projects scope.
- Apply the same scope to threads, tasks, OpenSpec, search, automations, and
  plugin-owned project data.
- Keep deep links, browser history, refreshes, and multiple tabs deterministic.
- Preserve the distinction between a project and the concrete workspace in
  which a thread executes.
- Give end users and agents equivalent UI, SDK, and CLI capabilities.

## Non-goals

- Project scope does not replace environments or worktrees.
- All projects is not a special database project and does not own entities.
- Global settings, machines, plugin management, and account-level data do not
  become project-owned.
- The proposal does not require every plugin object to be project-scoped.
  Plugins may intentionally expose global data, but must declare that behavior.

## Domain model

### Project

A Project is the durable identity of a repository, service, or independently
operated codebase. For a microservice architecture, a typical setup is one bb
project per microservice.

A project owns or scopes:

- one or more sources describing where its repository lives on enrolled hosts;
- environments and worktrees created from those sources;
- threads;
- tasks and task configuration;
- OpenSpec documents;
- project instructions and skills;
- project automations;
- plugin-owned project data.

### Source and environment

A project's normal directory is represented by its default source on a host.
An environment is a concrete working copy in which work executes. It may point
at the normal project directory, an existing worktree, or a managed worktree.

The intended relationships are:

```text
Project 1 ----- N Sources
Project 1 ----- N Environments
Project 1 ----- N Threads
Thread  N ----- 1 Environment
```

The project is the global navigation context. The environment is selected for
a particular thread or operation. It is not a second global switch because
several threads in the same project may run concurrently in different
worktrees.

For ordinary work, selecting a project is sufficient. New-thread defaults use
the project's default source and execution settings. The user chooses another
environment only when they want a worktree, another host, or another existing
working copy.

## Application scope

Represent scope explicitly rather than overloading a nullable project ID:

```ts
type AppScope =
  | { kind: "all-projects" }
  | { kind: "project"; projectId: string };
```

`projectId: null` is not sufficient because it can ambiguously mean all
projects, no route context, an unresolved project, or a personal workspace.
System boundaries should parse an explicit scope and pass the typed value
internally.

Surfaces that are outside either scope, such as Settings, should use an
explicit route classification rather than pretending to be in All projects.

## Navigation and URLs

The app shell provides a project switcher with All projects as an explicit
option:

```text
All projects
------------
payments-service
billing-service
notifications-service
```

Scope is encoded in the URL:

```text
/all/threads
/all/tasks
/all/openspec

/projects/:projectId/threads
/projects/:projectId/threads/:threadId
/projects/:projectId/new
/projects/:projectId/tasks
/projects/:projectId/openspec
```

The exact route spelling may change, but project scope must not exist only in
React state or local storage. URLs must remain refreshable and allow two tabs
to show different projects.

Each project remembers its most recently visited surface and relevant local
view state. Switching from Tasks in one project to another project should
normally open Tasks in the destination project. All projects keeps its own
view state.

Entity detail routes preserve the scope from which they were opened. For
example, opening a task from an aggregated list may use
`/all/tasks/:taskKey`, show the owning project in the detail header, and return
to the same aggregated list and filters. An explicit **Open project** action
moves to the owning project's scoped route.

## Surface behavior

| Surface | Project scope | All projects scope |
| --- | --- | --- |
| Threads | Threads belonging to the selected project | Threads from every project, with project filter and identity |
| Tasks | Tasks belonging to the selected project | Aggregated tasks, with project column and filter |
| OpenSpec | Specifications belonging to the selected project | Aggregated specification index, with project identity and filter |
| Automations | Automations belonging to the selected project | Aggregated automation list |
| Search | Results from the selected project | Cross-project results |
| New Thread | Project is already resolved | Project selection is required |
| New Task | Project is already resolved | Project selection is required |

Aggregated lists must use targeted database queries rather than loading every
project's rows and filtering in the client. Filter state should be representable
in the URL when it materially affects navigation or sharing.

## Creation rules

Every newly created project-owned object has exactly one owning project.

- In project scope, creation inherits the selected project.
- In All projects, the user or agent must select a project before submission.
- A remembered project may seed the selection but must not be silently applied
  as an invisible default.
- Moving an object between projects is a separate, explicit operation with its
  own validation and side effects.

OpenSpec editing and agent execution additionally require a concrete source or
environment. Project scope resolves ownership; it does not by itself identify
the working copy to mutate.

## Plugin contract

Project-aware plugin surfaces need the current scope. A future public plugin
API should use an `experimental_`-prefixed member until the contract is audited,
as required for new plugin API.

Conceptually, a nav panel should receive:

```ts
interface PluginNavPanelProps {
  subPath: string;
  experimental_scope: AppScope;
}
```

The final contract may instead expose a hook if that produces safer route
updates, but it must provide the same explicit state and update when the shell
scope changes. Plugins should not infer project identity from thread state or
parse host URLs themselves.

Plugin backend operations should accept an explicit project ID or explicit
scope at their boundary:

```ts
listTasks({
  scope: { kind: "all-projects" },
  filters: { status: ["in_progress"] },
});
```

Mutations that create project-owned data accept a concrete project ID, never
All projects.

## Tasks model

Tasks should use the bb project as their ownership boundary instead of
maintaining an independently named Tasks Project linked to a bb project.
Task-specific project metadata becomes an extension keyed by the bb project
ID, for example:

```text
task_project_settings
  bb_project_id
  prefix
  color
  next_task_number
```

Tasks reference `bb_project_id` directly. Existing Tasks projects and their
`linked_bb_project_id` values require a migration and an explicit policy for
unlinked or multiply linked legacy records.

If multiple independent task groupings inside one bb project are needed later,
introduce a separately named entity such as **Task Space** or **Board**. It
should not create a second meaning of Project.

## OpenSpec model

OpenSpec belongs to a project because its documents live with that project's
repository. In project scope, the OpenSpec surface reads the selected project's
default available source unless a thread-specific environment is in view.

All projects cannot be implemented as one filesystem root. It requires an
aggregated index across the available sources of individual projects. Every
result carries its project identity. Creating or editing a specification first
resolves a concrete project and then a concrete source or environment.

The aggregate view must handle disconnected hosts and unavailable sources
without treating the entire result as failed.

## CLI and SDK

Every scoped UI capability needs corresponding SDK and CLI behavior. Commands
that read collections accept an explicit scope, for example a concrete
`--project <id>` or an `--all-projects` flag. Commands that create or mutate a
project-owned object require `--project <id>` unless a thread-bound context
already provides an unambiguous project.

Environment variables such as `BB_PROJECT_ID` remain useful inside a running
thread, where ownership is already concrete. They do not represent the All
projects application scope.

## Global surfaces

The following remain outside project scope unless a feature specifically
introduces a project-filtered view:

- Settings;
- Machines and hosts;
- plugin installation and management;
- account and server configuration;
- global inbox or search entry points.

Global surfaces may link into either `/all/...` or a concrete project route,
but they do not silently change project ownership.

## Migration outline

1. Introduce an explicit application scope type and scoped routes.
2. Add the project switcher and filter the built-in thread surface by scope.
3. Preserve All projects as the explicit version of the current aggregate
   thread view.
4. Expose scope to plugin app surfaces through an experimental plugin API and
   document it in `docs/api_to_audit.md` when implemented.
5. Migrate Tasks from its linked project model to bb project ownership while
   preserving existing task keys and data.
6. Add project and aggregate OpenSpec surfaces with unavailable-source states.
7. Add equivalent SDK and CLI selectors for every shipped surface.
8. Remove stale nullable or independently inferred project context after all
   callers use the explicit scope.

## Open questions

- Should the project switcher preserve the current surface when the destination
  project has never visited it, or open that project's last surface?
- Should Personal be an ordinary visible project, a separate workspace kind,
  or absent from project-scoped product surfaces?
- Which filters belong in scoped URLs, and which are local presentation
  preferences?
- How should an All projects OpenSpec index be refreshed across disconnected
  hosts and large repositories?
- What is the migration behavior for legacy Tasks projects without exactly one
  valid linked bb project?
