# Security Policy

## Current state

AscendOS is a **design blueprint with no executable code**. There is no software
to exploit yet, so there are no software vulnerabilities to report at this time.

What *is* worth reporting is a **security flaw in the design itself** — for
example, a way the capability model grants more authority than intended, a boot
chain that doesn't actually establish the trust it claims, or an isolation
boundary that doesn't hold. Design-level security feedback is exactly the kind
of review this project wants.

## Reporting

- **Design security concerns** that are safe to discuss openly: open a normal
  issue using the *Design question* template, or start a Discussion.
- **Anything you'd rather report privately:** use GitHub's private vulnerability
  reporting on this repository, or contact the maintainer
  [@AwaleSagar](https://github.com/AwaleSagar) directly.

## When there is code

Once an implementation exists, this policy will be updated with supported
versions, a coordinated disclosure timeline, and a hardening contact. Until
then, please treat all "vulnerabilities" as design review.
