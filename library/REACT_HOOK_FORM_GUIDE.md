# React Hook Form v7 Complete Guide

A full, study-friendly reference for **React Hook Form v7** — the standard library for building performant, flexible forms in React apps. Includes setup, every core API, validation, schema resolvers, dynamic fields, Next.js integration, and real-world patterns.

> **What it is:** React Hook Form manages **form state** using uncontrolled inputs and native HTML refs — so your components don't re-render on every keystroke. It gives you validation, error messages, dynamic fields, and full TypeScript support with a minimal API surface. It is faster than Formik and far lighter than Redux Form. Pair it with Zod for end-to-end type-safe validation.

---

## Table of Contents
1. [What React Hook Form Is & Why Use It](#1-what-react-hook-form-is--why-use-it)
2. [Installation & Setup](#2-installation--setup)
3. [useForm Basics](#3-useform-basics)
4. [The register API in Depth](#4-the-register-api-in-depth)
5. [Reading Errors & formState Flags](#5-reading-errors--formstate-flags)
6. [Validation Modes](#6-validation-modes)
7. [Controller — Controlled & Third-Party Components](#7-controller--controlled--third-party-components)
8. [Schema Validation with Resolvers (Zod)](#8-schema-validation-with-resolvers-zod)
9. [Utility Methods: watch, getValues, setValue, reset, trigger, setError & More](#9-utility-methods)
10. [Dynamic Fields with useFieldArray](#10-dynamic-fields-with-usefieldarray)
11. [Nested Fields & Complex Objects](#11-nested-fields--complex-objects)
12. [FormProvider & useFormContext](#12-formprovider--useformcontext)
13. [Async Validation & Async Default Values](#13-async-validation--async-default-values)
14. [Handling Submission](#14-handling-submission)
15. [Integration with shadcn/ui & Next.js](#15-integration-with-shadcnui--nextjs)
16. [File Inputs, Checkboxes, Radios & Multi-Select](#16-file-inputs-checkboxes-radios--multi-select)
17. [Performance Notes](#17-performance-notes)
18. [Tips, Tricks & Gotchas](#18-tips-tricks--gotchas)
19. [Study Path](#19-study-path)

---

## 1. What React Hook Form Is & Why Use It

### The problem with controlled forms

Every controlled form input looks like this:

```tsx
// ❌ Controlled pattern — a re-render fires on EVERY keystroke
function ControlledForm() {
  const [name, setName] = useState("")
  const [email, setEmail] = useState("")
  return (
    <form>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
    </form>
  )
}
```

Each `onChange` calls `setName`, which triggers a full component re-render. With 10 fields and a complex form, that's thousands of re-renders per minute. Formik uses this controlled pattern and suffers from the same overhead.

### How RHF solves it

React Hook Form registers inputs by attaching a **ref** to each one. The DOM itself holds the value — React is not involved until you submit. The result: **zero re-renders during typing**.

```tsx
// ✅ Uncontrolled pattern — no re-renders while typing
function RHFForm() {
  const { register, handleSubmit } = useForm()
  return (
    <form onSubmit={handleSubmit(data => console.log(data))}>
      <input {...register("name")} />
      <input {...register("email")} />
      <button type="submit">Submit</button>
    </form>
  )
}
```

### RHF vs Formik vs controlled state

| | React Hook Form v7 | Formik | Plain useState |
|---|---|---|---|
| **Re-renders on type** | None (uncontrolled) | One per field per keystroke | Yes |
| **Bundle size** | ~9 KB gzipped | ~15 KB gzipped | 0 |
| **TypeScript** | Excellent (generics) | Decent | Manual |
| **Schema validation** | Via resolvers (Zod, Yup…) | Built-in Yup | Manual |
| **Dynamic fields** | `useFieldArray` (built-in) | `FieldArray` (verbose) | Manual |
| **3rd-party inputs** | `Controller` | `Field` | Wrap manually |
| **Learning curve** | Low | Moderate | None |
| **Actively maintained** | Yes (2026) | Slower cadence | — |

> **Bottom line:** Use React Hook Form for any real form. Use Zod as the validation layer. Use `Controller` only when an input truly needs to be controlled (custom UI component, date picker, etc.).

---

## 2. Installation & Setup

```bash
# Core library — this is all you need to get started
npm install react-hook-form

# Schema validation resolver (install the companion package)
npm install @hookform/resolvers

# Popular schema libraries (pick one or more)
npm install zod          # recommended — best TypeScript integration
npm install yup          # Formik-era classic
npm install valibot      # tiny modern alternative
```

No provider, no context, no global setup required. `useForm` is self-contained. The form state lives inside the hook.

⚡ **Version note:** This guide targets **React Hook Form v7** (released 2021, still the current major as of 2026). v7 requires React 16.8+. The v6 → v7 migration changed several APIs (`register` now returns props instead of a ref callback). Always check you are on v7 when reading external tutorials.

---

## 3. useForm Basics

`useForm` is the entry point. It returns all the methods you need.

```tsx
"use client" // needed in Next.js App Router

import { useForm, SubmitHandler } from "react-hook-form"

// 1. Define the shape of your form data
type LoginInputs = {
  email: string
  password: string
  rememberMe: boolean
}

export function LoginForm() {
  // 2. Call useForm — pass your type as a generic
  const {
    register,       // connects inputs to RHF
    handleSubmit,   // wraps your submit handler
    formState: { errors, isSubmitting },
  } = useForm<LoginInputs>({
    // 3. Provide default values — RHF uses these to determine "dirty" state
    defaultValues: {
      email: "",
      password: "",
      rememberMe: false,
    },
  })

  // 4. Your submit handler receives a fully-typed data object
  const onSubmit: SubmitHandler<LoginInputs> = async (data) => {
    await loginUser(data) // data.email, data.password, data.rememberMe
  }

  return (
    // 5. Pass handleSubmit your handler — it validates before calling it
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        type="email"
        placeholder="Email"
        {...register("email", { required: "Email is required" })}
      />
      {errors.email && <p>{errors.email.message}</p>}

      <input
        type="password"
        placeholder="Password"
        {...register("password", { required: "Password is required" })}
      />
      {errors.password && <p>{errors.password.message}</p>}

      <input type="checkbox" {...register("rememberMe")} />

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Logging in…" : "Log in"}
      </button>
    </form>
  )
}
```

### useForm options reference

| Option | Default | Purpose |
|---|---|---|
| `defaultValues` | `{}` | Initial field values. Controls dirty state. Accepts a plain object or an async function (see §13). |
| `mode` | `"onSubmit"` | When validation runs. See §6. |
| `reValidateMode` | `"onChange"` | When to re-validate after first submission attempt. |
| `resolver` | `undefined` | Schema resolver (Zod, Yup…). Replaces inline rules when set. See §8. |
| `criteriaMode` | `"firstError"` | `"all"` collects all errors per field instead of stopping at the first. |
| `shouldFocusError` | `true` | Auto-focus the first invalid field on failed submit. |
| `disabled` | `false` | Disables all registered inputs. |
| `values` | — | Reactive external values that sync into the form (for edit forms driven by fetched data). |
| `resetOptions` | — | Passed to `reset` whenever `values` changes. |

---

## 4. The register API in Depth

`register(name, options?)` returns four props that you spread onto a native input:

```tsx
// What register returns under the hood
{
  name: string       // the field name
  ref: RefCallback   // connects the DOM element to RHF
  onChange: Handler  // notifies RHF of changes (for re-validation)
  onBlur: Handler    // notifies RHF on blur (for re-validation)
}

// Usage — always spread the return value
<input {...register("fieldName", options)} />
```

### Validation rules

All validation options live in the second argument to `register`:

```tsx
<input
  {...register("username", {
    // --- Required ---
    required: "Username is required",          // string = the error message
    // OR: required: true (no message)
    // OR: required: { value: true, message: "Required" }

    // --- String length ---
    minLength: { value: 3, message: "At least 3 characters" },
    maxLength: { value: 20, message: "Max 20 characters" },

    // --- Numeric bounds ---
    min: { value: 1, message: "Must be at least 1" },
    max: { value: 100, message: "Cannot exceed 100" },

    // --- Pattern (regex) ---
    pattern: {
      value: /^[A-Za-z0-9_]+$/,
      message: "Letters, numbers and underscores only",
    },

    // --- Custom validate ---
    validate: (value) => value !== "admin" || "Username 'admin' is reserved",
    // OR multiple validators:
    validate: {
      notAdmin: (v) => v !== "admin" || "Reserved username",
      notEmpty: (v) => v.trim().length > 0 || "Cannot be blank",
    },
  })}
/>
```

### Type coercion options

HTML inputs always return strings. Use these to get the right JS type:

```tsx
// valueAsNumber — converts the string to a Number before handing to submit
<input
  type="number"
  {...register("age", {
    valueAsNumber: true,
    required: "Age is required",
    min: { value: 0, message: "Must be positive" },
  })}
/>

// valueAsDate — converts the string to a Date object
<input
  type="date"
  {...register("birthDate", { valueAsDate: true })}
/>

// setValueAs — custom transform
<input
  {...register("code", {
    setValueAs: (v) => v.trim().toUpperCase(),
  })}
/>
```

### Registering without spreading (ref forwarding)

Sometimes you need to attach the ref separately (e.g. when a component doesn't accept `...rest` props):

```tsx
const { ref, ...rest } = register("email")
<CustomInput {...rest} inputRef={ref} />
```

### Unregistering

By default, when a registered input unmounts, its value is kept in the form state. To clear it:

```tsx
useForm({ shouldUnregister: true }) // global
register("field", { shouldUnregister: true }) // per-field
```

---

## 5. Reading Errors & formState Flags

`formState` is a reactive object containing all metadata about the form's current state. Destructure only what you need — RHF uses proxies to avoid unnecessary re-renders.

```tsx
const {
  formState: {
    errors,          // field error objects
    isDirty,         // true if ANY field differs from defaultValues
    dirtyFields,     // object: { fieldName: true } for changed fields
    isValid,         // true if no validation errors exist
    isSubmitting,    // true while the async submit handler is running
    isSubmitted,     // true after the first submit attempt
    isSubmitSuccessful, // true if submit completed without error
    touchedFields,   // object: { fieldName: true } for blurred fields
    submitCount,     // number of times form has been submitted
  }
} = useForm()
```

### Displaying error messages

```tsx
// Pattern 1: simple conditional
{errors.email && <p className="error">{errors.email.message}</p>}

// Pattern 2: optional chaining (safer)
<p>{errors.email?.message}</p>

// Pattern 3: error type discrimination
{errors.username?.type === "minLength" && <p>Too short</p>}
{errors.username?.type === "pattern" && <p>Invalid characters</p>}
```

### formState flags in practice

```tsx
function MyForm() {
  const { register, handleSubmit, formState } = useForm<Inputs>({
    defaultValues: { name: "" },
    mode: "onTouched",
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name", { required: true })} />

      {/* Show save button only when something changed */}
      <button type="submit" disabled={!formState.isDirty || formState.isSubmitting}>
        {formState.isSubmitting ? "Saving…" : "Save"}
      </button>

      {/* Success message */}
      {formState.isSubmitSuccessful && <p>Saved successfully!</p>}

      {/* Submission count */}
      <small>Submitted {formState.submitCount} time(s)</small>
    </form>
  )
}
```

### criteriaMode: "all" — collecting multiple errors per field

```tsx
useForm({
  criteriaMode: "all",  // default is "firstError"
})

// Now errors.fieldName.types contains ALL failed rules:
// errors.fieldName.types.required
// errors.fieldName.types.minLength
// errors.fieldName.types.pattern
```

---

## 6. Validation Modes

The `mode` option controls **when validation runs** — before the first submission and during user interaction.

| Mode | Validates on… | Best for |
|---|---|---|
| `"onSubmit"` (default) | Submit click only | Simple forms, lowest re-render count |
| `"onBlur"` | Field blur (leaving the field) | Good UX default — not too eager |
| `"onChange"` | Every keystroke | Real-time feedback, more re-renders |
| `"onTouched"` | First blur, then on change after | Best general-purpose UX |
| `"all"` | Both blur and change | Most aggressive |

```tsx
useForm<Inputs>({
  mode: "onTouched",        // trigger after field is touched
  reValidateMode: "onChange", // after first submit: revalidate on every change
})
```

### Re-validation mode

`reValidateMode` only applies **after the first submit attempt**. The default `"onChange"` means once you've submitted once and have errors, they clear in real time as you fix them — which is good UX.

```tsx
// Good UX combination:
useForm({
  mode: "onBlur",           // validate each field when user leaves it
  reValidateMode: "onChange", // after submit: fix errors in real time
})
```

### Triggering validation manually

```tsx
const { trigger } = useForm()

// Validate one field
await trigger("email")

// Validate multiple fields
await trigger(["email", "password"])

// Validate all fields
await trigger()
```

---

## 7. Controller — Controlled & Third-Party Components

Most native HTML inputs work fine with `register`. But some components — MUI inputs, shadcn Select, react-select, date pickers, rich text editors — need to control their own internal state. For these, use `Controller`.

`Controller` wraps your component and bridges RHF's uncontrolled model to the component's controlled API.

```tsx
import { useForm, Controller } from "react-hook-form"

type Inputs = {
  country: string
  birthDate: Date | null
}

function ProfileForm() {
  const { control, handleSubmit } = useForm<Inputs>({
    defaultValues: { country: "", birthDate: null },
  })

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {/* Example: react-select */}
      <Controller
        name="country"
        control={control}                          // pass the control object
        rules={{ required: "Country is required" }}
        render={({ field, fieldState }) => (
          <ReactSelect
            {...field}                             // value, onChange, onBlur, name, ref
            options={countryOptions}
            placeholder="Select country"
            className={fieldState.error ? "error" : ""}
          />
        )}
      />

      {/* Example: a date picker component */}
      <Controller
        name="birthDate"
        control={control}
        rules={{ required: "Birth date required" }}
        render={({ field }) => (
          <DatePicker
            selected={field.value}
            onChange={field.onChange}         // RHF gets notified of changes
            onBlur={field.onBlur}
            ref={field.ref}
          />
        )}
      />
    </form>
  )
}
```

### Controller render prop arguments

| Argument | Contents |
|---|---|
| `field.value` | Current field value |
| `field.onChange` | Call with new value to update RHF |
| `field.onBlur` | Call when the input loses focus |
| `field.name` | The field name (for accessibility) |
| `field.ref` | Ref to focus the element on validation error |
| `fieldState.error` | Error object (same as `errors.fieldName`) |
| `fieldState.isDirty` | Whether this field is dirty |
| `fieldState.isTouched` | Whether this field has been touched |
| `formState` | Full formState (same as from `useForm`) |

### useController hook

`useController` is the hook equivalent of `Controller` — useful when building reusable input components:

```tsx
import { useController, UseControllerProps } from "react-hook-form"

// A reusable controlled input that plugs into any RHF form
function FormInput({ name, control, rules, label }: UseControllerProps & { label: string }) {
  const {
    field,
    fieldState: { error },
  } = useController({ name, control, rules })

  return (
    <div>
      <label>{label}</label>
      <input {...field} />
      {error && <p className="error">{error.message}</p>}
    </div>
  )
}

// Usage
<FormInput name="username" control={control} label="Username" rules={{ required: true }} />
```

### MUI TextField example

```tsx
<Controller
  name="firstName"
  control={control}
  rules={{ required: "Required" }}
  render={({ field, fieldState }) => (
    <TextField
      {...field}
      label="First Name"
      error={!!fieldState.error}
      helperText={fieldState.error?.message}
    />
  )}
/>
```

---

## 8. Schema Validation with Resolvers (Zod)

Inline `register` validation rules work fine for simple forms. For complex forms, use a **schema validator** — it centralises all rules, gives you better TypeScript inference, and reuses the same schema for API validation.

### Installation

```bash
npm install zod @hookform/resolvers
```

### Full Zod example

```tsx
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

// 1. Define your schema
const signupSchema = z
  .object({
    username: z
      .string()
      .min(3, "At least 3 characters")
      .max(20, "Max 20 characters")
      .regex(/^[a-zA-Z0-9_]+$/, "Letters, numbers and underscores only"),
    email: z.string().email("Invalid email address"),
    password: z
      .string()
      .min(8, "Password must be at least 8 characters")
      .regex(/[A-Z]/, "Must contain an uppercase letter")
      .regex(/[0-9]/, "Must contain a number"),
    confirmPassword: z.string(),
    age: z.number({ invalid_type_error: "Age must be a number" }).min(18, "Must be 18+"),
    role: z.enum(["admin", "editor", "viewer"]),
    acceptTerms: z.literal(true, {
      errorMap: () => ({ message: "You must accept the terms" }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    path: ["confirmPassword"],     // which field the error appears on
    message: "Passwords do not match",
  })

// 2. Infer the TypeScript type from the schema — the single source of truth
type SignupInputs = z.infer<typeof signupSchema>

// 3. Wire it up
function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<SignupInputs>({
    resolver: zodResolver(signupSchema),  // all validation goes through Zod
    defaultValues: {
      username: "",
      email: "",
      password: "",
      confirmPassword: "",
      role: "viewer",
      acceptTerms: true,
    },
  })

  const onSubmit = async (data: SignupInputs) => {
    // data is fully typed — SignupInputs
    await createUser(data)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("username")} placeholder="Username" />
      {errors.username && <p>{errors.username.message}</p>}

      <input {...register("email")} type="email" placeholder="Email" />
      {errors.email && <p>{errors.email.message}</p>}

      <input {...register("password")} type="password" />
      {errors.password && <p>{errors.password.message}</p>}

      <input {...register("confirmPassword")} type="password" placeholder="Confirm" />
      {errors.confirmPassword && <p>{errors.confirmPassword.message}</p>}

      {/* valueAsNumber not needed when using zodResolver — Zod handles coercion */}
      <input {...register("age", { valueAsNumber: true })} type="number" />
      {errors.age && <p>{errors.age.message}</p>}

      <select {...register("role")}>
        <option value="viewer">Viewer</option>
        <option value="editor">Editor</option>
        <option value="admin">Admin</option>
      </select>

      <input type="checkbox" {...register("acceptTerms")} />
      {errors.acceptTerms && <p>{errors.acceptTerms.message}</p>}

      <button disabled={isSubmitting}>Sign Up</button>
    </form>
  )
}
```

⚡ **Version note:** `zodResolver` from `@hookform/resolvers/zod` is compatible with Zod v3 (2021–) and Zod v4 (released 2025). If you're on Zod v4, import from `@hookform/resolvers/zod` — the API is the same, but Zod v4 has faster parsing and a revised `.email()` default. Check the resolvers README for any version-specific compatibility notes.

### Yup example (brief)

```tsx
import { yupResolver } from "@hookform/resolvers/yup"
import * as yup from "yup"

const schema = yup.object({ name: yup.string().required() })
type Inputs = yup.InferType<typeof schema>

useForm<Inputs>({ resolver: yupResolver(schema) })
```

### Valibot example (brief)

```tsx
import { valibotResolver } from "@hookform/resolvers/valibot"
import * as v from "valibot"

const schema = v.object({ name: v.pipe(v.string(), v.minLength(1)) })
type Inputs = v.InferOutput<typeof schema>

useForm<Inputs>({ resolver: valibotResolver(schema) })
```

> **Sharing the schema type:** with Zod, `z.infer<typeof schema>` is your form type AND your API type. Define the schema once in a shared file (e.g. `lib/schemas/user.ts`) and import it in both your form component and your API route handler for full stack type safety.

---

## 9. Utility Methods

### watch — subscribe to field values (causes re-renders)

```tsx
const { watch, register } = useForm<Inputs>()

// Watch a single field — re-renders on every change to "country"
const country = watch("country")

// Watch multiple fields
const [firstName, lastName] = watch(["firstName", "lastName"])

// Watch all fields — returns the entire form data object
const allValues = watch()

// Subscribe with a callback (no re-render in the component itself)
useEffect(() => {
  const subscription = watch((value, { name, type }) => {
    console.log("Changed:", name, value)
  })
  return () => subscription.unsubscribe()
}, [watch])
```

### useWatch — optimised watching (isolates re-renders to the subscriber)

```tsx
import { useWatch } from "react-hook-form"

function ShippingCostDisplay({ control }: { control: Control<Inputs> }) {
  // Only THIS component re-renders when "weight" changes
  // The parent form component is untouched
  const weight = useWatch({ control, name: "weight", defaultValue: 0 })
  return <p>Estimated shipping: ${(weight * 0.5).toFixed(2)}</p>
}
```

### getValues — read without subscribing

```tsx
const { getValues } = useForm()

// No re-render — just reads the current value synchronously
const email = getValues("email")
const all = getValues()           // all fields
const subset = getValues(["email", "name"])
```

### setValue — programmatically update a field

```tsx
const { setValue } = useForm()

// Basic
setValue("email", "user@example.com")

// With options
setValue("email", "user@example.com", {
  shouldDirty: true,    // mark as dirty
  shouldTouch: true,    // mark as touched
  shouldValidate: true, // run validation immediately
})
```

### reset — reset the form

```tsx
const { reset } = useForm()

// Reset to original defaultValues
reset()

// Reset to specific values (useful after a successful save)
reset({ name: "Jane", email: "jane@example.com" })

// Reset with options
reset(undefined, {
  keepErrors: false,
  keepDirty: false,
  keepValues: false,
  keepTouched: false,
  keepIsSubmitted: false,
  keepSubmitCount: false,
})
```

### resetField — reset a single field

```tsx
const { resetField } = useForm()

// Reset "email" to its defaultValue, clear dirty/touched/error state
resetField("email")

// Reset to a specific value
resetField("email", { defaultValue: "new@default.com" })

// Keep dirty/touched state but clear errors
resetField("email", { keepError: false, keepDirty: true })
```

### trigger — manually run validation

```tsx
const { trigger } = useForm()

// Validate one field, returns true/false
const isEmailValid = await trigger("email")

// Validate multiple
await trigger(["email", "password"])

// Validate all
const isFormValid = await trigger()
```

### setError — set a server-side or manual error

```tsx
const { setError } = useForm()

// Set an error on a specific field (e.g. after a server 422 response)
setError("email", {
  type: "manual",
  message: "This email is already registered",
})

// Set a non-field error (root-level)
setError("root.serverError", {
  type: "server",
  message: "Something went wrong on the server",
})

// errors.root?.serverError?.message — access it like any other error
```

### clearErrors

```tsx
const { clearErrors } = useForm()

clearErrors("email")          // clear one
clearErrors(["email", "name"]) // clear several
clearErrors()                  // clear all
```

### setFocus

```tsx
const { setFocus } = useForm()

// Focus a field programmatically (e.g. after an async error response)
setFocus("email")

// Focus and select all text
setFocus("email", { shouldSelect: true })
```

---

## 10. Dynamic Fields with useFieldArray

`useFieldArray` manages arrays of fields — think line items in an invoice, multiple phone numbers, a list of tasks, or a repeating section.

```tsx
import { useForm, useFieldArray, SubmitHandler } from "react-hook-form"

type LineItem = {
  name: string
  quantity: number
  price: number
}

type InvoiceForm = {
  clientName: string
  items: LineItem[]
}

function InvoiceForm() {
  const {
    register,
    control,
    handleSubmit,
    watch,
    formState: { errors },
  } = useForm<InvoiceForm>({
    defaultValues: {
      clientName: "",
      items: [{ name: "", quantity: 1, price: 0 }], // start with one row
    },
  })

  const { fields, append, remove, insert, move, replace, prepend } = useFieldArray({
    control,           // required — links to the form
    name: "items",     // path to the array in your form data
  })

  const onSubmit: SubmitHandler<InvoiceForm> = (data) => {
    console.log(data.items) // fully typed LineItem[]
  }

  // Compute totals reactively
  const items = watch("items")
  const total = items.reduce((sum, item) => sum + item.quantity * item.price, 0)

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("clientName", { required: true })} placeholder="Client" />

      <table>
        <thead>
          <tr>
            <th>Item</th><th>Qty</th><th>Price</th><th></th>
          </tr>
        </thead>
        <tbody>
          {fields.map((field, index) => (
            // IMPORTANT: use field.id (not index) as the React key
            <tr key={field.id}>
              <td>
                <input
                  {...register(`items.${index}.name`, { required: "Item name required" })}
                  placeholder="Item name"
                />
                {errors.items?.[index]?.name && (
                  <p>{errors.items[index].name.message}</p>
                )}
              </td>
              <td>
                <input
                  type="number"
                  {...register(`items.${index}.quantity`, {
                    valueAsNumber: true,
                    min: { value: 1, message: "Min 1" },
                  })}
                />
              </td>
              <td>
                <input
                  type="number"
                  step="0.01"
                  {...register(`items.${index}.price`, { valueAsNumber: true })}
                />
              </td>
              <td>
                {/* Don't let the user remove the last row */}
                <button
                  type="button"
                  onClick={() => remove(index)}
                  disabled={fields.length === 1}
                >
                  Remove
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>

      <p>Total: ${total.toFixed(2)}</p>

      <button
        type="button"
        onClick={() => append({ name: "", quantity: 1, price: 0 })}
      >
        + Add Item
      </button>

      <button type="submit">Save Invoice</button>
    </form>
  )
}
```

### useFieldArray methods

| Method | Signature | What it does |
|---|---|---|
| `append` | `append(value, options?)` | Add one or more items to the end |
| `prepend` | `prepend(value, options?)` | Add one or more items to the beginning |
| `insert` | `insert(index, value)` | Insert at a specific position |
| `remove` | `remove(index?)` | Remove by index (or all if omitted) |
| `move` | `move(from, to)` | Reorder — drag and drop |
| `swap` | `swap(indexA, indexB)` | Swap two items |
| `replace` | `replace(items)` | Replace the entire array |
| `update` | `update(index, value)` | Replace one item at an index |

### Key rules for useFieldArray

```tsx
// ✅ Always use field.id as the React key — NOT the index
fields.map((field, index) => <div key={field.id}>…</div>)

// ✅ Use template literal names for nested fields
register(`items.${index}.name`)

// ✅ Provide defaultValues for the array — otherwise fields are uncontrolled
defaultValues: { items: [{ name: "" }] }

// ❌ Don't use index as key — reordering will corrupt state
fields.map((_, index) => <div key={index}>…</div>)
```

### Nested useFieldArray (arrays within arrays)

```tsx
type Form = {
  sections: Array<{
    title: string
    questions: Array<{ text: string }>
  }>
}

// In the inner component, create a nested field array
function SectionRow({ sectionIndex, control, register }) {
  const { fields, append, remove } = useFieldArray({
    control,
    name: `sections.${sectionIndex}.questions`, // nested path
  })
  // …
}
```

---

## 11. Nested Fields & Complex Objects

RHF supports deeply nested objects using dot-notation names. TypeScript enforces the path is valid.

```tsx
type Address = {
  street: string
  city: string
  zip: string
  country: string
}

type UserForm = {
  name: string
  contact: {
    email: string
    phone: string
  }
  shippingAddress: Address
  billingAddress: Address
  sameAsBilling: boolean
}

function UserProfileForm() {
  const { register, watch, setValue, handleSubmit, formState: { errors } } = useForm<UserForm>({
    defaultValues: {
      name: "",
      contact: { email: "", phone: "" },
      shippingAddress: { street: "", city: "", zip: "", country: "" },
      billingAddress:  { street: "", city: "", zip: "", country: "" },
      sameAsBilling: false,
    },
  })

  const sameAsBilling = watch("sameAsBilling")
  const billingAddress = watch("billingAddress")

  // Copy billing to shipping when checkbox is ticked
  useEffect(() => {
    if (sameAsBilling) {
      setValue("shippingAddress", billingAddress, { shouldDirty: true })
    }
  }, [sameAsBilling, billingAddress, setValue])

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register("name")} />

      {/* Dot notation for nested fields */}
      <input {...register("contact.email", { required: true })} />
      {errors.contact?.email && <p>{errors.contact.email.message}</p>}

      <input {...register("billingAddress.street")} />
      <input {...register("billingAddress.city")}   />
      <input {...register("billingAddress.zip")}    />

      <label>
        <input type="checkbox" {...register("sameAsBilling")} />
        Shipping same as billing
      </label>

      {!sameAsBilling && (
        <>
          <input {...register("shippingAddress.street")} />
          <input {...register("shippingAddress.city")}   />
        </>
      )}

      <button type="submit">Save</button>
    </form>
  )
}
```

### Default values for nested data (edit forms)

```tsx
// Fetched user from API
const user = await getUser(id)

useForm<UserForm>({
  defaultValues: {
    name: user.name,
    contact: {
      email: user.email,
      phone: user.phone ?? "",
    },
    shippingAddress: user.shippingAddress ?? {
      street: "", city: "", zip: "", country: ""
    },
  },
})
```

---

## 12. FormProvider & useFormContext

When your form is split across many components (multi-step forms, deeply nested layouts), passing `register`, `control`, and `errors` as props gets unwieldy. `FormProvider` puts the form methods into React context so any descendant can access them.

```tsx
import { useForm, FormProvider, useFormContext } from "react-hook-form"

type CheckoutForm = {
  personal: { name: string; email: string }
  payment: { cardNumber: string; expiry: string; cvv: string }
  shipping: { address: string; city: string; zip: string }
}

// Top-level form
function CheckoutPage() {
  const methods = useForm<CheckoutForm>({
    defaultValues: {
      personal: { name: "", email: "" },
      payment: { cardNumber: "", expiry: "", cvv: "" },
      shipping: { address: "", city: "", zip: "" },
    },
  })

  return (
    // FormProvider passes ALL methods into context
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        <PersonalSection />    {/* child — no props needed */}
        <ShippingSection />
        <PaymentSection />
        <button type="submit" disabled={methods.formState.isSubmitting}>
          Place Order
        </button>
      </form>
    </FormProvider>
  )
}

// Child component — pulls what it needs from context
function PersonalSection() {
  const { register, formState: { errors } } = useFormContext<CheckoutForm>()
  return (
    <fieldset>
      <legend>Personal Details</legend>
      <input {...register("personal.name",  { required: "Name required" })} placeholder="Full name" />
      {errors.personal?.name && <p>{errors.personal.name.message}</p>}

      <input {...register("personal.email", { required: "Email required" })} type="email" />
      {errors.personal?.email && <p>{errors.personal.email.message}</p>}
    </fieldset>
  )
}

function PaymentSection() {
  const { register, formState: { errors } } = useFormContext<CheckoutForm>()
  return (
    <fieldset>
      <legend>Payment</legend>
      <input {...register("payment.cardNumber", { required: true, minLength: 16 })} />
      <input {...register("payment.expiry",     { required: true })} placeholder="MM/YY" />
      <input {...register("payment.cvv",        { required: true, maxLength: 4 })} type="password" />
    </fieldset>
  )
}
```

> **Tip:** `useFormContext` throws if used outside a `FormProvider`. In shared component libraries, guard with a try/catch or check for context manually if the component can be used both inside and outside a form.

---

## 13. Async Validation & Async Default Values

### Async custom validate rule

```tsx
<input
  {...register("username", {
    required: "Username is required",
    validate: async (value) => {
      // Calls the server — RHF awaits this automatically
      const isTaken = await checkUsernameAvailable(value)
      return isTaken || "This username is already taken"
    },
  })}
/>
```

> **Gotcha:** async `validate` functions run on every validation trigger. If `mode: "onChange"`, this fires on every keystroke — add a debounce or use `mode: "onBlur"` to avoid hammering the server.

### Debounced async validation

```tsx
import { useDebouncedCallback } from "use-debounce"  // npm install use-debounce

const checkUsername = useDebouncedCallback(async (value: string) => {
  const taken = await api.checkUsername(value)
  if (taken) setError("username", { type: "manual", message: "Username taken" })
  else clearErrors("username")
}, 400)

// In register, skip validate and call checkUsername in onChange instead:
<input {...register("username")} onChange={(e) => checkUsername(e.target.value)} />
```

### Async default values

For edit forms where you need to fetch the current data before rendering the form:

```tsx
// Option 1: async defaultValues (RHF v7.36+ feature)
// The form renders with empty defaults, then re-populates when the promise resolves
const { register } = useForm<UserForm>({
  defaultValues: async () => {
    const user = await fetchUser(userId)  // RHF awaits this
    return { name: user.name, email: user.email, bio: user.bio ?? "" }
  },
})

// Option 2: reset() after data loads (more control)
const { reset, register } = useForm<UserForm>()
const { data: user } = useQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) })

useEffect(() => {
  if (user) {
    reset({             // populate form with server data
      name: user.name,
      email: user.email,
      bio: user.bio ?? "",
    })
  }
}, [user, reset])
```

> **Prefer Option 2** when using TanStack Query — `reset()` gives you explicit control over when the form populates, and you can react to loading/error states before resetting.

---

## 14. Handling Submission

### Async submit handler

```tsx
const onSubmit: SubmitHandler<Inputs> = async (data) => {
  // handleSubmit catches thrown errors — the form's isSubmitting stays true until this resolves/rejects
  const response = await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  })

  if (!response.ok) {
    const error = await response.json()
    // Map server validation errors back to specific fields
    if (error.field === "email") {
      setError("email", { type: "server", message: error.message })
    } else {
      // Non-field errors go on the root
      setError("root.serverError", { type: "server", message: "Submission failed. Try again." })
    }
    return
  }

  reset()            // clear the form after success
  router.push("/dashboard")
}

// handleSubmit runs validation first. If it fails, onSubmit is NOT called.
<form onSubmit={handleSubmit(onSubmit)}>
```

### Mapping multiple server errors

```tsx
const onSubmit = async (data: Inputs) => {
  const res = await fetch("/api/signup", { method: "POST", body: JSON.stringify(data) })
  if (res.status === 422) {
    const { errors } = await res.json()
    // errors: [{ field: "email", message: "Email in use" }, ...]
    errors.forEach(({ field, message }: { field: keyof Inputs; message: string }) => {
      setError(field, { type: "server", message })
    })
    setFocus(errors[0].field)  // focus the first erroring field
    return
  }
  // ...success
}
```

### Disabling the submit button

```tsx
const { formState: { isSubmitting, isValid } } = useForm({ mode: "onChange" })

// While submitting
<button type="submit" disabled={isSubmitting}>
  {isSubmitting ? "Submitting…" : "Submit"}
</button>

// Only enable when valid AND not submitting (stricter UX)
<button type="submit" disabled={!isValid || isSubmitting}>
  Submit
</button>
```

> **Gotcha:** `isValid` is `false` on the initial render when `mode: "onSubmit"` (nothing has been validated yet). Only combine `!isValid` with a non-`onSubmit` mode, or users will see a permanently disabled button.

### handleSubmit second argument — handle invalid submits

```tsx
// Second arg runs when submit is clicked but validation fails
handleSubmit(
  (data) => onSuccess(data),
  (errors) => {
    console.log("Validation failed:", errors)
    // Could send to analytics, log, focus the first error, etc.
  }
)
```

---

## 15. Integration with shadcn/ui & Next.js

### shadcn/ui Form components

shadcn/ui ships dedicated `<Form>` components that wrap React Hook Form. They handle the `FormProvider`, error display, and accessibility automatically.

```bash
npx shadcn-ui@latest add form input label button
```

```tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

const formSchema = z.object({
  username: z.string().min(2, "Must be at least 2 characters").max(50),
  bio: z.string().max(160).optional(),
})

type FormValues = z.infer<typeof formSchema>

export function ProfileForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { username: "", bio: "" },
  })

  async function onSubmit(values: FormValues) {
    await saveProfile(values)
  }

  return (
    // <Form> wraps FormProvider — passes 'form' methods into context
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* FormField uses Controller internally */}
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                <Input placeholder="johndoe" {...field} />
              </FormControl>
              <FormDescription>This is your public display name.</FormDescription>
              <FormMessage /> {/* automatically shows errors.username.message */}
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="bio"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Bio</FormLabel>
              <FormControl>
                <Input {...field} />
              </FormControl>
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

### Next.js App Router — client component forms

All RHF hooks require `"use client"` in the App Router:

```tsx
// app/login/page.tsx — server component (the page itself)
import { LoginForm } from "./login-form"
export default function LoginPage() {
  return (
    <main>
      <h1>Sign in</h1>
      <LoginForm />
    </main>
  )
}

// app/login/login-form.tsx — client component
"use client"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { loginSchema } from "@/lib/schemas"
import type { z } from "zod"

type LoginInputs = z.infer<typeof loginSchema>

export function LoginForm() {
  const { register, handleSubmit, setError, formState } = useForm<LoginInputs>({
    resolver: zodResolver(loginSchema),
  })
  // ...
}
```

### Next.js with Server Actions

```tsx
"use client"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { createPostAction } from "./actions"  // server action

type Inputs = z.infer<typeof postSchema>

function NewPostForm() {
  const { register, handleSubmit, setError, formState } = useForm<Inputs>({
    resolver: zodResolver(postSchema),
  })

  const onSubmit = async (data: Inputs) => {
    // Call the server action directly — it runs on the server
    const result = await createPostAction(data)

    if (result?.errors) {
      // Map server-side errors back into the client form
      result.errors.forEach(({ field, message }) => {
        setError(field as keyof Inputs, { type: "server", message })
      })
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("title")} />
      {formState.errors.title && <p>{formState.errors.title.message}</p>}
      <textarea {...register("body")} />
      <button type="submit" disabled={formState.isSubmitting}>Post</button>
    </form>
  )
}
```

```tsx
// app/posts/actions.ts — the server action
"use server"
import { postSchema } from "@/lib/schemas"

export async function createPostAction(data: unknown) {
  const parsed = postSchema.safeParse(data)
  if (!parsed.success) {
    return { errors: parsed.error.issues.map(i => ({ field: i.path[0], message: i.message })) }
  }
  await db.post.create({ data: parsed.data })
}
```

> **Pattern note:** RHF validates on the client before the server action is called. The server action re-validates with Zod for security (never trust client data). This dual-validation approach is the recommended pattern in 2026.

---

## 16. File Inputs, Checkboxes, Radios & Multi-Select

### File input

```tsx
type UploadForm = {
  avatar: FileList  // file inputs return a FileList
  documents: FileList
}

const { register, handleSubmit } = useForm<UploadForm>()

const onSubmit = (data: UploadForm) => {
  const formData = new FormData()
  formData.append("avatar", data.avatar[0])          // first file
  Array.from(data.documents).forEach((file) => {
    formData.append("documents", file)               // multiple files
  })
  await fetch("/api/upload", { method: "POST", body: formData })
}

<input type="file" accept="image/*" {...register("avatar", { required: true })} />
<input type="file" multiple {...register("documents")} />
```

### Checkboxes

```tsx
type PrefsForm = {
  notifications: boolean
  newsletter: boolean
  // Multiple checkboxes for one field → string[]
  interests: string[]
}

const { register } = useForm<PrefsForm>({
  defaultValues: { notifications: true, newsletter: false, interests: [] },
})

{/* Single checkbox → boolean */}
<input type="checkbox" {...register("notifications")} />

{/* Group of checkboxes → string[] (RHF collects checked values) */}
<input type="checkbox" value="react"       {...register("interests")} />
<input type="checkbox" value="typescript"  {...register("interests")} />
<input type="checkbox" value="graphql"     {...register("interests")} />
```

### Radio buttons

```tsx
type PlanForm = {
  plan: "free" | "pro" | "enterprise"
}

const { register } = useForm<PlanForm>({
  defaultValues: { plan: "free" },
})

<input type="radio" value="free"       {...register("plan")} /> Free
<input type="radio" value="pro"        {...register("plan")} /> Pro
<input type="radio" value="enterprise" {...register("plan")} /> Enterprise
```

### Native multi-select

```tsx
type Inputs = { roles: string[] }

const { register } = useForm<Inputs>({
  defaultValues: { roles: [] },
})

// RHF collects all selected option values into an array automatically
<select multiple {...register("roles")}>
  <option value="admin">Admin</option>
  <option value="editor">Editor</option>
  <option value="viewer">Viewer</option>
</select>
```

### react-select (multi) via Controller

```tsx
import Select from "react-select"
import { Controller } from "react-hook-form"

type Option = { value: string; label: string }
type Inputs = { tags: Option[] }

<Controller
  name="tags"
  control={control}
  render={({ field }) => (
    <Select
      {...field}
      isMulti
      options={tagOptions}
      // react-select onChange gives you the selected options array
    />
  )}
/>

// In onSubmit, data.tags is Option[]
// Extract just the values if needed:
const tagValues = data.tags.map(t => t.value)
```

---

## 17. Performance Notes

### Why RHF is fast by default

1. **Uncontrolled inputs:** the DOM holds values, not React state. No `setState` per keystroke.
2. **Proxy-based formState:** destructuring `formState.errors` only subscribes the component to error changes — not all formState changes. RHF uses ES proxies to track exactly which properties you read.
3. **Isolated re-renders:** a field's error display only re-renders when that field's error changes.

### Measuring re-renders

```tsx
// Add this to any component to see how often it renders during development
const renders = useRef(0)
renders.current++
console.log("Renders:", renders.current)
```

### Isolating re-renders with useWatch

```tsx
// ❌ This causes the PARENT form component to re-render on every change to "total"
function OrderForm() {
  const { watch } = useForm()
  const total = watch("total")  // parent re-renders
  return (
    <>
      <LineItems />
      <p>Total: {total}</p>
    </>
  )
}

// ✅ Move the subscription to a child — only TotalDisplay re-renders
function TotalDisplay({ control }: { control: Control<OrderForm> }) {
  const total = useWatch({ control, name: "total" })
  return <p>Total: {total}</p>
}

function OrderForm() {
  const { control } = useForm()
  return (
    <>
      <LineItems control={control} />
      <TotalDisplay control={control} />  {/* only this re-renders */}
    </>
  )
}
```

### watch vs useWatch

| | `watch()` | `useWatch()` |
|---|---|---|
| **Where** | Called inside the component with `useForm` | Called in any component with `control` prop |
| **Re-renders** | The component calling `watch` | Only the component calling `useWatch` |
| **Use case** | Quick prototyping, small forms | Optimized production forms |
| **Default value** | From defaultValues | Must specify `defaultValue` option |

### Avoiding unnecessary Controller usage

```tsx
// ❌ Don't use Controller for native inputs — register is faster
<Controller name="email" control={control} render={({ field }) => <input {...field} />} />

// ✅ Use register for native inputs
<input {...register("email")} />
```

### Memoizing expensive child components

```tsx
// If a child component is expensive and doesn't depend on form state,
// wrap in React.memo to prevent re-renders from formState changes
const AddressSection = React.memo(function AddressSection({ onComplete }) {
  const { register } = useFormContext()
  return (/* address fields */)
})
```

---

## 18. Tips, Tricks & Gotchas

### 1. Spread register — don't destructure

```tsx
// ✅ Spread the entire return value
<input {...register("email")} />

// ❌ If you pick individual properties, you may miss the ref
const { onChange, onBlur } = register("email") // missing ref — RHF can't focus on error
```

### 2. defaultValues vs value/defaultValue

```tsx
// ✅ Always set defaultValues in useForm — this is the RHF way
useForm({ defaultValues: { name: "Jane" } })

// ❌ Don't set defaultValue on the input directly — RHF won't know the field is "clean"
<input {...register("name")} defaultValue="Jane" />

// ❌ Don't set value (that makes it controlled — conflicts with uncontrolled register)
<input {...register("name")} value={name} />
```

### 3. The controlled vs uncontrolled warning

If you mix `register` (uncontrolled) with `value` (controlled), React warns:
`"A component is changing an uncontrolled input to be controlled."`

Fix: use either `register` + `defaultValues`, or `Controller` + `value`. Never both.

### 4. valueAsNumber for number inputs

```tsx
// ❌ Without valueAsNumber, "age" is the string "25" — not the number 25
<input type="number" {...register("age")} />

// ✅ Get an actual number
<input type="number" {...register("age", { valueAsNumber: true })} />

// ✅ Or let Zod coerce it
const schema = z.object({ age: z.coerce.number().min(0) })
```

### 5. reset() timing with async data

```tsx
// ❌ Won't work — reset() before data arrives
const { data } = useQuery(...)
const { reset } = useForm()
reset(data) // data is undefined on first render

// ✅ Reset only when data is ready
useEffect(() => {
  if (data) reset(data)
}, [data, reset])
```

### 6. useForm must be called in the component that renders the form

```tsx
// ❌ You can't call useForm in a custom hook and return JSX that uses register
// unless FormProvider is used to pass context down

// ✅ Call useForm in the component, pass methods down as props or use FormProvider
```

### 7. Don't include register in useEffect dependencies

```tsx
// ❌ register is stable but ESLint may warn — don't add it to deps
useEffect(() => {
  register("hiddenField")
}, [register]) // infinite loop risk if register reference changes

// ✅ Register in the render — or use useForm's values/defaultValues
```

### 8. Arrays need field.id — not index

```tsx
// ❌ Causes React key conflicts when items are reordered
{fields.map((_, i) => <div key={i}>…</div>)}

// ✅ RHF generates a stable unique id for each field
{fields.map((field) => <div key={field.id}>…</div>)}
```

### 9. isValid is false before first validation with mode: "onSubmit"

```tsx
// ❌ This button is always disabled until the user submits once
const { formState: { isValid } } = useForm({ mode: "onSubmit" })
<button disabled={!isValid}>Submit</button>

// ✅ Use isSubmitting to block double-submits, not isValid
<button disabled={isSubmitting}>Submit</button>

// ✅ Or switch to mode: "onChange" / "onTouched" — isValid updates reactively
useForm({ mode: "onTouched" })
```

### 10. Errors don't persist after successful submit by default

After a successful submit, `errors` is cleared. If you need to show a success state, use `isSubmitSuccessful`:

```tsx
{formState.isSubmitSuccessful && <p>Form submitted successfully!</p>}
```

### 11. Schema resolver vs inline rules — don't mix both

When a `resolver` is set, inline `register` rules are ignored. Pick one approach per form:

```tsx
// ❌ Mixed — inline rules are silently ignored when resolver is set
useForm({ resolver: zodResolver(schema) })
<input {...register("email", { required: true })} />  // required here does nothing

// ✅ Put all rules in the Zod schema
const schema = z.object({ email: z.string().email() })
useForm({ resolver: zodResolver(schema) })
<input {...register("email")} />
```

### 12. Checkbox value vs checked

```tsx
// ❌ Don't set value on a boolean checkbox — it will become a string
<input type="checkbox" {...register("agreed")} value="yes" />  // data.agreed = "yes"

// ✅ Let RHF use the checked state
<input type="checkbox" {...register("agreed")} />  // data.agreed = true/false
```

### 13. Performance — avoid watch in large forms

```tsx
// ❌ watch() with no arguments re-renders the form on every field change
const all = watch()

// ✅ Watch only specific fields
const email = watch("email")

// ✅ Or use getValues() when you only need the value at a point in time (no subscription)
const onBlurEmail = () => { const v = getValues("email"); doSomething(v) }
```

---

## 19. Study Path

Work through these in order. Each step builds on the previous. After completing all of them, you'll be able to build any real-world form from memory.

### Phase 1 — Core API (1–2 days)

1. **Setup + first form** (§2, §3): `npm install react-hook-form`, create a login form with `register`, `handleSubmit`, and a basic submit handler. See the data object log.
2. **Validation rules** (§4): add `required`, `minLength`, `pattern`. Display `errors.fieldName.message`.
3. **formState flags** (§5): wire `isSubmitting` to the submit button. Watch `isDirty` change as you type.
4. **Validation modes** (§6): switch between `onSubmit`, `onBlur`, `onTouched`. Feel the UX difference.

**Build:** A registration form (name, email, password, confirm password, terms checkbox). Validate all fields inline. Show real-time errors.

### Phase 2 — Schema & TypeScript (1 day)

5. **Zod resolver** (§8): add Zod, rewrite the registration form with a schema. Use `z.infer` as the form type.
6. **Cross-field validation**: add a `.refine()` for password confirmation. Share the schema with a mock API route.
7. **Async default values** (§13): build an edit profile form that fetches the user and populates with `reset()`.

**Build:** Rewrite the registration form using Zod. Add an edit form that loads user data from a fake API.

### Phase 3 — Complex Patterns (2–3 days)

8. **Controller** (§7): wrap a react-select dropdown and a date picker. Compare the `field` object to `register`.
9. **useFieldArray** (§10): build an invoice form with add/remove line items. Compute a running total with `watch`.
10. **Nested objects** (§11): build a checkout form with nested `shippingAddress` and `billingAddress`. Add a "same as billing" checkbox that copies values.
11. **FormProvider** (§12): split the checkout form into `<PersonalDetails>`, `<PaymentDetails>`, and `<ShippingDetails>` — each in its own file, accessing the form via `useFormContext`.

**Build:** A multi-section checkout form with dynamic line items. No prop-drilling — use FormProvider.

### Phase 4 — Real-World Integration (2 days)

12. **Server errors** (§14): simulate a 422 response from a fake API. Use `setError` to map errors to fields. Focus the first erroring field.
13. **shadcn/ui** (§15): rebuild any form using shadcn `<Form>`, `<FormField>`, `<FormItem>`, `<FormMessage>`. Notice how it handles accessibility automatically.
14. **Next.js Server Action** (§15): call a Server Action from `handleSubmit`. Re-validate on the server with the same Zod schema.
15. **File uploads** (§16): add an avatar upload to the profile form. Build the FormData in `onSubmit`.

**Build:** A full Next.js app with a product listing. Each product has an edit form (loaded via TanStack Query + `reset()`), submitted to a Server Action, with Zod validation on both client and server.

### Phase 5 — Performance & Mastery (1 day)

16. **useWatch** (§17): move a "derived total" calculation into a `<Total>` child component using `useWatch`. Verify with React DevTools that the parent form no longer re-renders on typing.
17. **Gotchas review** (§18): re-read the gotchas and verify you understand each one by writing a quick example that triggers the problem, then fixing it.

### Projects that cement mastery

| Project | Key skills |
|---|---|
| Login / Signup flow | `register`, Zod, `setError` for server errors |
| User profile edit | Async `defaultValues` + `reset()`, file upload |
| Invoice builder | `useFieldArray`, `watch` for computed totals |
| Multi-step checkout | `FormProvider`, nested objects, step-by-step validation |
| Job application form | File input, multiple selects, conditional fields |
| Settings page | Multiple independent forms, `resetField`, patch-on-blur |

### Key references to bookmark

- Official docs: [react-hook-form.com](https://react-hook-form.com) — API section and "Advanced Usage"
- Resolvers: [github.com/react-hook-form/resolvers](https://github.com/react-hook-form/resolvers)
- shadcn form guide: [ui.shadcn.com/docs/components/form](https://ui.shadcn.com/docs/components/form)
- Zod docs: [zod.dev](https://zod.dev)

> **Final tip:** open the React DevTools "Profiler" tab while building forms. Record a session, type in a few fields, then inspect which components re-rendered and why. You will immediately understand why uncontrolled inputs matter — and know exactly where to apply `useWatch` or `React.memo` when performance demands it.
