```python
import pandas as pd
import random
import uuid
from datetime import datetime, timedelta
import os

# Directory to store generated files
output_dir = "./generated_data"
os.makedirs(output_dir, exist_ok=True)

# Static dataset: country code mapping
countries = [
    {"country_code": "US", "country_name": "United States"},
    {"country_code": "CA", "country_name": "Canada"},
    {"country_code": "UK", "country_name": "United Kingdom"},
    {"country_code": "IN", "country_name": "India"},
    {"country_code": "AU", "country_name": "Australia"},
]

# Save static CSV
country_df = pd.DataFrame(countries)
static_path = os.path.join(output_dir, "country_codes.csv")
country_df.to_csv(static_path, index=False)

# Streaming dataset: random user events
event_types = ["login", "purchase", "logout", "click"]
now = datetime.now()
stream_events = []

for _ in range(500):
    event = {
        "user_id": str(uuid.uuid4())[:8],
        "country_code": random.choice([c["country_code"] for c in countries]),
        "event_time": (now - timedelta(minutes=random.randint(0, 30))).strftime("%Y-%m-%d %H:%M:%S"),
        "event_type": random.choice(event_types),
    }
    stream_events.append(event)

streaming_df = pd.DataFrame(stream_events)

# Save as newline-delimited JSON (suitable for Spark readStream)
stream_path = os.path.join(output_dir, "sample_streaming_events.json")
streaming_df.to_json(stream_path, orient="records", lines=True)

print(f"✔ Static file saved to: {static_path}")
print(f"✔ Streaming file saved to: {stream_path}")

```
