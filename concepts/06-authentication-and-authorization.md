# Authentication and authorization

> **The one idea:** authentication turns a request into *identity* — a bag of claims attached to `HttpContext.User` — and authorization is a *decision* made against that bag. Two middlewares, in that order, because you cannot evaluate a permission before you know who is asking.

---

## Why this concept exists

Old ASP.NET tied identity to the hosting model. `HttpContext.Current.User` was ambient, forms authentication was a cookie backed by a membership provider you configured in `web.config`, and Windows authentication came from IIS. The identity you got depended on where you were running.

That falls apart the moment there is no single "where". A React app calling an API on another origin, a mobile client, a gRPC service calling another service, a background worker with no user at all — none of those fit a cookie handed out by the web server that owns the page.

.NET Core's answer is to stop caring where identity came from. Every mechanism — a cookie, a bearer token, a client certificate, an OIDC handshake — produces the same thing: a **`ClaimsPrincipal`**. A set of statements about the caller. What produced it is a *scheme*, and the rest of the framework never has to know which one ran.

That normalisation is the concept. Everything else in this chapter is either "how do the claims arrive" or "what do we do with them once they have".

And chapter 01 already explained why there are two middlewares rather than one: the pipeline is ordered, position determines behaviour, and these two have a dependency between them.

---

## 1. Identity is data; permission is a decision made against it

The distinction sounds like pedantry until you are debugging, at which point it is the whole thing.

**Authentication** answers *who is this*. It reads something off the request — a cookie, an `Authorization` header, a certificate — validates it, and builds a `ClaimsPrincipal`:

```csharp
HttpContext.User;                              // ClaimsPrincipal
User.Identity?.IsAuthenticated;                // did any scheme succeed?
User.FindFirst("sub")?.Value;                  // a claim
User.IsInRole("Planner");                      // also just a claim, see §6
```

A `ClaimsPrincipal` holds one or more `ClaimsIdentity`, each of which is a list of `Claim` — type and value, nothing more. `sub`, `email`, `role`, `tenant_id`. That is the entire data model, and its flatness is deliberate: a claim is a *statement someone made about the caller*, not a permission.

**Authorization** answers *is this allowed*. It takes the principal and the thing being accessed, and returns yes or no.

The failure modes fall straight out of the split, and so do the status codes:

- **401 Unauthorized** — authentication did not establish an identity, or the one it established is not accepted. The correct response is a *challenge*: prove who you are.
- **403 Forbidden** — authentication worked perfectly. We know exactly who you are, and you may not do this. Re-authenticating will not help.

Getting a 403 when you expected a 401 means your token was fine and your permissions were not. Getting 401 on an endpoint you are certain you have rights to means the token never turned into a principal at all — and §2 is usually why.

---

## 2. Two middlewares, in that order, and what each actually does

```csharp
app.UseRouting();
app.UseAuthentication();   // reads credentials → sets HttpContext.User
app.UseAuthorization();    // reads endpoint metadata → allows or rejects
app.MapControllers();
```

The ordering constraint is not a convention to memorise — it is a data dependency, twice over:

- **`UseAuthentication` before `UseAuthorization`**, because the second one evaluates a principal the first one creates. Reversed, authorization runs against an anonymous user and every `[Authorize]` endpoint returns 401 no matter what token you send.
- **Both after `UseRouting`**, because authorization needs to know *which endpoint* was selected in order to read its `[Authorize]` metadata. Before routing, no endpoint has been chosen and there is no metadata to read. This is exactly chapter 01 §4's two-stage routing split, and it is the reason that split exists.

The detail people get wrong: **`UseAuthentication` does not reject anything.** If there is no credential, or the credential is invalid, it sets an unauthenticated principal and calls `next()`. The request carries on. Nothing rejects until authorization looks at endpoint metadata and finds a requirement the principal fails.

That is why `[Authorize]` on a controller with no `UseAuthorization()` in the pipeline is a **silently open endpoint**. The attribute is metadata. Metadata does nothing on its own — something has to read it, and if that something is not in the pipeline, the attribute is a comment. It is worth being blunt: that is not a subtle bug, it is an unauthenticated public endpoint that looks protected in the source.

---

## 3. A scheme is just how the same claims arrive

A *scheme* is a named handler that knows how to turn some part of a request into a principal — and how to challenge when there isn't one.

```csharp
builder.Services
  .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
  .AddJwtBearer(o => { /* §4 */ })
  .AddCookie();
```

The two you will actually use:

**Cookies** — the browser holds an opaque encrypted cookie; the server decrypts it into a principal. State lives server-side (or in the encrypted payload), so signing someone out is real: delete or invalidate the cookie. The cost is that cookies are sent automatically by the browser, which is what makes CSRF possible and why antiforgery tokens exist for cookie-authenticated forms.

**JWT bearer** — the client holds a signed token and sends it in `Authorization: Bearer <token>`. Nothing is stored server-side, which is the entire appeal and the entire problem (§4, §10). Nothing is sent automatically, so CSRF is not a concern.

The rule of thumb worth carrying: **cookies for a browser app the server renders, bearer tokens for APIs consumed by clients you do not control.** A React SPA talking to your API is the ambiguous middle, and either works — cookies with `SameSite` and antiforgery are meaningfully safer than a token in `localStorage`, which is readable by any script that gets injected.

Multiple schemes coexist fine. `[Authorize(AuthenticationSchemes = "Bearer")]` picks one per endpoint; a default scheme covers the rest.

**In GROS**, service-to-service gRPC calls carry a bearer token — the calling service is the subject, there is no user, and the claims describe the *service*. That is a useful thing to have internalised: a principal does not have to be a person, and "who is calling" is a more general question than "who is logged in".

---

## 4. What "validating a JWT" actually means

A JWT is three base64url segments: header, payload, signature. Configuring the handler looks small and every line is load-bearing.

```csharp
.AddJwtBearer(o =>
{
    o.Authority = "https://login.example.com";   // where to fetch signing keys (JWKS)
    o.Audience  = "gros-api";
    o.TokenValidationParameters = new()
    {
        ValidateIssuer           = true,   // who minted it
        ValidateAudience         = true,   // was it minted for *this* API
        ValidateLifetime         = true,   // exp / nbf, with ClockSkew
        ValidateIssuerSigningKey = true,   // is the signature real
    };
});
```

Four independent checks, and skipping any one of them breaks a different thing:

- **Signature** — proves the token was minted by someone holding the key and has not been altered. Without it, anyone can write any claims they like.
- **Issuer** — proves *which* someone. A valid signature from a different identity provider is still a valid signature.
- **Audience** — proves the token was minted for this API. Without it, a token your users legitimately obtained for a *different* service is accepted here. This is the one most often disabled during debugging and never re-enabled, and it turns every other service sharing your identity provider into a way in.
- **Lifetime** — `exp` and `nbf`. There is a default `ClockSkew` of five minutes, which surprises people writing tests around expiry.

On keys: **symmetric (HS256)** means the API holds the same secret used to sign, so anything that can validate can also mint. **Asymmetric (RS256/ES256)** means the provider signs with a private key and your API validates with the public one, fetched from the authority's JWKS endpoint and rotated without redeploying you. For anything beyond a single service signing its own tokens, asymmetric is the right default.

The historical attack worth knowing because it explains the design: `alg: none`, where a token declares it is unsigned and a naive library believes it. Modern handlers reject this, and the related class — algorithm confusion, submitting an RS256 public key as an HS256 shared secret — is why you configure *which* algorithms are acceptable rather than trusting the token's own header. **A token's header is attacker-controlled input.** Never let it choose its own validation rules.

---

## 5. ASP.NET Core Identity is a membership store, not an authentication mechanism

This confuses people constantly, and the confusion is in the name.

**ASP.NET Core Identity** is a library for *owning user accounts*: an EF Core-backed store of users and roles, password hashing (PBKDF2 with a sensible iteration count, salted, and upgradable), lockout after failed attempts, email and phone confirmation, two-factor, and password reset token generation. It answers "is this password correct for this user" and "here is that user's roles".

It is **not** an OIDC provider. It does not issue JWTs, it does not implement the authorization code flow, and it has no concept of a third-party client application. If you need to be an identity *provider* for other apps, that is Duende IdentityServer, OpenIddict, or a managed service like Entra ID or Auth0.

So the real decision is: **do you want to own accounts at all?**

Owning them means you own password storage, credential-stuffing defence, account recovery flows, the email deliverability of your reset links, and a breach if you get any of it wrong. Delegating to an OIDC provider means your app never sees a password — you receive an identity token, validate it (§4), and map claims onto your own authorisation model.

The honest position for most line-of-business software: **delegate authentication, own authorization.** Let the provider prove who someone is; keep the decision about what they may do in your own domain, because that decision is business logic and it lives with the data. This is also the shape that survives an acquisition, an SSO requirement, or a customer who insists on their own directory.

---

## 6. Authorization: roles, policies, and the point where roles run out

**A role is just a claim** of type `role`. `[Authorize(Roles = "Planner")]` is a string comparison against claims on the principal. That is the whole implementation, and knowing it explains exactly when roles stop being enough: they encode *who someone is*, and permission questions are usually about *what they may do* and *to which thing*.

The first time you write `Roles = "Admin,Supervisor,Planner"` you have discovered that you actually meant a permission, and you are enumerating everyone who has it. Add a fourth role and you edit every attribute.

**Policies** name the requirement instead of the people:

```csharp
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("CanApproveLayout", p => p.RequireClaim("permission", "layout:approve"));
    o.AddPolicy("InternalOnly",     p => p.RequireAssertion(c => c.User.HasClaim("tenant", "internal")));
});

[Authorize(Policy = "CanApproveLayout")]
```

Now the attribute states the requirement, and *who satisfies it* is a data question answered wherever claims are issued. That is the right seam.

For anything more involved, a policy is a set of `IAuthorizationRequirement`s with `AuthorizationHandler<T>` implementations resolved **from the container** — chapter 03, unchanged: constructor-inject what the decision needs, and the handler is testable without a web server.

**Resource-based authorization** is the level people reach last and need most. "May this user approve *this* layout" cannot be answered from claims alone, because the answer depends on the row:

```csharp
var result = await authz.AuthorizeAsync(User, layout, "CanApproveLayout");
if (!result.Succeeded) return Forbid();
```

This runs *inside* the handler, after loading the resource, because it has to. No attribute can express it — the attribute runs before you have the object. The moment a permission depends on ownership, tenancy or state, it stops being metadata and becomes code, and trying to keep it declarative is how multi-tenant data leaks happen.

---

## 7. Securing endpoints is metadata — and default-deny is one line

`[Authorize]` and `RequireAuthorization()` do the same thing through different syntax: they attach metadata to an endpoint, which `UseAuthorization` reads.

```csharp
app.MapControllers().RequireAuthorization();
app.MapGet("/health", () => "ok").AllowAnonymous();
```

The important part is the default, and this is the section's real content. By default, **an endpoint with no authorization metadata is public.** Protection is opt-in, which means the failure mode is silent: a new controller written on a Friday is world-readable and nothing in the build, the tests or the logs says so.

Invert it:

```csharp
builder.Services.AddAuthorization(o =>
{
    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

`FallbackPolicy` applies to every endpoint that has **no** authorization metadata at all. Forgetting `[Authorize]` now fails closed — the endpoint 401s until someone deliberately writes `[AllowAnonymous]`. That is a one-line change that converts an entire class of bug from "silently public" into "obviously broken", and it is close to free.

Worth keeping straight, because the names are unhelpfully similar: **`DefaultPolicy`** is what a bare `[Authorize]` means (by default, "any authenticated user"). **`FallbackPolicy`** is what applies when there is no attribute at all. Setting the first does nothing for the endpoint somebody forgot.

Two more that belong here:

- **`[AllowAnonymous]` wins.** It short-circuits authorization regardless of anything at the controller or fallback level. A stray one on a base class is worth grepping for.
- **CORS is not an authorization control.** It is a *browser* restriction on which origins may read a response. It does nothing to `curl`, a mobile app, or a server. An endpoint that is "protected by CORS" is protected against exactly one category of client and open to everyone else.

---

## 8. HTTPS, HSTS, and where identity meets transport

Every mechanism in this chapter is a bearer of some kind: whoever holds the cookie or the token *is* the user. So all of it rests on the transport, and this is where authentication becomes a deployment question.

```csharp
app.UseHttpsRedirection();   // 307 http → https
app.UseHsts();               // Strict-Transport-Security: tell the browser never to try http again
```

`UseHttpsRedirection` fixes the *current* request after it has already been sent in clear text — including its cookie. HSTS fixes the *next* one, by instructing the browser to refuse plaintext for this host for `max-age` seconds. That is the useful distinction: redirection is a courtesy, HSTS is the actual control.

Two things to know before enabling it: HSTS is excluded on localhost by default (so it does not poison your dev machine), and **it is difficult to undo**. A browser that has cached a one-year `max-age` will refuse http to your domain for a year, and `preload` is harder still — it ships in the browser's binary. Set a short `max-age` first and raise it once you are sure every subdomain can serve TLS.

Cookies get their own flags, and they are not optional: `Secure` (never sent over http), `HttpOnly` (invisible to JavaScript, so an XSS bug cannot exfiltrate the session), and `SameSite` (not sent on cross-site requests, which is the modern CSRF defence).

And then the part that breaks in production and not on your machine — **your TLS is usually terminated before your app.** nginx, an ingress controller or a load balancer holds the certificate and forwards plain HTTP inside the network. Chapter 01 §6 covered the mechanism; here is what it does to auth specifically:

- `Request.Scheme` is `http`, so `UseHttpsRedirection` issues a redirect to a URL that comes straight back — an infinite loop.
- `Secure` cookies are never set, because the framework believes the connection is insecure.
- OIDC generates a `redirect_uri` beginning `http://`, and the identity provider rejects it as unregistered.

All three are one cause, and one fix: `UseForwardedHeaders` early in the pipeline, configured with the proxies you actually trust. Do not skip the trusted-network configuration — `X-Forwarded-For` and `X-Forwarded-Proto` are client-supplied headers, and accepting them from anywhere lets a caller assert their own scheme and address.

---

## 9. Security is layers, and this chain is the spine

*How do you secure an application* is an umbrella question, and the temptation is to answer with a list. The better answer is that every control lives at a specific layer, and the layers you cannot bolt on afterwards are the ones worth getting right first.

Reading outward from the request:

| Layer | The control | Where it came from |
|---|---|---|
| Transport | TLS, HSTS, secure cookie flags, trusted proxy headers | §8, ch. 01 §6 |
| Identity | A validated principal — all four JWT checks, not three | §4 |
| Permission | Default-deny fallback, policies, resource checks | §6–7 |
| Input | Model validation, parameterised queries, output encoding | EF Core parameterises by default — ch. 04 §3 |
| Secrets | Configuration providers, never the repository | ch. 02 §5 |
| Data | Least privilege on the database account | ch. 04 §8 |
| Runtime | Correct environment, patched base images, no debug endpoints | ch. 02, ch. 03 §3 |

Three of those rows are threads this handbook has already pulled, and they are worth making explicit because they are the ones that get missed:

**Never ship with `ASPNETCORE_ENVIRONMENT=Development`.** The developer exception page renders stack traces, source snippets, and often connection strings, to whoever triggered the error. Chapter 03 §3 noted that scope validation is Development-only; this is the same switch with a far worse failure mode.

**The database account probably has more rights than it needs.** Chapter 04 §8 pointed out that migrating on startup requires permanent DDL permission for the application's runtime identity. An account that can `DROP TABLE` because of a convenience during the first ninety seconds of a deployment is a standing risk for the rest of the release.

**Secrets belong in the configuration stack, not the repository.** Chapter 02 §5 was explicit that user secrets are not encrypted — they are merely outside the repo. In production the layer is a vault or the orchestrator's secret store, and the signing key from §4 is exactly the kind of thing this is about: leak it and every token in the system is forgeable.

Deployment-specific, and easy to defer forever: turn off detailed errors, remove `Server` and other fingerprinting headers, keep base images patched, and make sure your logs do not contain the tokens you were debugging last month.

---

## 10. The detail most people miss: a JWT is signed, not secret

Signing proves a token has not been *altered*. It does nothing to stop it being *read*.

The payload is base64url — not encryption, just encoding. Anyone holding the token, and anyone who can see it in a log, a browser devtools panel, a proxy trace or a crash report, can paste it into jwt.io and read every claim in it. Teams put email addresses, internal user IDs, tenant names, role structures and occasionally worse into tokens, on the unexamined assumption that "signed" implies "private". It does not. If a claim would be a problem to disclose, it does not belong in a JWT unless you are using JWE, which almost nobody is.

The second half of the same misunderstanding is worse, and it is the reason bearer tokens are a genuine architectural trade rather than a free win:

**A JWT cannot be revoked.** It is self-contained by design — that is the whole point, the reason no server lookup is needed and the reason it scales. But it also means that from the moment a token is issued until `exp`, it is valid, and there is no "log out" that can reach it. A stolen token works. A fired employee's token works. A token leaked in a log works. Right up until it expires, and not one second less.

Which is why the practical design is always the same shape: **short-lived access tokens, long-lived refresh tokens.** The access token is minutes, so the exposure window is small. The refresh token is stored server-side, is revocable, and is the thing you actually invalidate on logout or compromise. If you find yourself issuing eight-hour access tokens because refresh is inconvenient, you have chosen an eight-hour window in which no revocation is possible — and that is a decision worth making deliberately rather than by omission.

If you genuinely need instant revocation, you need server-side state: a blocklist checked per request, or reference tokens instead of self-contained ones. Both give back the statelessness you chose JWTs for. That trade is the honest answer to "how do I invalidate a JWT", and the interview answer worth giving is that **you don't — you make the window short enough that it doesn't matter, or you stop using self-contained tokens.**

---

## Common failure modes

| Symptom | Cause |
|---|---|
| Every authenticated request gets 401 | `UseAuthorization` before `UseAuthentication`, so authorization ran against an anonymous principal |
| `[Authorize]` endpoints are reachable with no credentials | `UseAuthorization()` missing entirely — the attribute is metadata nothing reads |
| 403 rather than 401 with a valid token | Authentication worked. This is a permission failure — the claim or policy, not the token |
| 401 with a token that works elsewhere | Wrong audience or issuer for *this* API, or a different signing key |
| Tokens accepted that were minted for another service | `ValidateAudience = false`, usually left over from debugging |
| Token expiry behaves oddly in tests | The default five-minute `ClockSkew` |
| Infinite redirect loop behind a proxy | TLS terminated upstream; `Request.Scheme` is `http`. Needs `UseForwardedHeaders` |
| `Secure` cookies never set in production | Same cause — the framework believes the connection is plaintext |
| OIDC rejects `redirect_uri` as unregistered | Same cause again — the URI was generated as `http://` |
| A new endpoint is public and nobody noticed | No `FallbackPolicy`; authorization is opt-in by default |
| An endpoint is protected in the browser and open to `curl` | CORS mistaken for an access control |
| Logging out does not stop the old token working | JWTs are valid until `exp`. Revocation needs server-side state |
| A stack trace with a connection string is returned to a user | `ASPNETCORE_ENVIRONMENT=Development` in production |
| Personal data visible to anyone holding a token | The payload is encoded, not encrypted |

---

## Check yourself

Answer out loud, without looking:

1. `UseAuthentication` runs and finds no credentials at all. What does it do to the request — and what is the *next* thing that has any opinion about it?
2. Both auth middlewares must run after `UseRouting`. Give the reason in terms of what authorization needs to read.
3. You disable audience validation to unblock a demo. Name what is now accepted that was not before, and say who can exploit it.
4. Roles are claims, and policies are also evaluated against claims. So what does a policy actually buy you, and what can neither of them express?
5. A user's account is disabled at 10:00. Their access token expires at 10:45. What can they do in between, and what are the two ways to change that answer?

---

## Questions this chapter answers

Seven of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch06):

| # | As originally asked | Answered by section |
|---|---|---|
| 27 | How do you ensure the security of an application? | §9 — layers, with this chain as the spine |
| 48 | Security considerations when deploying | §8–9 — transport, environment, least privilege, secrets |
| 50 | How do you configure HTTPS and SSL? | §8 — redirection versus HSTS, and the proxy that terminated it |
| 80 | How does .NET Core handle authentication and authorization? | §1–3 — a principal, then a decision; a scheme is how it arrives |
| 81 | What is the Identity system? | §5 — a membership store, not an OIDC provider |
| 82 | How to implement JWT authentication | §4 — four checks, and why each one matters separately |
| 84 | How can you secure API endpoints? | §6–7 — policies over roles, and default-deny as one line |

Chapter 07 asks how you know any of this works: 41–45.

## Next

→ [`07-testing-and-tdd.md`](07-testing-and-tdd.md) — §6 argued that an authorization handler is testable without a web server, and chapter 03 argued that DI is what makes any of it substitutable. The next chapter is what you do with that.
