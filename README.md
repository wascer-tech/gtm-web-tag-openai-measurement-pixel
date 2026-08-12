# Wascer OpenAI Ads Pixel

A Google Tag Manager custom template for web containers. It loads the OpenAI Ads
measurement pixel and reports browser events to your OpenAI pixel.

## What it does

* Creates the `oaiq` queue before the SDK loads, so no call is lost while the
  script downloads.
* Loads `https://bzrcdn.openai.com/sdk/oaiq.min.js` once per page, even when the
  tag fires again in a single page app.
* Sends `init` with your pixel ID and, when you enable it, hashed user data.
* Sends `measure` with the event name, the event data and the options that
  OpenAI expects.
* Reads `value`, `currency` and `items` from the GA4 data layer and converts the
  amounts to the minor unit of the currency.
* Waits for `ad_storage` consent when you ask it to, and sends the event as soon
  as the visitor grants it.

## Install

1. In your web container, open **Templates**, then **Tag Templates**, then
   **New**.
2. In the template editor menu, choose **Import**, and pick `template.tpl`.
3. Save the template.
4. Open **Tags**, create a new tag and choose **Wascer OpenAI Ads Pixel**.
5. Paste your pixel ID, pick the event, and attach a trigger.

For a page view, use the **All Pages** trigger. For ecommerce events, use a
custom event trigger that matches the data layer event that pushes `items` and
`value`.

## Fields

### Configuration

| Field | What it does |
|---|---|
| OpenAI Pixel ID | Identifies your pixel. You find it in the conversions tab of OpenAI Ads Manager. |
| Event name | Choose between a standard event and a custom one. |
| Event | The standard event to report. Defaults to `page_viewed`. |
| Custom event name | The name of your own event. Letters, numbers, underscores and dashes, up to 64 characters. |
| Event ID | The key that ties this hit to the server hit. See the deduplication section. |

### Data layer

| Field | What it does |
|---|---|
| Read amount, currency and items from the data layer | Picks up `value`, `currency` and `items`, falling back to the same keys inside `ecommerce`. |
| Send hashed user data from the data layer | Picks up `user_data` and turns it into the fields OpenAI accepts. |

### User data

A table where you set the identifiers by hand. Anything you type here wins over
the data layer. The field names are the ones OpenAI accepts: `email_sha256`,
`external_id_sha256`, `country`, `city` and `zip_code`.

### Consent

| Field | What it does |
|---|---|
| Consent mode | `Wait for ad_storage consent` holds the event until the visitor grants it. `Always send` fires right away. |
| Opt this user out of personalization | Sends `opt_out: true` in the event options. |

### Advanced

`Log SDK debug output to the browser console` turns on the SDK debug flag and
prints the payload the template builds. Use it while you set the tag up, then
turn it off.

## Advanced matching

The template hashes what needs hashing and leaves alone what is already hashed.

| Field | Normalization | Hashed |
|---|---|---|
| `email_sha256` | Trim, then lowercase | Yes, SHA-256 in lowercase hex |
| `external_id_sha256` | Trim | Yes, SHA-256 in lowercase hex |
| `country` | Trim, then uppercase | No |
| `city` | Trim, then lowercase, capped at 128 characters | No |
| `zip_code` | Trim, capped at 32 characters | No |

A value that already looks like a SHA-256 digest, meaning 64 hexadecimal
characters, goes through untouched. So you can feed the template either the raw
email or the hash your backend produced.

From the GA4 `user_data` object the template reads:

* `email_sha256`, `sha256_email_address`, `email_address` or `email` into
  `email_sha256`
* `external_id_sha256`, `external_id` or `user_id` into `external_id_sha256`
* `country`, `city` and `zip_code` at the top level
* `address.country`, `address.city` and `address.postal_code`, including the
  case where `address` is an array and the first entry holds the values

## Deduplication with the server tag

Run the pixel and the OpenAI Conversions API side by side and OpenAI counts one
conversion instead of two, as long as both hits carry the same ID. Put the same
value in the **Event ID** field here and in the `id` field of the server tag. A
transaction ID works well for purchases. For other events, a GTM variable that
generates a value once per event works too.

## Events

Every event carries a `type` in its data. The template fills it for you.

| Event | `type` | Data it may carry |
|---|---|---|
| `page_viewed` | `contents` | `amount`, `currency`, `contents[]` |
| `contents_viewed` | `contents` | `amount`, `currency`, `contents[]` |
| `items_added` | `contents` | `amount`, `currency`, `contents[]` |
| `checkout_started` | `contents` | `amount`, `currency`, `contents[]` |
| `order_created` | `contents` | `amount`, `currency`, `contents[]` |
| `lead_created` | `customer_action` | `amount`, `currency` |
| `registration_completed` | `customer_action` | `amount`, `currency` |
| `appointment_scheduled` | `customer_action` | `amount`, `currency` |
| `subscription_created` | `plan_enrollment` | `plan_id`, `amount`, `currency`, `contents[]` |
| `trial_started` | `plan_enrollment` | `plan_id`, `amount`, `currency`, `contents[]` |
| Custom event | `custom` | `plan_id`, `amount`, `currency`, `contents[]` |

`app_installed` and `app_opened` are missing on purpose. The web pixel does not
accept them. Send those from the server tag.

## Amounts

OpenAI reads `amount` in the minor unit of the currency, so 2599 means 25.99 US
dollars. The GA4 data layer carries the regular unit, so the template converts.

* Currencies with no decimals keep the number as it is: BIF, CLP, DJF, GNF, IDR,
  ISK, JPY, KMF, KRW, MGA, PYG, RWF, UGX, VND, VUV, XAF, XOF and XPF.
* Currencies with three decimals multiply by 1000: BHD, IQD, JOD, KWD, LYD, OMR
  and TND.
* Everything else multiplies by 100.

When the data layer has no `value`, the template adds up the items instead.
It never sends an `amount` without a `currency`, because OpenAI rejects that.

## Content Security Policy

If the site runs a CSP, allow these origins:

```
script-src  https://bzrcdn.openai.com
connect-src https://bzr.openai.com https://bzrcdn.openai.com
img-src     https://bzr.openai.com
```

## Reference

OpenAI measurement pixel documentation:
https://developers.openai.com/ads/measurement-pixel
