# Frontend Security & Storage — Combined Notes


Table of Contents

- Token storage patterns (Split storage, Memory+Cookie, BFF)
- Client-side storage options (localStorage, sessionStorage, Cookies, IndexedDB)
- Web browser security fundamentals (XSS, CSRF, CORS)

---

## Token storage patterns 

The most secure and industry-standard way to store tokens in a frontend application is to keep the Access Token in application memory (RAM) and the Refresh Token in an  Secure Cookie. Storing both tokens in  or  is highly discouraged for production apps because any malicious script running on your site (Cross-Site Scripting or XSS) can easily steal them. [1, 2, 3]  
🛡️ Option 1: The Modern Best Practice (Split Storage) 
This approach isolates the long-lived refresh token from frontend JavaScript entirely. 

• Access Token: Store it in application memory (e.g., inside a React/Vue/Angular state variable or a plain JavaScript closure). 

	• Pros: Completely immune to XSS token theft because the variable is never written to disk or browser storage. 
	• Cons: The token is cleared if the user hard-refreshes the browser page. [1, 2, 4, 5, 6]  

• Refresh Token: The backend must set this in an  cookie. 

	• Pros: JavaScript cannot read or touch this cookie, keeping it perfectly safe from XSS. The browser automatically attaches it to the token-refresh API calls behind the scenes. 
	• Cons: Vulnerable to Cross-Site Request Forgery (CSRF), which you must mitigate using specific cookie flags. [1, 2, 7, 8]  

Required Cookie Flags for the Backend:To make the refresh token cookie safe, the backend must apply these exact flags: 

• : Blocks all client-side JavaScript access. 
• : Ensures the cookie is only transmitted over encrypted HTTPS connections. 
•  or : Protects your application against CSRF attacks. 
• : Restricts the browser to only send this cookie when hitting your token-refresh endpoint, rather than on every single image or asset request. [1, 7, 9, 12, 13]  

How the page refresh issue is solved: When a user refreshes the page, the access token disappears from memory. Your frontend immediately kicks off a background request to the backend refresh endpoint (). The browser sends the  cookie automatically, the backend verifies it, and returns a brand-new access token back into your frontend state seamlessly. [2, 14, 15, 16, 17]  
📦 Option 2: The Easiest (But Least Secure) Way 
This approach involves storing both tokens directly inside the browser's persistent storage, such as  or . 

• Access Token: Stored as a plain text string. 
• Refresh Token: Stored as a plain text string. 
• Pros: Extremely simple to implement. Survives browser page reloads and tab closures effortlessly. No complex backend cookie configuration required. 
• Cons: Highly Insecure. If your web application includes a compromised third-party script, NPM package, or an input field vulnerable to XSS, an attacker can extract these tokens globally using . [1, 3, 25, 26]  

🏛️ Option 3: Backend-for-Frontend (BFF) Pattern [27, 28]  
For enterprise-grade security, applications use the BFF (Backend-for-Frontend) architecture. In this architecture, the frontend Single Page Application (SPA) never sees or handles any tokens at all. [15, 29, 30, 31, 32]  
The frontend talks exclusively via an encrypted session cookie to a lightweight backend proxy (the BFF). That proxy securely stores both the access and refresh tokens server-side, attaches them to downstream microservice requests, and leaves the client completely decoupled from token management. [13, 29, 30, 33, 34]  

📊 Summary Comparison 

| Storage Location [1, 2, 7, 8, 29, 33, 35, 36] | Access Token | Refresh Token | XSS Security | CSRF Security  |
| --- | --- | --- | --- | --- |
| — | Yes | Yes | ❌ Highly Vulnerable | Safe  |
| Memory + Cookie | JS State (RAM) | Cookie | Highly Secure | ⚠️ Needs  flags  |
| BFF Pattern | Server-side | Server-side | Maximum Security | ⚠️ Needs SameSite flags  |


## Client-side storage options

Frontend engineers use four primary client-side data storage mechanisms. Each has unique characteristics regarding capacity, lifespan, and security boundaries. [1, 2] 
------------------------------
## 📦 1. localStorage [3] 
Persistent, long-term storage built directly into the browser. [4, 5, 6, 7] 

* Lifespan: Permanent. Data stays forever until manually cleared by the user or deleted via JavaScript code. Closing the tab or browser will not remove it. [8, 9, 10, 11, 12] 
* Capacity: Roughly 5MB per domain. [13, 14] 
* Accessibility: Accessible by any JavaScript code running on that exact same domain (Same-Origin Policy). [15, 16, 17] 
* Security Risk: High risk for sensitive data. It is highly vulnerable to Cross-Site Scripting (XSS). Any malicious script or compromised npm package running on your site can read all data inside it. [18, 19, 20, 21, 22] 
* Best Used For: Non-sensitive UI states, user theme preferences (light/dark mode), cached offline data, or items in a shopping cart before checkout. [23, 24, 25, 26, 27] 

------------------------------
## ⏱️ 2. sessionStorage [28] 
A temporary, isolated sandbox tied to a specific browser tab session. [29, 30, 31, 32] 

* Lifespan: Tied strictly to the specific browser tab. Closing the tab immediately destroys the data. [33, 34] 
* Capacity: Roughly 5MB per domain. [35, 36] 
* Accessibility: Limited only to that specific tab. Even if you open the exact same URL in a second tab, that new tab gets its own completely blank sessionStorage instance. [37, 38, 39] 
* Security Risk: Still vulnerable to XSS while the tab remains open, but the exposure window is smaller since closing the tab wipes the memory. [40, 41, 42] 
* Best Used For: Multi-step wizard forms, filtering states on a search results page, or tracking deep links to navigate back within a single session. [43] 

------------------------------
## 🍪 3. Cookies [44] 
Small text files originally designed for server-client communication that travel automatically with every network request. [45, 46] 

* Lifespan: Configurable via an expiration date (Expires or Max-Age). Can act as a temporary "session cookie" (wiped when browser closes) or a persistent cookie.
* Capacity: Tiny. Only about 4KB per cookie, with a strict limit of around 20–50 cookies per domain.
* Accessibility: Can be read by both frontend JavaScript and the backend server. However, if the server sets the HttpOnly flag, JavaScript is completely blocked from reading or altering it.
* Network Behavior: The browser automatically attaches cookies to every single HTTP request made to that domain (images, API calls, pages).
* Security Risk: If unprotected, they are vulnerable to XSS. If HttpOnly is on, they are immune to XSS but become vulnerable to Cross-Site Request Forgery (CSRF), which requires mitigation using the SameSite=Strict or Lax flag.
* Best Used For: Session identifiers, authentication tokens (using HttpOnly), and cross-site tracking or analytics. [47, 48, 49, 50, 51] 

------------------------------
## 🗄️ 4. IndexedDB
A full-scale, transactional, object-oriented database running directly inside the user's browser. [52, 53, 54, 55] 

* Lifespan: Permanent, surviving browser restarts until cleared by the user or codebase.
* Capacity: Huge. Usually limited only by the user's available hard drive space (often hundreds of megabytes or gigabytes).
* Accessibility: Accessible via asynchronous JavaScript APIs following the Same-Origin Policy.
* Best Used For: Heavy web apps that need extensive offline support (like Progressive Web Apps or Google Docs), caching massive datasets, storing files/blobs locally, or client-side indexing. [56, 57, 58, 59, 60] 

------------------------------
## 📊 Direct Comparison

| Feature [61, 62, 63, 64, 65] | localStorage | sessionStorage | Cookies | IndexedDB |
|---|---|---|---|---|
| Capacity | ~5 MB | ~5 MB | ~4 KB | Virtually Unlimited |
| Lifespan | Permanent | Tab closure | Explicit expiry | Permanent |
| Sent to Server? | No | No | Automatically | No |
| JS Access | Yes | Yes | Yes (Unless HttpOnly) | Yes |
| Data Type | Strings only | Strings only | Strings only | Objects, Blobs, Binary |


---

## Web browser security fundamentals  

These three concepts form the foundation of web browser security, but they address entirely different mechanisms. 

• XSS (Cross-Site Scripting): An attack where a site unknowingly runs malicious code inserted by an attacker. 
• CSRF (Cross-Site Request Forgery): An attack where an untrusted site tricks a user's browser into performing an unwanted action on a trusted site where the user is already logged in. 
• CORS (Cross-Origin Resource Sharing): A browser mechanism used by servers to permit or restrict which outside websites are allowed to access their resources. [3, 6]  

Key Differences at a Glance 

| Feature [1, 3, 4, 5, 6, 7, 8] | XSS (Cross-Site Scripting) | CSRF (Cross-Site Request Forgery) | CORS (Cross-Origin Resource Sharing)  |
| --- | --- | --- | --- |
| What is it? | A security vulnerability. | A security vulnerability. | A security policy (a feature).  |
| Who executes it? | The user's browser, running an attacker's injected code. | The user's browser, tricked by the attacker into sending a valid HTTP request. | The user's browser, which enforces server-approved origins for API requests.  |
| What is stolen/abused? | Cookies, session tokens, and the ability to act completely as the user. | Executing actions on the user's behalf (e.g., changing passwords or transferring money). | Reading data from a different domain than the one hosting the frontend.  |
| Primary Defense | Output encoding, input validation, and Content Security Policy (CSP). | Anti-CSRF tokens and the  cookie attribute. | Properly configured HTTP headers (e.g., ).  |

Understanding the Details 
1. XSS (Cross-Site Scripting) An XSS vulnerability allows a malicious third party to inject and execute arbitrary JavaScript inside your users' browsers. Because this script runs in the context of your website, it has access to everything your site does—including reading your users' cookies and session tokens. 

• Example: An attacker adds a comment on a blog with . When other users view the comment, their browser executes the script, and their session data is sent to the attacker's server. [1, 4]  

2. CSRF (Cross-Site Request Forgery) CSRF tricks an authenticated user into performing actions they did not intend to. While XSS steals the user's session, CSRF piggybacks on it. The target website trusts the request because the browser automatically sends the user's session cookies, even though the request originated from an attacker's site. 

• Example: You log into your banking site. You then navigate to a malicious site, which forces your browser to submit a hidden form to your bank requesting a transfer of funds to the attacker's account. [10]  

3. CORS (Cross-Origin Resource Sharing) By default, browsers enforce the Same-Origin Policy (SOP). This means a website loaded from  cannot normally make API calls to . CORS is the browser-native mechanism that allows a server at  to safely loosen this restriction and declare: "It is okay for siteA.com to request my data". 

• Example: A frontend running on  making an API request to . Stripe's server must return CORS headers indicating that  is permitted to access the data. [12, 13]  


