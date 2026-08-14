# lit-view

A view that works like a custom element, without being one.

lit-html templates are plain functions — enough until a view has to hold something with a lifetime: a service subscription, a timer, an owned DOM node. The usual fix is to promote the view to a custom element for its lifecycle callbacks, and with it take on a tag name, a registry entry, shadow DOM, and a component boundary. `lit-view` gives you just the lifecycle: connect and acquire, disconnect and release, re-render yourself — in a class you call like a function inside any template. It is a small base over lit-html's own [`AsyncDirective`](https://lit.dev/docs/templates/custom-directives/#async-directives), four lifecycle members and nothing else.

```bash
npm install lit-view lit-html
```

## Example

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

The view subscribes when it enters the page, re-renders itself whenever the service notifies, and unsubscribes when it leaves. The host template just calls a function.

## Lifecycle

`view(Class)` turns the subclass into that template-callable. lit creates **one instance per template position** and reuses it for every later render at that position — which is what makes instance state meaningful.

A subclass defines up to four members:

- **`connected()`** — the view is live: acquire. Subscribe, start timers. Runs on first render, and again each time the view is reattached (a keyed list move, a [`cache()`](https://lit.dev/docs/templates/directives/#cache) swap back in), so acquisition must be re-runnable.

- **`disconnected()`** — the view is paused or gone: release. Fires when the view's DOM leaves the document. It may be followed by `connected()` again.

- **`template(...args)`** — derive output from the latest host arguments; return any lit-renderable value. The default renders nothing, for views that paint entirely by hand.

- **`this.render(t?)`** — commit. Called bare, it commits `template()` with the latest host arguments; `render(t)` commits `t` directly. Safe to call from anywhere — subscriptions, timers, event handlers. While the view is detached it is a no-op, and during a host render the host's own result carries, so nothing double-commits.

Host arguments are typed on the class: `class Row extends View<[label: string, count: number]>` receives `RowView("a", 1)` in `template`.

`reconnected()` is used by the base itself — do not override it.

## Guarantees

- `connected()` runs exactly once per period of attachment; ordinary host re-renders never repeat it.
- After a reattachment, the committed output reflects current state with no subclass code. If the same render pass re-renders the view, that render carries it; otherwise the base commits once in a microtask. A restored view cannot show stale state.
- `render()` never throws for lifecycle reasons — detached and mid-render calls are no-ops, including calls triggered from inside `connected()`.

## Owning a DOM node

For identity-critical content — a live iframe, a hand-managed canvas — create the node once, mutate it, and return the same node every time:

```ts
class Frame extends View<[src: string]> {
  #iframe?: HTMLIFrameElement;
  template(src: string) {
    this.#iframe ??= document.createElement("iframe");
    if (this.#iframe.src !== src) this.#iframe.src = src;
    return this.#iframe;
  }
}
```

lit dirty-checks the committed value by identity, so returning the same node is a no-op: the iframe survives host re-renders with its document intact.

## What it is not

No scheduling or batching — commits are synchronous, the one exception being the reconnect catch-up, deferred one microtask. No reactive properties — arguments arrive from the host, and shared state belongs to whatever owns it. No shadow DOM, no styles, no element identity: if a view needs to be a real custom element, make one. This class is for views that don't.
