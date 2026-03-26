# Flask in This Service — A Guide for New Interns

Welcome! This document explains what Flask is, how web frameworks work in general, and how **this specific service** uses Flask. No prior experience required.

---

## What is Flask?

Flask is a **Python web framework**. A web framework is a library that handles the boring, repetitive parts of building a web server so you can focus on your actual business logic.

Without a framework, you'd have to write code that:
- Opens a network socket and listens for connections
- Parses raw HTTP text (methods, headers, paths, bodies)
- Sends correctly formatted HTTP responses back

Flask does all of that for you. You just write Python functions.

---

## The Mental Model — Waiter at a Restaurant

Think of your service as a restaurant:

```
Client (caller)  →  Waiter (Flask)  →  Kitchen (your Python code)
                 ←                  ←
```

- The **client** places an order (HTTP request)
- The **waiter** (Flask) receives it, identifies what was ordered, and passes it to the right cook
- The **kitchen** (your business logic) prepares the response
- The **waiter** delivers the response back to the client

Flask is the waiter. It never cooks food — it just routes orders to the right place.

---

## How Flask Works — Step by Step

### Step 1: Create the App

```python
# controller.py
from flask import Flask

app = Flask(__name__)
```

`Flask(__name__)` creates your web application. This is the central object — you register all routes on it.

---

### Step 2: Define a Route

A **route** tells Flask: *"When someone sends a request to this URL, call this Python function."*

```python
@app.route('/hello', methods=['GET'])
def say_hello():
    return 'Hello, World!'
```

The `@app.route(...)` part is a Python **decorator** — think of it as a label you stick on a function telling Flask to watch for that URL.

```
HTTP GET /hello
       |
       Flask matches → say_hello()
       |
       Returns "Hello, World!" as HTTP response
```

---

### Step 3: Run the Server

```python
if __name__ == '__main__':
    app.run(debug=True)
```

Or via the command line (how this service does it):

```bash
FLASK_APP=controller.py flask run --host=0.0.0.0 --port=10087
```

`flask run` starts a web server that listens for HTTP requests on port 10087.

---

## URL Parameters — Dynamic Routes

URL parameters let you capture values directly from the URL path.

```python
@app.route('/users/<user_id>', methods=['GET'])
def get_user(user_id):
    return f"Fetching user: {user_id}"
```

- `<user_id>` is a **placeholder** in the URL
- Flask automatically extracts the value and passes it as a function argument
- `GET /users/42` → `user_id = "42"`

---

## Reading Request Headers

HTTP headers are key-value pairs sent alongside the request (like metadata). Flask exposes them via `request.headers`:

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/example', methods=['POST'])
def example():
    client_name = request.headers.get('client')  # e.g., "kli", "ifl"
    domain = request.headers.get('domain')        # e.g., "incentihub"
    return f"Hello {client_name} from {domain}"
```

---

## How THIS Service Uses Flask

Now that you understand the basics, here's exactly what this service does — it maps directly to everything above.

### The App — `controller.py`

```python
from flask import Flask, request
from common_filters.server import apply_filters
import service_handler
from logger import print

app = Flask(__name__)

apply_filters(app)  # Registers internal middleware (auth, filters)
```

`apply_filters(app)` is an internal library call that attaches pre-built filters to the Flask app (similar to installing security cameras before the waiter starts working).

---

### The One and Only Route

```python
@app.route('/accounting-ledger-service/<payout_cycle_id>/<gl_acc_rule_id>', methods=['POST'])
def accountingledgercontroller(payout_cycle_id, gl_acc_rule_id):
    ...
```

This service has **exactly one route**. Flask extracts two URL parameters:

| URL Parameter | Example | Meaning |
|---|---|---|
| `payout_cycle_id` | `100-InsuranceAdvisor-20240301-20240331` | Which payout cycle to process |
| `gl_acc_rule_id` | `COMMACCRUEAGT` | Which accounting rule to apply |

A real request looks like:
```
POST /accounting-ledger-service/100-InsuranceAdvisor-20240301-20240331/COMMACCRUEAGT
```

---

### Reading Headers Inside the Route

The route then reads **four headers** from the incoming request:

```python
def accountingledgercontroller(payout_cycle_id, gl_acc_rule_id):
    request_data = {}
    request_data["client"]   = request.headers.get('client')    # Tenant ID
    request_data["domain"]   = request.headers.get('domain')    # Domain
    request_data["stage_id"] = request.headers.get('stage_id')  # e.g., "accrual"
    request_data["stage"]    = request.headers.get('stage')     # e.g., "CLIENT_SB"
```

These four values + the two URL parameters are bundled into a `request_data` dict and passed to the actual business logic.

---

### Handing Off to Business Logic

```python
    return service_handler.accounting_handler(request_data)
```

Flask doesn't know anything about accounting rules or GL codes. Its only job was to receive the HTTP request, extract the data, and hand it off. From here, `service_handler.py` does all the real work.

---

### Error Handling

```python
    try:
        ...
        return service_handler.accounting_handler(request_data)
    except Exception as e:
        return {'status': 500, 'message': 'Internal Server Error'}
```

If anything goes wrong, Flask returns a simple error dictionary. Flask automatically converts Python dicts to JSON HTTP responses.

---

### Starting the Server (Development Mode)

The controller has this at the bottom:

```python
if __name__ == '__main__':
    app.run(debug=True)
```

This means if you run `python controller.py` directly, Flask starts a development server. In production, the `flask run` command is used instead.

---

## The Full Picture — Visualised

```
Incoming HTTP Request
POST /accounting-ledger-service/cycle123/COMMACCRUEAGT
Headers: client=kli, domain=incentihub, stage_id=accrual, stage=CLIENT_SB
          |
          v
     [ Flask App ]
     - Matches URL pattern
     - Extracts: payout_cycle_id="cycle123", gl_acc_rule_id="COMMACCRUEAGT"
          |
          v
  accountingledgercontroller()
     - Reads headers → builds request_data dict
     - Calls service_handler.accounting_handler(request_data)
          |
          v
  [ Business Logic — NOT Flask ]
     - Load properties from DynamoDB
     - Fetch payout cycle data
     - Select correct accounting class (Factory Pattern)
     - Fetch entitlement data, build CSV
     - Upload via Batch API
          |
          v
     Returns result string/dict
          |
          v
     [ Flask ]
     - Converts return value to HTTP Response (200 OK)
          |
          v
HTTP Response back to caller
```

---

## What Flask Does vs What This Service Does

| Task | Who handles it |
|---|---|
| Listen on port 10087 | Flask |
| Parse the URL, extract parameters | Flask |
| Read HTTP headers | Flask (`request.headers`) |
| Route to the correct function | Flask (`@app.route`) |
| Load config from DynamoDB | `service_handler.py` + `property-fetcher` |
| Fetch payout/entitlement data | `Util.py`, `account.py` |
| Build accounting CSV | `account.py`, `impl/*.py` |
| Upload to batch API | `BulkBatchAction.py` |

Flask is a thin shell. 95% of this service's complexity lives outside of Flask entirely.

---

## Key Flask Concepts — Quick Reference

| Concept | Code | Meaning |
|---|---|---|
| App instance | `app = Flask(__name__)` | The central Flask object |
| Route decorator | `@app.route('/path', methods=['POST'])` | Maps a URL to a function |
| URL parameter | `<variable_name>` in the path | Captured and passed as function arg |
| Read a header | `request.headers.get('key')` | Access incoming HTTP headers |
| Return a response | `return "text"` or `return {"key": "value"}` | Flask converts to HTTP response |
| Run the server | `flask run` or `app.run()` | Starts the web server |

---

## Recommended Next Steps

If you want to learn Flask more deeply, try these in order:

1. **Official Flask quickstart** — https://flask.palletsprojects.com/en/1.1.x/quickstart/
2. **Build a tiny API** — create a `/ping` route that returns `{"status": "ok"}`
3. **Read request body** — try `request.get_json()` to parse a JSON POST body
4. **Read this service's `controller.py`** — it will now make complete sense
