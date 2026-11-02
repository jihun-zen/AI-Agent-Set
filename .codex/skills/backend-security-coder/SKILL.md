---
name: backend-security-coder
description: Implement or review backend input validation, authentication, authorization, secrets handling, and API defenses. Use for security-sensitive backend changes or focused security code reviews.
---

# Backend Security

1. Identify trust boundaries, caller identity, sensitive data, and externally controlled inputs.
2. Validate type, length, range, and allowed values at the boundary. Use parameterized queries and never construct commands from request values.
3. Require authentication and authorize each sensitive action with least privilege; do not rely only on hidden UI controls.
4. Keep secrets out of source, logs, URLs, and error responses. Return project-standard safe error responses.
5. Use framework-provided cryptography and token validation. Do not implement custom password hashing, encryption, or signature validation.
6. Review changes for injection, broken access control, sensitive-data exposure, unsafe redirects, and insecure defaults.
