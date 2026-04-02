# Exercises & Review Questions

> **ADHD tip:** Do one block at a time. Check your answers in the study guide only *after* you've written something down. Getting it wrong and then seeing the answer is how memory actually forms.
>
> Mark each question `[ ]` → `[x]` as you complete it. Track your streak.

---

## Block 1 — Vocabulary Warm-Up (5 min)
*Match the term to the definition. Write the letter next to the number.*

**Terms:**
- A. REST API
- B. HTTP Status Code
- C. Bearer Token
- D. JSON
- E. Pagination
- F. Serialization
- G. Server-side scripting
- H. Event-driven programming

**Definitions:**

- [ ] 1. ___ A number in an HTTP response that tells you if the request succeeded or failed.
- [ ] 2. ___ Code that runs on a server in response to a user request, not in the browser.
- [ ] 3. ___ Converting a data structure (object) into a text format that can be sent over a network.
- [ ] 4. ___ A credential sent in every HTTP request header to prove identity, with no server-side session.
- [ ] 5. ___ A text-based data format used by most web APIs to exchange structured data.
- [ ] 6. ___ A style of API that uses HTTP methods (GET, POST) to perform operations on resources.
- [ ] 7. ___ Breaking a large dataset into chunks that are fetched one at a time.
- [ ] 8. ___ A programming model where code waits for something to happen, then runs in response.

---

## Block 2 — True / False (5 min)
*Write T or F. If False, write one sentence explaining why.*

- [ ] 9. ___ The Fabman API key is sent to Google's servers once at login and stored in a session.

- [ ] 10. ___ A POST request typically includes a body, while a GET request typically does not.

- [ ] 11. ___ An HTTP status code of 200 means the request failed.

- [ ] 12. ___ `JSON.stringify()` converts a string into a JavaScript object.

- [ ] 13. ___ Google Apps Script runs inside the user's browser.

- [ ] 14. ___ The 3DPrinterOS integration failing will crash the entire sign-up process.

- [ ] 15. ___ The Field Mappings sheet tells the script which form question maps to which Fabman member field.

---

## Block 3 — Code Reading (10 min)
*Read the code snippet, then answer the question below it. All snippets are from your actual project.*

**Question 16:** Read this from `shared.js`:

```javascript
function try_send_request(api_key, method, url, payload) {
    const full_url = `https://fabman.io/api/v1${url}`;
    const request = {
        method: method.toLowerCase(),
        muteHttpExceptions: true,
        headers: {
            Authorization: `Bearer ${api_key}`,
        },
    };
    if (payload) {
        request.payload = JSON.stringify(payload);
        request.headers['Content-Type'] = 'application/json';
    }
    const response = UrlFetchApp.fetch(full_url, request);
    return response;
}
```

- [ ] a. What does `muteHttpExceptions: true` do? Why is it important here?
- [ ] b. Why is `Content-Type` only set when there is a `payload`?
- [ ] c. What is `JSON.stringify(payload)` doing and why is it necessary?

---

**Question 17:** Read this from `shared.js`:

```javascript
function fetch_all(api_key, url) {
    let results = [];
    while (true) {
        const offset = results.length;
        const page_url = `${url}?limit=1000&offset=${offset}`;
        const response = try_send_request(api_key, 'GET', page_url);
        const data = JSON.parse(response.getContentText());
        results.push(...data);

        let total_count = 0;
        const headers = response.getHeaders();
        for (const name of Object.keys(headers)) {
            if (name.toLowerCase() == 'x-total-count') {
                total_count = parseInt(headers[name], 10);
                break;
            }
        }
        if (results.length >= total_count) break;
    }
    return results;
}
```

- [ ] a. What does `offset` represent in the context of pagination?
- [ ] b. Why does the code do a case-insensitive search for the `x-total-count` header instead of just `headers['x-total-count']`?
- [ ] c. What would happen if `total_count` stayed at 0 and the API returned an empty array `[]`?

---

**Question 18:** Read this from `shared.js`:

```javascript
function get_3dpos_login_session() {
    const payload = `username=${encodeURIComponent(username)}&password=${encodeURIComponent(password)}`;
    const response = UrlFetchApp.fetch(login_url, {
        method: 'post',
        contentType: 'application/x-www-form-urlencoded',
        payload: payload,
    });
    const data = JSON.parse(response.getContentText());
    return data.message.session;
}
```

- [ ] a. What is `encodeURIComponent()` protecting against?
- [ ] b. This uses `application/x-www-form-urlencoded` instead of `application/json`. What is the difference in how the data looks when sent?
- [ ] c. Describe the two-step process this function is part of (login → create user) and why two steps are needed.

---

## Block 4 — Short Answer (10 min)
*Write 2–4 sentences for each.*

- [ ] 19. Explain the difference between **stateless** (Bearer token) and **stateful** (session-based) authentication. Which does Fabman use and which does 3DPrinterOS use?

- [ ] 20. Describe the **client-server model** as it applies to your project. Name at least three different "clients" and "servers" in the system.

- [ ] 21. What is the purpose of the **Field Mappings sheet**? Why is this configuration stored in a spreadsheet instead of hardcoded in the script?

- [ ] 22. Explain what **server-side scripting** means and why it matters for this project specifically (think about security and access).

---

## Block 5 — Scenario Questions (10 min)
*These are the kinds of questions an interviewer or professor might ask. Think it through before writing.*

- [ ] 23. **The intern scenario:** A new intern is setting up the plugin for the first time. They enter an invalid API key in the Settings sheet. Trace through the code — what happens? Which functions are called and what does the user see? *(Hint: look at `validate_api_key()` and `on_api_key_changed()` in admin.js)*

- [ ] 24. **The duplicate user scenario:** Someone tries to sign up but already has a Fabman account with that email. Walk through exactly what happens — which HTTP status code comes back, which function catches it, and what response does the user get?

- [ ] 25. **The design question:** Why does the project use Google Sheets as its "database" instead of a real database like MySQL? List one advantage and one disadvantage of this choice.

- [ ] 26. **The networking question:** Your CMSC 242 professor asks: "How is this project related to socket programming?" Write a response that connects what you know about TCP/IP sockets to what `UrlFetchApp.fetch()` is doing under the hood.

---

## Block 6 — Diagram Challenge (10 min)
*Draw or describe the flow in words.*

- [ ] 27. Draw (or write out as a numbered list) the complete sequence of events that happens from the moment a user clicks "Submit" on the Google Form to the moment their account appears in Fabman. Include every major function called and every network request made.

- [ ] 28. Draw the HTTP request-response pair for creating a new member. Label: method, URL, headers, body (request side) and status code, body (response side).

---

## Block 7 — Presentation Prep (5 min)
*Pretend a professor is asking you these questions live.*

- [ ] 29. "Can you explain what an API is to someone who has never heard the term, using your project as the example?"

- [ ] 30. "What CMSC 210 concepts appear in this project? Give me two specific examples from the code."

- [ ] 31. "What CMSC 242 concepts appear in this project? Give me two specific examples from the code."

- [ ] 32. "If I asked you to add support for a third external service to this project, what pattern would you follow? Which existing code would serve as your template?"

---

## Answer Key Hints
*(Read only after attempting — seriously, it helps more)*

- **Q9:** Bearer tokens are sent with *every* request. There is no "login once" step for Fabman.
- **Q12:** It's the opposite — `JSON.parse()` does that. `JSON.stringify()` goes object → string.
- **Q13:** Apps Script runs on Google's servers. Browser = client-side. Server = not the browser.
- **Q14:** 3DPrinterOS errors are caught and swallowed in a try/catch in signup.js:107-115.
- **Q16a:** It prevents the HTTP library from throwing an exception on 4xx/5xx — the code handles errors manually.
- **Q17b:** HTTP spec says header names are case-insensitive, but Google's implementation doesn't normalize them, so the script must check manually.
- **Q24:** Status 422. Caught by `is_error(member_response, 422, 'duplicateEmailAddress')` in signup.js:70. An email is sent to the user.
- **Q26:** `UrlFetchApp.fetch()` opens a TCP socket, sends an HTTP request, reads the response, and closes the socket — it's the entire socket lifecycle abstracted into one function call.
