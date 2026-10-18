# Spec Delta

## Purpose

Defines how agents act on GitHub: under a dedicated identity with limited permissions and short-lived tokens, so that their changes reach a default branch only through a pull request the owner approves.

## ADDED Requirements

### Requirement: Agents act on GitHub as a dedicated App
Agents SHALL act on GitHub only as a dedicated GitHub App, never with the owner's account or tokens. The App SHALL be installed only on the repositories agents work on, with only the permissions needed to push branches and open pull requests and issues.

#### Scenario: Agent opens a pull request
- **WHEN** an agent pushes a branch and opens a pull request
- **THEN** GitHub shows the App's bot account as the one who pushed and opened it

#### Scenario: Repository outside the installation
- **WHEN** an agent tries to read or push to a private repository the App is not installed on
- **THEN** GitHub denies access

#### Scenario: Agent tries to change repository settings
- **WHEN** an agent tries to change settings, rulesets, secrets or environments
- **THEN** GitHub denies it, because the App has no administration permissions

### Requirement: Tokens are short-lived and minted outside the agent's machine
Agents SHALL use installation tokens that expire within one hour. The App's private key SHALL NOT be stored on any machine where an agent runs. Tokens are minted on the owner's machine, or later by a dedicated service, and delivered to the agent's machine.

#### Scenario: Token leaks
- **WHEN** a token from the agent's machine is copied elsewhere
- **THEN** it stops working within one hour of being minted

#### Scenario: Owner's laptop is asleep in pair mode
- **WHEN** the owner's laptop stops delivering tokens
- **THEN** the agent loses GitHub write access within one hour, and regains it when the owner reconnects

#### Scenario: Searching the agent's machine for the private key
- **WHEN** the agent's machine is searched for the App's private key
- **THEN** it is not found

### Requirement: Default branch changes need the owner's approval
On each repository the App is installed on, the default branch SHALL accept changes only through a pull request with at least one approval, and SHALL reject force pushes and deletion. The App SHALL NOT be able to bypass these rules.

#### Scenario: Agent pushes directly to the default branch
- **WHEN** an agent pushes a commit to the default branch
- **THEN** GitHub rejects the push

#### Scenario: Agent merges its own pull request
- **WHEN** an agent tries to merge its pull request before anyone approved it
- **THEN** GitHub blocks the merge

#### Scenario: Owner reviews an agent's pull request
- **WHEN** the owner approves a pull request opened by the App
- **THEN** the pull request can be merged, because its author is the App and not the owner

### Requirement: Agent access can be revoked at once
The owner SHALL be able to cut all agent access to GitHub in one action, without rotating any of the owner's own credentials.

#### Scenario: Suspected misuse
- **WHEN** the owner suspends the App's installation
- **THEN** every token minted for it stops working, and the owner's own access is unaffected
