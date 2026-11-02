---
name: frontend-react
description: Build, refactor, or review React interfaces with reusable components, responsive Tailwind styling, accessible interaction states, and local-state-first design. Use for React or Tailwind UI work.
---

# React and Tailwind UI

## Scope

- For questions, explanations, reviews, and planning requests, inspect and explain only. Use the implementation guidance below as review criteria. Run checks that generate artifacts only when explicitly requested; a request to run checks does not authorize source edits.
- Implement only when requested, and limit changes to the requested result and necessary related code. Leave unrelated refactoring, component extraction, dependency updates, and visual improvements as suggestions.
- Never execute remote deployments, HTTP or AD requests, or database writes. This includes read-only HTTP requests, localhost HTTP requests, and local or test database writes. Related code may be authored, but execution belongs to the user; requests for verification do not create an exception.

## Implementation guidance

1. Inspect `package.json`, the installed Tailwind version, and existing component conventions before adding dependencies or choosing syntax.
2. Put reusable components in the project's component location (normally `src/components/`); keep page-specific composition close to its route.
3. Prefer Tailwind utilities and CSS Grid for responsive layout. Use `min-h-[100dvh]` instead of `h-screen` for viewport-filling mobile layouts.
4. Start with local state and custom hooks. Introduce global state only when state is genuinely shared across independent branches.
5. Implement loading, empty, error, focus, and disabled states where relevant to the requested change. Do not expand a small edit into a page-wide state audit or redesign. Use semantic controls and labels before adding ARIA attributes.
6. Preserve the existing visual system; do not impose a new icon library, animation library, or visual style without a project need.

After requested implementation changes, run relevant local lint and type-check commands without automatic fixes only after confirming their scripts and hooks do not perform prohibited operations. Do not run package downloads, browser checks that issue HTTP requests, deployment hooks, or integration tests that issue HTTP/AD requests or write to a database. Use offline checks and mocks where possible; if side effects cannot be ruled out, leave execution to the user and report the skipped verification. Prefer check-only formatting; limit any required formatting to the edited code. Report unrelated existing failures without expanding the task to fix them.

## Code style

- Keep function declarations and calls on one line when readable. Do not wrap parameters or arguments solely because there are several of them.
- Do not split strings or log messages into concatenated fragments merely to wrap a line. Avoid unnecessary blank lines.
- Wrap long JSX, nested expressions, and callbacks when it improves readability. Do not compress complex logic or introduce cryptic names to save lines.
- Preserve formatting outside the edited code. If project formatter settings conflict with this preference, report the conflict without changing the settings on your own.

```tsx
const result = calculateTotal(price, quantity, discount);
const handleSelect = (id: string) => onSelect(id);
```
