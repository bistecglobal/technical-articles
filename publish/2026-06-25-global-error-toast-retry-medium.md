# Every Mutation Gets Retry for Free: A One-Provider Error Pattern for Next.js

*The worst bug is the one your user notices before you do — like a Save button that silently does nothing. Here's how one provider and one hook killed the whole class of it.*

The worst kind of bug is the one your user notices before you do. A "Save" button that silently does nothing is exactly that bug — and for a while, parts of Keyflow had it.

Keyflow is our OKR and performance-management platform. Like any app with a lot of forms, it's a lot of mutations: create an objective, edit a key-result score, pick a template. And like a lot of apps that grew feature-by-feature, its error handling had grown the same way — unevenly. Some mutations showed an inline error. Some swallowed the failure into a `.catch(console.error)` that only existed in the dev console. From the user's chair, those last ones looked identical to success: click, nothing, shrug. The network blipped, the write never landed, and nobody was told.

We fixed the whole class of problem with one provider and one hook — and the payoff was that every mutation in the app got a retryable error toast without any of them being individually rewritten.

## The shape of the problem

The mistake wasn't any single missing `try/catch`. It was that error handling lived at the call site, so its quality varied with whoever wrote that call site. Two failure modes recurred:

1. **Silent catches.** `somePromise().catch(console.error)` — technically handled, invisibly. The Template picker and the new-objective page both had one.
2. **Inconsistent surfacing.** Where errors *were* shown, each form did it its own way, and none offered the one thing a user actually wants after a transient failure: a way to try again without re-entering everything.

Fixing these one form at a time is a treadmill. New forms reintroduce the problem. We wanted the *default* path — the one a developer gets by doing nothing special — to be the correct one.

## One toast host, mounted once

The first piece is a context provider mounted a single time, at the dashboard layout root, so it wraps every page beneath it. It owns a stack of toasts and exposes exactly one verb — `push` — to the rest of the app:

```tsx
export const ErrorToastContext = createContext<ErrorToastContextValue>({ push: () => {} });

export function ErrorToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  const idRef = useRef(0);

  const dismiss = useCallback((id: number) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  const push = useCallback((message: string, retry?: () => void) => {
    const id = ++idRef.current;
    setToasts((prev) => [...prev, { id, message, retry: retry ?? null }]);
    setTimeout(() => dismiss(id), 8000);
  }, [dismiss]);
  // ...renders the stack bottom-right
}
```

A few deliberate choices in this small surface:

- **`push` takes an optional `retry` callback.** A toast with one renders a Retry button; a toast without one is purely informational. The toast doesn't know or care what retrying *means* — it just calls the closure.
- **Toasts stack and auto-dismiss after 8 seconds.** Three failures in a row give you three toasts, not one that clobbers the last. Each clears itself on a timer or on explicit dismiss.
- **A monotonic `idRef` counter** keys each toast, so React reconciles the list cleanly and the dismiss-by-id filter is unambiguous.

The render layer uses a small pointer-events trick worth noting: the fixed container is `pointer-events-none` so it never blocks clicks on the page behind it, while each toast card is `pointer-events-auto` so its own buttons stay clickable. The error UI floats over the app without stealing interaction from it.

## The hook that makes retry automatic

The provider is inert on its own — something has to call `push`. That something is `useRetryableAction`, the hook the app already used to run mutations. The change that made the whole feature click into place was wiring the hook to the toast context so it pushes errors itself:

```ts
const run = useCallback(async (action: () => Promise<T>): Promise<T | null> => {
  lastAction.current = action;
  setState({ loading: true, error: null, fieldHint: null });
  try {
    const result = await action();
    setState({ loading: false, error: null, fieldHint: null });
    return result;
  } catch (err) {
    const msg = err instanceof Error ? err.message : "Something went wrong — please try again.";
    const fieldHint = extractFieldHint(msg);
    setState({ loading: false, error: msg, fieldHint });
    pushToast(msg, lastAction.current ? () => { run(lastAction.current!); } : undefined);
    return null;
  }
}, [pushToast]);
```

The key is `lastAction.current`. Every call stashes the action thunk in a ref before running it. When it throws, the hook builds a retry closure that re-invokes `run` with *that exact same action* — same inputs, same endpoint — and hands it to the toast. The user clicks Retry, the original mutation re-fires, and on success the toast dismisses itself.

Because every form in the app already routed its mutations through `useRetryableAction`, this one wiring change gave all of them a retryable toast at once. No form was touched. That's the leverage: the correct behaviour was installed at the chokepoint every mutation already passed through, so the default path became the right path.

There's a small bonus in `extractFieldHint`: it scans the error message for keywords like "title", "cycle", "score", "date", or "owner" and returns a hint a form can use to highlight the offending field. The error message does double duty — a human-readable toast *and* a machine-readable pointer to what to fix.

## Closing the silent catches

The pattern only helps where errors reach it, so the same change hunted down the two `.catch(console.error)` callsites — in the Template picker and the new-objective page — and routed them through the toast instead. Those were the exact spots where a failure had been invisible; now a dropped template load or a failed objective fetch surfaces like everything else.

## What it changed

The change touched no database and no API — it's purely a client-side resilience layer. But it converted a whole category of silent failures into visible, recoverable ones, and it did so in a way that's hard to regress: a new form that uses the existing mutation hook inherits the toast and the retry button for free, with no extra code.

Three takeaways that travel to any front-end:

- **Install correctness at the chokepoint, not the call site.** Error handling scattered across call sites degrades to the level of the least careful one. Moved into the one hook every mutation flows through, it became uniform and free.
- **Make retry re-fire the captured action, not a reload.** Stashing the action thunk in a ref means Retry repeats *exactly* what failed — no page refresh, no re-entered form, no lost context.
- **A silent catch is a bug, not error handling.** `console.error` satisfies the linter and nobody else. If a write can fail, the person who made it happen has to be told.

The button that silently does nothing is no longer possible by default in Keyflow — and the fix was one provider, one hook, and deleting two `console.error`s.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
