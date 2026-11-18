---
name: react-data-hook
description: Use when fetching server data in a React + TypeScript component — writing or reviewing a data-fetching custom hook with typed responses, loading/error state, request cancellation, and no state updates after unmount, or replacing an ad-hoc useEffect + fetch. Triggers on "fetch data in React", "useEffect fetch", "loading state", "custom hook", "API call in a component", "useFetch", "race condition in fetch", "cancel request on unmount".
metadata:
  version: 1.0.0
---

# React Data Hook

A playbook for fetching server data in React the right way: **typed, cancellable, and with loading + error states handled every time.**

## Principles

- A component should **render state, not orchestrate fetching**. Put the fetch in a hook so the component stays declarative.
- Always model **three states** — `loading`, `error`, `data`. Never assume the request succeeded.
- **Cancel in-flight requests** when the component unmounts or the inputs change. A late response that updates unmounted or stale UI is a real bug (flicker, wrong data, React warnings), not a theoretical one.
- **Type the response.** A hook that returns `any` just moves the bug downstream.

## First: prefer a real data library when one is present

If the project already uses **TanStack Query (React Query)** or **SWR**, use it — do not hand-roll fetching beside it. They give you caching, dedup, retries, and cancellation for free:

```tsx
const { data, isLoading, error } = useQuery({
  queryKey: ["article", id],
  queryFn: ({ signal }) => fetchArticle(id, signal),
});
```

Only build the hook below when the project has **no** data layer and you don't want to add one.

## The baseline typed hook (zero dependencies)

```ts
// useFetch.ts
import { useEffect, useState } from "react";

type State<T> =
  | { status: "loading"; data: null; error: null }
  | { status: "success"; data: T; error: null }
  | { status: "error"; data: null; error: Error };

export function useFetch<T>(url: string, init?: RequestInit): State<T> {
  const [state, setState] = useState<State<T>>({
    status: "loading",
    data: null,
    error: null,
  });

  useEffect(() => {
    const controller = new AbortController();
    setState({ status: "loading", data: null, error: null });

    fetch(url, { ...init, signal: controller.signal })
      .then(async (res) => {
        if (!res.ok) throw new Error(`Request failed: ${res.status}`);
        return (await res.json()) as T;
      })
      .then((data) => setState({ status: "success", data, error: null }))
      .catch((err: unknown) => {
        if (err instanceof DOMException && err.name === "AbortError") return; // expected
        setState({
          status: "error",
          data: null,
          error: err instanceof Error ? err : new Error(String(err)),
        });
      });

    return () => controller.abort(); // cancel on unmount / url change
  }, [url]); // re-run only when the request identity changes

  return state;
}
```

Why it's correct:

- The **discriminated union** makes illegal states unrepresentable — you can't read `data` while `loading`.
- `AbortController` + the cleanup return **cancels** the previous request before starting a new one, killing the classic race where responses resolve out of order.
- The `AbortError` branch stays silent, so a normal cancel never flashes an error.

## Using it — render all three states

```tsx
function Article({ id }: { id: string }) {
  const result = useFetch<ArticleDto>(`/api/v1/articles/${id}`);

  if (result.status === "loading") return <Spinner aria-label="Loading article" />;
  if (result.status === "error") return <ErrorBanner message={result.error.message} />;

  return <ArticleView article={result.data} />;
}
```

Rendering a loading and an error branch is not optional polish — it's the difference between a blank screen and a usable one.

## Common mistakes this skill prevents

- `useEffect(() => { fetch(url).then(r => r.json()).then(setData) }, [])` — no loading state, no error handling, no cancellation, and `[]` ignores a changing `url`.
- Putting non-stable values (an object/array literal, an inline `init`) in the dependency array, causing an infinite refetch loop. Memoize them or depend on primitives.
- `setData` firing after unmount. The cleanup `abort()` makes the late `.then` short-circuit on `AbortError`.
- Swallowing non-2xx responses — `fetch` only rejects on network failure, so you must check `res.ok` yourself.

## Checklist before you call it done

- [ ] Fetching lives in a hook; the component only reads state.
- [ ] Loading, error, and data are all represented and all rendered.
- [ ] The response is typed (no `any`); non-2xx is turned into an error.
- [ ] An `AbortController` cancels on unmount and when inputs change.
- [ ] The dependency array holds only stable values that define the request.
- [ ] If the project has React Query / SWR, that is used instead of this hook.
