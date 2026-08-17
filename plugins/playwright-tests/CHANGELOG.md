# Changelog

## 0.1.0
- Initial release
- Condition-over-duration core rule, with the actionability matrix that explains why most waits are redundant
- Decision table mapping "what you're waiting for" to the assertion or API that waits for it
- Timeout budget: which of the seven timeouts bounds what, and how to read the error each one produces
- Reference catalogue of waiting recipes — element state, network, events, `page.clock`, eventual consistency, overlays, frames
- Fifteen anti-patterns with before/after rewrites and a grep-based audit
- Annotated `playwright.config.ts` baseline plus a flake triage playbook
