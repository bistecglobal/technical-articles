The worst kind of bug is the one your user notices before you do — like a "Save" button that silently does nothing.

Keyflow (our OKR platform) had patches of that bug. Error handling had grown form-by-form, so its quality varied with whoever wrote each call site. Some mutations swallowed failures into `.catch(console.error)` — invisible to the user. Click, nothing, shrug. The write never landed and nobody was told.

We killed the whole class of problem with one provider and one hook:

🔹 **One toast host, mounted once** at the layout root. It exposes a single `push(message, retry?)` verb. Toasts stack, auto-dismiss after 8s, and render a Retry button only when a retry closure is supplied.

🔹 **The hook makes retry automatic.** `useRetryableAction` stashes each mutation in a ref before running it. On failure it pushes a toast whose Retry closure re-fires *that exact action* — same inputs, same endpoint. No reload, no re-entered form.

🔹 **Leverage at the chokepoint.** Every form already ran mutations through that hook — so one wiring change gave all of them a retryable error toast. Zero forms rewritten.

The principles travel to any frontend:
✦ Install correctness at the chokepoint, not the call site — scattered handling degrades to the least careful call site.
✦ Make retry re-fire the captured action, not a page reload.
✦ A silent `.catch(console.error)` is a bug, not error handling.

No DB or API changes — pure client-side resilience. The button that silently does nothing is no longer possible by default.

Full article → [link placeholder — operator fills in after Medium publish]

#React #NextJS #Frontend #WebDev #UX
