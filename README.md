# Ganzir — OmniCTF 2026 Write-up

**Challenge:** Ganzir  
**Category:** Web  
**Theme:** SCP‑5000 (Site‑19 emergency intranet mirror)  
**Tools Used:** Burp Suite, `curl`, Browser DevTools

<img width="788" height="533" alt="Screenshot_From_2026-07-18_01-02-13 (1)" src="https://github.com/user-attachments/assets/da2ee762-1407-43b7-8d5b-03ea48d94aa4" />

---

## 1. Initial Reconnaissance

Visiting the challenge URL presents a dashboard styled as a **Secure Facility Dossier System** for Site-19.

<img width="1123" height="581" alt="Capture d’écran 2026-07-19 231754" src="https://github.com/user-attachments/assets/ef61de8c-74c8-490d-83d6-75c132b268c3" />


The page contains an **Employee Ingress** button. Clicking this brings us to `/employee`, which returns a **403 Forbidden** page.  
This 403 page is unusually verbose and reveals multiple technical clues:

- `accepted job endpoint: POST /employee`
- `accepted body formats: raw_request form field or text/plain raw HTTP`
- `edge parser: honors Transfer-Encoding: chunked`
- `bridge parser: trusts Content-Length before forwarding remaining bytes`
- `internal request: GET /employee/session HTTP/1.1`
- `required internal header: X-Employee-Gate: internal`

<img width="951" height="941" alt="Capture d’écran 2026-07-19 154623" src="https://github.com/user-attachments/assets/eb90e376-274c-48db-885a-f4627cf8d8c9" />


Additionally, inspecting the source of `/login` reveals a useful bulletin:

```html
<p class="muted">Recovery bulletin: Cassie from Records has been assigned to the harness intake desk. Temporary access should use the identity relay.</p>
```

<img width="953" height="975" alt="Capture d’écran 2026-07-19 153044" src="https://github.com/user-attachments/assets/60359959-5750-4e2f-99d8-2fc4b5759d9a" />


---

## 2. Dead Ends (HTTP Smuggling & JWT)

Because of the `/employee` hints, we initially focused on request smuggling with Burp Suite.

- **TE.CL Smuggling:** `POST /employee` with both `Transfer-Encoding: chunked` and `Content-Length: 0` repeatedly returned `400 Bad Request`.
- **H2.CL Smuggling:** Attempted to smuggle `GET /employee/session` via the `raw_request` form field.
- **Header Injection:** Tried to inject `X-Employee-Gate: internal` using various splitting/encoding techniques.

<img width="951" height="946" alt="Capture d’écran 2026-07-19 153558" src="https://github.com/user-attachments/assets/98da42d1-ec3e-4ae1-ab9c-792ce23ef5bb" />


We also examined `/jwt`, which displays a signed JWT and this note:

> *"The signed browser token is used for telemetry correlation, not authorization. Signature validation is enforced."*

<img width="951" height="946" alt="Capture d’écran 2026-07-19 153558" src="https://github.com/user-attachments/assets/dad09d9b-6662-4537-bbe0-63dd8df92580" />


This confirmed JWT manipulation was a deliberate red herring.

---

## 3. Breakthrough: Weak Credentials

After exhausting complex attack paths, we returned to basics: the login form.

The bulletin referenced **Cassie**, so we tried short password guesses for user `cassie`:

```bash
curl -s -X POST 'https://ganzir-....inst.omnictf.com/login' \
  -d 'username=cassie&password=cassie'
```

✅ **Result:** Login succeeded, and a valid `site19_session` cookie was issued.

So the “identity relay” hint pointed directly to a weak credential: **`cassie:cassie`**.

---

## 4. Post-Auth Enumeration & SSTI Discovery

Once authenticated, new navigation links appeared, including **Templates**.

Opening `/briefing-template` shows **Briefing Template Dry Run** with a template input and an **Operator Notes** panel.  
The notes explicitly disclose:

```text
engine: Jinja2
variables: wave, vector
helper: read_file(path)
flag copy: /flag.txt
```

<img width="958" height="972" alt="Capture d’écran 2026-07-19 153205" src="https://github.com/user-attachments/assets/c6f4fcca-e72f-4cdf-8b0c-eea4a2f8d785" />


This is a straightforward **Server-Side Template Injection (SSTI)** surface with a dangerous file-read helper exposed.

---

## 5. Exploitation & Flag Retrieval

First, we confirmed template execution:

```bash
curl -s -b cookies.txt -X POST 'https://ganzir-....omnictf.com/briefing-template' \
  --data-urlencode 'template={{ 7*7 }}'
```

The rendered output showed `49`, confirming Jinja2 execution.

Then we used the provided helper directly:

```text
{{ read_file("/flag.txt") }}
```

<img width="952" height="909" alt="Capture d’écran 2026-07-19 152532" src="https://github.com/user-attachments/assets/4952be42-71f1-4ca8-8071-8b1adf23b9b4" />


The rendered preview returned the flag immediately.

---

## 6. Flag

```text
CTF{ganzir_was_already_in_the_fire_plan}
```

---

## 7. Key Takeaways

1. **Read every hint carefully.**  
   The login bulletin leaked the target username, and operator notes exposed the exact file-read path.

2. **Don’t overcommit to the intended path.**  
   The challenge strongly suggested request smuggling, but the real foothold was weak authentication.

3. **Re-enumerate after auth.**  
   New routes/features often appear only after login and may expose entirely different classes of vulnerabilities.

4. **Expect red herrings in CTF web challenges.**  
   The 403 parser details and JWT endpoint consumed time by design.
