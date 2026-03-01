# Evolynt Blueprints

> A Blueprint is a YAML file that describes an entire business setup on Evolynt. Instead of configuring resources one by one through the UI, you describe everything in a single file and import it. This repository contains example Blueprints, JSON schemas for validation, and this reference guide.

## What is a Blueprint?

A Blueprint is a text file that describes your business setup on Evolynt. It defines products, offers, pricing, portals, funnels, workflows, email templates, and more in a single YAML document. When imported, Evolynt reads the file and creates all the resources automatically.

```yaml
EvolyntVersion: "1.0"

Resources:
  mini-course:
    Type: Evolynt::Commerce::Product
    Properties:
      Name: "5-Day Email Marketing Bootcamp"
      ProductType: course
```

Blueprints can be exported from an existing workspace and re-imported into another, making them fully round-trippable.

## Document Structure

Every Blueprint file follows this structure:

```yaml
EvolyntVersion: "1.0"

Metadata:
  Name: "My Business Setup"
  Description: "Complete configuration for my coaching business"

Resources:
  resource-slug:
    Type: Evolynt::Domain::Resource
    Properties:
      # Resource-specific properties
```

### EvolyntVersion

Required. Currently `"1.0"` is the only supported version.

### Metadata (Optional)

Human-readable name and description for the Blueprint. Useful when sharing templates.

### Resources

The `Resources` section contains all resources to create. Each resource has:

- **Slug** (the map key): A unique identifier like `my-tag` or `coaching-offer`. Used for `!Ref` references.
- **Type**: The resource type in `Evolynt::Domain::Resource` format.
- **Properties**: Configuration specific to that resource type.

## Resource Types

All resource types follow the format `Evolynt::<Domain>::<Resource>`:

| Domain | Resource Types | Description |
|--------|---------------|-------------|
| `CRM` | `Pipeline`, `Tag`, `BookingLink`, `CustomField`, `Agreement` | Customer relationship management |
| `Commerce` | `Product`, `PricingPlan`, `Offer` | Products, pricing, and offers |
| `Portal` | `Portal`, `OnboardingFlow` | Client-facing portals |
| `Funnel` | `Funnel`, `Form` | Lead capture and sales flows |
| `Automation` | `Workflow`, `EmailTemplate`, `EmailSequence` | Workflows and sequences |
| `Settings` | `BusinessProfile` | Business configuration |

## Intrinsic Functions

### !Ref - Resource References

The `!Ref` function references another resource by its slug. Use it when one resource needs to link to another.

```yaml
Resources:
  coaching-portal:
    Type: Evolynt::Portal::Portal
    Properties:
      Name: "Coaching Members Area"

  coaching-offer:
    Type: Evolynt::Commerce::Offer
    Properties:
      Name: "Elite Coaching"
      DefaultPortal: !Ref coaching-portal
      Items:
        - Product: !Ref private-coaching
```

Behavior:
- Validated at parse time. If the referenced resource does not exist, you get an immediate error.
- References must point to resources defined in the same Blueprint file.
- Returns the slug string at runtime (resolved to UUID internally).

### !Sub - Variable Substitution

The `!Sub` function inserts variables into strings. Use it for environment-specific values.

```yaml
Subject: !Sub "Welcome to ${COMPANY_NAME}!"
```

Variables come from environment variables or import-time parameters.

Common variables:

| Variable | Use Case |
|----------|----------|
| `${COMPANY_NAME}` | Brand name in emails and templates |
| `${ENVIRONMENT}` | Different settings for dev/staging/prod |
| `${SUPPORT_EMAIL}` | Contact email that varies by workspace |

## Dependency Order

Resources are imported in this sequence. A resource can only `!Ref` resources at the same level or earlier.

| Order | Resource Types | Can Reference |
|-------|---------------|---------------|
| 1 | Pipeline, Tag, BookingLink, CustomField | (none) |
| 2 | Portal | (none) |
| 3 | Product | BookingLink |
| 4 | PricingPlan | (none) |
| 5 | EmailTemplate | (none) |
| 6 | EmailSequence | EmailTemplate |
| 7 | Agreement | (none) |
| 8 | Form | CustomField |
| 9 | OnboardingFlow | Form, Agreement, BookingLink |
| 10 | Offer | Portal, Product, PricingPlan, OnboardingFlow |
| 11 | Workflow | Tag, EmailTemplate, EmailSequence, Offer, Pipeline, Form, Agreement |
| 12 | Funnel | Form, BookingLink, Offer, Agreement |
| 13 | BusinessProfile | (none) |

---

## CRM Resources

### Pipeline

**Type:** `Evolynt::CRM::Pipeline`

Organizes Deals through stages. Think of it as a visual board where each column is a stage in your sales process.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | Display name for the pipeline |
| Description | string | No | Pipeline description |
| Stages | array | Yes | Ordered list of stages |

#### Stage Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | Stage display name |
| Slug | string | Yes | Stage slug (unique within the pipeline) |
| Order | integer | Yes | Display order (1-based) |
| Description | string | No | Stage description |

#### Example

```yaml
sales-pipeline:
  Type: Evolynt::CRM::Pipeline
  Properties:
    Name: "High-Ticket Sales"
    Description: "Main sales pipeline for coaching offers"
    Stages:
      - Name: "Lead"
        Slug: "lead"
        Order: 1
      - Name: "Qualified"
        Slug: "qualified"
        Order: 2
      - Name: "Call Scheduled"
        Slug: "call-scheduled"
        Order: 3
      - Name: "Closed Won"
        Slug: "closed-won"
        Order: 4
      - Name: "Closed Lost"
        Slug: "closed-lost"
        Order: 5
```

---

### Tag

**Type:** `Evolynt::CRM::Tag`

A label you attach to Contacts to organize and segment them.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | Display name for the tag |
| Description | string | No | Tag description |

#### Example

```yaml
hot-lead:
  Type: Evolynt::CRM::Tag
  Properties:
    Name: "Hot Lead"
    Description: "High-intent prospects"
```

---

### BookingLink

**Type:** `Evolynt::CRM::BookingLink`

Creates a scheduling page where Contacts can book appointments.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Display name for the booking link |
| OwnerUserEmail | string | Yes | - | Email of the team member who owns this booking link |
| Description | string | No | - | Description shown on the public booking page |
| DurationMinutes | number | Yes | - | Appointment duration (5-480 minutes) |
| BufferBeforeMinutes | number | No | 0 | Buffer time before appointments |
| BufferAfterMinutes | number | No | 0 | Buffer time after appointments |
| MinNoticeHours | number | No | 24 | Minimum notice required for booking |
| MaxAdvanceDays | number | No | 60 | Maximum days in advance to book |
| AutoConfirm | boolean | No | true | Auto-confirm appointments |
| CreateZoomMeeting | boolean | No | false | Auto-create Zoom meeting |
| CreateCalendarEvent | boolean | No | true | Create calendar event for appointment |
| ZoomWaitingRoom | boolean | No | true | Enable Zoom waiting room |
| ZoomJoinBeforeHost | boolean | No | false | Allow joining before host |
| ZoomRecording | enum | No | none | Recording option: `cloud`, `local`, or `none` |
| ZoomMuteOnEntry | boolean | No | true | Mute participants on entry |

#### Example

```yaml
strategy-call:
  Type: Evolynt::CRM::BookingLink
  Properties:
    Name: "Strategy Call"
    OwnerUserEmail: "coach@example.com"
    Description: "30-minute discovery call to discuss your goals"
    DurationMinutes: 30
    BufferAfterMinutes: 15
    MinNoticeHours: 24
    CreateZoomMeeting: true
    ZoomWaitingRoom: true
    ZoomRecording: cloud
```

---

### CustomField

**Type:** `Evolynt::CRM::CustomField`

Captures additional data on Contacts beyond the standard fields (name, email, phone).

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Label | string | Yes | - | Display label for the field |
| FieldType | enum | Yes | - | Field type (see table below) |
| Options | string[] | Conditional | - | Required for `select` and `multi_select` |
| DefaultValue | string | No | - | Default value |
| Placeholder | string | No | - | Placeholder text |
| HelpText | string | No | - | Help text shown to users |

#### Field Types

| Field Type | Description |
|------------|-------------|
| `text` | Single line of text |
| `email` | Email address with validation |
| `phone` | Phone number |
| `number` | Numeric value |
| `textarea` | Multiple lines of text |
| `select` | Dropdown with single selection |
| `multi_select` | Dropdown with multiple selections |
| `checkbox` | Yes/no toggle |
| `date` | Date picker |
| `url` | Website URL with validation |

#### Example

```yaml
revenue-range:
  Type: Evolynt::CRM::CustomField
  Properties:
    Label: "Revenue Range"
    FieldType: select
    Options:
      - "Under $100k"
      - "$100k - $500k"
      - "$500k - $1M"
      - "Over $1M"
    Placeholder: "Select your revenue range"
```

---

### Agreement

**Type:** `Evolynt::CRM::Agreement`

A legal document that Contacts can sign electronically.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Internal name for the agreement |
| Title | string | Yes | - | Title displayed on the signing page |
| Content | string | Yes | - | Agreement body text (supports Markdown and variables) |
| SignatureType | enum | No | typed | Signature method: `typed` or `checkbox` |
| CheckboxLabel | string | No | - | Label for checkbox (when SignatureType is `checkbox`) |
| CollectFields | array | No | - | Additional fields to collect from signer |
| IncludeCompanySignature | boolean | No | false | Show company signature block on signed document |

#### CollectFields Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Field | enum | Yes | - | `initials`, `companyName`, `billingAddress`, `dateOfBirth`, or `phone` |
| Required | boolean | No | false | Whether the field is required |
| Label | string | No | - | Custom label (uses default if not specified) |

#### Content Variables

| Variable | Description |
|----------|-------------|
| `{{workspace.name}}` | Your business/workspace name |
| `{{contact.name}}` | Signer's name |
| `{{contact.email}}` | Signer's email |

#### Example

```yaml
coaching-agreement:
  Type: Evolynt::CRM::Agreement
  Properties:
    Name: "Coaching Agreement"
    Title: "COACHING SERVICES AGREEMENT"
    SignatureType: typed
    CollectFields:
      - Field: initials
        Required: true
      - Field: phone
        Required: false
    IncludeCompanySignature: true
    Content: |
      This Agreement is entered into between {{workspace.name}}
      ("Coach") and {{contact.name}} ("Client").

      ## 1. SERVICES
      Coach agrees to provide coaching services as outlined.

      ## 2. CONFIDENTIALITY
      Both parties agree to keep all discussions confidential.
```

---

## Commerce Resources

### Product

**Type:** `Evolynt::Commerce::Product`

What customers receive in their Portal after purchasing an Offer. Evolynt supports 5 product types.

#### Common Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | The name shown to customers in their Portal |
| ProductType | enum | Yes | `course`, `community`, `page`, `coaching`, or `feed` |
| Description | string | No | A brief description of what is included |

---

#### Course Products (`ProductType: course`)

Structured training with video lessons organized into modules.

**Course Properties:**

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Course | object | Yes | - | Course content container |
| Course.Title | string | Yes | - | Title shown at the top of the course |
| Course.Description | string | No | - | Description of what students will learn |
| Course.Modules | array | No | [] | Modules that make up the course |

**Module Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Title | string | Yes | Module name |
| Order | integer | Yes | Display order (1, 2, 3...) |
| Lessons | array | No | Lessons within this module |

**Lesson Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Title | string | Yes | Lesson name |
| Order | integer | Yes | Display order within the module |
| ContentType | enum | Yes | `video`, `text`, or `video_and_text` |
| VideoEmbedUrl | string | Conditional | Required when ContentType is `video` or `video_and_text` |
| TextContent | string | Conditional | Required when ContentType is `text` or `video_and_text` |

**Supported Video Providers:** YouTube, Vimeo, Wistia, Loom, Bunny Stream.

```yaml
my-course:
  Type: Evolynt::Commerce::Product
  Properties:
    Name: "12-Week Business Transformation"
    ProductType: course
    Course:
      Title: "Business Transformation Curriculum"
      Modules:
        - Title: "Week 1: Foundations"
          Order: 1
          Lessons:
            - Title: "Welcome & Overview"
              Order: 1
              ContentType: video
              VideoEmbedUrl: "https://vimeo.com/123456789"
            - Title: "Setting Your Revenue Goal"
              Order: 2
              ContentType: video_and_text
              VideoEmbedUrl: "https://vimeo.com/123456790"
              TextContent: |
                ## Your First Assignment
                1. Write down your 90-day revenue goal
                2. Break it into weekly targets
```

---

#### Community Products (`ProductType: community`)

Discussion spaces and events for member engagement.

**Community Properties:**

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Community | object | Yes | - | Community content container |
| Community.Name | string | Yes | - | Community name shown to members |
| Community.Description | string | No | - | Welcome message or description |
| Community.Spaces | array | No | [] | Discussion channels |
| Community.Events | array | No | [] | Scheduled events |

**Space Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | Space name |
| Slug | string | Yes | URL-friendly identifier (unique within community) |
| Description | string | No | What this space is for |
| Order | integer | Yes | Display order in sidebar |

**Event Properties:**

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Title | string | Yes | - | Event name |
| Description | string | No | - | Event details |
| SpaceSlug | string | No | - | Post event in a specific space |
| StartAt | string | Yes | - | Start time (ISO 8601) |
| EndAt | string | Yes | - | End time (ISO 8601) |
| LocationType | enum | Yes | - | `virtual`, `physical`, or `hybrid` |
| LocationDetails | string | No | - | Physical address (for physical/hybrid) |
| MeetingUrl | string | No | - | Video call link (for virtual/hybrid) |
| HostUserEmail | string | Yes | - | Email of the team member hosting |
| Capacity | integer | No | - | Maximum attendees |
| RecurrenceRule | object | No | - | For repeating events |

**RecurrenceRule Properties:**

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Frequency | string | Yes | - | `daily`, `weekly`, or `monthly` |
| Interval | integer | No | 1 | Repeat every N periods |
| DaysOfWeek | array | No | - | For weekly: days to repeat (0=Sun, 6=Sat) |
| DayOfMonth | integer | No | - | For monthly: day of month (1-31) |
| EndType | string | No | never | `never`, `count`, or `until` |
| Count | integer | Conditional | - | Required if EndType is `count` |
| Until | string | Conditional | - | Required if EndType is `until` (ISO date) |

```yaml
mastermind-community:
  Type: Evolynt::Commerce::Product
  Properties:
    Name: "Mastermind Inner Circle"
    ProductType: community
    Community:
      Name: "Inner Circle Community"
      Description: "Welcome to the Inner Circle!"
      Spaces:
        - Name: "Announcements"
          Slug: announcements
          Order: 1
        - Name: "Wins & Celebrations"
          Slug: wins
          Order: 2
      Events:
        - Title: "Weekly Group Coaching Call"
          StartAt: "2025-01-15T14:00:00Z"
          EndAt: "2025-01-15T15:00:00Z"
          LocationType: virtual
          MeetingUrl: "https://zoom.us/j/123456789"
          HostUserEmail: "sarah@example.com"
          RecurrenceRule:
            Frequency: weekly
            DaysOfWeek: [3]
```

---

#### Page Products (`ProductType: page`)

A single page of content. Use for bonus resources, downloads, or simple information.

**Page Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Page | object | Yes | Page content container |
| Page.Title | string | Yes | Title shown at the top |
| Page.Content | string | Yes | Page content (supports Markdown) |

```yaml
bonus-resources:
  Type: Evolynt::Commerce::Product
  Properties:
    Name: "Bonus Resources"
    ProductType: page
    Page:
      Title: "Your Bonus Resources"
      Content: |
        ## Templates
        - **Client Onboarding Checklist**
        - **Discovery Call Script**
        - **Content Calendar Template**
```

---

#### Coaching Products (`ProductType: coaching`)

Scheduled sessions with a BookingLink. Customers book their included sessions from the Portal.

**Coaching Properties:**

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Coaching | object | Yes | - | Coaching configuration |
| Coaching.Title | string | Yes | - | Title shown in the portal |
| Coaching.Description | string | No | - | What customers can expect |
| Coaching.Quantity | integer | Yes | - | Number of sessions included |
| Coaching.BookingLink | string (ref) | Yes | - | Reference to a BookingLink |
| Coaching.ExpirationDays | integer | No | - | Days until unused sessions expire |

```yaml
strategy-call:
  Type: Evolynt::CRM::BookingLink
  Properties:
    Name: "Strategy Session"
    OwnerUserEmail: "coach@example.com"
    DurationMinutes: 60
    CreateZoomMeeting: true

private-coaching:
  Type: Evolynt::Commerce::Product
  Properties:
    Name: "VIP Coaching Package"
    ProductType: coaching
    Coaching:
      Title: "Private Strategy Sessions"
      Quantity: 10
      BookingLink: !Ref strategy-call
      ExpirationDays: 180
```

---

#### Feed Products (`ProductType: feed`)

A content stream you post to over time, like a blog within the portal.

**Feed Properties:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Feed | object | Yes | Feed configuration |
| Feed.Title | string | Yes | Title shown at the top |
| Feed.Description | string | No | Description of what content to expect |

```yaml
insider-updates:
  Type: Evolynt::Commerce::Product
  Properties:
    Name: "Insider Feed"
    ProductType: feed
    Feed:
      Title: "Insider Updates"
      Description: "Weekly insights and behind-the-scenes content"
```

Note: Feed posts are created through the Evolynt admin interface after import. The Blueprint creates the feed container.

---

### PricingPlan

**Type:** `Evolynt::Commerce::PricingPlan`

Defines how customers pay for Offers.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Name shown to customers |
| Description | string | No | - | Explains this payment option |
| Amount | integer | Yes | - | Price in **cents** (e.g., 9700 = $97.00) |
| Currency | string | No | USD | Currency code (`USD`, `EUR`, `GBP`, etc.) |
| BillingType | enum | Yes | - | `one_time`, `subscription`, or `split_pay` |
| BillingInterval | enum | Conditional | - | Required for subscription/split_pay: `daily`, `weekly`, `biweekly`, `monthly`, `yearly` |
| NumberOfPayments | integer | Conditional | - | Required for `split_pay` (minimum 2) |

**Important:** Amounts are always in cents. `yearly` interval is only available for subscriptions, not split_pay.

#### Billing Types

| Type | Description |
|------|-------------|
| `one_time` | Single payment, lifetime access |
| `subscription` | Recurring payments until cancelled |
| `split_pay` | Fixed number of payments for a total amount |

#### Examples

```yaml
# One-time payment
course-pif:
  Type: Evolynt::Commerce::PricingPlan
  Properties:
    Name: "Pay in Full"
    Amount: 99700
    Currency: USD
    BillingType: one_time

# Monthly subscription
monthly-plan:
  Type: Evolynt::Commerce::PricingPlan
  Properties:
    Name: "Monthly Membership"
    Amount: 9700
    Currency: USD
    BillingType: subscription
    BillingInterval: monthly

# 6-month payment plan
payment-plan:
  Type: Evolynt::Commerce::PricingPlan
  Properties:
    Name: "6-Month Payment Plan"
    Amount: 599700
    Currency: USD
    BillingType: split_pay
    BillingInterval: monthly
    NumberOfPayments: 6
```

---

### Offer

**Type:** `Evolynt::Commerce::Offer`

A collection of Products that can be sold with PricingPlans. When purchased, customers access the Products through the assigned Portal.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Offer name shown to customers |
| Description | string | No | - | Description of what is included |
| DefaultPortal | string (ref) | Yes | - | Portal where customers access products |
| Items | array | No | [] | Products included in this offer |
| Items[].Product | string (ref) | Yes | - | Reference to a Product |
| PricingPlans | array | No | [] | PricingPlans available for this offer |

#### Example

```yaml
elite-offer:
  Type: Evolynt::Commerce::Offer
  Properties:
    Name: "Elite Mastermind 2025"
    Description: "Course, community, and coaching"
    DefaultPortal: !Ref mastermind-portal
    Items:
      - Product: !Ref transformation-course
      - Product: !Ref mastermind-community
      - Product: !Ref private-coaching
    PricingPlans:
      - !Ref mastermind-pif
      - !Ref mastermind-monthly
```

---

## Portal Resources

### Portal

**Type:** `Evolynt::Portal::Portal`

A private website where customers log in and access their purchased Offers.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Display name for the portal |
| Description | string | No | - | Portal description |
| ThemeConfig | object | No | - | Theme configuration |
| ThemeConfig.PrimaryColor | string | No | - | Primary brand color (hex) |

#### Example

```yaml
elite-portal:
  Type: Evolynt::Portal::Portal
  Properties:
    Name: "Elite Accelerator Portal"
    Description: "Exclusive area for Elite Accelerator members"
    ThemeConfig:
      PrimaryColor: "#0368EE"
```

---

### OnboardingFlow

**Type:** `Evolynt::Portal::OnboardingFlow`

Steps new customers complete in their Portal after purchasing an Offer. Used to welcome members, collect information, sign agreements, and book calls.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Display name for the flow |
| Description | string | No | - | Flow description |
| Steps | array | No | [] | Ordered list of steps |

#### Step Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Step display name |
| Slug | string | Yes | - | URL slug (unique within flow) |
| Order | integer | Yes | - | Step order (minimum 1) |
| Sections | array | No | [] | List of sections |

#### Section Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Order | integer | Yes | Section order (minimum 1) |
| Elements | array | No | List of elements |

OnboardingFlow and Funnel share the same element types. See [Element Types](#element-types-funnels-and-onboardingflows) below.

#### Example

```yaml
welcome-flow:
  Type: Evolynt::Portal::OnboardingFlow
  Properties:
    Name: "Elite Mastermind Onboarding"
    Steps:
      - Name: "Welcome"
        Slug: welcome
        Order: 1
        Sections:
          - Order: 1
            Elements:
              - Type: headline
                Order: 1
                Content:
                  Text: "Welcome to the Elite Mastermind!"
              - Type: video
                Order: 2
                Content:
                  Url: "https://vimeo.com/123456789"
              - Type: button
                Order: 3
                Content:
                  Text: "Continue"
                  NextStep: agreement
      - Name: "Sign Agreement"
        Slug: agreement
        Order: 2
        Sections:
          - Order: 1
            Elements:
              - Type: agreement
                Order: 1
                Content:
                  AgreementSlug: !Ref coaching-agreement
                  OnSuccess:
                    Action: next_step
```

Note: In OnboardingFlows, buttons support a `NextStep` property (a step slug) for direct navigation between steps.

---

## Funnel Resources

### Funnel

**Type:** `Evolynt::Funnel::Funnel`

A public set of web pages that turns visitors into Contacts. Each page (step) leads visitors toward an action: purchasing an offer, submitting a form, or booking a call.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Display name for the funnel |
| Description | string | No | - | Internal description |
| Steps | array | No | [] | Pages in the funnel |

#### Step Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Step display name |
| Slug | string | Yes | - | URL slug (unique within funnel) |
| Order | integer | Yes | - | Step order (minimum 1) |
| Sections | array | No | [] | Sections on this step |

#### Section Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Order | integer | Yes | Display order within step |
| Rows | array | No | Rows inside this section |

#### Row Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Order | integer | Yes | Display order within section |
| Columns | array | No | Columns inside this row |

#### Column Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Order | integer | Yes | Display order within row |
| Elements | array | No | Elements inside this column |

#### Element Types (Funnels and OnboardingFlows)

Elements are the building blocks inside sections. Each element has `Type`, `Order`, `Content` (varies by type), and optional `Settings`.

**Text Elements:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `headline` | `Text` (string, required) | Large heading text |
| `subheadline` | `Text` (string, required) | Smaller supporting text |
| `text` | `Text` (string, required) | Regular paragraph text |

**Media Elements:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `image` | `Alt` (string) | Display an image (upload via UI after import) |
| `video` | `Url` (url, required), `Provider` (enum: `youtube`, `vimeo`, `wistia`) | Embed a video |

**Interactive Elements:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `button` | `Text` (required), `Action` (`next_step`, `prev_step`, `external_link`), `LinkUrl` (conditional), `NextStep` (OnboardingFlow only) | Clickable button |

**Layout Elements:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `spacer` | `Height` (integer, required) | Vertical space in pixels |
| `divider` | `Style` (enum: `solid`, `dashed`, `dotted`) | Horizontal line |

**Social Proof Elements:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `faq_item` | `Question` (required), `Answer` (required) | FAQ entry |

**Form Element:**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `form` | `FormSlug` (!Ref, required), `OnSuccess.Action` (`next_step` or `stay`) | Embedded form |

**Reference Elements (checkout, booking, agreements):**

| Type | Content Properties | Description |
|------|-------------------|-------------|
| `checkout` | `OfferSlug` (!Ref, required), `PricingPlanSlug` (!Ref), `ButtonText` | One-step Stripe checkout |
| `two_step_checkout` | `OfferSlug` (!Ref, required), `PricingPlanSlug` (!Ref), `ButtonText` | Two-step checkout (captures lead info first) |
| `claim_offer` | `OfferSlug` (!Ref, required), `ButtonText` | Claim a free offer |
| `booking_link` | `BookingLinkSlug` (!Ref, required), `OnSuccess.Action` | Embedded booking calendar |
| `agreement` | `AgreementSlug` (!Ref, required), `OnSuccess.Action` | Agreement signing |

#### Example

```yaml
registration-form:
  Type: Evolynt::Funnel::Form
  Properties:
    Name: "Webinar Registration"
    Fields:
      - Type: standard
        FieldName: name
        Required: true
      - Type: standard
        FieldName: email
        Required: true
    SubmitButton:
      Text: "Save My Spot"
      BackgroundColor: "#2563eb"

webinar-funnel:
  Type: Evolynt::Funnel::Funnel
  Properties:
    Name: "Free Training Registration"
    Steps:
      - Name: "Landing Page"
        Slug: landing
        Order: 1
        Sections:
          - Order: 1
            Rows:
              - Order: 1
                Columns:
                  - Order: 1
                    Elements:
                      - Type: headline
                        Order: 1
                        Content:
                          Text: "Scale Your Business in 90 Days"
                      - Type: button
                        Order: 2
                        Content:
                          Text: "Register Now"
                          Action: next_step
      - Name: "Registration"
        Slug: form
        Order: 2
        Sections:
          - Order: 1
            Rows:
              - Order: 1
                Columns:
                  - Order: 1
                    Elements:
                      - Type: form
                        Order: 1
                        Content:
                          FormSlug: !Ref registration-form
                          OnSuccess:
                            Action: next_step
      - Name: "Thank You"
        Slug: thank-you
        Order: 3
        Sections:
          - Order: 1
            Rows:
              - Order: 1
                Columns:
                  - Order: 1
                    Elements:
                      - Type: headline
                        Order: 1
                        Content:
                          Text: "You're Registered!"
```

---

### Form

**Type:** `Evolynt::Funnel::Form`

Captures information about a Contact. Every form submission creates or updates a Contact in the CRM.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Form name (internal reference) |
| Description | string | No | - | Form description |
| Fields | array | Yes | - | List of form fields |
| SubmitButton | object | Yes | - | Submit button configuration |
| Settings | object | No | - | Visual settings |

#### Standard Field Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Type | string | Yes | - | Must be `standard` |
| FieldName | enum | Yes | - | `name`, `email`, or `phone` |
| Required | boolean | No | false | Whether the field is required |
| Placeholder | string | No | - | Placeholder text |

#### Custom Field Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Type | string | Yes | - | Must be `custom` |
| CustomFieldSlug | string (ref) | Yes | - | Reference to a CustomField |
| Required | boolean | No | false | Whether the field is required |
| LabelOverride | string | No | - | Custom label override |

#### SubmitButton Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Text | string | Yes | - | Button label text |
| BackgroundColor | string | No | - | Button background color (hex) |
| TextColor | string | No | - | Button text color (hex) |

#### Settings Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| BackgroundColor | string | No | - | Form background color (hex) |
| MaxWidth | integer | No | - | Maximum width in pixels |
| Padding | integer | No | - | Padding in pixels |
| BorderRadius | integer | No | - | Corner roundness in pixels |

#### Example

```yaml
revenue-range:
  Type: Evolynt::CRM::CustomField
  Properties:
    Label: "Revenue Range"
    FieldType: select
    Options:
      - "Under $100k"
      - "$100k - $500k"
      - "$500k - $1M"
      - "Over $1M"

application-form:
  Type: Evolynt::Funnel::Form
  Properties:
    Name: "Mastermind Application"
    Fields:
      - Type: standard
        FieldName: name
        Required: true
      - Type: standard
        FieldName: email
        Required: true
      - Type: custom
        CustomFieldSlug: !Ref revenue-range
        Required: true
    SubmitButton:
      Text: "Submit Application"
      BackgroundColor: "#0368EE"
    Settings:
      MaxWidth: 600
      BorderRadius: 12
```

---

## Automation Resources

### Workflow

**Type:** `Evolynt::Automation::Workflow`

Listens for events and automatically takes action: sending emails, adding tags, granting offers, and more.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Workflow name |
| Description | string | No | - | Description of what this workflow does |
| Triggers | array | No | [] | Events that start this workflow |
| Steps | array | No | [] | Actions the workflow takes, in order |

#### Trigger Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| EventType | string | Yes | Event that starts the workflow |
| FilterConfig | object | No | Filter for specific conditions |
| FilterConfig.Field | string | Yes | Field to check (e.g., `offer.slug`) |
| FilterConfig.Operator | string | Yes | Comparison operator (UPPERCASE) |
| FilterConfig.Value | string | Yes | Value to match (supports `!Ref`) |

#### Available Events

| Event | Description |
|-------|-------------|
| `FormSubmitted` | Contact submitted a Form |
| `PaymentSucceeded` | Payment completed |
| `PaymentFailed` | Payment failed |
| `PaymentPartiallyRefunded` | Payment partially refunded |
| `PaymentFullyRefunded` | Payment fully refunded |
| `OfferGranted` | Contact gained access to an Offer |
| `OfferRevoked` | Contact lost access to an Offer |
| `AppointmentConfirmed` | Appointment was confirmed |
| `AgreementSigned` | Agreement was signed |

#### Filter Operators

`EQUALS`, `NOT_EQUALS`, `CONTAINS`, `NOT_CONTAINS`, `GREATER_THAN`, `LESS_THAN`, `GREATER_THAN_OR_EQUALS`, `LESS_THAN_OR_EQUALS`, `INCLUDES`, `NOT_INCLUDES`, `IS_EMPTY`, `IS_NOT_EMPTY`

#### Step Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Type | string | Yes | Step type (see tables below) |
| Order | integer | Yes | Execution order (1, 2, 3...) |
| Config | object | Yes | Settings specific to this step type |
| Then | array | Conditional | Steps if condition is true (`if_else` only) |
| Else | array | Conditional | Steps if condition is false (`if_else` only) |

#### Action Steps

| Type | Config Properties | Description |
|------|-------------------|-------------|
| `send_email` | `TemplateSlug: !Ref slug` | Send email to contact |
| `send_internal_notification` | `TemplateSlug: !Ref slug`, `RecipientType: 'owner'` or `'specific_user'`, `RecipientUserId?: string` | Send email to team member |
| `add_tag` | `TagSlug: string` | Add tag to contact |
| `remove_tag` | `TagSlug: string` | Remove tag from contact |
| `grant_offer` | `OfferSlug: !Ref slug` | Give contact access to an Offer |
| `revoke_offer` | `OfferSlug: !Ref slug` | Remove contact's access to an Offer |
| `enroll_sequence` | `SequenceSlug: !Ref slug` | Start an Email Sequence |
| `update_deal` | `Field: string`, `Value: string` | Update a deal field |
| `move_deal_stage` | `PipelineSlug: string`, `StageSlug: string` | Move deal to stage |

#### Time Steps

| Type | Config Properties | Description |
|------|-------------------|-------------|
| `wait_duration` | `Days`, `Hours`, `Minutes` (numbers) | Wait for set time |
| `wait_until_time` | `Time: "HH:mm"`, `Timezone?`, `DayOfWeek?` | Wait until specific time |
| `wait_relative_to_appointment` | `Relative: "before"` or `"after"`, `Days/Hours/Minutes` | Wait relative to appointment |

#### Control Steps

| Type | Config Properties | Description |
|------|-------------------|-------------|
| `if_else` | `Condition: { Field, Operator, Value }` | Branch based on condition. Uses `Then` and `Else` arrays. |

#### Example

```yaml
onboarding-workflow:
  Type: Evolynt::Automation::Workflow
  Properties:
    Name: "New Client Onboarding"
    Triggers:
      - EventType: OfferGranted
        FilterConfig:
          Field: offer.slug
          Operator: EQUALS
          Value: !Ref mastermind-offer
    Steps:
      - Type: send_email
        Order: 1
        Config:
          TemplateSlug: !Ref welcome-email
      - Type: add_tag
        Order: 2
        Config:
          TagSlug: active-client
      - Type: wait_duration
        Order: 3
        Config:
          Days: 1
      - Type: if_else
        Order: 4
        Config:
          Condition:
            Field: contact.lesson_progress
            Operator: EQUALS
            Value: 0
        Then:
          - Type: send_email
            Order: 1
            Config:
              TemplateSlug: !Ref nudge-email
        Else:
          - Type: enroll_sequence
            Order: 1
            Config:
              SequenceSlug: !Ref engagement-sequence
```

---

### EmailTemplate

**Type:** `Evolynt::Automation::EmailTemplate`

A reusable email design for Workflows and Email Sequences.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| Name | string | Yes | Template name |
| Subject | string | Yes | Email subject line (supports variables) |
| Elements | array | Yes | Email content elements (minimum 1) |

#### Element Types

| Type | Content | Settings (defaults) |
|------|---------|---------------------|
| `headline` | `Text` | FontSize (32), TextColor (#1f2937), Alignment (left) |
| `subheadline` | `Text` | FontSize (24), TextColor (#1f2937), Alignment (left) |
| `text` | `Text` | FontSize (16), TextColor (#374151), Alignment (left), LineHeight (1.6) |
| `button` | `Text`, `LinkUrl` | BackgroundColor (#0368EE), TextColor (#ffffff), Alignment (left), BorderRadius (6), PaddingVertical (12), PaddingHorizontal (24) |
| `spacer` | (none) | Height (20) |
| `divider` | (none) | Style (solid), Color (#e5e7eb), Thickness (1) |

#### Available Variables

| Variable | Description |
|----------|-------------|
| `{{contact.name}}` | Contact's full name |
| `{{contact.email}}` | Contact's email |
| `{{contact.phone}}` | Contact's phone |
| `{{contact.custom_field.<slug>}}` | Custom field value |
| `{{deal.title}}` | Deal title |
| `{{deal.value}}` | Deal value |
| `{{workspace.name}}` | Business name |
| `{{offer.name}}` | Related offer name |
| `{{portal.url}}` | Client portal URL |
| `{{unsubscribe_url}}` | Unsubscribe link (required for compliance) |

#### Example

```yaml
welcome-email:
  Type: Evolynt::Automation::EmailTemplate
  Properties:
    Name: "Welcome Email"
    Subject: "Welcome to {{workspace.name}}, {{contact.name}}!"
    Elements:
      - Type: headline
        Order: 1
        Content:
          Text: "Welcome, {{contact.name}}!"
        Settings:
          Alignment: center
      - Type: text
        Order: 2
        Content:
          Text: "Thank you for joining us. We're excited to have you!"
      - Type: button
        Order: 3
        Content:
          Text: "Access Your Portal"
          LinkUrl: "{{portal.url}}"
        Settings:
          BackgroundColor: "#0368EE"
          Alignment: center
      - Type: divider
        Order: 4
      - Type: text
        Order: 5
        Content:
          Text: "{{unsubscribe_url}}"
        Settings:
          FontSize: 12
          TextColor: "#6b7280"
```

---

### EmailSequence

**Type:** `Evolynt::Automation::EmailSequence`

Sends automated Email Templates to Contacts over time. Contacts are enrolled via the `enroll_sequence` step in a Workflow.

#### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| Name | string | Yes | - | Sequence name |
| Description | string | No | - | Description of what this sequence does |
| Steps | array | No | [] | Emails with timing |

#### Step Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| TemplateSlug | string (ref) | Yes | Reference to the EmailTemplate to send |
| DelayAmount | integer | Yes | How long to wait (number) |
| DelayUnit | enum | Yes | `minutes`, `hours`, or `days` |
| Order | integer | Yes | Step order (1, 2, 3...) |

Each step's delay is measured from the previous step, not from enrollment. Use `DelayAmount: 0` with `DelayUnit: minutes` for immediate send.

#### Example

```yaml
onboarding-sequence:
  Type: Evolynt::Automation::EmailSequence
  Properties:
    Name: "New Client Onboarding"
    Description: "5-day drip sequence for new clients"
    Steps:
      - TemplateSlug: !Ref welcome-email
        DelayAmount: 0
        DelayUnit: minutes
        Order: 1
      - TemplateSlug: !Ref getting-started-tips
        DelayAmount: 1
        DelayUnit: days
        Order: 2
      - TemplateSlug: !Ref course-reminder
        DelayAmount: 3
        DelayUnit: days
        Order: 3
```

---

## Settings Resources

### BusinessProfile

**Type:** `Evolynt::Settings::BusinessProfile`

Configures business information for contracts, invoices, and legal documents. This is a singleton resource: the slug must always be `business-profile`.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| LegalName | string | No | Legal business name |
| Email | string | No | Business contact email |
| Phone | string | No | Business phone number |
| Address | object | No | Business address |
| Address.Line1 | string | No | Street address line 1 |
| Address.Line2 | string | No | Street address line 2 |
| Address.City | string | No | City |
| Address.State | string | No | State/province |
| Address.PostalCode | string | No | Postal/ZIP code |
| Address.Country | string | No | Country |
| Timezone | string | No | IANA timezone (e.g., `America/New_York`) |
| BusinessType | string | No | Business structure (LLC, Corporation, etc.) |
| Industry | string | No | Industry classification |
| RegistrationId | string | No | Business registration ID (EIN, VAT, etc.) |
| RegistrationIdType | string | No | Type of registration ID |
| AuthorizedRepresentative | object | No | Representative info |
| AuthorizedRepresentative.FirstName | string | No | First name |
| AuthorizedRepresentative.LastName | string | No | Last name |
| AuthorizedRepresentative.Email | string | No | Email |
| AuthorizedRepresentative.Phone | string | No | Phone |
| AuthorizedRepresentative.Position | string | No | Title/position |
| SignatureText | string | No | Default signature text for agreements |

#### Example

```yaml
business-profile:
  Type: Evolynt::Settings::BusinessProfile
  Properties:
    LegalName: "Acme Coaching LLC"
    Email: "hello@acmecoaching.com"
    Phone: "+1-555-123-4567"
    Address:
      Line1: "123 Main Street"
      City: "San Francisco"
      State: "CA"
      PostalCode: "94105"
      Country: "USA"
    Timezone: "America/Los_Angeles"
    BusinessType: "LLC"
    SignatureText: "Jane Smith, CEO"
```

---

## Validation Rules

### Amount Format

All monetary amounts are in **cents** (smallest currency unit). Do not use decimals.

| You Want to Charge | Enter This Amount |
|-------------------|-------------------|
| $97.00 | `9700` |
| $997.00 | `99700` |
| $4,997.00 | `499700` |

### Property Key Format

All property keys use **PascalCase**:

```yaml
# Correct
Properties:
  Name: "My Product"
  ProductType: course
  DefaultPortal: !Ref my-portal
  BillingInterval: monthly

# Wrong
Properties:
  name: "My Product"           # lowercase
  product_type: course         # snake_case
  default-portal: !Ref ...     # kebab-case
```

### Required Properties

Each resource type has specific required properties. Missing required properties produce a validation error. For example, every Product requires both `Name` and `ProductType`.

### Common Validation Errors

**ReferenceError:** Invalid `!Ref` target:
```
ReferenceError: !Ref 'member-portal' not found in Resources
```

**ValidationError:** Invalid property value:
```
ValidationError: Amount must be a positive integer (cents). Received: 99.97
```

**TypeError:** Wrong data type:
```
TypeError: Stages - Expected array, received string
```

## Slug Naming Rules

Resource slugs (the keys in the `Resources` map) must follow these rules:

| Rule | Requirement |
|------|-------------|
| Start character | Must start with a lowercase letter (`a-z`) |
| Allowed characters | Lowercase letters, numbers, and hyphens (`a-z`, `0-9`, `-`) |
| Uniqueness | Must be unique within the workspace |
| Max length | 255 characters |

Valid: `my-offer`, `coaching-2025`, `vip-portal`, `email-template-v2`

Invalid: `My-Offer` (uppercase), `123-offer` (starts with number), `offer_v2` (underscores), `offer.v2` (periods)

---

## Complete Example: Fitness Inner Circle

This real-world Blueprint creates a fitness membership business with a community, workout library, sales funnel, automated onboarding workflow, email sequences, and business profile.

```yaml
EvolyntVersion: "1.0"

Resources:
  # ============================================
  # OFFER
  # ============================================
  inner-circle-offer:
    Type: Evolynt::Commerce::Offer
    Properties:
      Name: "Fitness Inner Circle"
      Description: "Monthly membership with community access, weekly coaching calls, and workout library"
      DefaultPortal: !Ref inner-circle-portal
      Items:
        - Product: !Ref inner-circle-community
        - Product: !Ref workout-library
      PricingPlans:
        - !Ref inner-circle-monthly

  # ============================================
  # PRODUCT (Community)
  # ============================================
  inner-circle-community:
    Type: Evolynt::Commerce::Product
    Properties:
      Name: "Fitness Inner Circle Community"
      ProductType: community
      Description: "Private community with weekly group coaching calls"
      Community:
        Name: "Fitness Inner Circle"
        Description: "Your private community for accountability, support, and expert coaching"
        Spaces:
          - Name: "General"
            Slug: general
            Description: "Introduce yourself and connect with other members"
            Order: 1
          - Name: "Wins & Progress"
            Slug: wins
            Description: "Share your victories, big and small"
            Order: 2
          - Name: "Q&A"
            Slug: qa
            Description: "Ask questions and get answers from coaches and members"
            Order: 3
          - Name: "Resources"
            Slug: resources
            Description: "Guides, templates, and helpful materials"
            Order: 4
        Events:
          - Title: "Weekly Group Coaching Call"
            Description: "Live Q&A and coaching with Coach Sarah. Bring your questions!"
            StartAt: "2025-01-07T19:00:00-05:00"
            EndAt: "2025-01-07T20:00:00-05:00"
            LocationType: virtual
            MeetingUrl: "https://zoom.us/j/example"
            HostUserEmail: "sarah@fitlifecoaching.com"
            RecurrenceRule:
              Frequency: weekly
              Interval: 1
              DaysOfWeek: [2]
              EndType: never

  # ============================================
  # PRODUCT (Course - Workout Library)
  # ============================================
  workout-library:
    Type: Evolynt::Commerce::Product
    Properties:
      Name: "Workout Library"
      ProductType: course
      Description: "Complete workout library with upper body, lower body, full body, and recovery routines"
      Course:
        Title: "Workout Library"
        Description: "Your complete collection of follow-along workout routines"
        Modules:
          - Title: "Upper Body Workouts"
            Order: 1
            Lessons:
              - Title: "Push Day - Chest, Shoulders & Triceps"
                Order: 1
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/upper-push"
              - Title: "Pull Day - Back & Biceps"
                Order: 2
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/upper-pull"
          - Title: "Lower Body Workouts"
            Order: 2
            Lessons:
              - Title: "Quad-Focused Leg Day"
                Order: 1
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/lower-quads"
              - Title: "Glute & Hamstring Focus"
                Order: 2
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/lower-glutes"
          - Title: "Full Body Workouts"
            Order: 3
            Lessons:
              - Title: "30-Minute Full Body Blast"
                Order: 1
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/full-body-30"
              - Title: "45-Minute Strength Builder"
                Order: 2
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/full-body-45"
          - Title: "Recovery & Mobility"
            Order: 4
            Lessons:
              - Title: "Active Recovery Day"
                Order: 1
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/recovery-active"
              - Title: "Full Body Stretch Routine"
                Order: 2
                ContentType: video
                VideoEmbedUrl: "https://www.youtube.com/embed/recovery-stretch"

  # ============================================
  # PRICING PLAN
  # ============================================
  inner-circle-monthly:
    Type: Evolynt::Commerce::PricingPlan
    Properties:
      Name: "Monthly Membership"
      Description: "Recurring monthly access to the Fitness Inner Circle"
      Amount: 9700
      Currency: USD
      BillingType: subscription
      BillingInterval: monthly

  # ============================================
  # FUNNEL
  # ============================================
  inner-circle-funnel:
    Type: Evolynt::Funnel::Funnel
    Properties:
      Name: "Fitness Inner Circle Sales Page"
      Description: "Sales page for the Fitness Inner Circle membership"
      Steps:
        - Name: "Sales Page"
          Slug: sales
          Order: 1
          Sections:
            - Order: 1
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: headline
                          Order: 1
                          Content:
                            Text: "Stop Working Out Alone (And Finally Get the Results You Deserve)"
                        - Type: subheadline
                          Order: 2
                          Content:
                            Text: "Join hundreds of members getting expert coaching, accountability, and proven workouts for just $97/month"
                        - Type: video
                          Order: 3
                          Content:
                            Url: "https://www.youtube.com/watch?v=VSL_PLACEHOLDER"
                            Provider: youtube
                        - Type: two_step_checkout
                          Order: 4
                          Content:
                            OfferSlug: !Ref inner-circle-offer
                            PricingPlanSlug: !Ref inner-circle-monthly
                            ButtonText: "Join the Inner Circle - $97/month"
            - Order: 2
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: subheadline
                          Order: 1
                          Content:
                            Text: "Does this sound familiar?"
                        - Type: text
                          Order: 2
                          Content:
                            Text: |
                              You start a new workout program, excited and motivated. Week one goes great.
                              
                              Then life happens. You miss a day. Then two. Before you know it, you're back to square one, wondering why you can't stick to anything.
                              
                              Here's what's really going on:
                              
                              - You're working out alone with no one to hold you accountable
                              - When you hit a plateau, you have no idea what to change
                              - You're not sure if you're doing exercises correctly (and you're worried about injury)
                              - You don't have anyone to celebrate your wins with
                              
                              The truth is, fitness isn't just about having a program. It's about having support.
                        - Type: spacer
                          Order: 3
                          Content:
                            Height: 20
            - Order: 3
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: subheadline
                          Order: 1
                          Content:
                            Text: "What if you never had to figure it out alone?"
                        - Type: text
                          Order: 2
                          Content:
                            Text: |
                              Imagine having a certified coach available every week to answer your questions, fix your form, and adjust your program.
                              
                              Imagine being part of a community where everyone is working toward the same goals, cheering each other on, and sharing what's working.
                              
                              That's the Fitness Inner Circle.
                              
                              It's not just another membership with videos you'll never watch. It's a complete support system designed to keep you consistent, motivated, and making progress month after month.
            - Order: 4
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: subheadline
                          Order: 1
                          Content:
                            Text: "Here's everything you get as a member"
                        - Type: text
                          Order: 2
                          Content:
                            Text: |
                              ✓ Private Community Access (Value: $197/month)
                              Connect with members who get it. Share wins, ask questions, and stay accountable with people on the same journey.
                              
                              ✓ Weekly Live Coaching Calls (Value: $297/month)
                              Every Tuesday at 7pm ET, get your questions answered live. Form checks, program adjustments, mindset coaching. No question is too small.
                              
                              ✓ Complete Workout Library (Value: $97)
                              Upper body, lower body, full body, and recovery routines you can follow along anytime. New workouts added monthly.
                              
                              ✓ Direct Access to Coach Sarah
                              Post in the Q&A space anytime and get a response within 24 hours. It's like having a personal trainer in your pocket.
                              
                              Total Value: $591+/month
                        - Type: spacer
                          Order: 3
                          Content:
                            Height: 10
                        - Type: text
                          Order: 4
                          Content:
                            Text: "Your investment: Just $97/month (cancel anytime)"
            - Order: 5
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: text
                          Order: 1
                          Content:
                            Text: "\"I've joined gyms and bought programs before, but I always quit after a few weeks. The Inner Circle is different. The weekly calls keep me accountable and the community actually cares. Down 23lbs and still going strong!\" — Michelle K., 38, Marketing Manager"
                        - Type: text
                          Order: 2
                          Content:
                            Text: "\"The live calls are worth the membership alone. Coach Sarah helped me fix my squat form and my knee pain is completely gone. Plus I've made real friends in the community.\" — James T., 45, Dad of 3"
                        - Type: text
                          Order: 3
                          Content:
                            Text: "\"I was skeptical about an online community, but these people actually show up for each other. When I hit my first pull-up, I had 50+ people celebrating with me. That's priceless.\" — Andrea L., 29, Nurse"
            - Order: 6
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: faq_item
                          Order: 1
                          Content:
                            Question: "I'm a complete beginner. Is this right for me?"
                            Answer: "Absolutely. The workout library has routines for all levels, and Coach Sarah specializes in helping beginners build proper foundations. Many of our most successful members started with zero gym experience."
                        - Type: faq_item
                          Order: 2
                          Content:
                            Question: "What if I can't make the live calls?"
                            Answer: "All calls are recorded and posted in the community within 24 hours. You can also submit questions in advance and get them answered even if you can't attend live."
                        - Type: faq_item
                          Order: 3
                          Content:
                            Question: "Can I cancel anytime?"
                            Answer: "Yes, no contracts or commitments. Cancel anytime with one click in your account. We want you here because you're getting results, not because you're locked in."
                        - Type: faq_item
                          Order: 4
                          Content:
                            Question: "I don't have access to a gym. Can I still participate?"
                            Answer: "Yes! Our workout library includes home-friendly routines with minimal equipment. Many members work out with just dumbbells or resistance bands."
            - Order: 7
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: headline
                          Order: 1
                          Content:
                            Text: "Your Fitness Journey Shouldn't Be a Solo Mission"
                        - Type: text
                          Order: 2
                          Content:
                            Text: "For less than a single personal training session per month, get expert coaching, a supportive community, and a complete workout library. Join today and see what's possible when you stop going it alone."
                        - Type: two_step_checkout
                          Order: 3
                          Content:
                            OfferSlug: !Ref inner-circle-offer
                            PricingPlanSlug: !Ref inner-circle-monthly
                            ButtonText: "Join the Inner Circle - $97/month"
        - Name: "Thank You"
          Slug: thank-you
          Order: 2
          Sections:
            - Order: 1
              Rows:
                - Order: 1
                  Columns:
                    - Order: 1
                      Elements:
                        - Type: headline
                          Order: 1
                          Content:
                            Text: "Welcome to the Inner Circle!"
                        - Type: text
                          Order: 2
                          Content:
                            Text: |
                              You're officially part of the Fitness Inner Circle. Check your email for login details.
                              
                              Here's what to do first:
                              
                              1. Log into your portal and introduce yourself in the General space
                              2. Check out the workout library and pick your first routine
                              3. Mark your calendar for Tuesday at 7pm ET for the live coaching call
                              
                              Welcome to the team. We can't wait to see your progress!

  # ============================================
  # PORTAL
  # ============================================
  inner-circle-portal:
    Type: Evolynt::Portal::Portal
    Properties:
      Name: "Fitness Inner Circle Portal"
      Description: "Member portal for Fitness Inner Circle"
      ThemeConfig:
        PrimaryColor: "#10b981"

  # ============================================
  # TAG
  # ============================================
  inner-circle-member:
    Type: Evolynt::CRM::Tag
    Properties:
      Name: "Inner Circle Member"
      Description: "Active Fitness Inner Circle subscriber"

  # ============================================
  # WORKFLOW
  # ============================================
  inner-circle-purchase-workflow:
    Type: Evolynt::Automation::Workflow
    Properties:
      Name: "Inner Circle Purchase"
      Description: "Triggers when someone subscribes to the Fitness Inner Circle"
      Triggers:
        - EventType: OfferGranted
          FilterConfig:
            Field: offer.slug
            Operator: EQUALS
            Value: !Ref inner-circle-offer
      Steps:
        - Type: add_tag
          Order: 1
          Config:
            TagSlug: !Ref inner-circle-member
        - Type: enroll_sequence
          Order: 2
          Config:
            SequenceSlug: !Ref inner-circle-sequence

  # ============================================
  # EMAIL SEQUENCE
  # ============================================
  inner-circle-sequence:
    Type: Evolynt::Automation::EmailSequence
    Properties:
      Name: "Inner Circle Welcome Sequence"
      Description: "2-week welcome and engagement sequence for new members"
      Steps:
        - TemplateSlug: !Ref inner-circle-welcome-email
          DelayAmount: 0
          DelayUnit: minutes
          Order: 1
        - TemplateSlug: !Ref inner-circle-engagement-email
          DelayAmount: 3
          DelayUnit: days
          Order: 2
        - TemplateSlug: !Ref inner-circle-value-email
          DelayAmount: 7
          DelayUnit: days
          Order: 3

  # ============================================
  # EMAIL TEMPLATES
  # ============================================
  inner-circle-welcome-email:
    Type: Evolynt::Automation::EmailTemplate
    Properties:
      Name: "Inner Circle Welcome"
      Subject: "Welcome to the Fitness Inner Circle, {{contact.name}}!"
      Elements:
        - Type: headline
          Order: 1
          Content:
            Text: "You're In! Welcome to the Inner Circle"
          Settings:
            Alignment: center
        - Type: text
          Order: 2
          Content:
            Text: |
              Hi {{contact.name}},
              
              Welcome to the Fitness Inner Circle! You've just joined a 
              community of people committed to getting stronger, healthier, 
              and more confident.
              
              Here's how to make the most of your membership:
              
              **Step 1: Introduce Yourself**
              Head to the General space and tell us a bit about yourself 
              and your fitness goals. The community loves welcoming new 
              members!
              
              **Step 2: Explore the Workout Library**
              Browse our collection of follow-along workouts and pick one 
              to try this week. Start wherever feels right for your level.
              
              **Step 3: Join the Live Call**
              Every Tuesday at 7pm ET, we meet for live Q&A and coaching. 
              Bring your questions! Replays are always available if you 
              can't make it live.
              
              This is your time. Let's make it count.
        - Type: button
          Order: 3
          Content:
            Text: "Enter Your Portal"
            LinkUrl: "{{portal.url}}"
          Settings:
            Alignment: center
            BackgroundColor: "#10b981"
        - Type: spacer
          Order: 4
          Settings:
            Height: 30
        - Type: text
          Order: 5
          Content:
            Text: "Questions? Just reply to this email. We're here to help."
        - Type: divider
          Order: 6
        - Type: text
          Order: 7
          Content:
            Text: "{{unsubscribe_url}}"
          Settings:
            FontSize: 12
            TextColor: "#6b7280"

  inner-circle-engagement-email:
    Type: Evolynt::Automation::EmailTemplate
    Properties:
      Name: "Inner Circle Engagement Check-in"
      Subject: "Have you posted in the community yet?"
      Elements:
        - Type: headline
          Order: 1
          Content:
            Text: "Quick Check-in"
        - Type: text
          Order: 2
          Content:
            Text: |
              Hi {{contact.name}},
              
              You've been a member for a few days now. How's it going?
              
              I wanted to remind you about tomorrow's live coaching call. 
              It's happening Tuesday at 7pm ET, and it's a great chance to:
              
              - Get your form checked on any exercise
              - Ask questions about your program
              - Connect with other members live
              
              If you haven't introduced yourself in the community yet, now's 
              a perfect time. The members here are incredibly supportive, and 
              you'll get way more out of your membership when you engage.
              
              Remember: You don't have to go through this alone anymore.
        - Type: button
          Order: 3
          Content:
            Text: "Join the Community"
            LinkUrl: "{{portal.url}}"
          Settings:
            Alignment: center
            BackgroundColor: "#10b981"
        - Type: divider
          Order: 4
        - Type: text
          Order: 5
          Content:
            Text: "{{unsubscribe_url}}"
          Settings:
            FontSize: 12
            TextColor: "#6b7280"

  inner-circle-value-email:
    Type: Evolynt::Automation::EmailTemplate
    Properties:
      Name: "Inner Circle Value Reminder"
      Subject: "One week in - how are you feeling?"
      Elements:
        - Type: headline
          Order: 1
          Content:
            Text: "One Week Down!"
        - Type: text
          Order: 2
          Content:
            Text: |
              Hi {{contact.name}},
              
              You've been part of the Inner Circle for a week now. 
              Congrats on taking action!
              
              I wanted to share a quick win from the community:
              
              "I've tried so many programs but always quit. The 
              accountability here is different. I actually look forward 
              to posting my workouts because I know people will celebrate 
              with me. Finally seeing real progress!" - Sarah M.
              
              That's what this community is all about. Real people, real 
              support, real results.
              
              If you haven't been as active as you'd like, that's okay. 
              Every day is a fresh start. Here's a simple challenge:
              
              1. Do one workout from the library this week
              2. Post about it in Wins & Progress
              3. Celebrate someone else's win
              
              Small steps lead to big transformations.
              
              See you inside!
        - Type: button
          Order: 3
          Content:
            Text: "Check Out Member Wins"
            LinkUrl: "{{portal.url}}"
          Settings:
            Alignment: center
            BackgroundColor: "#10b981"
        - Type: spacer
          Order: 4
          Settings:
            Height: 20
        - Type: text
          Order: 5
          Content:
            Text: |
              P.S. If the Inner Circle isn't what you expected, you can 
              cancel anytime with one click. But I hope you'll give it a 
              real shot first. The members who stick around for 30 days 
              almost never leave.
        - Type: divider
          Order: 6
        - Type: text
          Order: 7
          Content:
            Text: "{{unsubscribe_url}}"
          Settings:
            FontSize: 12
            TextColor: "#6b7280"

  # ============================================
  # BUSINESS PROFILE
  # ============================================
  business-profile:
    Type: Evolynt::Settings::BusinessProfile
    Properties:
      LegalName: "FitLife Coaching LLC"
      Email: "hello@fitlifecoaching.com"
      Phone: "+1-555-987-6543"
      Timezone: "America/New_York"
      BusinessType: "LLC"
      Industry: "Fitness & Health"
      SignatureText: "Coach Sarah, FitLife Coaching"
```

## JSON Schemas

Machine-readable JSON Schema files are available for validating Blueprints before import:

| Resource Type | Schema File |
|--------------|-------------|
| `Evolynt::CRM::Pipeline` | [`schemas/pipeline.json`](schemas/pipeline.json) |
| `Evolynt::CRM::Tag` | [`schemas/tag.json`](schemas/tag.json) |
| `Evolynt::CRM::BookingLink` | [`schemas/booking-link.json`](schemas/booking-link.json) |
| `Evolynt::CRM::CustomField` | [`schemas/custom-field.json`](schemas/custom-field.json) |
| `Evolynt::CRM::Agreement` | [`schemas/agreement.json`](schemas/agreement.json) |
| `Evolynt::Commerce::Product` | [`schemas/product.json`](schemas/product.json) |
| `Evolynt::Commerce::PricingPlan` | [`schemas/pricing-plan.json`](schemas/pricing-plan.json) |
| `Evolynt::Commerce::Offer` | [`schemas/offer.json`](schemas/offer.json) |
| `Evolynt::Portal::Portal` | [`schemas/portal.json`](schemas/portal.json) |
| `Evolynt::Portal::OnboardingFlow` | [`schemas/onboarding-flow.json`](schemas/onboarding-flow.json) |
| `Evolynt::Funnel::Funnel` | [`schemas/funnel.json`](schemas/funnel.json) |
| `Evolynt::Funnel::Form` | [`schemas/form.json`](schemas/form.json) |
| `Evolynt::Automation::Workflow` | [`schemas/workflow.json`](schemas/workflow.json) |
| `Evolynt::Automation::EmailTemplate` | [`schemas/email-template.json`](schemas/email-template.json) |
| `Evolynt::Automation::EmailSequence` | [`schemas/email-sequence.json`](schemas/email-sequence.json) |
| `Evolynt::Settings::BusinessProfile` | [`schemas/business-profile.json`](schemas/business-profile.json) |

## More Resources

- [Full Blueprint Documentation](https://docs.evolynt.com) - Complete reference with detailed examples for every resource type
- [Blueprint Examples](https://docs.evolynt.com/docs/blueprints/examples) - Real-world Blueprints for fitness, ad agencies, content creators, freelancers, and more
- [Getting Started Guide](https://docs.evolynt.com/docs/blueprints/getting-started) - Step-by-step guide to importing your first Blueprint
- [Concepts Guide](https://docs.evolynt.com/docs/blueprints/concepts) - Deep dive into `!Ref`, `!Sub`, dependency order, and validation
