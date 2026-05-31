The BQ code shared is to better understand consent mode and what selection users made around analytics_storage and ads_storage. If Advanced Consent Mode is implemented the values of analytics_storage and ads_storage it could be a mix of Yes, No or null.

# OVERVIEW
This repository contains Big Query code using Google Analytics raw data allowing to track **Consent Mode v2**. It is engineered specifically to validate **Consent Mode v2** configurations, and spot compliance leaks at scale.

## DAY PART DEFINITIONS
The data is segmented into the following buckets:
* **Early Risers**: 4 AM - 5 AM
* **Breakfast**: 6 AM - 9 AM
* **Morning**: 10 AM - 11 AM
* **Lunchtime**: 12 PM - 2 PM
* **Afternoon**: 3 PM - 4 PM
* **Early Evening**: 5 PM - 7 PM
* **Evening**: 8 PM - 11 PM
* **Lates**: 12 AM - 3 AM

## 📖 The Technical Mechanics: How the Code Recognizes Consent Mode v2

When analyzing raw event data inside Google BigQuery, understanding the architectural shifts between Consent Mode v1 and v2 is vital. Google did not modify the flat database schema to add explicit columns for the new v2 privacy fields (`ad_user_data` and `ad_personalization`). Instead, the platform leverages behavioral fingerprints and downstream variable mapping to process v2 data:

1. **Multi-Parameter Parameter Binding (`privacy_info.ads_storage`):**
   In a certified Consent Mode v2 implementation (e.g., CookieHub, OneTrust), the user's frontend interaction with an advertising or marketing toggle simultaneously controls three backend parameters: `ads_storage`, `ad_user_data`, and `ad_personalization`. Google bundles the evaluation of these components and updates the state within the existing `privacy_info.ads_storage` database field. 
   
2. **The Advanced Consent Mode Fingerprint (`analytics_storage = 'No'`):**
   * **Under v1 Rules:** If a user clicked "Decline", tags were completely blocked on the client side. **Zero rows were ever written to BigQuery.**
   * **Under Advanced v2 Rules:** Google introduces secure **cookieless pings**. When a user denies consent, GA4 still securely transmits hit-level data to the cloud database while stripping out user identifiers. 
   
   *Therefore, the mere presence of rows in your query results displaying an explicit `No` status for analytics or advertising storage is structural database proof that the engine recognizes and executes the Advanced Consent Mode v2 framework.*

3. **Database Flag Population States:**
   * **`Yes` / `No` Strings:** Confirms Consent Mode v2 is active and passing structured evaluations.
   * **`null` / Blank Cells:** Indicates unassigned state tracking where tags fired entirely outside of the privacy control wrapper.
