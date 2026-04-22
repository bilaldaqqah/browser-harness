# allegiant

Allegiant Air (allegiantair.com) direct-distribution booking flow. No OTA alternative — pure-direct airline.

## Initial load

- Chrome with a warm profile passes Allegiant's bot check on first hit. `page_info()` shows `title: "Just a moment..."` for ~5–10s, then the real title loads. Sleep 8s after `new_tab` before doing anything.
- No Akamai / PerimeterX / CAPTCHA observed when using the user's real Chrome session (stock anti-bot header check only). Stagehand/Playwright-out-of-the-box would likely get stuck on this first screen — the thesis holds.

## Two pre-checkout overlays

1. **Allways Rewards Visa promo** — dismiss with `document.querySelector('.Popup__CloseIcon-sc-1kasz48-2').click()`.
2. **OneTrust cookie banner** — click the button whose trimmed text is `"I understand"`.

On the `/payment` page a **third** Allways modal appears with an X in the top-right corner (~x=1610 y=100 at 1699 viewport). It's not in the main DOM; coordinate click works, `aria-label="Close"` does not match it.

## Checkout URL structure

6-step funnel, predictable paths under a per-session hash:

```
/booking/<session-hash>/flights?tt=ONEWAY&o=AVL&d=FLL&ta=1&tc=0&tis=0&til=0&ds=2026-05-08&c=1&h=1
/booking/<session-hash>/travelers
/booking/<session-hash>/seats
/booking/<session-hash>/ancillaries     (displayed as step 4 "Bags")
/booking/<session-hash>/cars
/booking/<session-hash>/payment
```

Query params on `/flights`: `tt=ONEWAY|ROUNDTRIP`, `o`/`d`=IATA, `ta|tc|tis|til` adults/children/infant-seat/infant-lap, `ds`=YYYY-MM-DD depart, `de`=YYYY-MM-DD return. Starting from this URL directly skips the homepage search form but you still need a valid session hash — can't deeplink cold.

## Stable selectors

| Field | Selector | Notes |
|---|---|---|
| Origin combo | `#select-origin` | react-select; type IATA, ArrowDown, Enter |
| Destination combo | `#select-destination` | same pattern; click the wider wrapper (`.css-1hwfws3`) because the input itself is 2px wide |
| Departure date input | `#departure_date` | opens a 2-month calendar |
| Return date | `#return_date` | only when Round Trip |
| Calendar day | `[aria-label="Friday, May 8th 2026"]` | verbose, includes weekday + ordinal + year |
| Traveler name fields | `#adults.0.first-name`, `#adults.0.middle-name`, `#adults.0.last-name` | CSS.escape the dots if using `querySelector` |
| DOB | `#adults.0.dob-month`, `#adults.0.dob-day` (react-select), `#adults.0.dob-year` (plain input) | |
| Phone | `#adults.0.primary-phone-number` | placeholder `123-456-7890` |
| Email | `#adults.0.email` | |
| Card PAN | `#card-number` | **NOT iframed** — raw input, accepts `type_text`. Auto-formats `xxxx-xxxx-xxxx-xxxx`. |
| CVV | `#card-cvv` | raw input |
| Cardholder | `#card-holder-name` | pre-filled from traveler `TEST PASSENGER` |
| Expiration | `[class*="-control"]` whose `textContent` is `"Month"` / `"Year"` | react-select (see gotcha below) |
| Billing street | `#address-line-1`, `#address-line-2` | |
| City | `#city` | |
| State | `#state` react-select | |
| Zip | `#zip-code` | |
| Billing phone | `#phone-number` | |
| Receipt email | `#email-address` (second occurrence, y>500) | there are two `#email-address` elements on the payment page — login form and receipt |

## Selectors that fail

- `[role="listbox"]` on the react-select dropdowns — no role is set. Use `.css-1ixjdef-menu` and enumerate `.css-11unzgr > div` descendants.
- Typing into the month/year react-select inputs does **not** filter options. Typing `"12"` lands on January, `"2029"` lands on 2026. You must **click the option div directly**. See gotcha below.

## Gotchas

- **react-select typeahead is broken for numeric expiry selects.** Don't type the value and press Enter. Instead:
  1. Click the control to open the menu.
  2. Find the leaf `<div>` whose `textContent.trim()` exactly equals the target (e.g. `"December"`, `"2029"`) and whose `offsetParent !== null` and `children.length === 0`.
  3. Click that div via coordinates — `.click()` alone sometimes misses react-select's mousedown handler. The helper above (coordinate click) is reliable.
- **Month dropdown options are not all in the initial viewport.** Scrolling the option into view before clicking is necessary if the container clips.
- **There are two Continue buttons on Bags step.** A `HOLD ON!` baggage upsell popup renders a second `<button>Continue</button>` with class `fQjMhp` (non-footer). The footer button is `.PageFooter__ContinueButton-sc-12arybe-1`. Click the footer → popup appears → click the non-footer one.
- **The "No thanks" rental-car skip is an `<a>`, not a button.** Text starts with `"No thanks, I don't need a deal on ground"`. Skip via text match.
- **Payment form is NOT Stripe Elements / Spreedly / hosted iframe.** Raw inputs. Good for automation; watch for PCI impact if you're doing anything real with this.
- **Trip-level receipt email shares `#email-address` with a hidden login form on the payment page.** Query by id + `getBoundingClientRect().y > 500` to pick the right one.

## Waits beyond `wait_for_load()`

- 8s hard sleep after first `new_tab` — Cloudflare-style "Just a moment..." gate.
- 3s after clicking step-boundary `Continue` — the next route renders after a short spinner that `wait_for_load` doesn't catch.
- 0.6–0.8s after opening any react-select before querying its `.css-1ixjdef-menu`.

## Private APIs observed

None mined (coordinate clicks were fast enough). Worth capturing the `/flights` search POST next time — likely a JSON endpoint under the `/booking/<hash>/` tree that returns the fare grid.

## Traps

- Two consecutive `Continue` button clicks on Travelers are sometimes needed — the first fires but the form validates asynchronously. If still on `/travelers` after a click, re-scroll the button into view and re-click. Don't treat the first no-op as a failure.
- `screenshot()` images are downscaled roughly 2× from viewport width. Don't infer click coords from the PNG; always `getBoundingClientRect()`.
