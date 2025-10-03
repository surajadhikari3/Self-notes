

Yes — Deephaven supports **PSK (Pre-Shared Key) authentication**, but it depends on how your server is deployed.

Here’s the breakdown:

---

### 🔑 1. What PSK auth is in Deephaven

- Deephaven has a **Pre-Shared Key authentication mode**, where a static secret key is configured on the server.
    
- Any client (Python, Java, JS API, gRPC) that knows this key can connect, without needing user accounts, LDAP, or OAuth.
    
- It’s a simple **“shared secret”** mechanism, mainly for internal deployments, testing, or when you’re running in a controlled environment.
    

---

### ⚙️ 2. How to enable PSK authentication

When starting **Deephaven server**, you can set the authentication provider to `psk`:

```bash
./start --auth psk --psk MY_SUPER_SECRET_KEY
```

Or in Docker:

```yaml
services:
  deephaven:
    image: ghcr.io/deephaven/server
    environment:
      - DH_AUTH_TYPE=psk
      - DH_PSK=MY_SUPER_SECRET_KEY
    ports:
      - "10000:10000"
```

---

### 💻 3. How clients connect with PSK

- **Python client**:
    

```python
from deephaven import Session

session = Session("localhost:10000", auth_type="psk", auth_token="MY_SUPER_SECRET_KEY")
```

- **Java client**:
    

```java
try (Session session = SessionFactory.builder()
        .host("localhost")
        .port(10000)
        .authenticationType("psk")
        .authenticationValue("MY_SUPER_SECRET_KEY")
        .build()
        .newSession()) {
    // use the session...
}
```

- **JavaScript client** (`dh-core`):
    

```js
import dh from "@deephaven/jsapi";

const client = new dh.Client({
  url: "http://localhost:10000",
  type: "psk",
  value: "MY_SUPER_SECRET_KEY",
});
```

---

### 📌 4. Where PSK fits

- ✅ Good for **internal dev/test** or when you don’t want to stand up OAuth/SSO.
    
- ✅ Easy for **Angular / Spring Boot → DH integration**, since you just pass the PSK in headers.
    
- ❌ Not recommended for production across the open internet (use mTLS or OAuth instead).
    

---

👉 Do you want me to **adapt your current Spring Boot + Deephaven dynamic Kafka setup** to use PSK authentication (instead of OAuthBearer SASL) so it’s simpler to wire up?