# Study Guide: Your Project Through the Lens of CMSC 210 & CMSC 242

> **How to use this guide (ADHD tip):** Read one section at a time. Set a 10-minute timer per section. Answer the Quick Check before moving on — don't skip them, they're what actually make things stick.

---

## The Big Picture First

Your project is a **Google Apps Script plugin** that:
1. Collects member sign-up data via a **Google Form** (CMSC 210)
2. Sends that data to an external service over the **internet using an API** (CMSC 242)
3. Runs entirely on **Google's servers** — no browser, no desktop app (CMSC 210: server-side scripting)

Every concept below is something you can point to in your own code during the presentation.

---

---

# PART 1 — CMSC 242: Networking & API Programming

---

## Concept 1: What Is an API?

**API** = Application Programming Interface. It's a defined contract for how two programs talk to each other over a network.

Your project uses **REST APIs** — the most common style on the web. REST APIs communicate over **HTTP**, which itself runs on top of **TCP/IP** (the network stack you study in CMSC 242).

- **HTTP** = the language programs use to make requests (the "envelope")
- **TCP/IP** = the postal system that actually delivers it across the network
- **REST API** = the agreed-upon rules for what goes in the envelope and what the response looks like

**In your project:** Two external APIs are used:
- Fabman API (`https://fabman.io/api/v1`) — creates members, assigns packages
- 3DPrinterOS API (`https://acorn.3dprinteros.com/`) — creates printer accounts

> **Quick Check 1:** In plain English, what does the Fabman API allow your script to do that it couldn't do by itself?

---

## Concept 2: HTTP Requests — The Anatomy

Every API call in your project is an HTTP request. Each one has four parts:

| Part | What it is | Example from your code |
|---|---|---|
| **Method** | What action to take | `GET` (read), `POST` (create) |
| **URL** | Where to send it | `https://fabman.io/api/v1/members` |
| **Headers** | Metadata | `Authorization: Bearer <key>` |
| **Body** | Data to send (POST only) | JSON object with member info |

**Find it in your code — `shared.js` lines 377–397, the `try_send_request()` function:**

```javascript
const request = {
    method: method.toLowerCase(),        // GET or POST
    headers: {
        Authorization: `Bearer ${api_key}`,  // <-- authentication header
    },
};
if (payload) {
    request.payload = JSON.stringify(payload);  // <-- body, serialized as JSON
    request.headers['Content-Type'] = 'application/json';
}
const response = UrlFetchApp.fetch(full_url, request);
```

`UrlFetchApp.fetch()` is Google Apps Script's equivalent of sending a TCP connection, sending an HTTP request over it, and reading the response — the entire network stack abstracted into one call.

> **Quick Check 2:** Looking at `try_send_request()`, what two things does the `Authorization` header communicate to the server?

---

## Concept 3: HTTP Status Codes

The server's response always includes a **status code** — a number that tells you what happened.

| Range | Meaning | Example in your project |
|---|---|---|
| 200–299 | Success | Member created (201) |
| 400 | Bad request — your fault | Malformed data sent |
| 422 | Unprocessable — data is valid but logically wrong | Duplicate email address |
| 500+ | Server error — their fault | Fabman's server crashed |

**Find it in your code:**

- `shared.js:334` — `if (response.getResponseCode() > 299)` — anything above 299 is treated as failure
- `signup.js:70` — `is_error(member_response, 422, 'duplicateEmailAddress')` — checks specifically for a 422 with a duplicate email tag
- `shared.js:405` — `if (response.getResponseCode() == 400)` — special handling for bad request

This is the same concept as checking return codes in C system calls — the number tells you what happened, and you branch your logic accordingly.

> **Quick Check 3:** Why does the code check for 422 specifically instead of just treating any error the same way?

---

## Concept 4: Authentication — Two Different Methods

Your project demonstrates **two completely different authentication strategies**. This is a great talking point.

### Method A: Bearer Token (Fabman)

- User creates an API key once in the Fabman web app
- That key is stored in the Settings sheet
- **Every single request** includes it in the `Authorization` header
- The server validates it each time — no "session" is maintained
- This is called **stateless** authentication

```javascript
// shared.js:383-385
headers: {
    Authorization: `Bearer ${api_key}`,
}
```

### Method B: Session-Based Auth (3DPrinterOS)

- Script logs in with username + password
- Server returns a **session token** (a temporary key)
- That session token is used for the actual operation
- This is called **stateful** authentication — the server remembers your session

```javascript
// shared.js:215-239  get_3dpos_login_session()
// Step 1: POST username+password → get session token
const response = UrlFetchApp.fetch(login_url, { method: 'post', payload: payload });
const data = JSON.parse(response.getContentText());
return data.message.session;   // <-- the session token

// shared.js:244-274  create_3dpos_user()
// Step 2: POST session token + new user data
payload_components.push(`session=${encodeURIComponent(session)}`);
```

**Connection to CMSC 242:** This is analogous to different socket authentication patterns — one uses a persistent credential per-connection, one establishes a handshake first.

> **Quick Check 4:** What is the risk of Bearer token auth vs session-based auth if someone intercepts your network traffic?

---

## Concept 5: JSON — Serialization and Deserialization

APIs can't send JavaScript objects over a network — they can only send **text**. JSON solves this.

- **Serialization** = object → JSON string (before sending)
- **Deserialization** = JSON string → object (after receiving)

**In your code:**

```javascript
// Serialization — shared.js:389
request.payload = JSON.stringify(payload);   // { firstName: "Jane", ... } → '{"firstName":"Jane",...}'

// Deserialization — shared.js:335
const data = JSON.parse(response.getContentText());  // '{"id":42,...}' → { id: 42, ... }
```

**Why this matters for networking:** JSON is a text format that travels safely over HTTP (which is also text-based). It's the standard "language" for REST APIs.

> **Quick Check 5:** What would happen if you sent a raw JavaScript object (without `JSON.stringify`) in an HTTP request body?

---

## Concept 6: URL Encoding

URLs can't contain spaces, `&`, `=`, or many special characters — these have special meaning in URLs. **URL encoding** replaces them with safe equivalents (e.g., space → `%20`).

**In your code — `shared.js:223`:**

```javascript
const payload = `username=${encodeURIComponent(username)}&password=${encodeURIComponent(password)}`;
```

If someone's password was `p@ss w0rd!`, without encoding this would break the URL. `encodeURIComponent()` turns it into `p%40ss%20w0rd%21`.

This is also why the 3DPrinterOS requests use `application/x-www-form-urlencoded` content type — it's a format where key-value pairs are joined with `&` and each value is URL-encoded.

> **Quick Check 6:** Why is it important to URL-encode user-supplied values (like email addresses) specifically?

---

## Concept 7: Pagination — Fetching Large Datasets

APIs rarely return all data at once — they **paginate** it, sending chunks. Your script needs to keep asking until it has everything.

**In your code — `shared.js:339–366`, the `fetch_all()` function:**

```javascript
let results = [];
while (true) {
    const offset = results.length;                              // "start where I left off"
    const page_url = `${url}?limit=${limit}&offset=${offset}`; // ask for next chunk
    const response = try_send_request(api_key, 'GET', page_url);
    const data = JSON.parse(response.getContentText());
    results.push(...data);                                      // accumulate

    // Check X-Total-Count header to know when done
    if (results.length >= total_count) break;
}
```

The API tells the total record count via a response **header** (`X-Total-Count`). The script reads the header and stops when it has accumulated that many records.

**Connection to CMSC 242:** This is analogous to reading from a socket in a loop — you don't get all the data in one read call, so you keep reading until you have it all.

> **Quick Check 7:** Why can't the script just request all records with an infinitely high limit?

---

## Concept 8: Error Handling in Networked Code

Networks fail. Servers fail. Your code must handle this gracefully.

Your project shows **three layers** of error handling:

**Layer 1 — Retry logic** (for transient failures):
```javascript
// shared.js:40-51  get_form() — retries up to 3 times with a 2-second sleep
while (true) {
    try {
        return FormApp.openByUrl(form_url);
    } catch (e) {
        retries += 1;
        if (retries > 3) throw e;
        Utilities.sleep(2 * 1000);
    }
}
```

**Layer 2 — HTTP error detection** (for API failures):
```javascript
// shared.js:368-375  send_request() vs try_send_request()
// try_send_request() always returns, even on error
// send_request() throws if the response is an error
```

**Layer 3 — Application-level error handling** (for business logic errors):
```javascript
// signup.js:70 — duplicate email isn't a crash, it's a known case
if (is_error(member_response, 422, 'duplicateEmailAddress')) {
    // send a helpful email instead of crashing
}
```

**Layer 4 — Graceful degradation** (for non-critical failures):
```javascript
// signup.js:107-115 — 3DPrinterOS failure doesn't stop the whole signup
try {
    create_3dpos_user(member_data);
} catch(e) {
    Logger.log(`3DPrinterOS account creation failed: ${e.toString()}`);
    // Don't fail the whole process if 3DPOS fails
}
```

> **Quick Check 8:** Why is it important that 3DPrinterOS errors are caught and swallowed instead of re-thrown?

---

---

# PART 2 — CMSC 210: Interactive Web Pages & Server-Side Scripting

---

## Concept 9: The Client-Server Model

Your project is a perfect example of a multi-tier client-server architecture:

```
[User] → [Google Form (client)] → submits data
                                        ↓
                          [Google Apps Script (server)]
                          processes data, runs logic
                                        ↓
                [Fabman API (external server)] ← script is now a client
                creates member, returns ID
                                        ↓
                [3DPrinterOS API (external server)]
                creates printer account
```

The same code is **a server** (receiving the form submission) and **a client** (making API calls). This is called a **middleware** or **backend** layer.

> **Quick Check 9:** In the architecture above, what role does the Google Sheet play?

---

## Concept 10: Server-Side Scripting

In CMSC 210 you write scripts that run on the server. Google Apps Script is exactly that — JavaScript that runs on **Google's servers**, not in anyone's browser.

**Why server-side and not client-side (browser JavaScript)?**

1. **Security** — The Fabman API key is stored in the sheet. If this ran in a browser, anyone could open DevTools and steal it.
2. **Access** — Server-side code can talk to other servers (Fabman, 3DPrinterOS). Browser code has CORS restrictions.
3. **Persistence** — Server-side code can write to a database (here, the Google Sheet). Browser code can't.
4. **Triggers** — The code runs in response to server events (form submitted), not browser events (button clicked).

**Analogy for your class:** In CMSC 210, when a visitor submits a form, a PHP or Python script runs on the web server to process it. Here, Google Apps Script plays that same role — it's just hosted by Google instead of a web server you manage.

> **Quick Check 10:** If the API key were exposed to the browser (client-side), what could a malicious user do with it?

---

## Concept 11: Event-Driven Programming

Your script doesn't run continuously — it **waits for events** and responds to them. This is the foundation of interactive web applications.

**Three events your project handles:**

| Event | Handler | What triggers it |
|---|---|---|
| Plugin installed | `onInstall(e)` | User installs from Marketplace |
| Spreadsheet opened | `onOpen(e)` | User opens the sheet |
| Form submitted | `on_form_submitted(e)` | Someone submits the sign-up form |
| Sheet edited | `on_installed_edit(e)` | Admin changes a setting |

**Two types of triggers — this is a subtle but important distinction:**

- **Simple triggers** (`onOpen`, `onInstall`) — built-in, run automatically, but have **limited permissions**
- **Installed triggers** (`on_form_submitted`, `on_installed_edit`) — registered programmatically, run with **full permissions**

```javascript
// admin.js:166-181  The script registers its own trigger at install time
ScriptApp.newTrigger('on_form_submitted')
    .forSpreadsheet(spreadsheet)
    .onFormSubmit()
    .create();
```

The comment at `admin.js:184` explains why: simple `onEdit` runs in "LIMITED mode" and can't access the Google Forms document. The installed trigger has full access.

> **Quick Check 11:** Why would it be a problem if `on_form_submitted` ran with limited permissions?

---

## Concept 12: Forms as Structured Data Collection

A Google Form is a user interface for **collecting structured data**. When submitted, each field becomes a named key-value pair.

**In your code — `signup.js:10`:**
```javascript
const submitted_data = e.namedValues;
// Result: { "First name": ["Jane"], "Email": ["jane@example.com"], ... }
```

The script then iterates over these fields, looks each one up in the Field Mappings sheet, and routes the value to the right Fabman member field. The form is the **input interface**; the script is the **data processor**.

**The mapping chain:**
```
Form field name ("What is your email?")
    → Field Mappings sheet lookup
    → Fabman field key ("emailAddress")
    → member_data.emailAddress = value
    → JSON body of POST request to Fabman
```

This is the core transformation your project performs — structured form data → API-compatible data format.

> **Quick Check 12:** What happens in your code if a form field has no mapping in the Field Mappings sheet?

---

## Concept 13: Dynamic / Data-Driven Interfaces

Static web pages show the same content to everyone. **Dynamic** pages show content that changes based on data.

Your project's admin interface is data-driven: the dropdown in the Package Mappings sheet isn't hardcoded — it's **populated live from the Fabman API**.

**In your code — `admin.js:500–506`:**
```javascript
const packages = fetch_packages(api_key);           // GET /packages from Fabman
const package_names = packages.map(p => `${p.name} (ID: ${p.id})`);
validation_rule = SpreadsheetApp.newDataValidation()
    .requireValueInList(package_names, true)        // dropdown options = API data
    .build();
```

If you add a new package to Fabman, the admin clicks "Update from Fabman" and the dropdown refreshes. The UI reflects the current state of the database.

> **Quick Check 13:** What would break if the package dropdown were hardcoded instead of fetched from the API?

---

## Concept 14: Data Persistence — The Sheet as a Database

In a traditional web app, a database (MySQL, PostgreSQL) stores data. Here, **Google Sheets acts as the database**.

| Database concept | Google Sheets equivalent |
|---|---|
| Table | Sheet tab |
| Row | Spreadsheet row |
| Column | Spreadsheet column |
| Primary key | Row number |
| Query | `getRange().getValues()` |
| Insert | `getRange().setValue()` |
| Index | Map object built from sheet data |

**Settings storage example — `shared.js:71–84`:**
```javascript
// Reads key-value pairs from the Settings sheet (like SELECT WHERE key = 'API Key')
const settings = sheet.getRange(first_row, 1, last_row - 1, 2).getValues();
for (const setting of settings) {
    if (setting[0] == SETTINGS.api_key) {
        return sheet.getRange(row, 2, 1, 1);  // returns the cell with the value
    }
}
```

**Audit log example — `shared.js:277–287`:**
```javascript
// Appends a row to the 3DPrinterOS sheet (like INSERT INTO)
const new_row = last_row + 1;
sheet.getRange(new_row, 1, 1, 1).setValue(email);
sheet.getRange(new_row, 2, 1, 1).setValue(timestamp);
```

> **Quick Check 14:** What is one limitation of using a spreadsheet as a database compared to a real database like MySQL?

---

---

# Presentation Cheat Sheet

Use this to anchor your talking points during the actual presentation.

| If they ask about... | Point to... | Say... |
|---|---|---|
| How the form connects to Fabman | `on_form_submitted()` in signup.js | "The form submission fires a server-side trigger, which maps form fields to API fields and POSTs to Fabman" |
| How authentication works | `try_send_request()` in shared.js:377 | "Bearer token in every request header — stateless auth" |
| How 3DPrinterOS is different | `get_3dpos_login_session()` in shared.js:215 | "Session-based — login first, get a token, then use the token" |
| What happens if something fails | `on_form_submitted()` error handling in signup.js | "Three layers: retry, HTTP error detection, graceful degradation" |
| Where data is stored | Settings sheet, Field Mappings sheet | "Google Sheets acts as our configuration database" |
| Why it runs server-side | Anywhere | "Security — API keys can't be exposed to the browser" |

---

*Next: Open `exercises.md` and test yourself. Aim to answer each question before looking back at this guide.*
