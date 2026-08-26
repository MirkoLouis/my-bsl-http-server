# Project Evaluation and Changelog

## 1. Evaluation against Lab Requirements

| Criteria | Status | Remarks |
| :--- | :---: | :--- |
| **Documentation (README & Screenshots)** (60%) | **Accomplished** | The `README.md` includes the Project Description, Installation & Setup, Usage, and the Screenshots section exactly as requested. The user has populated the `documentation/` folder with official screenshots. |
| **Folder & File Structure, License, & Attributes** (20%) | **Accomplished** | The project correctly contains `.gitattributes` (enforcing JavaScript highlighting for BSL), an MIT `LICENSE`, `src/http.bzg`, and the `documentation/` folder. The user has pushed this to a Public GitHub Repository. |
| **Actual Code Implementation** (20%) | **Accomplished** | The HTTP server successfully binds to port 8080 and handles proper HTTP routing (`/`, `/about`, and `404`) as mandated. Additionally, it features a bonus `/fun` route with a working boot animation. |

**Overall Status:** **100% Complete.** The repository perfectly aligns with all instructions in `Lab1_ Building an HTTP Server using Socket.md` and is ready for submission to MOLE.

---

## 2. Changes Made from the Example Template

While implementing the provided template from the lab, several necessary deviations and architectural changes were made to overcome native bugs in the BSL interpreter (v1.3.1) and to support the bonus `/fun` boot animation. 

Here is a detailed breakdown of every change made to the starter code:

### A. Replacing `.indexOf()` with String Array Indexing
**The Problem:** 
The starter code used `data.indexOf("GET / ")` to evaluate the request route. However, in this version of the Bonezegei interpreter, primitive strings are not objects and do not have an `.indexOf` method (yielding an `Attempt to call method on non-object type` crash). Furthermore, there is no built-in `indexOf` function.

**The Fix:** 
The routing logic was entirely rewritten to use explicit character-by-character array indexing. 
*Example:* 
```javascript
// Old Template:
if (data.indexOf("GET / ") >= 0) { ... }

// New Implementation:
if (sizeof(data) >= 10 && data[0] == "G" && data[1] == "E" && data[2] == "T" && data[4] == "/" && (data[5] == " " || data[5] == "?")) { ... }
```
This safely validates the HTTP verb and path without relying on unimplemented string prototype methods, and uses `sizeof(data)` to prevent out-of-bounds indexing.

### B. Splitting `socket_write` to Prevent Segmentation Faults
**The Problem:** 
The starter code sent the response by concatenating the headers and the body:
`socket_write(client, HTTP + BODY, sizeof(HTTP + BODY));`
When implementing the `/fun` route, the HTML payload was ~134KB (because it embedded a base64 font and ASCII art). Attempting to perform string concatenation (`HTTP + BODY`) on a 134KB string caused the BSL interpreter to crash with a `Segmentation fault (core dumped)`.

**The Fix:** 
The concatenation was removed, and the response was dispatched sequentially in two distinct TCP socket writes:
```javascript
// Old Template:
socket_write(client, HTTP + BODY, sizeof(HTTP + BODY));

// New Implementation:
socket_write(client, HTTP, sizeof(HTTP));
socket_write(client, BODY, sizeof(BODY));
```
This immediately resolved the memory fault and made the server resilient to large payloads.

### C. Asset Inlining for `fun.html`
To prevent the need for building a full static file server (since the lab only required specific hardcoded routes), all assets for the boot animation were embedded directly into `public/fun.html`:
- `PxPlus_IBM_VGA_8x16.ttf` was converted to a base64 data URI and injected directly into the CSS.
- `frover.txt` (the ASCII art) was injected directly as a multiline JavaScript string.
- The `index.html` shell logic was modified so that executing `get /` or `get /about` inside the mock TTY emulator redirects the browser to the actual BSL server routes.

### D. Implementation of All Routes (`/`, `/fun`, `/about`, and `404`)
As mandated by the lab instructions and extended for the bonus feature, the server actively intercepts the requested path and responds with the appropriate HTTP payload:
- **Default Landing Page (`/`):** The logic explicitly checks if the HTTP request begins with `GET / ` (ensuring it is the root path and not a sub-directory). If matched, it constructs a complete HTTP/1.1 200 OK header combined with a hardcoded HTML string body containing a welcome message.
- **Fun Route (`/fun`):** A custom `else if` block was injected into the routing table to listen for `GET /fun`. Instead of hardcoding the HTML as a string, it uses BSL's `readFile("public/fun.html")` to dynamically load the TTY boot animation file and serves it with a 200 OK header.
- **About Page (`/about`):** If the requested path does not match `/`, the server checks for the sequence matching `GET /about`. When identified, it similarly sets up an HTTP/1.1 200 OK header, but instead populates the `BODY` variable with HTML text describing the HTTP Server project.
- **404 Not Found (Fallback):** Finally, an `else` block serves as the catch-all for any unmapped route (such as `/anything` or `/home`). It responds with a strict `HTTP/1.1 404 Not Found` header paired with an HTML error page, informing the user that the route does not exist. All HTTP headers correctly formulate the `Content-Length` by using `sizeof(BODY)` to ensure protocol compliance.

---

## 3. Known Issues & Troubleshooting

### System Shutdown Hang (UID 1000)
When attempting to reboot the host machine or stop the Distrobox container after running the server in the background, users may experience a 90-second hang with the following systemd message:
> `A stop job is running for User Manager for UID 1000`

**Root Cause:**
This issue is caused by the interaction between the `bonezegei` interpreter, systemd, and Distrobox:
1. **Bonezegei:** The server operates on an infinite `while (1)` loop calling `socket_accept()`. Because the minimal BSL interpreter (v1.3.1) lacks built-in `SIGTERM` signal handling, it ignores graceful shutdown requests.
2. **Systemd:** When shutting down, Linux systemd asks all processes owned by the user (`UID 1000`) to terminate via `SIGTERM`. If a process ignores this, systemd waits for a default 90-second timeout before sending a forced `SIGKILL`.
3. **Distrobox:** Because the container binds its background processes directly to the host user's session, the hanging `bonezegei` process successfully blocks the host's `UID 1000` session manager from closing.

**Solution:**
To avoid the 90-second timeout penalty, manually kill the background interpreter before shutting down the container or host system by running:
```bash
killall bonezegei
```
