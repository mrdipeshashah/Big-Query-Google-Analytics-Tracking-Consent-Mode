# OVERVIEW
This repository contains Big Query code using Google Analytics raw data allowing to track **Consent Mode v2**. It is engineered specifically to validate **Consent Mode v2** configurations, and spot compliance leaks at scale. This repository also explains the finer details in how Big Query tracks consent mode 

## DASHBOARD

To monitor the impact of consent mode v2. this dashboard (https://datastudio.google.com/reporting/c9121e14-d639-4ac8-87ad-21b37f69fe86) will help quantify the impact 

## METRICS TO TRACK

The metrics come from the event-view code shared that will provide the data to build the following metrics: 

### METRIC DICTIONARY 

| ID | Metric Name | Type | Calculation Logic / Expression | Rule |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **Consented Sessions** | Volume | `SUM(total_sessions)` conditioned by Explicit Consent filter rules. | Inlcude Filter = `analytics_storage = 'Yes'` & `event_name = 'session_start'` |
| **2** | **Cookieless Recovered Sessions** | Volume | `SUM(total_sessions)` conditioned by Anonymous Mode filter rules. | Inlcude Filter = `analytics_storage = 'No'` & `event_name = 'session_start'` |
| **3** | **Total Baseline Sessions** | Volume | `SUM(total_sessions)` conditioned by Baseline Entry filter rules. | Inlcude Filter = `event_name = 'session_start'` |
| **4** | **Standard Consent Rate** | Calculated | `SUM(CASE WHEN analytics_storage = 'Yes' THEN unique_users ELSE 0 END) / SUM(unique_users)` | n/a |
| **5** | **Advanced Consent Rate** | Calculated | `SUM(CASE WHEN analytics_storage = 'No' THEN total_events ELSE 0 END) / SUM(total_events)` | n/a |
| **6** | **Critical Leaks** | Calculated | `SUM(CASE WHEN analytics_storage = 'null' AND event_name = 'session_start' THEN total_sessions ELSE 0 END)` | n/a |

## HOW CONSENT MODE V2 IS RECOGNISED 
When analyzing raw event data inside Google BigQuery, understanding the architectural shifts between Consent Mode v1 and v2 is vital. The platform leverages behavioral fingerprints, state rulesets, and downstream variable mapping to process v2 data:

1. **Multi-Parameter Parameter Binding (`privacy_info.ads_storage`):**
   In a certified Consent Mode v2 implementation (e.g., CookieHub, OneTrust), the user's frontend interaction with an advertising or marketing toggle simultaneously controls three backend parameters: `ads_storage`, `ad_user_data`, and `ad_personalization`. Google bundles the evaluation of these components and updates the state within the existing `privacy_info.ads_storage` database field. 
   
2. **The Advanced Consent Mode Fingerprint (`analytics_storage = 'No'`):**
   * **Under v1 Rules:** If a user clicked "Decline", tags were completely blocked on the client side. **Zero rows were ever written to BigQuery.**
   * **Under Advanced v2 Rules:** Google introduces secure **cookieless pings**. When a user denies consent, GA4 still securely transmits hit-level data to the cloud database while stripping out user identifiers. 
   
   *The query results displaying an explicit `No` status for analytics or advertising storage is structural database proof that the engine recognizes and executes the Advanced Consent Mode v2 framework.*

3. **Database Flag Population States:**
   * **`Yes` / `No` Strings:** Confirms Consent Mode v2 is active and passing structured evaluations.
   * **`null` / Blank Cells:** Indicates unassigned state tracking where tags fired entirely outside of the privacy control wrapper.

## THE AUDIT MATRIX: FRONTEND V CLOUD WAREHOUSE 

This master reference grid maps out exactly how browser-layer network requests (`gcs` string payloads) cross-reference to BigQuery outputs, with technical translation for each combination.

| GCS Payload Code | Ad Storage State | Analytics State | BigQuery `ads_storage` | BigQuery `analytics_storage` | What it means |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **G111** | 🟢 Granted | 🟢 Granted | **`Yes`** | **`Yes`** | **Full Active Consent:** Tags read and write cookies freely. If this state fires *before* a user interacts with the banner, it represents an illegal pre-consent tracker leak. |
| **G100** | 🔴 Denied | 🔴 Denied | **`No`** | **`No`** | **Advanced Mode Default / Opt-Out:** Cookies are blocked. Tags send completely anonymous cookieless pings to BigQuery for backend conversion modeling. |
| **G101** | 🔴 Denied | 🟢 Granted | **`No`** | **`Yes`** | **Partial Consent (Analytics Only):** Google Analytics runs natively with functional cookies, but all Google Ads remarketing lists and personalization arrays are explicitly blocked. |
| **G110** | 🟢 Granted | 🔴 Denied | **`Yes`** | **`No`** | **Partial Consent (Marketing Only):** Paid advertising loops track conversions via cookies normally, but standard website behavior analytics are restricted to cookieless tracking profiles. |
| **N/A** | 🛑 Null | 🛑 Null | **`null`** | **`null`** | **Critical Tracking Leakage:** The engine recorded tracking records, but the fields are entirely unassigned. Tags fired before the container received a programmatic default or update signal from the banner. |

### The Audit Matrix: Frontend Payload vs. Cloud Data Warehouse

This master reference grid maps out exactly how browser-layer network requests (`gcs` and `gcd` string payloads) cross-reference to BigQuery outputs, along with the technical translation for each combination.

| GCS Code | GCD Signature | Ad Storage State | Analytics State | BigQuery `ads_storage` | BigQuery `analytics_storage` | What it means / Cookieless Output |
| :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **G111** | `13r3r...` <br> `13v3v...` | 🟢 Granted | 🟢 Granted | **Yes** | **Yes** | **Full Active Consent:** Tags read/write cookies freely. If this fires *before* interaction, it represents an illegal pre-consent tracker leak. |
| **G100** | `13q3q...` | 🔴 Denied | 🔴 Denied | **No** | **No** | **Advanced Mode Default / Opt-Out:** Cookies are blocked. Tags send anonymous cookieless pings. `user_pseudo_id` and `ga_session_id` are exported as `null`. |
| **G101** | `13q3q...r...` | 🔴 Denied | 🟢 Granted | **No** | **Yes** | **Partial Consent (Analytics Only):** Google Analytics runs natively with functional cookies, but all Google Ads remarketing and personalization arrays are explicitly blocked. |
| **G110** | `13r3r...q...` | 🟢 Granted | 🔴 Denied | **Yes** | **No** | **Partial Consent (Marketing Only):** Paid advertising loops track conversions natively, but website behavior analytics are restricted to cookieless tracking profiles. |
| **N/A** | `13l3l3l3l1l1` | 🔴 Null | 🔴 Null | **null** | **null** | **Integration Mismatch (Critical Leak):** The tag manager fired before receiving a default or update status signal from the banner. |

> 📌 **Audit Pro-Tip:** Do not rely solely on the BigQuery `privacy_info` columns during an audit. An integration mismatch (`13l3l3...`) will often result in a `null` or unconfigured state in the data warehouse, which can be misread as a compliant advanced mode opt-out. Always cross-reference the frontend `gcd` network signature to confirm if GTM actually received the user's choice.

## UNDERSTANDING THE PRIVACY_INFO FIELDS
privacy_info fields in Big Query are split into 3: 

**analytics_storage**

What it Means & Its Role: 

This field determines whether Google Analytics is legally allowed to store or read cookies on the user's browser to track their behavior, count visits, and measure page engagement.

What the Values Mean:

Yes: Full Tracking Granted. Google Analytics drops a persistent cookie (_ga) on the user’s device. All behavioral actions (clicks, scrolls, conversions) are permanently stitched to a unique user_pseudo_id.

No: Cookieless Modeling Mode Active. No behavioral cookies are read or written. However, if the site runs Advanced Consent Mode v2, GA4 still sends anonymous "cookieless pings" to BigQuery. The data is saved, but personal identifiers like user_pseudo_id and ga_session_id are completely stripped out (null).

null: Untracked / Misconfigured State. The tracking engine recorded an event, but it has no idea what the consent status is. This indicates that your tracking code initialized and fired before the cookie banner (like CookieHub) could pass its default policy settings.

**ads_storage**

What it Means & Its Role:

This field governs advertising cookies and marketing permissions. In Consent Mode v2, this field carries a massive hidden role: it doesn't just look at basic advertising cookies; Google uses it as a combined metric to evaluate whether a brand has permission to send user data to Google Ads (ad_user_data) and whether that data can be used for remarketing lists and targeted ads (ad_personalization).

What the Values Mean:

Yes: Advertising Trackers Fully Operational. Google Ads and GA4 can link user actions to ad clicks, build retargeting audiences, and send conversion data directly to Google's ad platforms.

No: Ad Personalization Blocked. Google’s advertising tags are barred from reading or writing advertising cookies. Any cookieless pings that arrive in your database cannot be used for building remarketing audiences or personalized ad tracking.

null: Data Privacy Leakage. The tag fired and recorded an ad-related event profile without checking or assigning a legal consent state. This represents an immediate compliance warning for an enterprise.

**uses_transient_token**

What it Means & Its Role:

This is an advanced technical parameter that identifies how data was collected when cookies were rejected. A Transient Token is a temporary, short-lived, encrypted session key generated entirely inside a secure server's memory. It lives only for that specific page interaction and self-destructs the absolute millisecond a user closes their tab. It never leaves a physical file footprint on the user's computer.

What the Values Mean:

Yes: Enterprise Server-Side Architecture. This indicates the client is running a highly sophisticated Server-Side Google Tag Manager (sGTM) setup using a custom Server-to-Server Transient API. When the user clicked "Decline", the site didn't just send standard browser pings; instead, their own private cloud server securely mapped the session using temporary cryptographic tokens to guarantee 100% user privacy.

No: Standard Client-Side/Server-Side Configuration. This is what you see on 99% of normal sites (including your current CookieHub staging site tests). It means the tracking is running via standard browser scripts. When a user declines consent, the system relies on Google's default client-side cookieless pings rather than a custom server-side token architecture.

null: Incomplete Privacy Evaluation. The database engine recorded the interaction, but the data stream completely bypassed the cloud warehouse's compliance evaluation engine.
