# Northstar Inventory Sync 

A live inventory sync service for Northstar Retail Co.'s support tool.
It polls a warehouse API on a timer, caches the result, and exposes
a query endpoint + dashboard so the support tool (or a human) always
gets an instant, up-to-date answer to "is this in stock?"

## How it works

```
#Backend

warehouse_api.py          sync_service.py                 browser
(simulated warehouse) --> polls every 10s --> cache      --> templates/index.html
   port 5001                  |               (in-memory       + static/css/style.css
                               v                + cache.json)   + static/js/app.js
                     GET /inventory   (query endpoint)              |
                     GET /            (dashboard shell)   fetch()---+ every 10s
                          port 5000
```

- `warehouse_api.py` — stands in for Northstar's real warehouse system.
  Serves `GET /api/inventory`.
- `sync_service.py` —  Polls the warehouse API
  every 10 seconds(POLL_INTERVAL_SECONDS=10) for easy grading and testing (specification calls for
  5 minutes in production). Caches the latest
  stock, and serves:
  - `GET /inventory` — returns the full cached inventory as JSON (the query endpoint)
  - `GET /inventory/<id>` — performs a single product lookup
  - `GET /` — provides a human-readable dashboard that auto-refreshes every 10s to display real-time updates

 #Frontend
 
templates/index.html — the dashboard page structure
static/css/style.css — dark warehouse-ops visual style: status colors (green/amber/red) borrowed from warehouse floor signage, monospace type for anything that's a "reading" (IDs, quantities, timestamps)
static/js/app.js — polls 'GET /inventory' every 10s (same as the backend sync) and updates the table without a full page reload; flashes a row when its quantity changes, and pulses a status dot on every successful sync

The frontend never talks to 'warehouse_api.py' directly — only ever to 'sync_service.py's /inventory' endpoint. That's the whole point of the cache: the user interface stays fast and responsive even if the warehouse API is slow or temporarily down.

## Run it locally

```bash
pip install -r requirements.txt

# terminal 1
python3 warehouse_api.py

# terminal 2
python3 sync_service.py
```

Then open:
- Dashboard: http://127.0.0.1:5000/
- Query endpoint: http://127.0.0.1:5000/inventory

## Starting inventory

| Product ID | Product Name     | Quantity |
| ---------- | -----------------| -------- |
| ELYV-001   | Elysia Vanilla   | 12       |
| ATH-001    | Atheeri          | 8        |
| KHC-001    | Khair Confection | 5        |
| YY-001     | Yum Yum          | 0        |
| JZG-001    | Jazaab Gold      | 7        |

(`warehouse_api.py` randomly nudges quantities on each poll so the
dashboard has something real to show changing over time.)





# Northstar-Co-Ltd
Syncing service for Northstar Company inventory
from flask import Flask, jsonify
import threading
import time
import requests

app = Flask(__name__)

# -------------------------------------------------
# 1.WAREHOUSE INVENTORY
# -------------------------------------------------

warehouse_inventory = {
    "ANG-001": {
        "name": "Angham",
        "quantity": 10
    },
    "JZG-001": {
        "name": "Jazaab Gold",
        "quantity": 7
    },
    "ELY-VAN-001": {
        "name": "Elysia Vanilla",
        "quantity": 8
    },
    "ATH-001": {
        "name": "Atheeri",
        "quantity": 12
    },
    "KHC-001": {
        "name": "Khair Confection",
        "quantity": 15
    }
}


# -------------------------------------------------
# 2. CACHED INVENTORY
# -------------------------------------------------

cached_inventory = {}


# -------------------------------------------------
# 3.WAREHOUSE API
# -------------------------------------------------

@app.route("/warehouse/inventory")
def warehouse_api():
    """
    Pretends to be the warehouse's inventory API.
    """
    return jsonify(warehouse_inventory)


# -------------------------------------------------
# 4. RETRY + EXPONENTIAL BACKOFF
# -------------------------------------------------

def fetch_warehouse_inventory():

    max_retries = 3
    delay = 1

    for attempt in range(max_retries):

        try:
            response = requests.get(
                "http://127.0.0.1:5000/warehouse/inventory",
                timeout=5
            )

            response.raise_for_status()

            print("Warehouse API request successful.")

            return response.json()

        except requests.RequestException as error:

            print(
                f"Attempt {attempt + 1} failed: {error}"
            )

            if attempt == max_retries - 1:
                print("All retry attempts failed.")
                return None

            print(f"Waiting {delay} seconds before retrying...")

            time.sleep(delay)

            # Exponential backoff
            delay *= 2


# -------------------------------------------------
# 5. POLLER
# -------------------------------------------------

def poll_warehouse():

    while True:

        latest_inventory = fetch_warehouse_inventory()

        if latest_inventory is not None:

            cached_inventory.clear()
            cached_inventory.update(latest_inventory)

            print("Inventory cache updated.")
            print(cached_inventory)

        else:

            print(
                "Could not update inventory. "
                "Keeping existing cache."
            )

        # Wait 5 minutes before polling again
        time.sleep(300)


# -------------------------------------------------
# 6. SUPPORT QUERY ENDPOINT
# -------------------------------------------------

@app.route("/inventory/<product_id>")
def check_stock(product_id):

    if product_id not in cached_inventory:
        return jsonify({
            "error": "Product not found"
        }), 404

    product = cached_inventory[product_id]

    return jsonify({
        "product_id": product_id,
        "product_name": product["name"],
        "quantity": product["quantity"],
        "in_stock": product["quantity"] > 0
    })


# -------------------------------------------------
# 7. START THE POLLER
# -------------------------------------------------

poller_thread = threading.Thread(
    target=poll_warehouse,
    daemon=True
)

poller_thread.start()


# -------------------------------------------------
# 8. START THE SERVER
# -------------------------------------------------

if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=5000
    )
