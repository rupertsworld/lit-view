# lit-view

Stateful views for lit-html without a component framework: a small base class over `AsyncDirective` with a plain lifecycle vocabulary — `connected`/`disconnected` to acquire and release, `template` to derive, `render` to commit.

Use it when a view holds something with a lifetime — a service subscription, a timer, an owned DOM node — and a plain template function stops being enough. Its one dependency is `lit-html` (peer).

```bash
npm install lit-view lit-html
```

## API

```ts
import { html } from "lit-html";
import { view, View } from "lit-view";

class SkillList extends View {
  #unsubscribe?: () => void;

  connected()    { this.#unsubscribe = service.subscribe(() => this.render()); }
  disconnected() { this.#unsubscribe?.(); this.#unsubscribe = undefined; }

  template() { return html`…derived from current state…`; }
}

export const SkillListView = view(SkillList);

// elsewhere, in any template, at any depth:
html`<section>${SkillListView()}</section>`;
```

- **`view(Class)`** — calls lit's `directive(Class)` and returns the resulting template-callable. Invoking it in a template produces a `DirectiveResult`; lit constructs **one instance per bound child position** and reuses that instance for every subsequent host render at the position. Convention: the class defined on its own, the export wrapping it once.
- **The host path** — the base overrides `Directive.update(part, args)`, lit's host-render entry: it stores the arguments, then — under the commit guard — runs `connected()` on first activation and returns `template(...args)` as the value lit commits. `connected()` running guarded is load-bearing: a subscription that notifies synchronously calls `render()`, which no-ops, and the returned value already carries that state. The directive's own `render()` slot is never used — which is what frees the `render` name for committing on the instance.
- **`connected()`** — the view is live: acquire. Invoked from `update()` on first activation, and again from lit's `reconnected()` callback after every reattachment, so acquisition must be re-runnable. Both invocations run under the commit guard: anything `connected()` triggers synchronously defers to the path that invoked it.
- **`disconnected()`** — the view is paused or gone: release. This *is* lit's `AsyncDirective.disconnected()` callback, implemented directly by the subclass. Lit fires it when the part's tree leaves the document — a removal, a cached swap-out, a keyed list move — and may follow it with `reconnected()`.
- **`template(...args)`** — derive output from the latest host arguments; any lit-renderable value. Defaults to `noChange` (lit: leave the committed value as it is), for views that paint entirely by hand.
- **`this.render(t?)`** — commit via `AsyncDirective.setValue()`: `t`, or `template()` called with the stored host arguments when bare. Guarded twice: it no-ops when `this.isConnected` is false (lit warns on detached `setValue`), and no-ops during a host render (calling `setValue` from inside `update()` is illegal in lit — there, the host path's return value carries).
- **`reconnected()`** — reserved by the base; subclasses do not override it. It maps lit's reattachment callback to `connected()` under the commit guard, then a conditional, deferred catch-up: if the same render pass re-runs `update()`, the host path carries current state and the catch-up is dropped; otherwise a microtask commits `render()` after the pass unwinds — where `setValue` is legal. Catch-up for notifications missed while detached, never committed mid-render.

## Guarantees

- `connected()` runs exactly once per period of attachment; host re-renders do not repeat it.
- After a reconnect, the committed output reflects current state with no subclass code — via the host path when the reattachment re-renders the view, via a microtask-deferred commit when it doesn't.
- `render()` never throws for lifecycle reasons: detached and mid-host-render calls are no-ops — including calls from inside `connected()`, which always runs under the guard.
- A stable node returned from `template()` keeps its identity across host renders: lit's `ChildPart` dirty-checks the committed value by identity and no-ops on the same node — the recipe for identity-critical interiors (live iframes, keyed syncs), where the view mutates its owned subtree and returns the same element every time.

## Non-goals

No scheduling or batching (commits are synchronous — the reconnection catch-up, deferred one microtask, is the sole exception), no reactive properties (arguments arrive from the host; shared state belongs to whatever owns it), no shadow DOM, no styles, no element identity. If a view needs to be a real custom element, make one — this class is for views that don't.
