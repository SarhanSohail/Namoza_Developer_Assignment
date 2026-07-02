# Task 01: GTM Event Schema - OrthoNow

## 1. GTM Event Schema

The table below covers every interaction listed in the brief: the 3-step booking form, Call Now buttons, the WhatsApp chat widget, the gated Download Patient Guide PDF, the 9 clinic location pages and blog article scroll depth.

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_step_complete` (step 1) | Custom Event trigger, listening for `dataLayer.push({event: 'booking_step_complete', step_number: 1})` fired on the front-end when the user submits clinic and specialty selection | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (entry step and step 1 to step 2 transition); feeds a "Specialty Interest" audience used for specialty-specific remarketing |
| `booking_step_complete` (step 2) | Custom Event trigger, listening for `dataLayer.push({event: 'booking_step_complete', step_number: 2})` fired when name, phone and preferred date are submitted | `step_number`, `step_name`, `clinic_location`, `preferred_date` (do not pass raw name or phone into GA4, these are PII) | Funnel Exploration (step 2 to step 3 transition); feeds a "High Intent, Not Booked" remarketing audience for users who drop before confirmation |
| `booking_confirmed` (step 3, final booking submission) | Custom Event trigger, listening for `dataLayer.push({event: 'booking_confirmed', step_number: 3})` fired on successful booking confirmation | `step_number`, `clinic_location`, `specialty`, `booking_id` (non-PII reference ID, not name/phone) | Conversions report (marked as a key event); this is the primary conversion event for the property |
| `call_now_click` | Click trigger with a Click URL condition containing `tel:` (this matches all `<a href="tel:...">` Call Now links across the site), plus a Page Path condition to capture which page the click came from | `page_location`, `page_type` (homepage / clinic_page / landing_page), `clinic_location` (where applicable) | Engagement report, broken out by `page_type` to compare call intent across homepage, clinic pages and the paid landing page; feeds a "Call Intent" audience |
| `whatsapp_widget_click` | Click trigger, Click ID or Click Class filter scoped to the floating WhatsApp button element, with a Click URL condition matching `wa.me` | `page_location`, `widget_position` (floating_button), `clinic_location` (if determinable from page context) | Engagement report; feeds a "WhatsApp Engaged" audience for retargeting users who showed conversation intent but did not book |
| `patient_guide_form_submit` | Custom Event trigger, listening for a `dataLayer.push({event: 'patient_guide_form_submit'})` fired by the front-end once the gating form (name + phone) validates and submits successfully | `form_location` (page where the gate form sits), `guide_title`, `lead_type` (set to `content_download`) | Conversions report (marked as a secondary, lower-intent key event); feeds a "Content Downloaders" nurture audience, separate from the booking funnel audience |
| `patient_guide_download` | Custom Event trigger, listening for a `dataLayer.push({event: 'patient_guide_download'})` fired only after the gate form succeeds and the actual PDF download is triggered (kept separate from the form submit event to isolate form friction from file delivery issues) | `guide_title`, `file_url`, `form_location` | Engagement report, used to confirm that gated submissions actually result in a delivered asset (diagnostic event, not a conversion in itself) |
| `clinic_location_page_view` | Page View trigger (or History Change trigger if the site is a single-page application), condition matching the URL pattern for the 9 clinic location pages, for example `/clinics/*` | `clinic_location`, `clinic_city` (Bengaluru / Hyderabad / Chennai), `page_location` | Pages and Screens report, with `clinic_location` and `clinic_city` set as a custom dimension to break down traffic per clinic; feeds city-level remarketing audiences |
| `blog_scroll_depth` | Scroll Depth trigger, configured with vertical scroll percentage thresholds at 25, 50, 75 and 90 percent, scoped to blog article page paths only | `percent_scrolled`, `page_location`, `article_title` (pulled from a page-level data layer variable, not hardcoded per article) | Engagement report, used to build a "Content Engaged Reader" audience (for example, users who hit the 75 percent threshold) for blog-to-booking remarketing campaigns |

## 2. Booking Form Funnel Drop-Off Tracking

The 3-step booking form is a JavaScript-driven, client-side multi-step form. GTM cannot natively detect a step transition inside a form like this. GTM's triggers only fire in response to standard browser events (clicks, page loads, form submissions to a new URL, history changes) or in response to a `dataLayer.push()` that the page itself executes. A step change inside a single-page form does not produce a URL change, a new page load or a native DOM submit event that GTM can listen to on its own. This means the front-end developer must manually instrument each step transition in the application code by firing a `dataLayer.push()` at the exact moment a user completes a step and GTM is then configured with a Custom Event trigger to listen for that specific event name.

### Step-by-step trigger and dataLayer design

**Step 1: Location and specialty selection**

GTM trigger: Custom Event trigger matching `event: 'booking_step_complete'` with a trigger condition where `step_number` equals 1.

DataLayer push fired by the front-end the moment the user completes location and specialty selection and advances to step 2:

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee Pain"
}
```

**Step 2: Contact details and preferred date**

GTM trigger: Custom Event trigger matching `event: 'booking_step_complete'` with a trigger condition where `step_number` equals 2.

DataLayer push fired by the front-end the moment the user submits name, phone and preferred date and advances to step 3. Name and phone are intentionally excluded from the push since these are PII and should not be sent into GA4.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_submitted",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-04"
}
```

**Step 3: Booking confirmation**

GTM trigger: Custom Event trigger matching `event: 'booking_confirmed'` (named distinctly from the step-complete events since this is the terminal conversion action, not just another step transition).

DataLayer push fired by the front-end once the booking is confirmed server-side and the confirmation screen renders:

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee Pain",
  "booking_id": "ORN-2026-00481"
}
```

### Briefing the front-end developer

Since GTM is purely a listener and cannot reach into the form's internal state, the brief to the front-end developer for step 2, as an example, is as follows: at the exact point in the code where the step 2 form passes client-side validation and the application transitions the user to the step 3 view, insert a `window.dataLayer.push()` call with the exact object shown above, before or immediately as the step 3 view renders. The event name and parameter names must match exactly what is specified here, since GTM's Custom Event trigger is doing an exact string match on `event: 'booking_step_complete'` and the `step_number` value. If the front-end developer renames a field or fires the push before validation passes (for example, on button click rather than on successful validation), the funnel data will either be incomplete or will overcount step completions that were never actually valid.

### Surfacing this in GA4 Funnel Exploration

In GA4, build a Funnel Exploration using the three `booking_step_complete` / `booking_confirmed` events as funnel steps, in order: `booking_step_complete` filtered to `step_number = 1` as the entry step, `booking_step_complete` filtered to `step_number = 2` as the second step and `booking_confirmed` as the final step. Set the funnel to "open funnel" rather than "closed funnel" initially to understand true entry behavior, then switch to "closed funnel" to measure strict step-by-step progression. GA4 will display the percentage drop-off between each consecutive step directly in the funnel visualization, so it becomes immediately visible whether the larger drop is between step 1 and step 2 (location and specialty selection failing to convert into contact details submission) or between step 2 and step 3 (contact details submitted but booking never confirmed which usually points to a server-side or payment/scheduling failure rather than a form UX failure). Segment the funnel by `clinic_location` and by `specialty` to identify whether drop-off is concentrated at specific clinics or specialties which is useful since OrthoNow operates across 9 locations and drop-off patterns are unlikely to be uniform.

## 3. Recommended Google Ads Conversion Import

The single event to import into Google Ads as a conversion action is `booking_confirmed` (step 3 of the booking funnel).

The other candidate events in this schema (`call_now_click`, `whatsapp_widget_click`, `patient_guide_form_submit`) all represent engagement or soft-lead signals, not completed transactions. A Call Now click only confirms that a user opened their phone dialer; it does not confirm a call took place or that an appointment resulted from it. A WhatsApp widget click only confirms that a chat window opened. A Patient Guide download confirms interest in educational content which is a useful top-of-funnel nurture signal but sits well above actual purchase intent. `booking_confirmed` is the only event in this schema that represents a fully completed transaction, that is, a patient has selected a clinic and specialty, provided contact details and a preferred date and the booking has been confirmed server-side with a `booking_id`.

Importing a softer event such as `patient_guide_form_submit` as the optimization target would train Google Ads' bidding algorithms to find more users who download a PDF, not more users who actually book a consultation. Since OrthoNow's stated problem is a landing page that converts at 2.1 percent against a 6 to 8 percent benchmark and since performance marketing has been "flying blind" without proper tracking, the optimization target needs to be tied directly to the business outcome that matters which is a confirmed appointment. `booking_confirmed` is therefore the only event in this schema that should be marked as a primary conversion action; the remaining events are better suited to GA4 audiences for remarketing rather than direct bid optimization in Google Ads.

## 4. Implementation Notes

Two setup details that sit outside the schema table itself but affect whether it works correctly in practice:

**Trigger firing frequency.** For `call_now_click` and `whatsapp_widget_click`, the GTM trigger should be configured to fire "Once per page" rather than the default "Once per event" (all elements) or "Every occurrence." A user can tap Call Now multiple times while deciding whether to call and without this setting each tap logs as a separate event, inflating the Engagement report and the Call Intent audience with duplicate signals from a single visit.

**Marking the key event in GA4.** The table lists `booking_confirmed` as feeding the Conversions report but this does not happen automatically. In GA4 Admin, under Events, `booking_confirmed` must be manually toggled as a Key Event before it will appear as a conversion and before it can be imported into Google Ads. This is a one-time manual step, not something GTM or the dataLayer push handles on its own.
