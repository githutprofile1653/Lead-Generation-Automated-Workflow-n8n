# Lead Generation Automation Workflow – Technical Documentation

---

## **1. Overview**

<img width="1101" height="313" alt="lead generation workflow" src="https://github.com/user-attachments/assets/5f5aae3b-0b3c-40ef-968a-16a89b391cc8" />

This document provides an in-depth technical explanation of the **Lead Generation Workflow** implemented in **n8n**.
The workflow automates the process of collecting business leads based on **industry and location**, enriching them with **contact information (emails & phones)**, and storing the results in **Google Sheets**, separated into leads **with emails** and **without emails**.

---

## **2. High-Level Architecture**

**User Input → Business Discovery → Data Filtering → Email Enrichment → Deduplication → Storage**

**External Services Used**

* **n8n** – Workflow orchestration
* **Apify (Google Maps Scraper Actor)** – Business discovery
* **Tavily Search API** – Web search for emails
* **Google Sheets API** – Lead storage

---

## **3. Workflow Trigger**

### **3.1 Form Trigger – “On form submission”**

**Node Type:** `n8n-nodes-base.formTrigger`
**Purpose:** Collects lead generation criteria from the user.

**Form Fields**

* `Industry` – Business category (e.g., restaurant)
* `Location` – City or region (e.g., Abuja)
* `Number of rows` – Maximum number of businesses to scrape

**Output**

```json
{
  "Industry": "restaurant",
  "Location": "Abuja",
  "Number of rows": "5"
}
```

This data drives the entire downstream workflow.

---

## **4. Business Discovery Layer**

### **4.1 Apify Actor – “Run an Actor and get dataset”**

**Node Type:** `@apify/n8n-nodes-apify.apify`
**Operation:** Run actor and retrieve dataset
**Actor Used:** Google Maps Places Scraper

**Dynamic Configuration**

* `locationQuery` ← Form `Location`
* `searchStringsArray` ← Form `Industry`
* `maxCrawledPlacesPerSearch` ← Form `Number of rows`

**Key Disabled Features (for performance & compliance)**

* No social media scraping
* No contact scraping from Apify
* No website crawling
* No enrichment records

**Output**
A structured dataset of businesses including:

* `title`
* `website` (if available)
* `phoneUnformatted`
* `state`
* `address`
* Metadata (ratings, categories, etc.)

---

## **5. Data Normalization & Routing**

### **5.1 Split Out – “Split Out”**

**Node Type:** `n8n-nodes-base.splitOut`
**Purpose:** Flattens the Apify dataset into individual business records.

**Fields Extracted**

* `website`
* `title`
* `phoneUnformatted`
* `state`

Each business now becomes a single n8n item.

---

### **5.2 Conditional Routing – “Switch”**

**Node Type:** `n8n-nodes-base.switch`
**Purpose:** Separates businesses based on website availability.

**Rules**

* **Path A:** Website is empty → No email enrichment
* **Path B:** Website is not empty → Attempt email discovery

This ensures efficient processing and avoids unnecessary searches.

---

## **6. Leads Without Website (No Email Path)**

### **6.1 Split Out – “Split Out1”**

Extracts:

* `title`
* `phoneUnformatted`
* `state`

### **6.2 Google Sheets – “Append row in sheet”**

**Purpose:** Store businesses **without emails**.

**Sheet:** `leads-no-email`
**Columns Written**

* Business → `title`
* Phone Number → `phoneUnformatted`
* State → `state`

This dataset is ideal for:

* Cold calling
* SMS outreach
* Manual follow-ups

---

## **7. Email Enrichment Path**

### **7.1 Tavily Search – “Search”**

**Node Type:** `@tavily/n8n-nodes-tavily.tavily`
**Purpose:** Search the web for potential contact emails.

**Query Pattern**

```
site: <Business Title> contact email
```

This targets:

* Official websites
* Contact pages
* Directory listings

---

### **7.2 JavaScript Processing – “Code in JavaScript”**

**Purpose**

* Parse Tavily search results
* Extract emails and phone numbers
* Associate emails with business metadata

**Key Operations**

* Regex-based email extraction
* Phone number pattern matching
* Record creation per email found
* Fallback record if no email exists
* URL + email deduplication

**Output Structure**

```json
{
  "title": "Business Name",
  "url": "https://example.com",
  "phone": "+234...",
  "email": "info@example.com",
  "score": 0.87
}
```

---

### **7.3 Advanced Deduplication – “Code in JavaScript1”**

**Purpose**
Ensures **one best record per email**.

**Logic**

* Group records by email
* Keep the record with the **highest relevance score**
* Discard duplicates

This improves data quality and prevents redundant outreach.

---

## **8. Final Data Preparation**

### **8.1 Split Out – “Split Out2”**

Extracts final fields:

* `title`
* `phone`
* `email`

---

## **9. Leads With Email Storage**

### **9.1 Google Sheets – “Append row in sheet1”**

**Purpose:** Store enriched leads with email addresses.

**Sheet:** `leads`
**Columns**

* Business → `title`
* Phone → `phone`
* Email → `email`

**Matching Rule**

* Matches on `Business` to reduce duplicates

This dataset is optimized for:

* Email campaigns
* CRM imports
* Automated outreach

---

## **10. Error Handling & Data Quality**

* Conditional routing prevents unnecessary API calls
* Deduplication at two levels:

  * URL + email
  * Email + relevance score
* Empty results safely skipped
* Phone extraction is tolerant of international formats

---

## **11. Security & Credentials**

**Stored Securely in n8n**

* Apify API Key
* Tavily API Key
* Google Sheets OAuth2 credentials

No secrets are hardcoded in JavaScript.

---

## **12. Scalability & Optimization Notes**

* Can be extended to:

  * LinkedIn enrichment
  * CRM integration (HubSpot, Zoho, Salesforce)
  * Email validation APIs
* Batch size controlled by form input
* Tavily usage limited to businesses with websites

---

## **13. Summary**

This workflow provides a **fully automated, scalable lead generation pipeline** that:

* Accepts structured user input
* Discovers local businesses
* Enriches leads with emails intelligently
* Separates leads by quality
* Stores results in clean, actionable datasets

It is suitable for **B2B sales teams, agencies, and growth marketers** looking to generate leads with minimal manual effort.
