# Spec Delta

## Purpose

Keeps the repository vendor-agnostic by isolating provider-specific knowledge in public provider adapters that fill named slots of generic runbooks, while real deployment values never enter the repository.

## ADDED Requirements

### Requirement: Generic documents are provider-free
The architecture document and generic runbooks SHALL NOT name hosting providers, their products, or their regions. They refer to provider-specific steps only through slots and the provider index.

#### Scenario: Searching generic documents for provider names
- **WHEN** generic documents are searched for the names of providers that have an adapter
- **THEN** no matches are found outside `providers/`

### Requirement: Runbooks declare slots
Each generic runbook that depends on a provider SHALL declare its provider-specific steps as named slots. Each slot states the question it answers, and its name is stable so adapters can reference it.

#### Scenario: Reader reaches a provider-specific step
- **WHEN** a reader of a generic runbook reaches a step that differs by provider
- **THEN** the step names a slot, says what the slot must answer, and points to the provider index

### Requirement: Adapters fill slots
Provider-specific knowledge SHALL live in `providers/<name>/`, with one adapter file per generic runbook it supports. The adapter addresses each slot of that runbook by name.

#### Scenario: Adapter does not cover a slot yet
- **WHEN** an adapter has no content for one of the runbook's slots
- **THEN** that slot is listed as not yet covered, with the change or roadmap item expected to fill it

#### Scenario: Adding a new provider
- **WHEN** someone adds support for another provider
- **THEN** they create `providers/<name>/` and fill the same slots, without editing the generic runbook

### Requirement: Provider index
`providers/README.md` SHALL list every provider adapter, explain the slot convention, and state which provider the reference deployment uses for each role (LAB, PROD, backups).

#### Scenario: Reader wants the reference deployment's choice
- **WHEN** a reader wants to know which provider the reference LAB uses
- **THEN** the provider index answers it, and no generic document has to

### Requirement: No deployment values in the repository
The repository SHALL NOT contain real IP addresses, hostnames, account or customer IDs, or order or contract numbers of any deployment. Documents use placeholders such as `<lab-host>` instead.

#### Scenario: Adapter needs to refer to the ordered server
- **WHEN** an adapter or runbook step needs the address or name of the actual server
- **THEN** it uses a placeholder, and the real value lives in the password manager or the overlay network's naming

### Requirement: Volatile facts are dated
Prices, offer names, stock and other facts in adapters that change over time SHALL state the date they were checked and link to their source.

#### Scenario: Reader compares a recorded price with today's
- **WHEN** a reader sees a price in an adapter
- **THEN** the date it was checked and its source are shown next to it, so the reader knows to re-check it
