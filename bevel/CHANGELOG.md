# Changelog

## 0.1.0

The first release.

- Signals: `signal`, `computed`, `effect`, `untracked`, and `batch`, on a
  reactive core of Bevel's own. Bevel depends on no other package.
- Owners: `root` and `on_destroy`. A component, an effect run, and a
  `provide` each clean up what they made.
- Templates: `create`, the `.alx` factory, with `each`, `when`, and `model`
  for Angular's `@for`, `@if`, and `[(ngModel)]`.
- Components: `@component`, `mount`, and `bootstrap`, with the optional
  hooks `on_init` and `on_destroy`.
- Services: `@injectable`, `inject`, and `provide`, with scoped services and
  injectors that walk up the tree.
- A `how_to` tour in numbered steps, and three example games.
