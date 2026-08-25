# Epic EMS Shared Config

Runtime configuration shared across Epic EMS internal portals and scripts.

Files in this repo are the source of truth. Downstream consumers fetch them at runtime and keep a local hardcoded fallback for resilience.

## providers.json

Provider dropdown, AngelTrack tenant URL, and default patient state. Consumed by:

- **Stripe Credit Card Portal** (Node/Vercel) — fetched at cold-start by `api/providers.js`.
- Additional portals and Python scripts (see snippets below).

### Fetch URL

```
https://raw.githubusercontent.com/ericmoore705/epic-ems-config/main/providers.json
```

Cache: GitHub raw content is CDN-cached for ~5 minutes. Edits become visible everywhere within that window.

### Consumer snippets

**Node.js:**
```js
const CONFIG_URL = "https://raw.githubusercontent.com/ericmoore705/epic-ems-config/main/providers.json";
const FALLBACK = [/* your last-known-good copy */];
let cache = null;
async function getProviders() {
  if (cache) return cache;
  try {
    const r = await fetch(CONFIG_URL);
    if (r.ok) cache = await r.json();
  } catch (_) {}
  return cache || FALLBACK;
}
```

**Python:**
```python
import json, urllib.request
CONFIG_URL = "https://raw.githubusercontent.com/ericmoore705/epic-ems-config/main/providers.json"
FALLBACK = [ ... ]  # last-known-good copy
def get_providers():
    try:
        with urllib.request.urlopen(CONFIG_URL, timeout=5) as r:
            return json.loads(r.read())
    except Exception:
        return FALLBACK
```

## Editing

Edit locally, commit, push. Changes propagate to all consumers on their next cold-start (Vercel functions) or next run (scripts) — typically within 5 minutes of the push landing.
