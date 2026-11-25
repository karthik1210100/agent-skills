---
name: react-form
description: Use when building or reviewing a React + TypeScript form that writes to an API — a create or edit form, controlled inputs, client-side validation, submit/loading/disabled states, mapping server validation errors back onto fields, or its test. Use this whenever the user mentions a form, "submit", a "create/edit page", "validation", "controlled input", react-hook-form, zod, "POST from React", or handling a 422 / field errors from an API — even if they don't say the word "form".
metadata:
  version: 1.0.0
---

# React Form

A playbook for a React + TypeScript form that **submits cleanly, can't double-post, and shows both client and server errors next to the field that caused them.** This is the write-side twin of `react-data-hook`: there the component reads state a hook owns; here the component renders fields and a submit handler owns the async mutation. Work through the steps in order; skip one only when it genuinely doesn't apply, and say why.

## Principles

- **Model the submit lifecycle explicitly** — `idle → submitting → success | error`. Most form bugs are a missing state: no disabled button, no error surface, no success signal.
- **Never let the user submit twice.** Disable the button (and guard the handler) while a request is in flight. A double-click that creates two records is a data bug, not a UI nitpick.
- **The server is the source of truth.** Client validation is for fast feedback; it is not a substitute. Always handle the server rejecting input you thought was fine, and put each error back on its field.
- **Accessibility is correctness, not polish.** An input with no label, or an error a screen reader never announces, is broken for real users.

## First: prefer react-hook-form + zod when present

If the project already uses **react-hook-form** (often with **zod**), use it — don't hand-roll state and validation beside it. You get controlled/uncontrolled handling, validation, and submit state for free:

```tsx
const schema = z.object({ title: z.string().min(1), body: z.string().min(1) });
const { register, handleSubmit, setError, formState: { errors, isSubmitting } } =
  useForm<z.infer<typeof schema>>({ resolver: zodResolver(schema) });
```

Build the baseline below only when the project has **no** form library and you don't want to add one. Either way, the server-error and accessibility steps still apply.

## The baseline controlled form (zero dependencies)

```tsx
type Values = { title: string; body: string };
type Errors = Partial<Record<keyof Values | "form", string>>;

function ArticleForm({ onCreated }: { onCreated: (id: string) => void }) {
  const [values, setValues] = useState<Values>({ title: "", body: "" });
  const [errors, setErrors] = useState<Errors>({});
  const [submitting, setSubmitting] = useState(false);

  function validate(v: Values): Errors {
    const e: Errors = {};
    if (!v.title.trim()) e.title = "Title is required";
    if (!v.body.trim()) e.body = "Body is required";
    return e;
  }

  async function handleSubmit(event: React.FormEvent) {
    event.preventDefault();
    if (submitting) return;                        // guard against double-submit
    const clientErrors = validate(values);
    setErrors(clientErrors);
    if (Object.keys(clientErrors).length) return;

    setSubmitting(true);
    try {
      const res = await fetch("/api/v1/articles", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ article: values }),
      });

      if (res.status === 422) {
        setErrors(mapServerErrors(await res.json()));  // field errors from the API
        return;
      }
      if (!res.ok) throw new Error(`Request failed: ${res.status}`);

      const { data } = await res.json();
      onCreated(data.id);                          // success: hand off / redirect / reset
    } catch (err) {
      setErrors({ form: err instanceof Error ? err.message : "Something went wrong" });
    } finally {
      setSubmitting(false);                        // always re-enable, success or fail
    }
  }
  // ...render (below)
}
```

## Map server validation errors back onto fields

A good API answers invalid input with a predictable shape — the companion `rails-api-endpoint` skill returns `{ errors: [{ field, detail }] }`. Turn that into per-field messages so the error lands on the input, not in a lonely banner:

```ts
function mapServerErrors(body: { errors: { field?: string; detail: string }[] }): Errors {
  const mapped: Errors = {};
  for (const e of body.errors) {
    if (e.field) mapped[e.field as keyof Values] = e.detail;
    else mapped.form = e.detail;                  // errors with no field go form-level
  }
  return mapped;
}
```

This is why a consistent error shape across the stack matters: the form consumes exactly what the endpoint produces.

## Render fields, states, and accessible errors

```tsx
<form onSubmit={handleSubmit} noValidate>
  {errors.form && <p role="alert">{errors.form}</p>}

  <label htmlFor="title">Title</label>
  <input
    id="title"
    value={values.title}
    onChange={(e) => setValues({ ...values, title: e.target.value })}
    aria-invalid={!!errors.title}
    aria-describedby={errors.title ? "title-error" : undefined}
  />
  {errors.title && <span id="title-error" role="alert">{errors.title}</span>}

  <button type="submit" disabled={submitting}>
    {submitting ? "Saving…" : "Save"}
  </button>
</form>
```

- Every input has a real `<label htmlFor>` — placeholder text is not a label.
- `aria-invalid` + `aria-describedby` tie the message to the input, so assistive tech reads them together; `role="alert"` announces an error the moment it appears.
- After a failed submit, move focus to the first invalid field (or the form-level `role="alert"`) so a keyboard user isn't stranded at the bottom of the form.

## Common mistakes this skill prevents

- **Double submit** — no `disabled`/guard, so a fast double-click POSTs twice and creates duplicates.
- **Client-only validation** — the server rejects with 422 and the user sees nothing, because the response was never handled.
- **Lonely error banner** — "Something went wrong" with no indication of *which* field, instead of mapping errors back to inputs.
- **No success or reset** — the form sits there after a successful save with stale values and no confirmation, so the user submits again.
- **Mixed controlled/uncontrolled** — an input with a `value` but no `onChange` (or `undefined` value) warns and silently drops keystrokes.

## Test it

Drive it like a user with Testing Library: fill, submit, assert the request fired and that a server error shows on the right field.

```tsx
it("shows the server's field error on the input", async () => {
  server.use(rest.post("/api/v1/articles", (_req, res, ctx) =>
    res(ctx.status(422), ctx.json({ errors: [{ field: "title", detail: "has already been taken" }] }))));

  render(<ArticleForm onCreated={vi.fn()} />);
  await userEvent.type(screen.getByLabelText("Title"), "Dupe");
  await userEvent.type(screen.getByLabelText("Body"), "...");
  await userEvent.click(screen.getByRole("button", { name: /save/i }));

  expect(await screen.findByText("has already been taken")).toBeInTheDocument();
});
```

Run it red first if you're adding behaviour, then green.

## Checklist before you call it done

- [ ] Submit lifecycle is explicit: the button disables while submitting and re-enables in `finally`.
- [ ] The handler guards against a second in-flight submit.
- [ ] Client validation runs, but a server `422` is handled and mapped back onto fields.
- [ ] Field errors use `aria-invalid` + `aria-describedby`; a form-level error uses `role="alert"`.
- [ ] Every input has a real `<label>`; focus moves to the first error after a failed submit.
- [ ] Success does something visible — redirect, reset, or a confirmation.
- [ ] A test fills the form, submits, and asserts a server field error renders on the right input.
- [ ] If the project has react-hook-form / zod, that is used instead of hand-rolled state.
