# Security policy

## Reporting a vulnerability

**Do not open a public issue for a security problem.**

Report it privately through GitHub's private vulnerability reporting on the
affected repository (Security → Report a vulnerability), or to the contact
listed in the platform repository.

Please include what you found, how to reproduce it, and what an attacker could
do with it. You will get an acknowledgement, and an assessment once the issue
has been reproduced.

## Scope

ChurchBridge handles live audio from church services and distributes captions to
displays and personal devices. The findings that matter most:

- **Session and stream access.** Anything that lets an uninvited party join,
  read, or inject into a live session stream.
- **Audio or transcript exposure.** Anything that exposes captured audio,
  transcripts, or stored session data beyond its intended audience.
- **Credential handling.** Speech, translation, and LLM provider credentials
  reaching a client, a log, or a repository.
- **Deployment surface.** Anything in the deployment tooling that widens access
  to a host running ChurchBridge.

Findings in third-party dependencies should go to that project upstream, unless
ChurchBridge's use of it is what creates the exposure.

## Not in scope

Reports generated solely by automated scanners with no demonstrated impact, and
findings that require an attacker to already control the host or the soundboard.

## Please do not

Test against a live church service or any production instance you do not own.
Run ChurchBridge locally instead — a congregation in the middle of a service is
not a test environment.
