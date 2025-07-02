
import os
import json
import time
from datetime import datetime, timedelta
import random

# Set local output directory
output_dir = r"C:\Users\YourUsername\Downloads\streaming_json_files"
os.makedirs(output_dir, exist_ok=True)

# Sample data pools
products = ["laptop", "phone", "tablet", "monitor", "keyboard", "mouse", "speaker"]
users = [f"user_{i}" for i in range(1, 21)]
locations = ["Toronto", "Vancouver", "Montreal", "Calgary", "Ottawa"]
payment_methods = ["Credit Card", "PayPal", "Bank Transfer", "Cash", "Crypto"]
categories = {
    "laptop": "electronics",
    "phone": "electronics",
    "tablet": "electronics",
    "monitor": "electronics",
    "keyboard": "accessories",
    "mouse": "accessories",
    "speaker": "audio"
}

# Generate multiple JSON files simulating streaming data


for file_index in range(10):  # 10 files
    records = []
    for _ in range(10):  # 10 records per file
        product = random.choice(products)
        record = {
            "event_id": random.randint(100000, 999999),
            "user_id": random.choice(users),
            "product": product,
            "category": categories[product],
            "price": round(random.uniform(50.0, 1500.0), 2),
            "quantity": random.randint(1, 5),
            "total_amount": 0,  # Will calculate below
            "location": random.choice(locations),
            "payment_method": random.choice(payment_methods),
            "event_time": (datetime.now() - timedelta(seconds=random.randint(0, 60))).isoformat(),
            "status": random.choice(["ORDERED", "SHIPPED", "DELIVERED", "CANCELLED"]),
            "discount_applied": random.choice([True, False]),
            "promo_code": random.choice(["NEW10", "SUMMER20", "FREESHIP", None]),
            "device_type": random.choice(["mobile", "desktop", "tablet"]),
        }
        record["total_amount"] = round(record["price"] * record["quantity"], 2)
        records.append(record)

    # Save as NDJSON
    file_path = os.path.join(output_dir, f"stream_data_{file_index}.json")
    with open(file_path, "w") as f:
        for rec in records:
            f.write(json.dumps(rec) + "\n")

    print(f"Generated {file_path}")
    time.sleep(2)  # simulate file arrival over time
