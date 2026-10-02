# Spec Delta

## ADDED Requirements

### Requirement: Reads are not changes

The desktop file watcher SHALL ignore events that only record a read: an open, a read, or a close without a write. These events SHALL NOT produce an `index:changed` entry and SHALL NOT trigger `index:reconcile`. A close after a write SHALL still count as a change.

#### Scenario: The reconcile walk does not trigger itself
- **WHEN** the watcher is running on Linux and a reconcile pass walks and reads every folder and note in the graph
- **THEN** the watcher emits no `index:reconcile` and no `index:changed` because of that walk

#### Scenario: Another program reads a note
- **WHEN** another program opens and reads a note without changing it
- **THEN** the watcher emits nothing

#### Scenario: Another program writes a note
- **WHEN** another program writes a note and closes it
- **THEN** the watcher emits an `upsert` for that note, as before

### Requirement: Watching costs no reads of hidden folders

The desktop file watcher's file-ID cache SHALL NOT open a hidden or pruned path (any component below the graph root that starts with `.`, or is on the shared prune list) when an event for that path arrives.

#### Scenario: A hidden folder is opened once
- **WHEN** the watcher is running on Linux and something opens `.git/objects` once
- **THEN** the watcher causes no further opens of that folder

#### Scenario: Local history commits while the watcher runs
- **WHEN** a local-history commit writes into `.git/`
- **THEN** the watcher thread goes idle once the commit ends
