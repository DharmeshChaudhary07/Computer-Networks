# Week 2 CN Notes (Short): HTTP/HTTPS and DNS

Only what your roadmap asks for:

- **HTTP/HTTPS:** request/response cycle, status codes, headers, REST principles
- **DNS:** how a domain resolves to an IP, and why it matters for connecting to cloud services

Written for a beginner. Every term is explained in simple words.

---

## 1. Basic words first

| Word | Simple meaning |
| --- | --- |
| **Client** | The one who asks for something (your browser, Python script, Airflow task) |
| **Server** | The computer that answers (an API, a website, a database) |
| **Protocol** | A set of rules two computers follow to talk, like a language |
| **URL** | The full address of something on the internet |
| **IP address** | The number address of a machine, like 142.250.195.46 |
| **Domain name** | The human-friendly name of a machine, like google.com |
| **API** | A way for one program to ask another program for data, without knowing how it works inside |
| **JSON** | A text format for data, like a Python dictionary: `{"name": "Asha", "qty": 2}` |

---

## 2. HTTP

### 2.1 What is HTTP?

**HTTP (HyperText Transfer Protocol)** is the set of rules a client uses to ask a server for data, and the rules the server uses to answer. It works at the Application layer (Layer 7) and runs on top of TCP, which you learned in Week 1.

It is always **request and response**:

1. Client sends a **request**.
2. Server sends back a **response**.

**HTTP is stateless:** the server does not remember you between requests. So every request must carry what the server needs (like your token).

### 2.2 The URL

```
https://api.shop.com/v1/orders?status=paid&page=2
\___/   \_________/ \________/ \______________/
scheme     host       path        query string
```

- **Scheme:** `https` is the protocol.
- **Host:** the server's domain name.
- **Path:** which thing on the server you want.
- **Query string:** extra options after `?` as `key=value`, joined with `&`.

### 2.3 The request

A request has: a **request line** (method + path), **headers**, a blank line, and an optional **body**.

```
POST /v1/orders HTTP/1.1
Host: api.shop.com
Authorization: Bearer abc123
Content-Type: application/json

{"item": "protein", "quantity": 2}
```

### 2.4 The response

A response has: a **status line** (with the status code), **headers**, a blank line, and the **body**.

```
HTTP/1.1 201 Created
Content-Type: application/json

{"order_id": 981, "status": "created"}
```

### 2.5 Methods: what do you want to do?

| Method | Meaning | Example |
| --- | --- | --- |
| **GET** | Read data | Get a list of orders |
| **POST** | Create new data | Create a new order |
| **PUT** | Replace data fully | Replace a whole order |
| **PATCH** | Change part of data | Update only the order status |
| **DELETE** | Remove data | Delete an order |

**Safe to retry:** GET, PUT, DELETE (same result if repeated). **Careful:** POST may create duplicates if you repeat it.

### 2.6 Status codes: did it work?

The server answers with a 3-digit code. The first digit is the group.

| Group | Meaning |
| --- | --- |
| 2xx | Success |
| 3xx | Redirect (go somewhere else) |
| 4xx | **Your** mistake (client error) |
| 5xx | **Server's** problem (server error) |

**Codes to remember:**

| Code | Name | Meaning | What to do |
| --- | --- | --- | --- |
| 200 | OK | Success | Use the data |
| 201 | Created | New thing created | Success |
| 400 | Bad Request | Your request is wrong | Fix it, do not retry |
| 401 | Unauthorized | Not logged in, or bad or expired token | Fix or refresh the token |
| 403 | Forbidden | Logged in, but no permission | Ask for access |
| 404 | Not Found | Wrong URL or ID | Check the URL |
| 429 | Too Many Requests | You hit the rate limit | Wait, then retry |
| 500 | Internal Server Error | Server crashed | Retry later |
| 503 | Service Unavailable | Server overloaded or down | Retry later |

**Easy memory:** 401 = "who are you?", 403 = "I know you, but you are not allowed." 4xx = fix your request. 5xx = server trouble, retry with waiting.

### 2.7 Headers

**Headers** are `Name: value` lines that carry extra information about the request or response.

| Header | What it does |
| --- | --- |
| **Host** | Which domain you want |
| **Content-Type** | Format of the body (`application/json`) |
| **Accept** | Format you want back |
| **Authorization** | Your credentials (`Bearer <token>` or API key) |
| **Content-Length** | Size of the body |
| **Retry-After** | How many seconds to wait before retrying (with 429 or 503) |

### 2.8 REST principles

A **REST API** is a web API built on HTTP using a set of simple rules:

| Principle | Meaning |
| --- | --- |
| **Resources have URLs** | Every "thing" has its own address: `/orders`, `/orders/981` |
| **Use HTTP methods as actions** | GET reads, POST creates, PUT/PATCH updates, DELETE removes |
| **Stateless** | Each request carries everything needed. The server remembers nothing |
| **Client and server are separate** | They can change independently |
| **Data as JSON** | Responses are usually JSON |

**Good URL style:** use nouns (not verbs) and plurals. The method is the action.

| Action | Good | Bad |
| --- | --- | --- |
| List orders | `GET /orders` | `GET /getAllOrders` |
| Create order | `POST /orders` | `POST /createOrder` |
| Delete order | `DELETE /orders/981` | `GET /deleteOrder/981` |

### 2.9 Three habits for calling APIs as a DE

1. **Pagination:** APIs return data **a page at a time**. Loop until there is no more data.
2. **Retries on failure:** on 429 or 5xx, wait and retry (wait longer each time: 1 s, 2 s, 4 s).
3. **Always set a timeout**, or one stuck call can freeze your pipeline.

```python
import os, requests

session = requests.Session()   # reuses the connection, faster
session.headers["Authorization"] = f"Bearer {os.environ['API_TOKEN']}"

rows, page = [], 1
while True:
    r = session.get("https://api.shop.com/v1/orders",
                    params={"page": page, "per_page": 100},
                    timeout=(5, 30))          # always set a timeout
    r.raise_for_status()                      # error on 4xx / 5xx
    batch = r.json()["data"]
    if not batch:                             # no more pages
        break
    rows.extend(batch)
    page += 1
```

---

## 3. HTTPS

### 3.1 The problem with HTTP

HTTP sends everything as **plain text**. Anyone between you and the server could read or change it (passwords, tokens, data).

### 3.2 What is HTTPS?

**HTTPS = HTTP + TLS.** TLS (Transport Layer Security) wraps HTTP in an **encrypted tunnel**. Same HTTP, but secure. Uses port **443** (HTTP uses port 80).

HTTPS gives you:

| Promise | Meaning |
| --- | --- |
| **Encryption** | Nobody in the middle can read the data |
| **Integrity** | Nobody can change the data without being noticed |
| **Authentication** | You are really talking to the real server (checked with a **certificate**) |

**Certificate:** the server's ID card, signed by a trusted organisation (a Certificate Authority). Your computer checks it. If it is expired or for a different domain, you get an SSL error.

**Why it matters for DE:** all cloud services (S3, APIs) use HTTPS. Never turn off certificate checks (`verify=False`) in real pipelines. Never put secrets in the URL; put them in headers.

---

## 4. DNS

### 4.1 What is DNS?

**DNS (Domain Name System)** is the **phonebook of the internet**. Computers use IP addresses, but humans use names. DNS turns a name (`api.shop.com`) into an IP address (`203.0.113.25`).

**Why do we need it?**

- Names are easy to remember, numbers are not.
- The IP behind a name can change (server moved, failover), but the **name stays the same**.

### 4.2 How a domain resolves to an IP

When your code connects to `api.shop.com`:

1. **Check the cache.** Your program, your OS, then your resolver may already know the answer.
2. If not, your computer asks a **recursive resolver** (from your ISP, or public ones like Google `8.8.8.8`). Its job is to find the answer for you.
3. The resolver asks a **root server**: "Who handles `.com`?"
4. Then the **TLD server** (`.com`): "Who handles `shop.com`?"
5. Then the **authoritative server** of `shop.com`: "What is the IP of `api.shop.com`?" It gives the final answer: `203.0.113.25`.
6. The resolver **saves (caches)** the answer and gives it to you.
7. Your computer now connects to that IP (TCP handshake, then TLS, then the HTTP request).

```
Your PC -> Resolver -> Root -> .com server -> shop.com server -> IP address
```

**TTL (Time To Live):** how many seconds an answer may be cached. TTL = 300 means "remember this for 5 minutes."

### 4.3 Two records to know

| Record | Meaning | Example |
| --- | --- | --- |
| **A** | Name to an IP address | `api.shop.com -> 203.0.113.25` |
| **CNAME** | Name to **another name** (an alias) | `www.shop.com -> shop.com` |

### 4.4 Why DNS matters for connecting to cloud services

- **Cloud services are reached by name, not IP.** Examples: an S3 endpoint, an RDS database `mydb.abc123.ap-south-1.rds.amazonaws.com`, a Redshift cluster, a Kafka broker. Your connection string uses the name, and DNS finds the current IP.
- **IPs change.** If a database fails over or a server is replaced, the cloud updates DNS. Your code keeps working because you used the **name**. Never hard-code the IP.
- **Stale cache problem:** right after a change, some clients may still use the old IP until the TTL ends. This causes "works for some, fails for others."
- **Private names inside a cloud network (VPC):** internal service names resolve to private IPs.
- **Errors that mean DNS failed:** `Name or service not known`, `getaddrinfo failed`, `UnknownHostException`, `Could not resolve host`.

### 4.5 Commands to check DNS

```bash
nslookup api.shop.com    # shows the IP for the name
dig api.shop.com         # more detail, including TTL
```

If a connection fails, first check: **can the name be resolved?** If not, it is a DNS problem. If yes, move on to checking the port and permissions (Week 1: TCP, firewall).

---

## 5. Everything together: what happens when Python calls an API

`requests.get("https://api.shop.com/v1/orders")`:

1. **DNS** finds the IP of `api.shop.com`.
2. **TCP handshake** opens a connection (port 443).
3. **TLS handshake** sets up encryption and checks the certificate.
4. **HTTP request** is sent: `GET /v1/orders` with headers.
5. **Server replies** with a status code, headers and a JSON body.
6. Your code checks the **status code**, then reads the **JSON**.

---

## 6. Practice problems (with answers)

**Problem 1. Break down the URL** `https://api.weather.com/v2/forecast?city=Bhopal&days=3`. Name the scheme, host, path and query parameters.

- **Answer:** scheme = `https`, host = `api.weather.com`, path = `/v2/forecast`, query = `city=Bhopal` and `days=3`.

**Problem 2. Pick the method:** (a) read all customers, (b) create a customer, (c) delete a customer. Which is risky to retry blindly?

- **Answer:** (a) GET, (b) POST, (c) DELETE. **POST** is risky to retry because it may create duplicates.

**Problem 3. Status codes.** What does each mean and what do you do? (a) 401, (b) 404, (c) 429, (d) 503.

- **Answer:** (a) Bad or missing credentials: fix or refresh the token. (b) Wrong URL or ID: check it, do not retry. (c) Rate limit hit: wait, then retry. (d) Server unavailable: retry later with waiting.

**Problem 4. Pagination.** An API has 2,350 records and returns 100 per page. How many calls do you need?

- **Answer:** 2350 / 100 = 23.5, round up to **24 calls**. The last call returns 50 records.

**Problem 5. Debug it.** Where is the problem? (a) `Name or service not known`, (b) HTTP 401, (c) `CERTIFICATE_VERIFY_FAILED`.

- **Answer:** (a) **DNS** cannot resolve the name. (b) **Authentication**: token missing, wrong or expired. (c) **HTTPS/TLS**: the certificate is expired, for a different domain, or from an untrusted authority.

**Problem 6. DNS cache.** A DNS record has TTL = 300 seconds. A resolver cached it at 10:00:00. You change the IP at 10:01:00. When will that resolver give the new IP?

- **Answer:** The cache lasts until 10:05:00 (10:00:00 + 5 minutes). So until **10:05:00** it may still return the old IP.

---

## 7. Basic interview questions with answers

**Q1. What is HTTP?** It is the application layer protocol a client uses to request data from a server, and the server uses to respond. It is request and response based, runs on TCP, and is stateless.

**Q2. What happens in the HTTP request/response cycle?** The client sends a request (method, URL, headers, optional body). The server processes it and sends back a response (status code, headers, body).

**Q3. What does stateless mean?** The server does not remember previous requests. Each request must carry all the information needed, like the auth token.

**Q4. Explain the status code groups.** 2xx = success, 3xx = redirect, 4xx = client error (your request is wrong), 5xx = server error.

**Q5. Difference between 401 and 403?** 401: you are not authenticated (missing or invalid credentials). 403: you are authenticated but not allowed to access that resource.

**Q6. What are HTTP headers? Name a few.** Extra information sent with a request or response as `Name: value` lines. Examples: `Content-Type` (body format), `Authorization` (credentials), `Accept` (format wanted), `Retry-After` (how long to wait).

**Q7. What is REST? What are its principles?** A style for designing web APIs on HTTP. Resources have URLs, HTTP methods are the actions (GET, POST, PUT, DELETE), requests are stateless, and data is usually JSON.

**Q8. Difference between HTTP and HTTPS?** HTTPS is HTTP inside a TLS encrypted tunnel. It gives encryption, integrity and server authentication. HTTP uses port 80, HTTPS uses port 443.

**Q9. What is DNS and why do we need it?** DNS translates domain names into IP addresses. We need it because names are easier to remember and the IP behind a name can change.

**Q10. How does a domain resolve to an IP?** The computer checks its cache, then asks a recursive resolver. The resolver asks a root server, then the TLD server, then the authoritative server, which returns the IP. The answer is cached for its TTL.

**Q11. Why does DNS matter when connecting to cloud services?** Cloud services (S3, RDS, Redshift, Kafka) are reached by hostname, and their IPs can change. DNS maps the stable name to the current IP, so your connection works after failover or scaling. If DNS fails, you get errors like "Name or service not known".

**Q12. What is TTL in DNS?** Time To Live: how long (in seconds) a DNS answer can be cached before it must be looked up again.

**Q13. How do you handle a 429 error when calling an API?** Wait (use the Retry-After header if present), then retry, increasing the wait each time (exponential backoff). Also slow down the rate of requests.

**Q14. How do you pull a large dataset from an API?** Use pagination: loop through pages (page number, offset or cursor) until no more data comes back, set a timeout, and retry on temporary failures.