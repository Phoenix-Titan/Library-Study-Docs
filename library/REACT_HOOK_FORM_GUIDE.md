# React Hook Form — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've built a form with `useState` and it felt awful" to "I architect type-safe, accessible, performant forms in production React and Next.js apps" — without an internet connection. Every concept comes with prose that explains *what it is*, *the logic / why it works this way*, *what it's for and when you reach for it*, *how to use it*, the *key options/props*, *best practices*, and *gotchas* — followed by runnable, heavily-commented code. Read top-to-bottom the first time; afterwards, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **React Hook Form v7** (the current major in 2026), **React 19**, **Zod v3 and v4**, and **Next.js 15** (App Router). Things worth knowing:
> - **React Hook Form v7** is the stable major line. v7 changed `register` to *return props you spread* (rather than passing a ref callback as in v6). All code here is v7.
> - **React 19** shipped the `use`, `useActionState`, `useFormStatus`, and `useOptimistic` hooks plus first-class `<form action={...}>` support. RHF coexists with these — this guide explains where each fits (§16, §18).
> - **Zod v4** (2025) is faster and tweaks a few APIs (e.g. error customization). `@hookform/resolvers/zod` supports both v3 and v4. Differences are flagged with **⚡ Version note**.
> - **shadcn/ui** ships a thin, accessible wrapper around RHF + Zod that has become the de-facto form pattern in 2026 (§15).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; commands use `npm` but `pnpm`/`yarn`/`bun` work identically. Confirm exact APIs at react-hook-form.com.

---

## Table of Contents

1. [Why React Hook Form (the re-render story)](#1-why-react-hook-form-the-re-render-story) **[B]**
2. [Installation & Mental Model](#2-installation--mental-model) **[B]**
3. [`useForm` — the entry point and every option](#3-useform--the-entry-point-and-every-option) **[B/I]**
4. [`register` — how ref-based registration works](#4-register--how-ref-based-registration-works) **[B/I]**
5. [`handleSubmit` — validating and submitting](#5-handlesubmit--validating-and-submitting) **[B]**
6. [`formState` — errors, isDirty, isValid, and friends](#6-formstate--errors-isdirty-isvalid-and-friends) **[B/I]**
7. [Validation modes — when validation runs](#7-validation-modes--when-validation-runs) **[I]**
8. [Built-in validation rules](#8-built-in-validation-rules) **[B/I]**
9. [Schema validation with Zod + `zodResolver`](#9-schema-validation-with-zod--zodresolver) **[I]**
10. [`Controller` & `useController` — controlled components](#10-controller--usecontroller--controlled-components) **[I]**
11. [`watch` vs `useWatch` — reacting to values](#11-watch-vs-usewatch--reacting-to-values) **[I]**
12. [`useFieldArray` — dynamic / repeating fields](#12-usefieldarray--dynamic--repeating-fields) **[I/A]**
13. [Imperative methods: setValue, getValues, reset, trigger, setError, setFocus](#13-imperative-methods-setvalue-getvalues-reset-trigger-seterror-setfocus) **[I]**
14. [Nested objects & `useFormContext` for large forms](#14-nested-objects--useformcontext-for-large-forms) **[I/A]**
15. [The shadcn/ui Form pattern](#15-the-shadcnui-form-pattern) **[I/A]**
16. [Next.js: client forms, Server Actions & React 19](#16-nextjs-client-forms-server-actions--react-19) **[A]**
17. [Async & server-side validation](#17-async--server-side-validation) **[A]**
18. [Inputs cookbook: files, checkboxes, radios, selects](#18-inputs-cookbook-files-checkboxes-radios-selects) **[I]**
19. [Performance best practices](#19-performance-best-practices) **[A]**
20. [Accessibility](#20-accessibility) **[I/A]**
21. [Tips, tricks & gotchas](#21-tips-tricks--gotchas) **[B/I/A]**
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. Why React Hook Form (the re-render story)

Before learning *how* to use React Hook Form (RHF), you need to understand *why* it exists, because the entire design follows from one performance insight. If you understand the re-render story, every API decision in this library will feel obvious instead of arbitrary.

### 1.1 The two ways React handles form inputs **[B]**

React offers two models for `<input>` elements, and the choice between them is the single most important fact about React forms.

**Controlled inputs** store the value in React state. React is the "single source of truth": on every keystroke an `onChange` handler fires, calls a state setter, the component re-renders, and React writes the new value back into the DOM. The value you see is always the value in state.

```tsx
// ❌ Controlled pattern — a full component re-render fires on EVERY keystroke.
import { useState } from "react"

function ControlledForm() {
  const [name, setName] = useState("")
  const [email, setEmail] = useState("")
  // Each keystroke -> setName/setEmail -> React re-renders the WHOLE component.
  // With 15 fields and some derived UI, this is thousands of renders per minute.
  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
    </form>
  )
}
```

**Uncontrolled inputs** let the *DOM itself* hold the value. You don't pass `value`; you read the value only when you need it (typically on submit) via a `ref`. React is not involved per-keystroke, so there is no re-render while typing.

```tsx
// ✅ Uncontrolled pattern — the browser holds the value; React stays out of the way.
import { useRef } from "react"

function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null)
  const emailRef = useRef<HTMLInputElement>(null)
  // No re-render while typing. We read the values only on submit.
  function onSubmit(e: React.FormEvent) {
    e.preventDefault()
    console.log(nameRef.current?.value, emailRef.current?.value)
  }
  return (
    <form onSubmit={onSubmit}>
      <input ref={nameRef} />
      <input ref={emailRef} />
    </form>
  )
}
```

### 1.2 The logic behind RHF

Plain uncontrolled inputs are fast but painful: you'd write your own `ref` for every field, your own validation, your own error display, your own "is this dirty?" tracking. **React Hook Form is uncontrolled-first, but gives you all the ergonomics of a controlled form library** — validation, errors, dirty/touched tracking, dynamic fields, TypeScript inference — without the per-keystroke re-render cost.

The mechanism: `register("fieldName")` returns a `ref` callback (plus `name`, `onChange`, `onBlur`). You spread it onto a native input. RHF stashes the DOM node in an internal map. When you submit, RHF reads every registered node's `.value` straight from the DOM. State lives *outside* React's render cycle, in a mutable store inside the hook, and RHF re-renders your component *surgically* — only when something you actually subscribed to (a specific error, `isDirty`, a watched field) changes.

This is why RHF feels almost too easy: you write less code than the controlled version **and** it's faster.

### 1.3 RHF vs the alternatives

| | React Hook Form v7 | Formik | Plain `useState` | React 19 `<form action>` |
|---|---|---|---|---|
| **Re-renders while typing** | None (uncontrolled) | One per keystroke | One per keystroke | None (native) |
| **Bundle size** | ~9 KB gzipped | ~13 KB gzipped | 0 | 0 |
| **TypeScript inference** | Excellent (generics + path types) | Decent | Manual | Limited |
| **Schema validation** | Resolvers (Zod, Yup, Valibot) | Built-in Yup | Manual | Manual / Zod |
| **Dynamic fields** | `useFieldArray` | `FieldArray` (verbose) | Manual | Manual |
| **3rd-party/controlled inputs** | `Controller` | `Field` | Wrap manually | n/a |
| **Per-field error UX, dirty/touched** | Built-in | Built-in | Manual | Manual |
| **Best for** | Any real interactive form | Legacy projects | Trivial 1–2 field forms | Progressive-enhancement, RSC |

> **Bottom line:** Reach for React Hook Form for any form with validation, more than a couple of fields, or any interactivity. Use **Zod** as the validation layer. Use `Controller` only when an input genuinely must be controlled (design-system components, date pickers, `react-select`). React 19's native `<form action>` is excellent for simple server-driven forms with progressive enhancement — but RHF still wins for rich client-side UX, and the two can be combined (§16).

---

## 2. Installation & Mental Model

### 2.1 Install **[B]**

```bash
# Core library — this alone gets you fully working forms with built-in validation.
npm install react-hook-form

# The resolver bridge — needed ONLY if you validate with a schema library (Zod/Yup/Valibot).
npm install @hookform/resolvers

# Pick a schema library. Zod is the recommended default in 2026 for its TypeScript inference.
npm install zod          # recommended
# npm install yup        # the Formik-era classic
# npm install valibot    # tiny, modular, tree-shakeable alternative
```

There is **no provider, no context, no global config** required to start. `useForm` is entirely self-contained: the form's state lives inside that one hook call. (You only add a provider — `FormProvider` — when you want to *share* one form across many components; see §14.)

⚡ **Version note:** Always confirm you are on **v7**. The v6→v7 migration changed `register`: in v6 you passed a ref callback (`ref={register}`); in v7 you spread the returned props (`{...register("name")}`). Old tutorials that show `ref={register}` are v6 and will not work.

### 2.2 The mental model you must hold

Hold these four facts in your head and RHF stops being magic:

1. **The DOM holds the values.** RHF reads them via refs. That's why there are no `value`/`onChange` props on your inputs by default.
2. **`useForm()` returns a toolbox.** Everything — `register`, `handleSubmit`, `watch`, `setValue`, `formState`, `control` — comes from that one call.
3. **`control` is the connector object.** It's the internal store handle. You pass it to `Controller`, `useFieldArray`, and `useWatch` so they can plug into the same form instance.
4. **Re-renders are opt-in.** Reading `formState.errors` subscribes you to error changes; calling `watch("x")` subscribes you to `x`. If you don't subscribe, you don't re-render. This is enforced by ES Proxies under the hood.

---

## 3. `useForm` — the entry point and every option

`useForm` is the hook that creates and owns a form instance. **What it is:** a factory that returns all the methods and state you need to wire up, validate, read, and submit a form. **The logic:** by centralizing form state in one hook, RHF can track subscriptions precisely and avoid global re-renders. **When you use it:** once per form, at the top of the component that owns that form.

### 3.1 A complete first form **[B]**

```tsx
"use client" // Required in the Next.js App Router — RHF runs on the client.

import { useForm, type SubmitHandler } from "react-hook-form"

// 1. Describe the SHAPE of your form data with a TypeScript type.
//    RHF uses this generic to type `register` names, `errors`, `data`, everything.
type LoginInputs = {
  email: string
  password: string
  rememberMe: boolean
}

export function LoginForm() {
  // 2. Call useForm<YourType>() and destructure the tools you need.
  const {
    register,                                   // connects native inputs to RHF
    handleSubmit,                               // wraps your submit fn with validation
    formState: { errors, isSubmitting },        // reactive form metadata (subscribe by reading)
  } = useForm<LoginInputs>({
    // 3. defaultValues seed the fields AND define what "dirty/clean" means.
    //    Always provide them — it prevents controlled/uncontrolled warnings.
    defaultValues: {
      email: "",
      password: "",
      rememberMe: false,
    },
    mode: "onTouched", // when validation runs (see §7); onTouched is a great default UX
  })

  // 4. Your submit handler receives FULLY TYPED, validated data.
  const onSubmit: SubmitHandler<LoginInputs> = async (data) => {
    // data.email / data.password / data.rememberMe are all correctly typed.
    await fakeLogin(data)
  }

  return (
    // 5. handleSubmit(onSubmit) validates first, then calls onSubmit only if valid.
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        // 6. Spread register(name, rules) — never pick its properties apart.
        {...register("email", { required: "Email is required" })}
        aria-invalid={errors.email ? "true" : "false"}
      />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        {...register("password", { required: "Password is required" })}
        aria-invalid={errors.password ? "true" : "false"}
      />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <label>
        <input type="checkbox" {...register("rememberMe")} /> Remember me
      </label>

      {/* isSubmitting is true while the async onSubmit promise is pending. */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Logging in…" : "Log in"}
      </button>
    </form>
  )
}
```

### 3.2 `useForm` options — full reference

These options shape how the whole form behaves. You'll use `defaultValues`, `mode`, and `resolver` constantly; the rest are situational but worth knowing.

| Option | Default | What it does & when to use it |
|---|---|---|
| `defaultValues` | `{}` | Initial field values. **Defines the dirty baseline** (a field is "dirty" when it differs from its default). Accepts a plain object *or an async function* that returns one (§17). Always set it. |
| `values` | — | **Reactive** external values that sync *into* the form whenever they change. Use for edit forms driven by fetched data — when `values` updates, RHF resets to it. Distinct from `defaultValues`, which is read once. |
| `mode` | `"onSubmit"` | When validation runs *before* the first submit. `onSubmit`/`onBlur`/`onChange`/`onTouched`/`all`. See §7. |
| `reValidateMode` | `"onChange"` | When validation re-runs *after* the first submit attempt. Usually leave as `onChange` so errors clear as the user fixes them. |
| `resolver` | — | Hooks in a schema validator (`zodResolver(schema)`). When set, inline `register` rules are ignored — put all rules in the schema. See §9. |
| `criteriaMode` | `"firstError"` | `"all"` collects *every* failing rule per field into `errors.field.types` (for showing a checklist of requirements). |
| `shouldFocusError` | `true` | Auto-focus the first invalid field on a failed submit (good for accessibility). |
| `shouldUnregister` | `false` | If `true`, a field's value is removed from form state when its input unmounts. Default `false` keeps values across conditional rendering. |
| `shouldUseNativeValidation` | `false` | Use the browser's native constraint validation UI instead of JS error objects. Rarely used. |
| `delayError` | — | Milliseconds to wait before *displaying* an error after it occurs — reduces flicker on fast typing. |
| `disabled` | `false` | Disables all registered inputs at once (e.g. while the form is read-only). |
| `resetOptions` | — | Options passed to the internal `reset` that fires when `values` changes (e.g. `{ keepDirtyValues: true }`). |

> **Best practice:** the three you should set on almost every form are `defaultValues` (always), `mode: "onTouched"` (good UX), and `resolver: zodResolver(schema)` (once you adopt Zod). Everything else is opt-in.

> **`defaultValues` vs `values` — the crucial distinction.** `defaultValues` is read **once** at mount and defines the clean baseline. `values` is **reactive**: change it and the form re-syncs. For an edit form where data arrives asynchronously, either pass `values={fetchedData}` (RHF handles the reset) or use `defaultValues: async () => fetch(...)`. Mixing both has subtle precedence rules — pick one approach per form.

---

## 4. `register` — how ref-based registration works

`register` is the function that connects a native input to the form. **What it is:** a function `register(name, rules?)` that returns the props a native input needs to participate in RHF. **The logic:** it hands the input a `ref` callback so RHF can grab the DOM node and read its value directly — that's the uncontrolled-first performance trick from §1. **When you use it:** for every native HTML input/select/textarea. (For non-native, controlled components, you use `Controller` instead — §10.)

### 4.1 What `register` returns

```tsx
// register(name, rules?) returns these four props:
{
  name: string                 // the field's name/path
  ref: RefCallback             // RHF captures the DOM node through this — the heart of it all
  onChange: ChangeHandler      // notifies RHF of changes (drives re-validation per `mode`)
  onBlur: ChangeHandler        // notifies RHF on blur (drives touched state + onBlur validation)
}

// Always SPREAD the whole object so the ref is included:
<input {...register("email")} />
```

The `ref` is the load-bearing part. When React mounts the input, it calls that ref callback with the DOM element; RHF stores it. Lose the ref (by destructuring only `onChange`/`onBlur`) and RHF can no longer read the value or focus the field on error.

### 4.2 Type coercion — HTML inputs are always strings **[I]**

Every HTML input gives you a *string*, even `type="number"` and `type="date"`. If you want a real `number` or `Date` in your submit data, tell RHF to coerce.

```tsx
// valueAsNumber — parse to a Number before it reaches your submit handler.
<input
  type="number"
  {...register("age", {
    valueAsNumber: true,                       // data.age becomes 25 (number), not "25"
    min: { value: 0, message: "Must be positive" },
  })}
/>

// valueAsDate — parse to a Date object.
<input type="date" {...register("birthDate", { valueAsDate: true })} />

// setValueAs — arbitrary transform (runs after valueAsNumber/valueAsDate if combined).
<input
  {...register("couponCode", {
    setValueAs: (v) => v.trim().toUpperCase(),  // normalize on the way in
  })}
/>
```

> **Gotcha:** an empty `type="number"` field with `valueAsNumber` yields `NaN`, not `0` or `""`. Handle it in validation (`z.coerce.number()` or a `validate` that allows empty for optional fields), or the user gets a confusing `NaN`.

> **⚡ Version note:** when using `zodResolver`, prefer `z.coerce.number()` / `z.coerce.date()` in the schema over `valueAsNumber`/`valueAsDate` so all coercion logic lives in one place. They can be combined, but doing coercion twice is a common source of `NaN`.

### 4.3 Separating the ref (when a component can't take `...rest`) **[A]**

Some component libraries expose the underlying input ref under a differently-named prop. Pull `ref` out and forward it explicitly:

```tsx
const { ref, ...rest } = register("email")
// CustomInput forwards its DOM ref via `inputRef` instead of `ref`:
<CustomInput {...rest} inputRef={ref} />
```

### 4.4 Unregistering and conditional fields

By default, when a registered input unmounts (e.g. a conditionally rendered field), **its value is kept** in form state. This is usually what you want (the user toggles a section closed and back open without losing data). To instead discard the value when the input leaves the DOM:

```tsx
useForm({ shouldUnregister: true })            // global: all fields
register("promoCode", { shouldUnregister: true }) // per-field
```

> **Best practice:** keep the default (`false`) unless conditionally-hidden fields are causing stale data to be submitted. If a field is only relevant when a checkbox is on, either set `shouldUnregister` for it or strip it in `onSubmit` / your Zod transform.

---

## 5. `handleSubmit` — validating and submitting

`handleSubmit` is the bridge between the form and your business logic. **What it is:** a higher-order function — you give it your submit callback and it returns an event handler for `<form onSubmit={...}>`. **The logic:** it intercepts the native submit event, calls `preventDefault()`, runs validation (inline rules or resolver), and only invokes *your* callback if validation passes. **When you use it:** on every form's `onSubmit`.

```tsx
const onValid: SubmitHandler<Inputs> = async (data) => {
  // Runs ONLY when all validation passes. `data` is typed and clean.
  await saveToServer(data)
}

const onInvalid: SubmitErrorHandler<Inputs> = (errors) => {
  // OPTIONAL second arg — runs when the user submits but validation fails.
  // Great for analytics, logging, or scrolling to the first error.
  console.warn("Validation failed:", errors)
}

<form onSubmit={handleSubmit(onValid, onInvalid)}>...</form>
```

### 5.1 Async submit and `isSubmitting`

If your callback returns a promise (e.g. an `async` function), RHF keeps `formState.isSubmitting` `true` until it resolves or rejects. This is how you disable the button and show a spinner without any extra state.

```tsx
const { handleSubmit, formState: { isSubmitting } } = useForm<Inputs>()

const onSubmit = async (data: Inputs) => {
  // isSubmitting is true for the entire duration of this await.
  const res = await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  })
  if (!res.ok) throw new Error("Request failed") // throwing keeps isSubmitting accurate
}

<button type="submit" disabled={isSubmitting}>
  {isSubmitting ? "Saving…" : "Save"}
</button>
```

> **Gotcha:** `handleSubmit` swallows the *return value* of your callback but **does not** swallow validation. If you `throw` inside `onSubmit`, the promise rejects and `isSubmitting` flips back to `false`, but `isSubmitSuccessful` stays `false` — useful for distinguishing "submitted and worked" from "submitted and errored." Map server errors with `setError` (§13, §17) rather than throwing if you want to show field-level messages.

---

## 6. `formState` — errors, isDirty, isValid, and friends

`formState` is the reactive metadata object describing the form's current condition. **What it is:** an object of flags and error data that updates as the user interacts. **The logic:** it's wrapped in an ES Proxy so that *reading* a property subscribes your component to changes in **that property only** — destructure `errors` and you re-render on error changes, not on every keystroke. **When you use it:** constantly — for error messages, button states, dirty checks, and success UI.

> **Critical best practice:** destructure `formState` at the top so the proxy can detect which fields you read. `const { formState: { errors, isDirty } } = useForm()` subscribes to exactly `errors` and `isDirty`. Accessing `formState.errors` lazily deep inside JSX also works, but destructuring up front is the clearest and is what the proxy optimization expects.

### 6.1 Every flag, explained

```tsx
const {
  formState: {
    errors,             // object of field errors: errors.email?.message, errors.email?.type
    isDirty,            // true if ANY field differs from its defaultValue
    dirtyFields,        // { fieldName: true } map of which fields changed
    touchedFields,      // { fieldName: true } map of which fields were blurred
    isValid,            // true when there are no validation errors (needs a non-onSubmit mode to be live)
    isValidating,       // true while async validation is in flight
    isSubmitting,       // true while the async submit handler is pending
    isSubmitted,        // true after the first submit attempt (success or fail)
    isSubmitSuccessful, // true only after a submit that completed without throwing
    submitCount,        // how many times the form was submitted
    defaultValues,      // the current default values (read-only snapshot)
  },
} = useForm<Inputs>()
```

| Flag | What it means | Typical use |
|---|---|---|
| `errors` | Per-field error objects | Render messages: `errors.email?.message` |
| `isDirty` | Form differs from defaults | Disable "Save" until something changed; warn on unsaved-changes navigation |
| `dirtyFields` | Which fields changed | Send only changed fields in a PATCH request |
| `touchedFields` | Which fields were blurred | Show errors only after a field has been visited |
| `isValid` | No errors exist | Enable submit only when valid (use with `mode: "onChange"`/`"onTouched"`) |
| `isValidating` | Async validation running | Show a spinner next to an async-checked field |
| `isSubmitting` | Submit in progress | Disable the submit button, show "Saving…" |
| `isSubmitted` | At least one submit attempt | Decide whether to show validation summaries |
| `isSubmitSuccessful` | Last submit succeeded | Show a success toast; trigger a `reset()` |
| `submitCount` | Number of submit attempts | Detect repeated failures; show extra help after N tries |

### 6.2 Error display patterns **[B/I]**

```tsx
// Pattern 1 — simple conditional (clearest for beginners):
{errors.email && <p role="alert">{errors.email.message}</p>}

// Pattern 2 — optional chaining (concise, null-safe):
<p role="alert">{errors.email?.message}</p>

// Pattern 3 — discriminate by which RULE failed (when you want different copy per rule):
{errors.username?.type === "minLength" && <p>Username is too short.</p>}
{errors.username?.type === "pattern"   && <p>Only letters, numbers, underscores.</p>}

// Pattern 4 — show ALL failing rules at once (requires criteriaMode: "all"):
//   useForm({ criteriaMode: "all" })  →  errors.password.types is { minLength: "...", pattern: "..." }
{errors.password?.types &&
  Object.values(errors.password.types).map((msg) => <p key={String(msg)}>{msg}</p>)}
```

> **`role="alert"`** makes screen readers announce the error the moment it appears — see §20. Always pair it with `aria-invalid` on the input.

### 6.3 `isDirty` and `dirtyFields` in practice

```tsx
function SettingsForm() {
  const {
    register, handleSubmit,
    formState: { isDirty, dirtyFields, isSubmitting },
  } = useForm<Settings>({ defaultValues: currentSettings })

  const onSubmit = async (data: Settings) => {
    // Only send the fields the user actually changed (efficient PATCH).
    const changed = Object.fromEntries(
      Object.keys(dirtyFields).map((k) => [k, data[k as keyof Settings]]),
    )
    await patchSettings(changed)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("displayName")} />
      <input {...register("email")} />
      {/* Disable Save until something changed — clear, honest UX. */}
      <button type="submit" disabled={!isDirty || isSubmitting}>Save changes</button>
    </form>
  )
}
```

> **Gotcha:** `isDirty` compares against `defaultValues`. If you type a value then delete it back to the default, `isDirty` returns to `false` — which is correct but can surprise you. Also, after a successful save, call `reset(data)` to make the *new* values the baseline, otherwise the form stays "dirty."

---

## 7. Validation modes — when validation runs

The `mode` option controls **when validation fires** during interaction, before the first submit. **The logic:** more eager validation = better real-time feedback but more re-renders; less eager = fewer renders but the user finds out about errors later. **When you tune it:** to balance UX against performance for your specific form.

| `mode` | Validates on… | Re-render cost | Best for |
|---|---|---|---|
| `"onSubmit"` (default) | Submit only | Lowest | Simple forms; minimal feedback |
| `"onBlur"` | Field blur | Low | Solid default; not nagging mid-typing |
| `"onChange"` | Every keystroke | Higher | Real-time feedback (password strength, etc.) |
| `"onTouched"` | First blur, then every change after | Medium | **Best general-purpose UX** |
| `"all"` | Blur **and** change | Highest | Maximum eagerness; use sparingly |

```tsx
useForm<Inputs>({
  mode: "onTouched",          // don't nag until the user has visited the field, then react live
  reValidateMode: "onChange", // after the first submit attempt, clear errors as they're fixed
})
```

### 7.1 `reValidateMode` — the after-submit behaviour

`reValidateMode` only kicks in **after the first submit attempt**. The default `"onChange"` gives the ideal experience: the user submits, sees errors, and as they correct each field the error vanishes immediately. You rarely change this.

### 7.2 Triggering validation manually

Sometimes you need to validate at a moment of your choosing — e.g. before advancing a wizard step. That's `trigger` (full details in §13):

```tsx
const { trigger } = useForm()
const ok = await trigger("email")              // validate one field, returns boolean
await trigger(["email", "password"])           // validate several
const formOk = await trigger()                 // validate everything
```

---

## 8. Built-in validation rules

For simple forms you can validate without any schema library at all, by passing rules as the second argument to `register`. **When to use:** quick forms, prototypes, or when you don't want a schema dependency. For anything non-trivial, prefer Zod (§9) — but knowing the built-ins is essential.

```tsx
<input
  {...register("username", {
    // --- required ---
    required: "Username is required",            // a string IS the error message
    // alternatives: required: true               (boolean, no custom message)
    //               required: { value: true, message: "Required" }

    // --- string length ---
    minLength: { value: 3,  message: "At least 3 characters" },
    maxLength: { value: 20, message: "At most 20 characters" },

    // --- numeric bounds (use with type="number" + valueAsNumber) ---
    min: { value: 1,   message: "Must be ≥ 1" },
    max: { value: 100, message: "Must be ≤ 100" },

    // --- regex pattern ---
    pattern: {
      value: /^[a-zA-Z0-9_]+$/,
      message: "Letters, numbers and underscores only",
    },

    // --- single custom validator: return true (valid) or an error string ---
    validate: (value) => value !== "admin" || "The name 'admin' is reserved",

    // --- OR multiple named validators (each keyed by its error 'type') ---
    // validate: {
    //   notReserved: (v) => v !== "admin" || "Reserved name",
    //   noSpaces:    (v) => !/\s/.test(v) || "No spaces allowed",
    //   async checkAvailable: async (v) => (await isFree(v)) || "Already taken",
    // },
  })}
/>
```

| Rule | Shape | Notes |
|---|---|---|
| `required` | `string \| boolean \| { value, message }` | String form is the shortest way to get a message |
| `minLength` / `maxLength` | `{ value: number, message }` | String length, not numeric value |
| `min` / `max` | `{ value: number, message }` | Numeric value; pair with `valueAsNumber` |
| `pattern` | `{ value: RegExp, message }` | Anchor your regex (`^…$`) to validate the whole value |
| `validate` | `fn \| Record<string, fn>` | Return `true`/`false`/`string`; may be `async` (§17). The richest rule. |

> **Best practice:** the `validate` object form is your escape hatch for cross-field rules without a schema (e.g. "confirm password matches" by reading `getValues("password")`). But once you have more than a couple of `validate` functions, switch to Zod — it's cleaner and reusable on the server.

> **Gotcha:** when a `resolver` (Zod/Yup) is set, **inline rules are ignored entirely.** Don't mix the two — pick schema *or* inline per form (§21 #11).

---

## 9. Schema validation with Zod + `zodResolver`

**What it is:** instead of scattering rules across `register` calls, you define one **schema** object that describes the entire form's valid shape, and plug it into `useForm` via a *resolver*. **The logic:** a schema is a single source of truth — it gives you (1) runtime validation, (2) a TypeScript type via inference, and (3) reusability on the server, all from one definition. **When you use it:** for essentially every real form. This is the modern standard.

Zod is the recommended schema library because `z.infer` turns your schema into your form type for free, eliminating drift between "what the form validates" and "what TypeScript thinks the data is."

### 9.1 Install & wire up

```bash
npm install zod @hookform/resolvers
```

```tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

// 1. Define the schema — this is BOTH your validation AND your type.
const signupSchema = z
  .object({
    username: z
      .string()
      .min(3, "At least 3 characters")
      .max(20, "At most 20 characters")
      .regex(/^[a-zA-Z0-9_]+$/, "Letters, numbers, underscores only"),
    email: z.string().email("Enter a valid email"),
    password: z
      .string()
      .min(8, "At least 8 characters")
      .regex(/[A-Z]/, "Needs an uppercase letter")
      .regex(/[0-9]/, "Needs a number"),
    confirmPassword: z.string(),
    // z.coerce.number() turns the input's "25" string into a number, then validates.
    age: z.coerce.number().min(18, "Must be 18 or older"),
    role: z.enum(["admin", "editor", "viewer"]),
    acceptTerms: z.literal(true, { message: "You must accept the terms" }),
  })
  // .refine validates ACROSS fields. `path` decides which field shows the error.
  .refine((d) => d.password === d.confirmPassword, {
    path: ["confirmPassword"],
    message: "Passwords do not match",
  })

// 2. Infer the form type from the schema — never hand-write it again.
type SignupInputs = z.infer<typeof signupSchema>

export function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<SignupInputs>({
    resolver: zodResolver(signupSchema), // ALL validation now flows through Zod
    defaultValues: {
      username: "", email: "", password: "", confirmPassword: "",
      age: 18, role: "viewer", acceptTerms: false,
    },
  })

  const onSubmit = async (data: SignupInputs) => {
    await createUser(data) // data is fully typed and already validated
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register("username")} placeholder="Username" aria-invalid={!!errors.username} />
      {errors.username && <p role="alert">{errors.username.message}</p>}

      <input {...register("email")} type="email" placeholder="Email" />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input {...register("password")} type="password" placeholder="Password" />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <input {...register("confirmPassword")} type="password" placeholder="Confirm password" />
      {errors.confirmPassword && <p role="alert">{errors.confirmPassword.message}</p>}

      {/* No valueAsNumber needed — z.coerce.number() handles the conversion. */}
      <input {...register("age")} type="number" />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      <select {...register("role")}>
        <option value="viewer">Viewer</option>
        <option value="editor">Editor</option>
        <option value="admin">Admin</option>
      </select>

      <label>
        <input type="checkbox" {...register("acceptTerms")} /> I accept the terms
      </label>
      {errors.acceptTerms && <p role="alert">{errors.acceptTerms.message}</p>}

      <button type="submit" disabled={isSubmitting}>Sign up</button>
    </form>
  )
}
```

### 9.2 `input` vs `output` types **[A]**

When a schema *transforms* values (`z.coerce`, `.transform()`, `.default()`), the data **going in** differs from the data **coming out**. Zod exposes both: `z.input<typeof schema>` (pre-transform — what the form fields hold) and `z.output<typeof schema>` (post-transform — what your submit handler receives). For type-correctness with transforms, pass both generics:

```tsx
// useForm<TFieldValues, TContext, TTransformedValues>
const form = useForm<
  z.input<typeof signupSchema>,   // field values BEFORE transform (e.g. age as string)
  unknown,
  z.output<typeof signupSchema>   // submit data AFTER transform (e.g. age as number)
>({ resolver: zodResolver(signupSchema) })
```

> **Best practice:** for most forms, `z.infer` (an alias of `z.output`) is enough. Only reach for the three-generic form when transforms make the input/output types genuinely diverge and TypeScript complains.

### 9.3 Sharing the schema with your backend

The biggest payoff: define the schema **once** in a shared module and import it in both the form and the API/Server Action. The client validates for UX; the server re-validates the same way for security. You can never trust the client, so this dual validation with one schema is the recommended pattern.

```tsx
// lib/schemas/user.ts — the single source of truth, imported everywhere
import { z } from "zod"
export const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
})
export type CreateUserInput = z.infer<typeof createUserSchema>
```

⚡ **Version note (Zod v3 vs v4):** `zodResolver` supports both. The most visible v4 change is error customization: v4 uses a unified `{ message: "..." }` / `error` API and deprecates the old `errorMap`/`invalid_type_error` style. `z.literal(true, { message })` (used above) works in v4; in v3 you may need `z.literal(true, { errorMap: () => ({ message }) })`. Zod v4 also parses faster and changed `.email()` internals. If you upgrade, run your form tests — most schemas need no changes.

### 9.4 Yup / Valibot (brief)

The resolver pattern is identical with other libraries; only the schema syntax differs.

```tsx
// Yup
import { yupResolver } from "@hookform/resolvers/yup"
import * as yup from "yup"
const yupSchema = yup.object({ name: yup.string().required() })
useForm({ resolver: yupResolver(yupSchema) })

// Valibot (tiny, tree-shakeable, modular)
import { valibotResolver } from "@hookform/resolvers/valibot"
import * as v from "valibot"
const vSchema = v.object({ name: v.pipe(v.string(), v.minLength(1)) })
useForm({ resolver: valibotResolver(vSchema) })
```

---

## 10. `Controller` & `useController` — controlled components

Native `<input>`, `<select>`, and `<textarea>` work perfectly with `register`. But many components — MUI `TextField`, shadcn `Select`, `react-select`, date pickers, rich-text editors, toggle switches — **manage their own internal state and expose a controlled API** (`value` + `onChange`) instead of accepting a `ref` you can read. `register` cannot read their value because there's no DOM input to grab. **That's exactly when you need `Controller`.**

**What `Controller` is:** an adapter that bridges RHF's uncontrolled model to a controlled component's `value`/`onChange` interface. It subscribes to the field internally and feeds the component a `value`, and gives you an `onChange` to push updates back into RHF. **The logic:** it isolates the controlled-component re-render to just that field rather than the whole form. **When you use it:** any component that isn't a plain native input.

### 10.1 `Controller` with `render`

```tsx
import { useForm, Controller } from "react-hook-form"
import Select from "react-select"
import DatePicker from "react-datepicker"

type Inputs = { country: string; birthDate: Date | null }

function ProfileForm() {
  const { control, handleSubmit } = useForm<Inputs>({
    defaultValues: { country: "", birthDate: null },
  })

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="country"
        control={control}                          // connect to THIS form instance
        rules={{ required: "Country is required" }} // rules live here (or in the schema)
        render={({ field, fieldState }) => (
          <Select
            options={countryOptions}
            // field provides: value, onChange, onBlur, name, ref, disabled.
            value={countryOptions.find((o) => o.value === field.value) ?? null}
            onChange={(opt) => field.onChange(opt?.value)} // push the chosen value into RHF
            onBlur={field.onBlur}
            name={field.name}
            className={fieldState.error ? "has-error" : ""}
          />
        )}
      />

      <Controller
        name="birthDate"
        control={control}
        render={({ field }) => (
          <DatePicker
            selected={field.value}
            onChange={field.onChange} // RHF receives the new Date directly
            onBlur={field.onBlur}
            ref={field.ref}
          />
        )}
      />
    </form>
  )
}
```

### 10.2 The `render` prop arguments

| Path | What it is |
|---|---|
| `field.value` | The field's current value (feed it to the component) |
| `field.onChange` | Call with the new value to update RHF |
| `field.onBlur` | Call when the component loses focus (drives touched/onBlur validation) |
| `field.name` | The field name (pass through for accessibility/forms) |
| `field.ref` | Ref RHF uses to focus the element on a validation error |
| `field.disabled` | Reflects `useForm({ disabled })` / `Controller` `disabled` |
| `fieldState.error` | This field's error object (same as `errors.country`) |
| `fieldState.isDirty` | Whether this field changed from its default |
| `fieldState.isTouched` | Whether this field was blurred |
| `fieldState.invalid` | Convenience boolean: has an error |
| `formState` | The whole form state, if you need it here |

### 10.3 `useController` — build reusable controlled inputs **[A]**

`useController` is the hook version of `Controller`. **When you use it:** to encapsulate a controlled field into your own reusable component (a design-system input), so the consumer just drops it in.

```tsx
import { useController, type UseControllerProps } from "react-hook-form"

type Props = UseControllerProps<any> & { label: string }

function TextField({ label, ...controllerProps }: Props) {
  // useController returns the same field/fieldState as Controller's render prop.
  const { field, fieldState } = useController(controllerProps)
  return (
    <div>
      <label htmlFor={field.name}>{label}</label>
      <input id={field.name} {...field} aria-invalid={!!fieldState.error} />
      {fieldState.error && <p role="alert">{fieldState.error.message}</p>}
    </div>
  )
}

// Usage — clean and declarative:
<TextField name="username" control={control} rules={{ required: true }} label="Username" />
```

### 10.4 MUI example (one of the most common cases)

```tsx
import { TextField } from "@mui/material"

<Controller
  name="firstName"
  control={control}
  rules={{ required: "Required" }}
  render={({ field, fieldState }) => (
    <TextField
      {...field}                                  // value, onChange, onBlur, name, ref
      label="First name"
      error={!!fieldState.error}                  // MUI's red-border state
      helperText={fieldState.error?.message}      // MUI's message slot
    />
  )}
/>
```

> **Best practice:** never use `Controller` for plain native inputs — it adds overhead `register` avoids (§19). Reserve it for genuinely controlled components. If you're unsure: does the component accept a `ref` to a real `<input>` and let you read `.value`? Then use `register`. Does it only expose `value`/`onChange`? Use `Controller`.

---

## 11. `watch` vs `useWatch` — reacting to values

Sometimes you need to *react* to a field's value while the user types — to show a live preview, compute a total, or conditionally render another field. RHF gives you two tools, and **choosing the right one is a performance decision.**

### 11.1 `watch` — quick, but re-renders the whole form

`watch` is a method from `useForm`. **The logic:** calling `watch("x")` subscribes the **component that called `useForm`** to changes in `x`, so that whole component re-renders on each change. **When to use:** small forms, prototyping, or when the watched value is needed right there in the form component.

```tsx
const { watch, register } = useForm<Inputs>()

const country = watch("country")                 // re-render this component when country changes
const [first, last] = watch(["firstName", "lastName"]) // watch several
const everything = watch()                       // watch ALL fields (expensive — avoid in big forms)

// Callback form: subscribe WITHOUT causing a render (great for side effects):
useEffect(() => {
  const sub = watch((value, { name, type }) => {
    console.log(`${name} changed`, value)
  })
  return () => sub.unsubscribe()                 // always clean up
}, [watch])
```

### 11.2 `useWatch` — isolates the re-render to one component

`useWatch` is a standalone hook. **The logic:** it subscribes only the component that calls it, leaving the parent form untouched. **When to use:** production forms where you want to confine re-renders. Move the "live total" or "preview" into a small child component that takes `control` and calls `useWatch` — the heavy form stops re-rendering on every keystroke.

```tsx
import { useWatch, type Control } from "react-hook-form"

// Only THIS component re-renders when `weight` changes — the parent form does not.
function ShippingCost({ control }: { control: Control<Inputs> }) {
  const weight = useWatch({ control, name: "weight", defaultValue: 0 })
  return <p>Estimated shipping: ${(weight * 0.5).toFixed(2)}</p>
}

function OrderForm() {
  const { control, register } = useForm<Inputs>({ defaultValues: { weight: 0 } })
  return (
    <form>
      <input type="number" {...register("weight", { valueAsNumber: true })} />
      <ShippingCost control={control} /> {/* the re-render is contained here */}
    </form>
  )
}
```

### 11.3 The comparison

| | `watch()` | `useWatch()` |
|---|---|---|
| Kind | Method from `useForm` | Standalone hook |
| Re-renders | The component holding `useForm` | Only the component calling `useWatch` |
| Needs `control`? | No (it's already inside `useForm`) | Yes (pass `control` in) |
| Default value | From `defaultValues` | Provide `defaultValue` to avoid `undefined` on first render |
| Best for | Small forms, quick reads, callback subscriptions | Isolated derived UI in large forms |

> **Best practice:** if a watched value drives derived UI in a big form, **always** push it into a child component with `useWatch`. Verify the win in React DevTools Profiler — you'll see the parent stop re-rendering on each keystroke.

---

## 12. `useFieldArray` — dynamic / repeating fields

**What it is:** a hook that manages an **array of fields** — invoice line items, multiple phone numbers, a to-do list, tags, a repeating questionnaire section. **The logic:** each array entry gets a stable internal `id`, so React can track items correctly even as you add, remove, and reorder them. **When you use it:** any time the number of fields isn't fixed and the user can add/remove rows.

```tsx
import { useForm, useFieldArray, type SubmitHandler } from "react-hook-form"

type LineItem = { name: string; quantity: number; price: number }
type Invoice = { client: string; items: LineItem[] }

function InvoiceForm() {
  const {
    register, control, handleSubmit, watch,
    formState: { errors },
  } = useForm<Invoice>({
    defaultValues: {
      client: "",
      items: [{ name: "", quantity: 1, price: 0 }], // start with one row
    },
  })

  // Connect the array to the form via `control` + the array's path `name`.
  const { fields, append, remove, insert, move, swap, replace, prepend, update } =
    useFieldArray({ control, name: "items" })

  const onSubmit: SubmitHandler<Invoice> = (data) => console.log(data.items)

  // Compute a running total. (In a big form, move this into a useWatch child — §11.)
  const items = watch("items")
  const total = items.reduce((s, i) => s + (i.quantity || 0) * (i.price || 0), 0)

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("client", { required: "Client is required" })} placeholder="Client" />

      {fields.map((field, index) => (
        // CRITICAL: key on field.id (RHF's stable id), NEVER the array index.
        <div key={field.id} className="row">
          <input
            {...register(`items.${index}.name` as const, { required: "Name required" })}
            placeholder="Item"
          />
          {errors.items?.[index]?.name && (
            <span role="alert">{errors.items[index]?.name?.message}</span>
          )}

          <input
            type="number"
            {...register(`items.${index}.quantity` as const, {
              valueAsNumber: true,
              min: { value: 1, message: "Min 1" },
            })}
          />
          <input
            type="number"
            step="0.01"
            {...register(`items.${index}.price` as const, { valueAsNumber: true })}
          />

          {/* Keep at least one row. */}
          <button type="button" onClick={() => remove(index)} disabled={fields.length === 1}>
            Remove
          </button>
        </div>
      ))}

      <p>Total: ${total.toFixed(2)}</p>
      <button type="button" onClick={() => append({ name: "", quantity: 1, price: 0 })}>
        + Add item
      </button>
      <button type="submit">Save invoice</button>
    </form>
  )
}
```

### 12.1 Methods

| Method | Signature | Does |
|---|---|---|
| `append` | `append(value \| value[], opts?)` | Add to the end |
| `prepend` | `prepend(value \| value[], opts?)` | Add to the beginning |
| `insert` | `insert(index, value)` | Insert at a position |
| `remove` | `remove(index? \| index[])` | Remove by index (or all if omitted) |
| `move` | `move(from, to)` | Reorder (drag-and-drop) |
| `swap` | `swap(a, b)` | Swap two entries |
| `replace` | `replace(value[])` | Replace the entire array |
| `update` | `update(index, value)` | Replace one entry (note: remounts that row) |

### 12.2 The non-negotiable rules

```tsx
// ✅ Key on field.id — RHF's stable identifier. Reordering stays correct.
{fields.map((field, i) => <Row key={field.id} index={i} />)}

// ❌ Keying on index corrupts state when items are removed/reordered.
{fields.map((_, i) => <Row key={i} index={i} />)}

// ✅ Use `as const` template-literal names so TypeScript types the path.
register(`items.${index}.name` as const)

// ✅ Provide an array in defaultValues, or the rows render uncontrolled.
defaultValues: { items: [{ name: "", quantity: 1, price: 0 }] }
```

> **Gotcha:** `field.id` is RHF's internal key — it is **not** a value in your form data and won't be submitted. Don't try to read it as data. If you need a real id, add your own field (e.g. `items.${i}.id`).

### 12.3 Nested field arrays (arrays inside arrays) **[A]**

For arrays within arrays (sections → questions), create a **child component per outer item** that runs its own `useFieldArray` on the nested path. This keeps each level's re-renders isolated.

```tsx
type Survey = { sections: { title: string; questions: { text: string }[] }[] }

function QuestionList({ sectionIndex, control, register }: {
  sectionIndex: number
  control: Control<Survey>
  register: UseFormRegister<Survey>
}) {
  const { fields, append, remove } = useFieldArray({
    control,
    name: `sections.${sectionIndex}.questions`, // nested path
  })
  return (
    <>
      {fields.map((f, i) => (
        <div key={f.id}>
          <input {...register(`sections.${sectionIndex}.questions.${i}.text` as const)} />
          <button type="button" onClick={() => remove(i)}>×</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ text: "" })}>+ Question</button>
    </>
  )
}
```

---

## 13. Imperative methods: setValue, getValues, reset, trigger, setError, setFocus

These are the "do something to the form programmatically" methods. They don't subscribe you to anything (no re-render just for calling them) — they *act*. Reach for them in effects, event handlers, and async flows.

### 13.1 `getValues` — read without subscribing

Reads the current value synchronously. **No re-render.** Use it when you need a value at a moment in time (e.g. inside a click handler) and don't want the reactivity of `watch`.

```tsx
const v = getValues("email")            // one field
const all = getValues()                 // the whole form
const some = getValues(["email", "name"]) // a subset, as an array
```

### 13.2 `setValue` — programmatically set a field

```tsx
setValue("email", "user@example.com")   // basic set
setValue("email", "user@example.com", {
  shouldDirty: true,                    // mark the field dirty (affects isDirty)
  shouldTouch: true,                    // mark it touched
  shouldValidate: true,                 // run validation immediately after setting
})
```

> **Gotcha:** by default `setValue` does **not** mark the field dirty or validate it. If you set a value programmatically and expect `isDirty`/errors to update, pass `shouldDirty`/`shouldValidate`.

### 13.3 `reset` — reset the whole form

`reset` returns the form to a clean state. **When you use it:** after a successful save (to make the saved values the new baseline), or to load fetched data into an edit form.

```tsx
reset()                                  // back to the original defaultValues
reset({ name: "Jane", email: "j@x.com" })// reset to NEW values (becomes the new baseline)
reset(data, {                            // reset with fine-grained control
  keepValues: false,
  keepDirty: false,
  keepErrors: false,
  keepTouched: false,
  keepIsSubmitted: false,
  keepDirtyValues: false,                // keep user edits but reset everything else
})
```

### 13.4 `resetField` — reset a single field

```tsx
resetField("email")                              // back to its default; clears dirty/touched/error
resetField("email", { defaultValue: "new@x.com" }) // and set a new default
resetField("email", { keepDirty: true, keepError: false })
```

### 13.5 `trigger` — run validation on demand

Returns a `Promise<boolean>`. **Essential for multi-step wizards:** validate the current step before allowing "Next".

```tsx
const isValid = await trigger("email")           // one field
await trigger(["email", "password"])             // several
const formValid = await trigger()                // everything

// Wizard step gate:
async function next() {
  const ok = await trigger(["firstName", "lastName"]) // only this step's fields
  if (ok) setStep((s) => s + 1)
}
```

### 13.6 `setError` / `clearErrors` — manual & server errors

`setError` injects an error that didn't come from validation — most often a **server-side error** (e.g. "email already registered" from a 422 response).

```tsx
setError("email", { type: "server", message: "This email is already registered" })

// Root-level (form-wide) error not tied to any single field:
setError("root.serverError", { type: "server", message: "Something went wrong." })
// Read it like any error: errors.root?.serverError?.message

clearErrors("email")                    // clear one
clearErrors(["email", "name"])          // clear several
clearErrors()                           // clear all
```

> **Gotcha:** errors set via `setError` (other than `root.*`) are cleared the next time that field is validated. That's usually desirable (the user fixes it, the error clears), but if you set a server error and the user doesn't edit the field, it persists until the next submit/validation. `root.*` errors are *not* tied to a field and must be cleared manually or via `reset`.

### 13.7 `setFocus` — move focus programmatically

```tsx
setFocus("email")                        // focus the field
setFocus("email", { shouldSelect: true })// focus and select its text
// Common after mapping a server error: setError(...) then setFocus(firstBadField)
```

---

## 14. Nested objects & `useFormContext` for large forms

### 14.1 Nested objects with dot-notation **[I]**

RHF addresses nested data with **dot/bracket paths**, and TypeScript validates the path against your form type. **When you use it:** structured data like addresses, contact blocks, settings groups.

```tsx
type UserForm = {
  name: string
  contact: { email: string; phone: string }
  shippingAddress: { street: string; city: string; zip: string }
  billingAddress: { street: string; city: string; zip: string }
  sameAsBilling: boolean
}

function ProfileForm() {
  const { register, watch, setValue, handleSubmit, formState: { errors } } =
    useForm<UserForm>({
      defaultValues: {
        name: "",
        contact: { email: "", phone: "" },
        shippingAddress: { street: "", city: "", zip: "" },
        billingAddress: { street: "", city: "", zip: "" },
        sameAsBilling: false,
      },
    })

  const sameAsBilling = watch("sameAsBilling")
  const billing = watch("billingAddress")

  // Copy billing → shipping when the box is ticked.
  useEffect(() => {
    if (sameAsBilling) setValue("shippingAddress", billing, { shouldDirty: true })
  }, [sameAsBilling, billing, setValue])

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register("name")} />
      <input {...register("contact.email", { required: "Email required" })} type="email" />
      {errors.contact?.email && <p role="alert">{errors.contact.email.message}</p>}

      <input {...register("billingAddress.street")} placeholder="Billing street" />
      <input {...register("billingAddress.city")} placeholder="Billing city" />

      <label>
        <input type="checkbox" {...register("sameAsBilling")} /> Shipping same as billing
      </label>

      {!sameAsBilling && (
        <>
          <input {...register("shippingAddress.street")} placeholder="Shipping street" />
          <input {...register("shippingAddress.city")} placeholder="Shipping city" />
        </>
      )}
      <button type="submit">Save</button>
    </form>
  )
}
```

### 14.2 `FormProvider` + `useFormContext` — stop prop-drilling **[A]**

**The problem:** when a form spans many components (multi-step wizards, deeply nested layouts, per-section components), threading `register`, `control`, and `errors` as props through every level is painful and brittle.

**The solution:** `FormProvider` puts the entire `useForm` toolbox into React context, and any descendant calls `useFormContext()` to grab what it needs — no props. **When you use it:** large or split-up forms.

```tsx
import { useForm, FormProvider, useFormContext } from "react-hook-form"

type Checkout = {
  personal: { name: string; email: string }
  payment: { cardNumber: string; expiry: string; cvv: string }
}

function CheckoutPage() {
  const methods = useForm<Checkout>({
    defaultValues: {
      personal: { name: "", email: "" },
      payment: { cardNumber: "", expiry: "", cvv: "" },
    },
  })

  return (
    // Spread ALL methods into context.
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit((d) => console.log(d))}>
        <PersonalSection /> {/* no props needed */}
        <PaymentSection />
        <button type="submit" disabled={methods.formState.isSubmitting}>Place order</button>
      </form>
    </FormProvider>
  )
}

function PersonalSection() {
  // Pull exactly what this section needs from context.
  const { register, formState: { errors } } = useFormContext<Checkout>()
  return (
    <fieldset>
      <legend>Personal</legend>
      <input {...register("personal.name", { required: "Name required" })} />
      {errors.personal?.name && <p role="alert">{errors.personal.name.message}</p>}
      <input {...register("personal.email", { required: "Email required" })} type="email" />
      {errors.personal?.email && <p role="alert">{errors.personal.email.message}</p>}
    </fieldset>
  )
}

function PaymentSection() {
  const { register } = useFormContext<Checkout>()
  return (
    <fieldset>
      <legend>Payment</legend>
      <input {...register("payment.cardNumber", { required: true, minLength: 16 })} />
      <input {...register("payment.expiry", { required: true })} placeholder="MM/YY" />
      <input {...register("payment.cvv", { required: true, maxLength: 4 })} type="password" />
    </fieldset>
  )
}
```

> **Gotcha:** `useFormContext()` returns `null` if there's no surrounding `FormProvider` — calling methods on it throws. If a component can be used both inside and outside a form, guard it or take props as a fallback.

> **Performance note:** prefer passing `control` as a prop to leaf components that use `useWatch`/`useController` when you can — it makes data flow explicit and is easy to memoize. Use `FormProvider` when prop-drilling becomes the bigger evil. They're not mutually exclusive.

---

## 15. The shadcn/ui Form pattern

**What it is:** shadcn/ui ships a small set of components — `Form`, `FormField`, `FormItem`, `FormLabel`, `FormControl`, `FormDescription`, `FormMessage` — that wrap RHF + a `Controller` and wire up **accessibility automatically** (label association, `aria-describedby`, `aria-invalid`, error announcement). **The logic:** it standardizes the boilerplate of label + control + error + description into a composable, accessible structure. **When you use it:** any shadcn/Tailwind project — this is the default form pattern in 2026.

```bash
# shadcn copies the component source into your repo (you own it).
npx shadcn@latest add form input button
```

```tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import {
  Form, FormControl, FormDescription, FormField,
  FormItem, FormLabel, FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

const schema = z.object({
  username: z.string().min(2, "At least 2 characters").max(50),
  bio: z.string().max(160).optional(),
})
type Values = z.infer<typeof schema>

export function ProfileForm() {
  const form = useForm<Values>({
    resolver: zodResolver(schema),
    defaultValues: { username: "", bio: "" },
  })

  async function onSubmit(values: Values) {
    await saveProfile(values)
  }

  return (
    // <Form> is FormProvider under the hood — spread the whole `form` object in.
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="username"
          // FormField wraps Controller — `field` is the controller field.
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                {/* FormControl injects id + aria-* so the label & error are linked. */}
                <Input placeholder="johndoe" {...field} />
              </FormControl>
              <FormDescription>Your public display name.</FormDescription>
              <FormMessage /> {/* renders errors.username.message automatically */}
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="bio"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Bio</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Saving…" : "Save profile"}
        </Button>
      </form>
    </Form>
  )
}
```

> **Why it's worth adopting:** `FormControl` + `FormLabel` + `FormMessage` generate the `id`/`htmlFor`/`aria-describedby`/`aria-invalid` wiring for you — accessibility you'd otherwise hand-write on every field (§20). Because shadcn copies the source into your project, you can read and customize it.

> **Note:** `FormField` always uses `Controller` internally, so even native inputs go through the controlled path here. For typical forms that's fine; for extreme-performance forms with hundreds of fields, plain `register` is lighter (§19).

---

## 16. Next.js: client forms, Server Actions & React 19

### 16.1 RHF needs `"use client"` **[A]**

All RHF hooks run on the client. In the App Router, the file using `useForm` must start with `"use client"`. Keep the *page* a Server Component and isolate the form in a client component.

```tsx
// app/login/page.tsx — Server Component (no "use client")
import { LoginForm } from "./login-form"
export default function Page() {
  return (
    <main>
      <h1>Sign in</h1>
      <LoginForm /> {/* the interactive client island */}
    </main>
  )
}
```

```tsx
// app/login/login-form.tsx — Client Component
"use client"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { loginSchema, type LoginInput } from "@/lib/schemas/auth"

export function LoginForm() {
  const { register, handleSubmit, setError, formState } = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
  })
  // ...
}
```

### 16.2 Calling a Server Action from `handleSubmit`

The most ergonomic 2026 pattern: RHF validates on the client for instant UX, then you call a typed **Server Action** from inside `onSubmit`. The action re-validates with the *same Zod schema* (never trust the client) and runs your mutation.

```tsx
// app/posts/actions.ts — runs on the server
"use server"
import { postSchema } from "@/lib/schemas/post"
import { db } from "@/lib/db"

export async function createPostAction(data: unknown) {
  // Re-validate on the server with the shared schema — security boundary.
  const parsed = postSchema.safeParse(data)
  if (!parsed.success) {
    // Return a serializable, field-keyed error shape the client can map.
    return {
      ok: false as const,
      errors: parsed.error.issues.map((i) => ({
        field: String(i.path[0]),
        message: i.message,
      })),
    }
  }
  await db.post.create({ data: parsed.data })
  return { ok: true as const }
}
```

```tsx
// app/posts/new-post-form.tsx
"use client"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { postSchema, type PostInput } from "@/lib/schemas/post"
import { createPostAction } from "./actions"

export function NewPostForm() {
  const { register, handleSubmit, setError, formState } = useForm<PostInput>({
    resolver: zodResolver(postSchema),
  })

  const onSubmit = async (data: PostInput) => {
    const result = await createPostAction(data) // call the server action directly
    if (!result.ok) {
      // Map server validation errors back onto fields.
      result.errors.forEach(({ field, message }) =>
        setError(field as keyof PostInput, { type: "server", message }),
      )
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("title")} />
      {formState.errors.title && <p role="alert">{formState.errors.title.message}</p>}
      <textarea {...register("body")} />
      {formState.errors.body && <p role="alert">{formState.errors.body.message}</p>}
      <button type="submit" disabled={formState.isSubmitting}>
        {formState.isSubmitting ? "Posting…" : "Post"}
      </button>
    </form>
  )
}
```

### 16.3 RHF vs React 19's native `<form action>`

React 19 added first-class `<form action={serverAction}>` plus `useActionState`, `useFormStatus`, and `useOptimistic`. **When to use which:**

- **Native `<form action>`** shines for simple, mostly-server-driven forms that should work **without JavaScript** (progressive enhancement). Validation typically lives in the action; feedback comes via `useActionState`.
- **React Hook Form** shines for rich client UX: instant per-field validation, `useFieldArray`, conditional fields, watched derived UI, `Controller`-wrapped design-system components. It does require JS.
- **Combine them:** use RHF for the client experience and call a Server Action from `onSubmit` (16.2). You get RHF's UX *and* the server boundary. You can even share the same Zod schema across `zodResolver`, the action, and a non-JS fallback.

```tsx
// Sketch: progressive-enhancement-friendly hybrid.
// The action works on its own (no-JS fallback); RHF enhances when JS loads.
const onSubmit = handleSubmit(async (data) => {
  const res = await createPostAction(data)
  // ...map errors with setError
})
// <form action={createPostAction} onSubmit={onSubmit}> ... </form>
```

> **⚡ Version note:** `useFormStatus` (React 19) reads the *native* form submission state of the nearest `<form action>`. It does **not** know about RHF's `isSubmitting` (RHF doesn't use the native action pipeline by default). For an RHF form, use `formState.isSubmitting` for button states; reserve `useFormStatus` for native-action forms.

---

## 17. Async & server-side validation

### 17.1 Async `validate` rule

`validate` may be `async` — RHF awaits it. **When you use it:** "is this username/email available?" checks against the server.

```tsx
<input
  {...register("username", {
    required: "Username is required",
    validate: async (value) => {
      const taken = await checkUsernameTaken(value) // RHF awaits this
      return !taken || "That username is already taken"
    },
  })}
/>
```

> **Gotcha:** with `mode: "onChange"`, an async `validate` fires on *every keystroke* — hammering your server. Use `mode: "onBlur"` or debounce (below). Watch `formState.isValidating` to show a spinner while the check runs.

### 17.2 Debounced async check

```tsx
import { useDebouncedCallback } from "use-debounce" // npm i use-debounce

function UsernameField() {
  const { register, setError, clearErrors, formState } = useFormContext()

  const check = useDebouncedCallback(async (value: string) => {
    if (!value) return
    const taken = await api.checkUsername(value)
    if (taken) setError("username", { type: "manual", message: "Already taken" })
    else clearErrors("username")
  }, 400) // wait 400ms after the user stops typing

  return (
    <>
      <input
        {...register("username")}
        onChange={(e) => check(e.target.value)} // drive the debounced check
      />
      {formState.isValidating && <span>Checking…</span>}
    </>
  )
}
```

### 17.3 Async default values (edit forms)

For edit forms you need the current data before rendering. Two approaches:

```tsx
// Option A — async defaultValues. The form shows empty, then populates on resolve.
const { register } = useForm<UserForm>({
  defaultValues: async () => {
    const u = await fetchUser(userId)        // RHF awaits this
    return { name: u.name, email: u.email, bio: u.bio ?? "" }
  },
})

// Option B — reset() after data loads (more control; pairs perfectly with TanStack Query).
const { register, reset } = useForm<UserForm>()
const { data: user } = useQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) })
useEffect(() => {
  if (user) reset({ name: user.name, email: user.email, bio: user.bio ?? "" })
}, [user, reset])
```

> **Best practice:** prefer **Option B with TanStack Query** in real apps — you control exactly when the form populates and can render loading/error states first. Or use the reactive `values` option (`useForm({ values: user })`) which resets automatically when `user` arrives.

### 17.4 Mapping server errors back to fields

```tsx
const onSubmit = async (data: Inputs) => {
  const res = await fetch("/api/signup", { method: "POST", body: JSON.stringify(data) })
  if (res.status === 422) {
    const { errors } = await res.json() // e.g. [{ field: "email", message: "In use" }]
    errors.forEach(({ field, message }: { field: keyof Inputs; message: string }) =>
      setError(field, { type: "server", message }),
    )
    setFocus(errors[0].field)             // focus the first bad field (accessibility)
    return
  }
  if (!res.ok) {
    setError("root.serverError", { type: "server", message: "Something went wrong." })
    return
  }
  reset(data) // success — make the saved values the new clean baseline
}
```

---

## 18. Inputs cookbook: files, checkboxes, radios, selects

### 18.1 File inputs **[I]**

File inputs are uncontrolled by nature (the browser won't let you set their value). RHF registers them; the value is a `FileList`.

```tsx
type Upload = { avatar: FileList; documents: FileList }

const { register, handleSubmit } = useForm<Upload>()

const onSubmit = async (data: Upload) => {
  const fd = new FormData()
  if (data.avatar?.[0]) fd.append("avatar", data.avatar[0])   // first file
  Array.from(data.documents ?? []).forEach((f) => fd.append("documents", f)) // many
  await fetch("/api/upload", { method: "POST", body: fd })
}

<input type="file" accept="image/*" {...register("avatar", { required: "Pick an image" })} />
<input type="file" multiple {...register("documents")} />
```

> **Zod tip:** validate files with `z.instanceof(FileList)` plus a `.refine` for size/type:
> ```tsx
> avatar: z.instanceof(FileList)
>   .refine((fl) => fl.length === 1, "One file required")
>   .refine((fl) => fl[0]?.size <= 2_000_000, "Max 2MB")
> ```

### 18.2 Checkboxes

```tsx
type Prefs = { notifications: boolean; interests: string[] }

useForm<Prefs>({ defaultValues: { notifications: true, interests: [] } })

// Single checkbox → boolean
<input type="checkbox" {...register("notifications")} />

// Group of checkboxes sharing one name → string[] (RHF collects checked `value`s)
<input type="checkbox" value="react" {...register("interests")} />
<input type="checkbox" value="ts"    {...register("interests")} />
<input type="checkbox" value="rust"  {...register("interests")} />
```

> **Gotcha:** never put a `value` on a boolean checkbox — it turns `data.notifications` into the string `"on"` instead of `true`. Only add `value` when grouping checkboxes into an array.

### 18.3 Radio buttons

```tsx
type PlanForm = { plan: "free" | "pro" | "enterprise" }
useForm<PlanForm>({ defaultValues: { plan: "free" } })

<label><input type="radio" value="free" {...register("plan")} /> Free</label>
<label><input type="radio" value="pro" {...register("plan")} /> Pro</label>
<label><input type="radio" value="enterprise" {...register("plan")} /> Enterprise</label>
```

### 18.4 Native multi-select

```tsx
type Inputs = { roles: string[] }
useForm<Inputs>({ defaultValues: { roles: [] } })

// RHF collects all selected option values into an array automatically.
<select multiple {...register("roles")}>
  <option value="admin">Admin</option>
  <option value="editor">Editor</option>
  <option value="viewer">Viewer</option>
</select>
```

### 18.5 `react-select` (controlled) via `Controller`

```tsx
import Select from "react-select"
import { Controller } from "react-hook-form"

type Option = { value: string; label: string }
type Inputs = { tags: Option[] }

<Controller
  name="tags"
  control={control}
  render={({ field }) => (
    <Select {...field} isMulti options={tagOptions} />
  )}
/>
// data.tags is Option[]; extract values with data.tags.map(t => t.value) if needed.
```

---

## 19. Performance best practices

RHF is fast by default, but large forms still benefit from deliberate patterns.

### 19.1 Why it's fast (recap)

1. **Uncontrolled inputs** — the DOM holds values; no `setState` per keystroke.
2. **Proxy-based `formState`** — reading `formState.errors` subscribes you to *errors only*, not all form changes.
3. **Surgical re-renders** — a field's error display re-renders when *that* field's error changes, not the whole form.

### 19.2 Isolate derived UI with `useWatch`

```tsx
// ❌ watch() in the parent re-renders the WHOLE form on every change.
function OrderForm() {
  const { watch } = useForm()
  const total = watch("total")  // parent re-renders constantly
  return <><LineItems /><p>Total: {total}</p></>
}

// ✅ Move the subscription into a child — only the child re-renders.
function TotalDisplay({ control }: { control: Control<Order> }) {
  const total = useWatch({ control, name: "total" })
  return <p>Total: {total}</p>
}
```

### 19.3 Prefer `register` over `Controller` for native inputs

```tsx
// ❌ Unnecessary overhead for a plain input:
<Controller name="email" control={control} render={({ field }) => <input {...field} />} />
// ✅ Lighter:
<input {...register("email")} />
```

### 19.4 Other levers

- **Don't `watch()` everything.** `watch()` with no args subscribes to all fields. Watch specific paths, or use `getValues()` for a one-time read.
- **`React.memo` heavy sections** that don't depend on changing form state.
- **`shouldUnregister: false`** (default) avoids remount churn for conditional fields.
- **`delayError`** smooths error flicker on fast typing.
- **Measure, don't guess.** Drop a render counter in suspect components, or use the React DevTools Profiler.

```tsx
// Quick render counter for debugging:
const renders = useRef(0); renders.current++
console.log("renders:", renders.current)
```

---

## 20. Accessibility

Forms are where accessibility most often breaks. RHF doesn't make forms accessible *for* you (unless you use the shadcn wrapper), so wire these up yourself.

**The essentials:**

1. **Associate a label with every control** via `htmlFor`/`id` (or wrap the input in `<label>`).
2. **Mark invalid fields** with `aria-invalid` so assistive tech announces the state.
3. **Announce errors** with `role="alert"` (or `aria-live`) so they're read out when they appear.
4. **Link the error text to the input** with `aria-describedby` so a screen-reader user hears the message when focused on the field.
5. **Focus the first error on failed submit** — `shouldFocusError: true` (the default) does this; keep it on.

```tsx
function AccessibleField() {
  const { register, formState: { errors } } = useFormContext<Inputs>()
  const id = "email"
  const errId = "email-error"
  return (
    <div>
      <label htmlFor={id}>Email</label>
      <input
        id={id}
        type="email"
        {...register("email", { required: "Email is required" })}
        aria-invalid={errors.email ? "true" : "false"}     // announce invalid state
        aria-describedby={errors.email ? errId : undefined} // link to the message
      />
      {errors.email && (
        <p id={errId} role="alert">{errors.email.message}</p> // announced on appearance
      )}
    </div>
  )
}
```

> **Best practice:** if you're on shadcn/ui, `FormControl`/`FormMessage`/`FormLabel` generate all of this (`id`, `htmlFor`, `aria-describedby`, `aria-invalid`) automatically (§15) — strong reason to adopt it. Also add `noValidate` to `<form>` so the browser's native validation bubbles don't compete with your RHF/ARIA messages.

---

## 21. Tips, tricks & gotchas

**1. Spread `register` — don't destructure away the ref.**
```tsx
<input {...register("email")} />                       // ✅
const { onChange } = register("email")                 // ❌ loses ref → no focus-on-error, no read
```

**2. Use `defaultValues`, not `value`/`defaultValue` on the input.**
```tsx
useForm({ defaultValues: { name: "Jane" } })           // ✅ the RHF way; sets the dirty baseline
<input {...register("name")} value={name} />           // ❌ makes it controlled → conflicts
<input {...register("name")} defaultValue="Jane" />    // ❌ RHF won't track clean/dirty correctly
```

**3. The controlled/uncontrolled warning** ("changing an uncontrolled input to be controlled") means you mixed `register` with `value`. Use either `register` + `defaultValues` *or* `Controller` + `value` — never both on one field.

**4. Coerce numbers.** `register("age")` gives `"25"` (string). Use `valueAsNumber` or `z.coerce.number()`.

**5. `reset()` timing with async data.** Don't call `reset(data)` before data exists; do it in an effect gated on the data, or use the `values` option.
```tsx
useEffect(() => { if (data) reset(data) }, [data, reset]) // ✅
```

**6. `useFieldArray`: key on `field.id`, never the index** — index keys corrupt state on reorder/remove.

**7. `isValid` is `false` before the first validation when `mode: "onSubmit"`.** Don't gate the submit button on `!isValid` with the default mode or it's permanently disabled. Use `isSubmitting` to prevent double-submit, or switch to `mode: "onTouched"`/`"onChange"`.

**8. Don't mix `resolver` with inline `register` rules** — when a resolver is set, inline rules are silently ignored. Put everything in the schema.

**9. Errors clear after a successful submit by default.** For a success message use `formState.isSubmitSuccessful`, not the absence of errors.

**10. `watch()` with no args is expensive** in big forms — it re-renders on every field change. Watch specific paths or move to `useWatch` in a child.

**11. `setValue` doesn't validate or dirty by default** — pass `{ shouldValidate: true, shouldDirty: true }` when you need those effects.

**12. `field.id` from `useFieldArray` is not form data** — it won't be submitted; add your own id field if you need one.

**13. `useFormContext()` throws/returns null outside a `FormProvider`** — guard shared components.

**14. After a successful save, `reset(savedData)`** to make the saved values the new clean baseline, otherwise the form stays `isDirty`.

**15. Server-side re-validate.** RHF validates on the client for UX only. Always re-validate on the server/Server Action with the same schema — clients are untrusted (§16).

**16. Empty `valueAsNumber` field is `NaN`, not 0.** Handle empty inputs in your schema for optional numbers (`z.coerce.number().optional()` or allow empty string).

---

## 22. Study Path & Build-to-Learn Projects

Work through these in order; each builds on the last. After all of them you can build any real-world form from memory.

### Phase 1 — Core API (1–2 days) [B]
1. **First form** (§3, §4, §5): build a login form with `register`, `handleSubmit`, and `defaultValues`. Log the submitted data.
2. **Built-in validation** (§8): add `required`, `minLength`, `pattern`; render `errors.field.message`.
3. **`formState`** (§6): wire `isSubmitting` to the button; watch `isDirty` flip as you type; show `isSubmitSuccessful`.
4. **Modes** (§7): switch between `onSubmit`/`onBlur`/`onTouched` and feel the UX difference.

**Build:** a registration form (name, email, password, confirm, terms) with inline rules and live errors.

### Phase 2 — Schema & TypeScript (1 day) [I]
5. **Zod resolver** (§9): rewrite the registration form with a schema; use `z.infer` as the form type.
6. **Cross-field rules** (§9): add `.refine()` for password confirmation; move the schema to `lib/schemas/`.
7. **Edit form** (§17): fetch a user and populate via `reset()` (or the `values` option).

**Build:** the registration form in Zod, plus an edit-profile form that loads from a fake API.

### Phase 3 — Complex patterns (2–3 days) [I/A]
8. **`Controller`** (§10): wrap `react-select` and a date picker; compare `field` to `register`.
9. **`useFieldArray`** (§12): build an invoice with add/remove rows and a `useWatch` total.
10. **Nested objects** (§14): checkout form with `shippingAddress`/`billingAddress` and a "same as" copy.
11. **`FormProvider`** (§14): split the checkout into per-section components using `useFormContext`.

**Build:** a multi-section checkout with dynamic line items and zero prop-drilling.

### Phase 4 — Real-world integration (2 days) [A]
12. **Server errors** (§17): simulate a 422; map with `setError`; `setFocus` the first bad field.
13. **shadcn/ui** (§15): rebuild a form with `<Form>`/`<FormField>`/`<FormMessage>`; note the free a11y.
14. **Server Actions** (§16): call a Server Action from `handleSubmit`; re-validate server-side with the shared schema.
15. **File uploads** (§18): add an avatar upload; build `FormData` in `onSubmit`; validate files with Zod.

**Build:** a Next.js app with product edit forms (TanStack Query + `reset()`), submitting to Server Actions with dual Zod validation.

### Phase 5 — Performance & mastery (1 day) [A]
16. **`useWatch`** (§19): move a derived total into a child; confirm in the Profiler that the parent stops re-rendering.
17. **Gotchas review** (§21): reproduce and fix each gotcha to lock in the mental model.

### Projects that cement mastery

| Project | Key skills |
|---|---|
| Login / signup flow | `register`, Zod, `setError` for server errors |
| Profile edit | async defaults + `reset()`, file upload, `dirtyFields` PATCH |
| Invoice builder | `useFieldArray`, `useWatch` totals |
| Multi-step wizard | `FormProvider`, `trigger` per step, nested objects |
| Job application | files, multi-selects, conditional fields, `shouldUnregister` |
| Settings page | multiple independent forms, `resetField`, patch-on-blur |

### Cross-references in this library
- **React 19** guide — `useActionState`, `useFormStatus`, `<form action>` (pairs with §16).
- **Zod** guide — schema design, `.refine`, transforms, `z.infer` (powers §9).
- **shadcn/ui** guide — the `Form` component internals (§15).
- **Next.js** guide — Server Actions, RSC boundaries, `"use client"` (§16).

### Key references to bookmark
- React Hook Form: [react-hook-form.com](https://react-hook-form.com) — API and "Advanced Usage".
- Resolvers: [github.com/react-hook-form/resolvers](https://github.com/react-hook-form/resolvers).
- Zod: [zod.dev](https://zod.dev).
- shadcn Form: [ui.shadcn.com/docs/components/form](https://ui.shadcn.com/docs/components/form).

> **Final tip:** open the React DevTools **Profiler**, record a typing session, and inspect which components re-rendered and why. Seeing uncontrolled inputs *not* re-render — and watching `useWatch`/`React.memo` shrink the render tree — turns the performance theory in this guide into intuition you'll carry to every form you build.
