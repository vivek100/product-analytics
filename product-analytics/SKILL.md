---
name: product-analytics
description: Use this skill when designing, implementing, or reviewing product analytics event tracking. Triggers include: defining tracking plans, implementing PostHog or Mixpanel events, setting up user identification, designing event schemas, auditing existing tracking, or planning analytics for new features or flow changes.
---

# Product Analytics Skill

This skill guides the design and implementation of high-quality, maintainable product analytics event tracking. Good analytics is not about tracking everything — it is about tracking the right things in the right way so you can actually measure what matters.

## Core Philosophy

Before writing a single tracking call, ask: **what decision will this data inform?** If you cannot answer that clearly, do not track it. Cluttered analytics pipelines with dozens of vague events are worse than clean pipelines with ten precise ones. Every event should map to a business question.

---

## User Identification

Getting identity right is the foundation of everything else. If users are not correctly identified, all downstream analysis — funnels, retention, cohorts — breaks silently.

**On every login or signup**, immediately call identify with the user's stable, server-assigned distinct ID:

```js
// PostHog
posthog.identify(user.id, {
  email: user.email,
  name: user.name,
  plan: user.plan,
  created_at: user.createdAt,
})

// Mixpanel
mixpanel.identify(user.id)
mixpanel.people.set({
  $email: user.email,
  $name: user.name,
  plan: user.plan,
})
```

**Never use emails or usernames as the distinct ID** — these can change. Use the immutable database ID.

**On logout**, call `posthog.reset()` or `mixpanel.reset()` so the next user on the same device starts with a clean anonymous identity. Failing to do this is one of the most common sources of corrupted identity graphs.

**Anonymous-to-identified linking**: Before login, events are tracked under an anonymous ID. PostHog and Mixpanel handle this automatically when you call `identify()` — they alias the anonymous ID to the authenticated one. You do not need to do anything extra, but be aware this is what is happening under the hood.

---

## Client-Side vs Server-Side Events

### Client-Side Events
Track UI interactions and page-level behavior on the client. These events have access to session context, device info, and the anonymous-to-identified flow automatically.

Use client-side for:
- Page views and navigation
- UI interactions (clicks, form submissions, modal opens)
- Feature engagement (expanded a section, played a video, used a filter)

### Server-Side Events
Some actions happen entirely on the backend and must be tracked server-side — payments, webhook triggers, background job completions, API calls from external systems.

**The critical challenge**: server-side events must be tied to a user. You have a few options:

1. **Pass the distinct ID through your application context.** The user's ID should always be available server-side from the session or JWT. Use it directly:

```python
posthog.capture(
    distinct_id=user.id,
    event="subscription_upgraded",
    properties={
        "plan_from": "starter",
        "plan_to": "pro",
        "revenue_usd": 49,
    }
)
```

2. **Embed the user ID in event properties when you cannot resolve it directly.** For async jobs or webhook handlers, pass the user ID as a property at the point of initiation so it travels with the event:

```js
// When initiating the async job, store user context
await queue.add("generate_report", {
  userId: currentUser.id,
  reportType: "monthly",
})

// In the worker
posthog.capture({
  distinctId: job.data.userId,
  event: "report_generated",
  properties: { report_type: job.data.reportType },
})
```

3. **Never fire server-side events with a generic or null distinct ID.** Events attributed to `"system"` or `null` are unqueryable and pollute your data.

---

## Event Naming Convention

A consistent naming convention is what separates a maintainable analytics system from a mess. Once you have dozens of events and multiple engineers adding tracking, inconsistency compounds quickly.

### Recommended Format: `object_action` (snake_case)

The pattern is always **noun first, verb second**:

```
user_signed_up
dashboard_viewed
report_exported
subscription_upgraded
onboarding_step_completed
invite_sent
```

**Avoid vague verbs** like `clicked`, `used`, or `touched`. Instead:

| Vague (avoid) | Specific (preferred) |
|---|---|
| `button_clicked` | `upgrade_cta_clicked` |
| `feature_used` | `filter_applied` |
| `page_visited` | `settings_page_viewed` |

### Page View and Click Events

For general page views and clicks, use two standard events with rich properties rather than dozens of individual page/button events:

```
page_viewed      → { page_name, page_path, referrer }
element_clicked  → { element_name, element_type, page_name, context }
```

This keeps your event list short while still letting you slice by page or element in the analytics tool. Adding a new page does not require a new event — just a new value in `page_name`.

### Naming Events for Funnel and Conversion Analysis

When events will be used in funnels, the naming and sequencing must be designed deliberately. A funnel is only as good as the events that form it — if the steps are named inconsistently, or fire in the wrong places, the funnel data will be unreliable.

**Design the funnel first, then name the events to match it.** Before writing any code, write down the sequence of steps you want to measure as a user journey:

```
signup_started
  → signup_email_submitted
  → email_verified
  → onboarding_started
  → onboarding_step_completed (step_name: "profile")
  → onboarding_step_completed (step_name: "connect_integration")
  → onboarding_step_completed (step_name: "invite_teammate")
  → user_activated
```

Each step in this funnel is a distinct event that fires at a clear, unambiguous moment. A few rules for funnel-friendly event design:

**One event per meaningful state transition.** Do not combine two steps into one event and do not fire the same event twice for the same conceptual step. If `signup_completed` fires both when the form is submitted and when the confirmation email is sent, your funnel will double-count.

**Name the terminal event after the outcome, not the action.** The final step in a funnel should capture the business outcome. Prefer `user_activated` over `dashboard_loaded_for_first_time`. The outcome name makes it obvious what you are measuring and survives UI refactors — the dashboard may be redesigned, but activation still means the same thing.

**Use a shared property to track funnel variant or experiment.** When running an A/B test on a flow, do not create separate event names per variant. Instead, add a property:

```js
posthog.capture("onboarding_started", {
  variant: "short_flow",
  experiment_id: "exp_onboarding_v2",
})
```

This lets you compare funnel conversion rates between variants in a single funnel report, filtered by `variant`.

**Conversion events need enough context to be useful.** A `subscription_upgraded` event with only `plan: "pro"` is marginally useful. One that also carries `plan_from`, `trial_days_remaining`, `triggered_from`, and `days_since_signup` lets you answer questions like "what share of upgrades happen in the last 2 days of a trial?" Add those properties at the time the event is designed, not as an afterthought.

**Distinguish intent from completion.** For high-stakes flows, track both the intention and the outcome as separate events:

```
checkout_started         → user clicked upgrade
checkout_completed       → payment succeeded
checkout_abandoned       → user left without completing
```

This lets you build abandonment funnels and identify where users drop off, not just what they successfully did.

---

## Event Count Discipline

**Aim for fewer than 50 core events for most products.** If you are approaching 100+, you have an event sprawl problem. Signs of sprawl:
- Multiple events that mean nearly the same thing (`signup_complete`, `user_registered`, `account_created`)
- Events that were added for a one-time analysis and never cleaned up
- Events named after UI elements rather than user intent

The solution is to **use properties, not more events**. Instead of:

```
report_pdf_exported
report_csv_exported
report_excel_exported
```

Use one event:
```
report_exported  → { format: "pdf" | "csv" | "excel" }
```

This is strictly better: fewer events, and you can still filter by format.

---

## Event Properties: Design Them Carefully

Properties are where the analytical power lives. A well-propertied event can answer many questions; a poorly propertied one answers almost none.

### Always include contextual properties

```js
posthog.capture("report_exported", {
  // What
  format: "pdf",
  report_type: "monthly_summary",
  page_count: 12,

  // When / state
  subscription_plan: user.plan,
  days_since_signup: daysSince(user.createdAt),
  is_first_export: !user.hasExportedBefore,

  // Where
  triggered_from: "dashboard",
})
```

### Property naming rules
- Always snake_case
- Booleans should be `is_` or `has_` prefixed: `is_paid`, `has_completed_onboarding`
- Dates should be ISO 8601 strings or Unix timestamps, not human-readable strings
- Enums should be lowercase with underscores: `"monthly_summary"`, not `"Monthly Summary"`
- Avoid nulls where possible — use an empty string or omit the property

### Super properties / user properties

Set properties that apply to every event (like `plan`, `company_size`, `locale`) as super properties in PostHog or as People properties in Mixpanel so you do not have to repeat them on every event:

```js
posthog.register({ plan: user.plan, company_size: user.org.size })
```

---

## User Properties: What Belongs There vs. on Events

User properties (People properties in Mixpanel, Person properties in PostHog) describe **who the user is** at any given point in time. Events describe **what the user did**. The distinction matters because user properties are mutable and always reflect the current state, while events are immutable records of a moment.

### What to track as a user property

User properties should capture facts about the user that are useful for segmentation across all their past and future behavior. Good candidates are things that change infrequently but affect how you interpret everything else:

**Identity and account**
- `plan` — current subscription tier (starter, pro, enterprise)
- `account_status` — active, trialing, churned, paused
- `signup_date` — ISO timestamp of account creation
- `company_name`, `company_size` — for B2B products
- `role` — their job function (engineer, marketer, executive)
- `locale`, `timezone`

**Lifecycle and activation**
- `onboarding_completed` — boolean, set once they finish
- `activation_date` — when they first hit your activation milestone
- `last_seen_at` — updated on each session (useful for dormancy detection)
- `total_sessions` — a running counter you increment on each login

**Subscription and revenue**
- `mrr` — their current monthly recurring revenue contribution
- `billing_cycle` — monthly or annual
- `trial_ends_at` — for targeted messaging before trial expiry

**Feature adoption flags**
- `has_connected_integration` — boolean
- `has_invited_teammate` — boolean
- `features_used` — array of feature keys they have engaged with

### What should stay on events, not user properties

Do not put ephemeral or action-specific data on the user profile. These belong on the event:

- The specific report format they exported (`format: "pdf"`)
- Which page they were on when they clicked something (`page_name`)
- The error message they encountered
- The variant they were shown in an A/B test

### Updating user properties correctly

Update user properties whenever the underlying fact changes, not just at signup. If a user upgrades their plan, immediately update `plan` on their profile:

```js
// PostHog: set or overwrite
posthog.setPersonProperties({ plan: "pro", mrr: 49 })

// Mixpanel: set or overwrite
mixpanel.people.set({ plan: "pro", mrr: 49 })

// Mixpanel: increment a counter safely
mixpanel.people.increment("total_sessions", 1)
```

For properties that should only be written once (like `signup_date` or `first_referrer`), use the `set_once` equivalent so a later update cannot overwrite your original acquisition data:

```js
mixpanel.people.set_once({ signup_date: new Date().toISOString(), first_referrer: document.referrer })
posthog.setPersonPropertiesForFlags({ signup_date: new Date().toISOString() })
```

---

## Marketing Automation and CRM Integration

Your analytics data does not live in isolation. User properties and events flow downstream into tools like Customer.io, Braze, Intercom, HubSpot, and ActiveCampaign where they power automated emails, in-app messages, and sales workflows. Designing your tracking with this downstream usage in mind will save significant re-work.

### How user properties become segments

Marketing automation tools sync user properties from PostHog or Mixpanel (via native integrations or tools like Segment) and use them to define audience segments. A few examples of what this enables:

- **Trial expiry campaigns**: Users where `account_status = "trialing"` and `trial_ends_at` is within 3 days → trigger an upgrade nudge email sequence
- **Activation gap**: Users where `onboarding_completed = false` and `signup_date` is more than 2 days ago → trigger a help email with setup tips
- **Plan-based content**: Users where `plan = "starter"` receive feature education for features available on pro; users on pro do not see it
- **Churn risk**: Users where `last_seen_at` is more than 14 days ago → trigger a win-back sequence
- **Upsell triggers**: Users where `has_invited_teammate = false` and `total_sessions > 5` → send a collaboration invite prompt

This only works if your user properties are consistently updated and correctly typed. A `plan` that still says `"starter"` for a user who upgraded three weeks ago will put them in the wrong campaign.

### How events trigger automations

Beyond static segments, events can fire real-time automation triggers:

- `subscription_upgraded` → immediately send an upgrade confirmation + onboarding tips for the new plan; notify sales if it is an enterprise deal
- `trial_started` → start a drip email onboarding sequence on day 0
- `onboarding_step_completed` where `step_name = "connect_integration"` → suppress the "connect your integration" onboarding email since they just did it
- `user_invited_teammate` → suppress the "invite your team" nudge campaign
- `subscription_cancelled` → trigger a cancellation survey and a win-back sequence with a delay

The suppression cases are often overlooked and cause embarrassing situations where a user receives an email urging them to do something they just did. Design your events with these suppressions in mind from the start.

### Properties to include on events specifically for marketing tools

When a marketing automation tool ingests an event, it often has limited context about the user unless you include it in the event properties. Be explicit:

```js
posthog.capture("report_exported", {
  format: "pdf",
  // Include these so downstream tools can act without a separate user lookup
  plan: user.plan,
  days_since_signup: daysSince(user.createdAt),
  onboarding_completed: user.onboardingCompleted,
})
```

This is especially important for server-side events that may be processed by a webhook or CDP pipeline where user profile enrichment is not guaranteed.

### Designing for Segment / CDP pipelines

If you are using a Customer Data Platform (Segment, Rudderstack, mParticle) as a router between your product and downstream tools, the same principles apply but with a few additions:

- Keep event names stable — a rename in your app will break any downstream destination that filters by event name
- Use `identify` calls generously — CDPs batch and forward these to CRMs, so every significant user property update should trigger a re-identify
- Avoid sending high-volume low-value events (like `page_viewed` on every scroll) to expensive downstream destinations; use the CDP's filtering layer to forward only what each destination needs

---

## Handling Feature Upgrades and Renames

When you redesign a feature, you have an analytics continuity problem: the old event and new event are the same user behavior but may have different names, which breaks historical charts.

### Decision tree for upgrades

1. **Minor UI change, same user intent** → Keep the same event name and properties. Add a new property like `ui_version: "v2"` if you need to compare pre/post.

2. **Significant redesign, same feature** → Keep the same event name. Document the date of the redesign. The continuity of the event name is more valuable than perfect purity.

3. **Feature genuinely renamed or replaced by something conceptually different** → Create a new event name. Keep firing the old one during a transition window (e.g., 30 days) so your historical dashboards do not break overnight. Add a `deprecated: true` label in your tracking plan.

4. **Feature removed** → Stop firing the event, but never delete it from your tracking plan documentation. Mark it as `archived` with the deprecation date so future engineers understand old dashboard data.

---

## Tracking Plan for Feature Changes

**Before shipping any change to a core user flow, you must write a tracking plan.** This is not optional if you want to measure impact. The tracking plan is a living document that travels with the ticket or PR. It forces the team to agree on what success looks like before building, and makes post-launch analysis far faster because the questions are already defined.

A complete tracking plan has two parts: the **analytics design** (what to measure and why) and the **implementation plan** (what code changes are needed to make it work).

### Part 1: Analytics Design

```
Feature: [Feature name or ticket ID]
Change: [What is changing — describe the user flow specifically, not just the UI]

Why we are tracking this:
  Business question:  What decision will this data inform?
                      e.g. "Does the redesigned checkout reduce drop-off?"
  Success metric:     The single number that would tell you the change worked.
                      e.g. "Checkout completion rate (checkout_started → subscription_upgraded)"
  Baseline (pre):     Current value of that metric if known. If unknown, note that
                      you need to capture a baseline before releasing.

Pre-change events still relevant:
  List the existing events from the old flow you will compare against post-launch.
  These must keep firing during any transition period.

Funnel definition (if applicable):
  Step 1: checkout_started
  Step 2: payment_info_entered
  Step 3: subscription_upgraded
  Comparison window: 30 days pre-launch vs 30 days post-launch

New events to add:
  Event name:   checkout_started
  Fires when:   User clicks any "Upgrade" CTA
  Properties:
    - triggered_from: string  (e.g. "pricing_page", "feature_gate_modal", "settings")
    - plan_selected: string   (e.g. "pro_monthly", "pro_annual")
    - current_plan: string    (user's current plan at time of click)

  Event name:   checkout_abandoned
  Fires when:   User exits checkout modal without completing payment
  Properties:
    - last_step_reached: string  (e.g. "payment_form", "plan_selection")
    - time_spent_seconds: int
    - plan_selected: string

Events to deprecate:
  Event: upgrade_modal_opened  (replaced by checkout_started)
  Deprecation date: [date 30 days after launch]
  Action: Keep firing in parallel until deprecation date, then remove

User property updates triggered by this change:
  On checkout_completed → set plan: "pro", mrr: 49, account_status: "active"

Dashboard to create post-launch:
  - Funnel: checkout_started → payment_info_entered → subscription_upgraded
  - Breakdown by triggered_from to identify highest-converting CTA location
  - Comparison: 30 days pre vs 30 days post
```

### Part 2: Implementation Plan

This is where the tracking plan becomes a technical spec. For each new event, describe exactly what code needs to change to make it fire with the right properties. Be specific enough that any engineer can pick this up without a verbal handoff.

```
Event: checkout_started

Where it fires:
  File: src/components/UpgradeButton.tsx
  Trigger: onClick handler of any element with data-upgrade-cta attribute

Data needed and where to get it:
  - triggered_from:   Read from the component's props or a data attribute on the
                      button (data-location="pricing_page"). The component must
                      accept and pass through a `ctaLocation` prop.
  - plan_selected:    Available from the PricingModal state (selectedPlan).
  - current_plan:     Available from the global user context (useUser().plan).

Code changes required:
  1. Add ctaLocation prop to <UpgradeButton> component.
  2. Update all call sites of <UpgradeButton> to pass ctaLocation.
     Locations: PricingPage.tsx, FeatureGateModal.tsx, SettingsBillingTab.tsx
  3. Add posthog.capture("checkout_started", { ... }) in the onClick handler.

Test cases to verify:
  - Click upgrade from pricing page → event fires with triggered_from: "pricing_page"
  - Click upgrade from feature gate → event fires with triggered_from: "feature_gate_modal"
  - current_plan is populated and not undefined

---

Event: checkout_abandoned

Where it fires:
  File: src/components/CheckoutModal.tsx
  Trigger: onClose handler (fires when modal dismissed without completing payment)

Data needed and where to get it:
  - last_step_reached:     Tracked in local component state (currentStep).
  - time_spent_seconds:    Compute from Date.now() minus a startTime ref set on modal open.
  - plan_selected:         From component state (selectedPlan).

Code changes required:
  1. Add a startTime ref initialized when CheckoutModal mounts.
  2. In the onClose handler, compute time_spent_seconds = (Date.now() - startTime.current) / 1000.
  3. Guard: only fire checkout_abandoned if checkout_completed has NOT fired in this session
     (track a completedRef boolean to avoid double-firing on the success redirect).
  4. Add posthog.capture("checkout_abandoned", { ... }) in the guarded onClose handler.

---

Server-side event: subscription_upgraded

Where it fires:
  File: src/api/billing/webhook.ts  (Stripe webhook handler)
  Trigger: customer.subscription.updated event where status transitions to "active"

Data needed and where to get it:
  - plan_to:           Derive from the Stripe price ID in the webhook payload.
                       Use the priceIdToPlanName() helper.
  - plan_from:         Look up the user's previous plan from the database before
                       updating it. Must be read BEFORE the db.update() call.
  - revenue_usd:       Stripe subscription amount_due / 100
  - user distinct_id:  Look up from the database using stripeCustomerId from the webhook.

Code changes required:
  1. In the webhook handler, before updating the user's plan in the database, read
     the current plan and store it as planBefore.
  2. After confirming the Stripe event is valid, call:
     posthog.capture({ distinctId: user.id, event: "subscription_upgraded", properties: { ... } })
  3. Also call posthog.identify to update the user's plan property on their profile:
     posthog.identify({ distinctId: user.id, properties: { plan: planTo, mrr: revenueUsd } })
  4. Ensure the posthog.capture call is inside the try block and failures are logged
     but do not throw — analytics failures must never break billing webhooks.
```

The implementation plan belongs in the same PR description or ticket as the analytics design. When the PR is reviewed, the reviewer should be able to verify that each event fires at the right moment with the right properties by reading this plan alongside the diff.

---

## What Not to Track

Just as important as what to track. Avoid:

- **Rage clicks and passive mouse movement** — noise without signal for most products
- **Every keystroke in a form** — track `form_submitted` with relevant properties instead
- **Internal admin actions** — filter these out at the user-identify level using a property like `is_internal: true`, or skip tracking for admin users entirely
- **Errors as custom events** — use your error monitoring tool (Sentry, Datadog) for this; analytics tools are not error trackers
- **PII in event properties** — never put email addresses, full names, phone numbers, or payment details in event properties. These flow into your analytics vendor's servers.

---

## Implementation Checklist

When implementing analytics for a new feature or flow, work through this list:

**Funnels and conversion**
- [ ] Funnel steps are defined before events are named, not after
- [ ] Each funnel step fires exactly once per user action — no double-firing
- [ ] Terminal conversion event is named after the business outcome, not the UI action
- [ ] A/B test variants are captured as event properties, not separate event names
- [ ] Intent and completion are tracked as separate events for high-stakes flows (e.g. checkout_started / checkout_completed / checkout_abandoned)

**Identity**
- [ ] User is identified on login with stable distinct ID and key user properties
- [ ] `reset()` is called on logout
- [ ] Server-side events carry the user's distinct ID — not null, not "system"

**Events**
- [ ] Events follow `object_action` naming convention
- [ ] No duplicate events that could be merged into one event + property
- [ ] Event properties include enough context to slice by plan, feature variant, and entry point
- [ ] No PII in event properties
- [ ] If replacing an old event, old event fires in parallel during transition window

**User properties**
- [ ] Key lifecycle properties are set at signup: `signup_date`, `plan`, `account_status`
- [ ] User properties are updated when the underlying fact changes (e.g. plan upgrade updates `plan` immediately)
- [ ] One-time properties use `set_once` so they are never overwritten
- [ ] Lifecycle flags updated correctly: `onboarding_completed`, `has_connected_integration`, etc.

**Tracking plan and implementation**
- [ ] Analytics design completed: business question, success metric, baseline, funnel definition
- [ ] Implementation plan completed: for each new event, where it fires, what data is needed, what code changes are required, and how to verify
- [ ] Super properties / People properties set for plan and key user segments
- [ ] Reviewer can verify event correctness from the implementation plan alongside the diff

**Marketing automation**
- [ ] Downstream tools (CRM, email platform) will receive the user property updates they need for segmentation
- [ ] Events that should suppress campaigns are implemented and tested
- [ ] High-cardinality or high-volume events are filtered at the CDP layer before forwarding to expensive destinations

---

## Quick Reference: PostHog vs Mixpanel Equivalents

| Concept | PostHog | Mixpanel |
|---|---|---|
| Identify user | `posthog.identify(id, props)` | `mixpanel.identify(id)` + `mixpanel.people.set(props)` |
| Track event | `posthog.capture(event, props)` | `mixpanel.track(event, props)` |
| Set persistent props | `posthog.register(props)` | `mixpanel.register(props)` |
| One-time props | `posthog.register_once(props)` | `mixpanel.register_once(props)` |
| Reset on logout | `posthog.reset()` | `mixpanel.reset()` |
| Server-side (Node) | `posthog.capture({ distinctId, event, properties })` | `mixpanel.track(event, { distinct_id, ...props })` |
