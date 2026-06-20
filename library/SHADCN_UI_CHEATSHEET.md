# shadcn/ui Component Cheatsheet (2026)

A practical reference for **shadcn/ui** — components, what they're for, key props, and copy-paste usage.

> **What shadcn/ui actually is:** Not an npm component library. It's a collection of **copy-in components** built on **Radix UI primitives** + **Tailwind CSS**. You run a CLI command and the component's source code is added to *your* project (usually `components/ui/`), so you own and can edit every file. This is why it's so popular for client work — no version lock-in, fully customizable.

---

## Table of Contents
1. [Setup](#1-setup)
2. [Adding Components](#2-adding-components)
3. [Core Concepts & Conventions](#3-core-concepts--conventions)
4. [Component Reference](#4-component-reference)
   - [Forms & Inputs](#forms--inputs)
   - [Buttons & Actions](#buttons--actions)
   - [Overlays & Dialogs](#overlays--dialogs)
   - [Navigation](#navigation)
   - [Data Display](#data-display)
   - [Feedback & Status](#feedback--status)
   - [Layout & Utility](#layout--utility)
5. [The Form Pattern (react-hook-form + zod)](#5-the-form-pattern)
6. [Theming & Dark Mode](#6-theming--dark-mode)
7. [Common Gotchas](#7-common-gotchas)

---

## 1. Setup

```bash
# Initialize shadcn in a Next.js / Vite / React project
npx shadcn@latest init
```

The init wizard asks about: base color, CSS variables, and paths. It creates:
- `components.json` — config (style, paths, aliases).
- `lib/utils.ts` — the `cn()` helper.
- CSS variables in your global stylesheet for theming.

```ts
// lib/utils.ts — the cn() helper you'll use everywhere
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

---

## 2. Adding Components

```bash
# Add one or several at once — source is copied into components/ui/
npx shadcn@latest add button
npx shadcn@latest add button card input dialog
```

Then import from your local path:
```tsx
import { Button } from "@/components/ui/button"
```

---

## 3. Core Concepts & Conventions

| Concept | What it means |
|---|---|
| **You own the code** | Components live in your repo. Edit them freely; updates are manual, not via npm. |
| **Built on Radix** | Accessibility (focus, ARIA, keyboard) is handled by Radix primitives underneath. |
| **`asChild` prop** | Merges the component's behavior onto your own child element instead of rendering an extra tag. e.g. `<Button asChild><Link href="/">Home</Link></Button>`. |
| **`cn()` for classes** | Pass `className` to override/extend styles; `cn()` merges them sanely. |
| **CVA variants** | Styling variants (`variant`, `size`) are defined with `class-variance-authority`. |
| **Composition** | Most components are *families*: `Dialog` + `DialogTrigger` + `DialogContent` etc. |
| **Controlled or uncontrolled** | Most overlays accept `open` + `onOpenChange` for control, or run uncontrolled. |
| **Sonner = toasts** | The old `Toast` is deprecated; use `Sonner` for notifications. |

---

## 4. Component Reference

### Forms & Inputs

---

#### Input
Single-line text field.
- **Use for:** text, email, password, search, number inputs.
- **Key props:** all native `<input>` props (`type`, `placeholder`, `value`, `onChange`, `disabled`), plus `className`.
```tsx
<Input type="email" placeholder="you@example.com" />
```

#### Textarea
Multi-line text input.
- **Use for:** messages, descriptions, comments.
- **Props:** native `<textarea>` props + `className`.
```tsx
<Textarea placeholder="Type your message..." rows={4} />
```

#### Label
Accessible label tied to a control.
- **Use for:** labeling any input; clicking it focuses the field.
- **Key props:** `htmlFor`.
```tsx
<Label htmlFor="email">Email</Label>
<Input id="email" />
```

#### Checkbox
Toggle a single boolean.
- **Use for:** "agree to terms", multi-select options.
- **Key props:** `checked`, `defaultChecked`, `onCheckedChange`, `disabled`.
```tsx
<Checkbox id="terms" onCheckedChange={(v) => setChecked(!!v)} />
```

#### Radio Group
Choose ONE from several options.
- **Use for:** mutually-exclusive choices (plan tiers, yes/no).
- **Key props:** `value`, `defaultValue`, `onValueChange`; children `RadioGroupItem value=...`.
```tsx
<RadioGroup defaultValue="month">
  <RadioGroupItem value="month" id="m" /><Label htmlFor="m">Monthly</Label>
  <RadioGroupItem value="year" id="y" /><Label htmlFor="y">Yearly</Label>
</RadioGroup>
```

#### Switch
On/off toggle (like a setting).
- **Use for:** boolean settings (notifications, dark mode).
- **Key props:** `checked`, `onCheckedChange`, `disabled`.
```tsx
<Switch checked={on} onCheckedChange={setOn} />
```

#### Select
Dropdown to pick one value.
- **Use for:** country, category, sort order.
- **Family:** `Select` > `SelectTrigger` > `SelectValue` + `SelectContent` > `SelectItem`.
- **Key props:** `value`, `onValueChange`, `defaultValue`.
```tsx
<Select onValueChange={setFruit}>
  <SelectTrigger className="w-48"><SelectValue placeholder="Pick fruit" /></SelectTrigger>
  <SelectContent>
    <SelectItem value="apple">Apple</SelectItem>
    <SelectItem value="banana">Banana</SelectItem>
  </SelectContent>
</Select>
```

#### Combobox
Searchable/filterable select. (Built by combining `Popover` + `Command` — not a single component.)
- **Use for:** long option lists where typing-to-filter helps (users, tags, repos).
- **Built from:** `Popover` + `Command` + `Button`.

#### Command
Command palette / fuzzy search menu (⌘K style).
- **Use for:** quick-search launchers, autocomplete menus.
- **Family:** `Command` > `CommandInput`, `CommandList`, `CommandEmpty`, `CommandGroup`, `CommandItem`.
```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandEmpty>No results.</CommandEmpty>
    <CommandGroup heading="Pages">
      <CommandItem>Dashboard</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

#### Input OTP
Segmented one-time-password / PIN input.
- **Use for:** 2FA codes, verification PINs.
- **Family:** `InputOTP` > `InputOTPGroup` > `InputOTPSlot`.
- **Key props:** `maxLength`, `value`, `onChange`.
```tsx
<InputOTP maxLength={6}>
  <InputOTPGroup>
    {[0,1,2,3,4,5].map(i => <InputOTPSlot key={i} index={i} />)}
  </InputOTPGroup>
</InputOTP>
```

#### Slider
Drag to pick a number in a range.
- **Use for:** volume, price range, opacity.
- **Key props:** `min`, `max`, `step`, `value` / `defaultValue` (array), `onValueChange`.
```tsx
<Slider defaultValue={[50]} max={100} step={1} />
```

#### Calendar
Inline date-picker grid (built on react-day-picker).
- **Use for:** standalone date selection; base for Date Picker.
- **Key props:** `mode` (`single` | `multiple` | `range`), `selected`, `onSelect`, `disabled`.
```tsx
<Calendar mode="single" selected={date} onSelect={setDate} />
```

#### Date Picker
Calendar inside a Popover with a trigger button. (Composed, not a single import.)
- **Use for:** form date fields.
- **Built from:** `Popover` + `Calendar` + `Button`.

#### Form
Wrapper integrating **react-hook-form** + **zod** validation with accessible markup.
- **Use for:** any real form with validation/error messages.
- **Family:** `Form` > `FormField` > `FormItem` > `FormLabel`, `FormControl`, `FormDescription`, `FormMessage`.
- See [section 5](#5-the-form-pattern) for the full pattern.

---

### Buttons & Actions

---

#### Button
The workhorse action element.
- **Use for:** submits, links (with `asChild`), triggers.
- **Key props:**
  - `variant`: `default` | `destructive` | `outline` | `secondary` | `ghost` | `link`
  - `size`: `default` | `sm` | `lg` | `icon`
  - `asChild`: render as a child element (e.g. a Next.js `<Link>`)
  - `disabled`, plus native button props.
```tsx
<Button variant="default">Save</Button>
<Button variant="destructive" size="sm">Delete</Button>
<Button variant="outline" size="icon"><Plus /></Button>
<Button asChild><Link href="/signup">Sign up</Link></Button>
```

#### Toggle
A button that holds an on/off pressed state.
- **Use for:** bold/italic toolbar buttons, single toggles.
- **Key props:** `pressed`, `onPressedChange`, `variant` (`default`|`outline`), `size`.
```tsx
<Toggle aria-label="Bold"><Bold className="size-4" /></Toggle>
```

#### Toggle Group
Set of toggles where one or many can be active.
- **Use for:** text-align pickers, view switchers.
- **Key props:** `type` (`single`|`multiple`), `value`, `onValueChange`.
```tsx
<ToggleGroup type="single" value={align} onValueChange={setAlign}>
  <ToggleGroupItem value="left"><AlignLeft /></ToggleGroupItem>
  <ToggleGroupItem value="center"><AlignCenter /></ToggleGroupItem>
</ToggleGroup>
```

---

### Overlays & Dialogs

---

#### Dialog
Modal window centered over the page.
- **Use for:** confirmations, forms, focused tasks.
- **Family:** `Dialog` > `DialogTrigger` + `DialogContent` > `DialogHeader` (`DialogTitle`, `DialogDescription`), `DialogFooter`, `DialogClose`.
- **Key props:** `open`, `onOpenChange`, `DialogTrigger asChild`.
```tsx
<Dialog>
  <DialogTrigger asChild><Button>Open</Button></DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Edit profile</DialogTitle>
      <DialogDescription>Make changes here.</DialogDescription>
    </DialogHeader>
    {/* form */}
    <DialogFooter><Button>Save</Button></DialogFooter>
  </DialogContent>
</Dialog>
```

#### Alert Dialog
Modal that REQUIRES a deliberate choice (cannot dismiss by clicking outside).
- **Use for:** destructive confirmations ("Are you sure you want to delete?").
- **Family:** `AlertDialog` > `AlertDialogTrigger` + `AlertDialogContent` > `AlertDialogHeader`, `AlertDialogFooter` > `AlertDialogAction`, `AlertDialogCancel`.
```tsx
<AlertDialog>
  <AlertDialogTrigger asChild><Button variant="destructive">Delete</Button></AlertDialogTrigger>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Are you absolutely sure?</AlertDialogTitle>
      <AlertDialogDescription>This cannot be undone.</AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction>Delete</AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

#### Sheet
Panel that slides in from an edge.
- **Use for:** mobile menus, filters, side forms.
- **Key props:** `side` (`top`|`right`|`bottom`|`left`), `open`, `onOpenChange`.
- **Family:** `Sheet` > `SheetTrigger` + `SheetContent` > `SheetHeader`, `SheetTitle`, `SheetFooter`.
```tsx
<Sheet>
  <SheetTrigger asChild><Button variant="outline">Menu</Button></SheetTrigger>
  <SheetContent side="left">{/* nav links */}</SheetContent>
</Sheet>
```

#### Drawer
Bottom drawer optimized for mobile (built on Vaul), with drag-to-dismiss.
- **Use for:** mobile-first sheets, action menus on touch.
- **Family:** `Drawer` > `DrawerTrigger` + `DrawerContent` > `DrawerHeader`, `DrawerFooter`, `DrawerClose`.

#### Popover
Floating panel anchored to a trigger.
- **Use for:** extra options, mini-forms, color pickers.
- **Family:** `Popover` > `PopoverTrigger` + `PopoverContent`.
```tsx
<Popover>
  <PopoverTrigger asChild><Button variant="outline">Open</Button></PopoverTrigger>
  <PopoverContent>Place content here.</PopoverContent>
</Popover>
```

#### Hover Card
Popover that opens on hover (with delay).
- **Use for:** profile previews, link/footnote previews.
- **Family:** `HoverCard` > `HoverCardTrigger` + `HoverCardContent`.

#### Tooltip
Tiny label on hover/focus.
- **Use for:** explaining icon buttons.
- **Family:** wrap app (or section) in `TooltipProvider`, then `Tooltip` > `TooltipTrigger` + `TooltipContent`.
```tsx
<TooltipProvider>
  <Tooltip>
    <TooltipTrigger asChild><Button size="icon"><Info /></Button></TooltipTrigger>
    <TooltipContent>More info</TooltipContent>
  </Tooltip>
</TooltipProvider>
```

#### Dropdown Menu
Click-triggered menu of actions.
- **Use for:** "..." action menus, account menus.
- **Family:** `DropdownMenu` > `DropdownMenuTrigger` + `DropdownMenuContent` > `DropdownMenuItem`, `DropdownMenuLabel`, `DropdownMenuSeparator`, `DropdownMenuCheckboxItem`, `DropdownMenuRadioGroup`, `DropdownMenuSub`.
```tsx
<DropdownMenu>
  <DropdownMenuTrigger asChild><Button variant="ghost"><MoreHorizontal /></Button></DropdownMenuTrigger>
  <DropdownMenuContent>
    <DropdownMenuLabel>Account</DropdownMenuLabel>
    <DropdownMenuSeparator />
    <DropdownMenuItem>Profile</DropdownMenuItem>
    <DropdownMenuItem>Logout</DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

#### Context Menu
Right-click menu.
- **Use for:** desktop-style right-click actions.
- **Family:** mirrors Dropdown Menu (`ContextMenu`, `ContextMenuTrigger`, `ContextMenuContent`, `ContextMenuItem`).

#### Menubar
Desktop app-style top menu bar.
- **Use for:** editor/app menus (File, Edit, View).
- **Family:** `Menubar` > `MenubarMenu` > `MenubarTrigger` + `MenubarContent` > `MenubarItem`.

---

### Navigation

---

#### Navigation Menu
Accessible top-nav with dropdown panels.
- **Use for:** marketing site headers with mega-menus.
- **Family:** `NavigationMenu` > `NavigationMenuList` > `NavigationMenuItem` > `NavigationMenuTrigger` + `NavigationMenuContent` / `NavigationMenuLink`.

#### Tabs
Switch between panels in the same space.
- **Use for:** settings sections, product detail tabs.
- **Family:** `Tabs` > `TabsList` > `TabsTrigger value=...` + `TabsContent value=...`.
- **Key props:** `defaultValue`, `value`, `onValueChange`.
```tsx
<Tabs defaultValue="account">
  <TabsList>
    <TabsTrigger value="account">Account</TabsTrigger>
    <TabsTrigger value="password">Password</TabsTrigger>
  </TabsList>
  <TabsContent value="account">Account settings…</TabsContent>
  <TabsContent value="password">Password settings…</TabsContent>
</Tabs>
```

#### Breadcrumb
Path/hierarchy navigation.
- **Use for:** "Home / Products / Shoes".
- **Family:** `Breadcrumb` > `BreadcrumbList` > `BreadcrumbItem` > `BreadcrumbLink` / `BreadcrumbPage`, `BreadcrumbSeparator`.

#### Pagination
Page-number navigation.
- **Use for:** paged tables/lists.
- **Family:** `Pagination` > `PaginationContent` > `PaginationItem` > `PaginationLink`, `PaginationPrevious`, `PaginationNext`, `PaginationEllipsis`.

#### Sidebar
Full, composable app sidebar system (collapsible, mobile-aware).
- **Use for:** dashboard/app navigation shells.
- **Family:** `SidebarProvider` > `Sidebar` > `SidebarHeader`, `SidebarContent` (`SidebarGroup`, `SidebarMenu`, `SidebarMenuItem`, `SidebarMenuButton`), `SidebarFooter` + `SidebarTrigger`, `SidebarInset`.
- **Note:** the most "batteries-included" component — great selling point for dashboard gigs.

---

### Data Display

---

#### Card
Bordered container grouping related content.
- **Use for:** stats, products, pricing tiers, dashboard widgets.
- **Family:** `Card` > `CardHeader` (`CardTitle`, `CardDescription`, `CardAction`), `CardContent`, `CardFooter`.
```tsx
<Card>
  <CardHeader>
    <CardTitle>Total Revenue</CardTitle>
    <CardDescription>This month</CardDescription>
  </CardHeader>
  <CardContent className="text-3xl font-bold">$12,400</CardContent>
  <CardFooter className="text-sm text-muted-foreground">+12% vs last month</CardFooter>
</Card>
```

#### Table
Styled static HTML table.
- **Use for:** simple tabular data.
- **Family:** `Table` > `TableHeader` > `TableRow` > `TableHead`; `TableBody` > `TableRow` > `TableCell`; `TableCaption`, `TableFooter`.

#### Data Table
Powerful table built with **@tanstack/react-table** (sorting, filtering, pagination, row selection). Composed in your code, not a single import.
- **Use for:** admin dashboards, real data grids.
- **Built from:** `Table` components + TanStack Table hooks (`useReactTable`, `getCoreRowModel`, etc.).
- **Note:** big value-add for dashboard clients — pairs naturally with **TanStack Query** for server data.

#### Avatar
User profile image with fallback initials.
- **Use for:** user/account pictures.
- **Family:** `Avatar` > `AvatarImage` + `AvatarFallback`.
```tsx
<Avatar>
  <AvatarImage src="/me.jpg" alt="Me" />
  <AvatarFallback>TN</AvatarFallback>
</Avatar>
```

#### Badge
Small status/category pill.
- **Use for:** tags, counts, statuses.
- **Key props:** `variant` (`default`|`secondary`|`destructive`|`outline`), `asChild`.
```tsx
<Badge variant="secondary">New</Badge>
```

#### Accordion
Collapsible stacked sections.
- **Use for:** FAQs, grouped settings.
- **Key props:** `type` (`single`|`multiple`), `collapsible`, `defaultValue`.
- **Family:** `Accordion` > `AccordionItem` > `AccordionTrigger` + `AccordionContent`.
```tsx
<Accordion type="single" collapsible>
  <AccordionItem value="q1">
    <AccordionTrigger>Do I own the code?</AccordionTrigger>
    <AccordionContent>Yes, 100%.</AccordionContent>
  </AccordionItem>
</Accordion>
```

#### Collapsible
Single show/hide region.
- **Use for:** "show more" toggles.
- **Family:** `Collapsible` > `CollapsibleTrigger` + `CollapsibleContent`.

#### Carousel
Sliding content slider (built on Embla).
- **Use for:** image galleries, testimonials, product showcases.
- **Family:** `Carousel` > `CarouselContent` > `CarouselItem` + `CarouselPrevious`, `CarouselNext`.
- **Key props:** `opts` (Embla options), `orientation`, `setApi`.

#### Chart
Composable charts wrapping **Recharts** with theme-aware styling.
- **Use for:** dashboards, analytics, reports.
- **Family:** `ChartContainer` + `ChartTooltip`, `ChartTooltipContent`, `ChartLegend` around Recharts elements (`Bar`, `Line`, `Area`, `Pie`).
- **Note:** strong dashboard-gig differentiator.

#### Aspect Ratio
Locks content (image/video/iframe) to a ratio.
- **Use for:** responsive media that won't shift layout.
- **Key props:** `ratio` (e.g. `16/9`).
```tsx
<AspectRatio ratio={16/9}><img src="/x.jpg" className="object-cover" /></AspectRatio>
```

---

### Feedback & Status

---

#### Sonner (Toast)
Notification toasts. **This replaces the deprecated `Toast` component.**
- **Use for:** "Saved!", "Error", async feedback.
- **Setup:** render `<Toaster />` once near the app root, then call `toast()` anywhere.
```tsx
// layout
import { Toaster } from "@/components/ui/sonner"
<Toaster />

// anywhere
import { toast } from "sonner"
toast.success("Profile saved")
toast.error("Something went wrong")
toast("Event created", { description: "Friday, 7pm", action: { label: "Undo", onClick: () => {} } })
```

#### Alert
Inline static callout box.
- **Use for:** page-level notices, warnings, info banners.
- **Key props:** `variant` (`default`|`destructive`).
- **Family:** `Alert` > `AlertTitle` + `AlertDescription`.
```tsx
<Alert variant="destructive">
  <AlertTitle>Error</AlertTitle>
  <AlertDescription>Your session expired.</AlertDescription>
</Alert>
```

#### Progress
Horizontal progress bar.
- **Use for:** uploads, loading %, steps.
- **Key props:** `value` (0–100).
```tsx
<Progress value={66} />
```

#### Skeleton
Gray placeholder shown while loading.
- **Use for:** loading states (cards, lists, avatars).
- **Usage:** just a styled box — size it with Tailwind classes.
```tsx
<Skeleton className="h-12 w-12 rounded-full" />
<Skeleton className="h-4 w-[200px]" />
```

---

### Layout & Utility

---

#### Separator
Thin divider line.
- **Use for:** separating sections/menu groups.
- **Key props:** `orientation` (`horizontal`|`vertical`), `decorative`.
```tsx
<Separator className="my-4" />
```

#### Scroll Area
Custom cross-browser styled scrollbars.
- **Use for:** scrollable panels, lists, chat windows.
- **Family:** `ScrollArea` (+ `ScrollBar` for horizontal).
```tsx
<ScrollArea className="h-72 w-48 rounded-md border p-4">{/* long list */}</ScrollArea>
```

#### Resizable
Draggable split panes.
- **Use for:** code editors, dashboards, side-by-side views.
- **Family:** `ResizablePanelGroup` (`direction`) > `ResizablePanel` + `ResizableHandle`.

---

## 5. The Form Pattern

shadcn `Form` wraps **react-hook-form** + **zod**. This is the standard way to do validated forms — worth memorizing for client work.

```tsx
"use client"
import { z } from "zod"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

const schema = z.object({
  email: z.string().email("Enter a valid email"),
})

export function SignupForm() {
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { email: "" },
  })

  function onSubmit(values: z.infer<typeof schema>) {
    console.log(values) // send to API
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input placeholder="you@example.com" {...field} /></FormControl>
              <FormMessage /> {/* auto-shows zod error */}
            </FormItem>
          )}
        />
        <Button type="submit">Sign up</Button>
      </form>
    </Form>
  )
}
```

**Why it matters:** `FormMessage` auto-renders validation errors, labels are wired for accessibility, and zod gives you typed, declarative rules. Clients love clean, validated forms.

---

## 6. Theming & Dark Mode

Colors are driven by CSS variables (e.g. `--primary`, `--background`, `--muted-foreground`) so components restyle globally.

| Token class | Used for |
|---|---|
| `bg-background` / `text-foreground` | Base page colors. |
| `bg-primary` / `text-primary-foreground` | Primary brand surfaces. |
| `bg-muted` / `text-muted-foreground` | Subtle/disabled text & fills. |
| `bg-card` / `bg-popover` | Surface backgrounds. |
| `border-border` / `ring-ring` | Borders & focus rings. |
| `bg-destructive` | Danger/delete. |

**Dark mode:** add the `dark` class to `<html>` (commonly via `next-themes`). All tokens flip automatically.
```bash
npm install next-themes
```

---

## 7. Common Gotchas

| Issue | Fix |
|---|---|
| Hooks error / "use client" | Interactive components (Dialog, Tabs, forms) need `"use client"` in Next.js App Router. |
| Tooltip not showing | Must be wrapped in `<TooltipProvider>`. |
| Toast not appearing | Render `<Toaster />` once at app root; import `toast` from `sonner`. |
| Button-as-link adds extra tag | Use `asChild`: `<Button asChild><Link/></Button>`. |
| Styles not merging / conflicting | Pass overrides via `className`; `cn()` + tailwind-merge resolves conflicts. |
| Select/Dialog clipped inside `overflow-hidden` | They portal to `body` by default — usually fine; check z-index if custom. |
| Updating a component | Re-run `npx shadcn@latest add <name>` (will overwrite — back up edits first). |
| Old Toast deprecated | Use **Sonner** instead. |

---

### Where each component shines (quick map for client projects)
- **Landing pages:** Button, Card, Accordion (FAQ), Carousel (testimonials), Navigation Menu, Badge, Sheet (mobile menu), Sonner.
- **Dashboards / apps:** Sidebar, Data Table, Chart, Card, Tabs, Dialog, Dropdown Menu, Skeleton, Form.
- **Auth flows:** Form, Input, Input OTP, Button, Alert, Sonner.

> Browse live examples + copy code at **ui.shadcn.com/docs/components** — every component has a working demo.
