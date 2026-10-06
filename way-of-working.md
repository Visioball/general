# Way of working

## General

- In person meetings on Thuesday, notify group if not able to attend, half hour
  in advance.

## Deliverables

- Source code (application logic, tests, configuration etc) will be located in
  relevant git repository.
- Code specific documentation will also be located in the relevant git
  repository.

- Role specific documentation, intended for personal portfolios, will not be in
  shared version control system.

## Guidelines

- GitHub will be the platform that is going to be used for the source code,
  tickets and Kanban board

- Git branches:
  - Branch convention is the following: `<ticket-no>-<ticket-name>`
  - Merging strategy: Only commits on feature/bugfix branches are allowed. Main
    branch is only accessible via PR merges.
  - Every PR requires 1 approver before merge.
  - If you're added on review, you need to reply in 3 days.
  - These rules will be enforced by GitHub platform.
- Every sprint, every application should release a new version. Use semantic
  version.
- Ticket conventions:
  - Tickets will include both documentation tickets + implementation tickets.
  - Tickets will be added to the Kanban board.
  - Every ticket that is being worked on should be assigned.
  - Bug reports should be created as tickets.
- Testing, CI/CD pipelines:
  - Every code base should have some components unit tested.
  - The execution of these tests should be automated via the pipelines.
  - End of the sprint should mean a working application. We will write a test
    plan, which will be executed and documented.
