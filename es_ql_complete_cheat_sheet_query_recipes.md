# ES|QL Complete Cheat Sheet & Query Recipes

**Elasticsearch Query Language (ES|QL)** is a piped query language for filtering, searching, transforming, and analyzing data directly inside Elasticsearch. Every query starts with a **source command** to generate a table, followed by **processing commands** chained with pipe (`|`) symbols.

```esql
FROM logs-* METADATA _score
| WHERE message : "error" AND response_code >= 500
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
| LIMIT 10
```

---

## 1. Source Commands
*Every ES|QL query must begin with one source command to produce the initial table.*

| Command | Usage | Example |
| :--- | :--- | :--- |
| **`FROM`** | Retrieves documents from one or more indices, data streams, or aliases. Add `METADATA` to include `_id`, `_index`, or `_score`. | `FROM logs-* METADATA _index, _score` |
| **`ROW`** | Creates a single-row table with literal values (useful for testing functions without querying an index). | `ROW a = "hello", b = 42` |
| **`SHOW`** | Returns cluster or index metadata (e.g., Elasticsearch version info). | `SHOW INFO` |
| **`TS`** | Queries time-series data streams (`TSDB`) optimized for metrics. | `TS metrics-*` |

---

## 2. Processing Commands
*Modify, filter, parse, enrich, or aggregate the table passed down from the previous pipe.*

| Command | Usage | Example |
| :--- | :--- | :--- |
| **`WHERE`** | Filters rows based on boolean conditions or search functions. | `\| WHERE status == 404 AND host != "localhost"` |
| **`EVAL`** | Creates new calculated columns or overwrites existing ones. | `\| EVAL mb = ROUND(bytes / 1048576.0, 2)` |
| **`STATS...BY`** | Aggregates rows into summary metrics grouped by one or more fields. | `\| STATS hits = COUNT(*) BY client.ip` |
| **`INLINE STATS`** | Computes aggregations and appends them as new columns without collapsing rows. | `\| INLINE STATS avg_lat = AVG(latency) BY host` |
| **`SORT`** | Orders rows by one or more columns (`ASC` or `DESC`). | `\| SORT @timestamp DESC, latency ASC` |
| **`LIMIT`** | Caps the number of returned rows (default is 1,000; max is 10,000). | `\| LIMIT 50` |
| **`KEEP`** | Retains only the specified columns in the output table. | `\| KEEP @timestamp, user.name, status` |
| **`DROP`** | Removes the specified columns from the output table. | `\| DROP raw_message, internal_id` |
| **`RENAME`** | Changes column names. | `\| RENAME src_ip AS source.ip` |
| **`DISSECT`** | Fast delimiter-based string extraction into new columns. | `\| DISSECT msg "%{ip} - %{user} [%{ts}]"` |
| **`GROK`** | Regex-based pattern extraction into new columns. | `\| GROK msg "%{IP:ip} %{WORD:method} %{URIPATH:path}"` |
| **`ENRICH`** | Adds fields from a pre-configured Elasticsearch enrich policy. | `\| ENRICH geo_policy ON client.ip WITH country` |
| **`LOOKUP JOIN`** | Left-joins rows with a `lookup` mode index on a matching key. | `\| LOOKUP JOIN users_lookup ON user.id` |
| **`MV_EXPAND`** | Splits a multivalued (array) field into one separate row per value. | `\| MV_EXPAND tags` |
| **`SAMPLE`** | Randomly samples a fraction of rows (between `0.0` and `1.0`). | `\| SAMPLE 0.1` |
| **`FORK` / `FUSE`** | Runs parallel query branches and merges/ranks results (`RRF` or `LINEAR`). | `\| FORK (WHERE msg:"a") (WHERE msg:"b") \| FUSE RRF` |

---

## 3. Search & Full-Text Functions
*Unlike exact filtering (`==`), search functions analyze text (tokenization, stemming, fuzziness) and calculate relevance scores when paired with `METADATA _score` and `SORT _score DESC`.*

| Function / Operator | Usage | Example |
| :--- | :--- | :--- |
| **`:` (Match Operator)** | Shorthand full-text match (uses `OR` logic between terms by default). | `\| WHERE description : "fluffy pancakes"` |
| **`MATCH`** | Full-text search with optional JSON parameters (`operator`, `fuzziness`, `minimum_should_match`). | `\| WHERE MATCH(title, "fluffy pancakes", {"operator": "AND"})` |
| **`MATCH_PHRASE`** | Matches exact phrases in exact word order. | `\| WHERE MATCH_PHRASE(message, "connection reset by peer")` |
| **`QSTR`** | Executes Lucene Query String syntax (wildcards, booleans, multi-field). | `\| WHERE QSTR("status:500 AND (timeout OR refused)")` |
| **`KQL`** | Runs Kibana Query Language syntax directly inside ES|QL. | `\| WHERE KQL("http.status_code >= 400 and service: auth")` |

---

## 4. Aggregate & Grouping Functions
*Used inside `STATS` (and `INLINE STATS`) commands.*

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`COUNT`** | Counts total rows (`COUNT(*)`) or non-null values (`COUNT(field)`). | `\| STATS total = COUNT(*), users = COUNT(user)` |
| **`COUNT_DISTINCT`** | Approximate count of unique values (optional precision threshold). | `\| STATS unique_ips = COUNT_DISTINCT(client.ip)` |
| **`SUM`** | Sums all numeric values in a group. | `\| STATS total_bytes = SUM(network.bytes) BY host` |
| **`AVG`** | Computes the arithmetic mean. | `\| STATS avg_ms = AVG(response_time) BY endpoint` |
| **`MIN` / `MAX`** | Returns the lowest or highest value (works on numbers and dates). | `\| STATS first = MIN(@timestamp), peak = MAX(cpu)` |
| **`MEDIAN`** | Returns the 50th percentile (resistant to outliers). | `\| STATS med_lat = MEDIAN(latency) BY service` |
| **`MEDIAN_ABSOLUTE_DEVIATION`** | Measures dispersion/jitter around the median. | `\| STATS jitter = MEDIAN_ABSOLUTE_DEVIATION(interval)` |
| **`PERCENTILE`** | Returns the value at a specific percentile (e.g., 95, 99). | `\| STATS p99 = PERCENTILE(latency, 99) BY api` |
| **`WEIGHTED_AVG`** | Computes an average weighted by a second numeric column. | `\| STATS w_avg = WEIGHTED_AVG(score, weight)` |
| **`TOP`** | Returns the top `N` values per group sorted `"ASC"` or `"DESC"`. | `\| STATS top3 = TOP(latency, 3, "DESC") BY host` |
| **`VALUES`** | Collects all unique values of a field into a multivalue array. | `\| STATS all_codes = VALUES(status) BY client.ip` |
| **`EARLIEST` / `LATEST`** | Returns the value associated with the earliest or latest `@timestamp`. | `\| STATS last_status = LATEST(status) BY host` |
| **`BUCKET`** | Groups timestamps or numbers into fixed intervals in `BY`. | `\| STATS c = COUNT(*) BY win = BUCKET(@timestamp, 5 minutes)` |
| **`CATEGORIZE`** | Automatically groups similar unstructured log messages into patterns. | `\| STATS c = COUNT(*) BY cat = CATEGORIZE(message)` |

---

## 5. String Functions
*Used in `EVAL` and `WHERE` to inspect, clean, slice, and decode text.*

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`CONCAT`** | Joins multiple strings together. | `\| EVAL full = CONCAT(first_name, " ", last_name)` |
| **`LENGTH`** | Returns the number of characters in a string. | `\| WHERE LENGTH(dns.question) > 50` |
| **`BYTE_LENGTH` / `BIT_LENGTH`** | Returns the byte or bit length of a string. | `\| EVAL sz = BYTE_LENGTH(payload)` |
| **`STARTS_WITH` / `ENDS_WITH`** | Checks if a string begins or ends with a substring (returns boolean). | `\| WHERE ENDS_WITH(file.name, ".exe")` |
| **`LEFT` / `RIGHT`** | Extracts `N` characters from the start or end of a string. | `\| EVAL prefix = LEFT(hash, 8)` |
| **`SUBSTRING`** | Extracts text starting at a 1-based index for a given length. | `\| EVAL code = SUBSTRING(id, 1, 4)` |
| **`LOCATE`** | Returns the 1-based position of a substring (`0` if not found). | `\| EVAL pos = LOCATE(url, "?", 1)` |
| **`SPLIT`** | Splits a string by a delimiter into a multivalue array. | `\| EVAL parts = SPLIT(domain, ".")` |
| **`REPLACE`** | Replaces regex matches with a replacement string. | `\| EVAL clean = REPLACE(cmd, "\\s+", " ")` |
| **`TRIM` / `LTRIM` / `RTRIM`** | Removes whitespace from both sides, left, or right. | `\| EVAL clean = TRIM(username)` |
| **`TO_LOWER` / `TO_UPPER`** | Converts text to lowercase or uppercase. | `\| EVAL norm = TO_LOWER(email)` |
| **`REPEAT`** | Repeats a string `N` times. | `\| EVAL sep = REPEAT("-", 10)` |
| **`TO_BASE64` / `FROM_BASE64`** | Encodes to or decodes from Base64 text. | `\| EVAL decoded = FROM_BASE64(b64_str)` |

---

## 6. Date-Time Functions

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`NOW`** | Returns the current UTC timestamp at query execution. | `\| WHERE @timestamp >= NOW() - 1 hour` |
| **`DATE_DIFF`** | Calculates elapsed time between two dates in a given unit. | `\| EVAL mins = DATE_DIFF("minutes", start_ts, end_ts)` |
| **`DATE_EXTRACT`** | Extracts a numeric part (`"hour_of_day"`, `"day_of_week"`, `"year"`). | `\| EVAL hr = DATE_EXTRACT("hour_of_day", @timestamp)` |
| **`DATE_FORMAT`** | Formats a date into a custom string pattern. | `\| EVAL day = DATE_FORMAT("yyyy-MM-dd", @timestamp)` |
| **`DATE_PARSE`** | Parses a text string into a `date` using a pattern. | `\| EVAL ts = DATE_PARSE("yyyy/MM/dd HH:mm", raw_ts)` |
| **`DATE_TRUNC`** | Truncates a date down to an interval boundary. | `\| EVAL hr_bin = DATE_TRUNC(1 hour, @timestamp)` |

---

## 7. Conditional & Math Functions

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`CASE`** | Evaluates if-else conditions sequentially and returns the first match. | `\| EVAL tier = CASE(ms < 100, "fast", ms < 500, "ok", "slow")` |
| **`COALESCE`** | Returns the first non-null value from a list of expressions. | `\| EVAL actor = COALESCE(user.name, client.ip, "unknown")` |
| **`GREATEST` / `LEAST`** | Returns the largest or smallest value across multiple arguments. | `\| EVAL peak = GREATEST(cpu_a, cpu_b, cpu_c)` |
| **`CLAMP`** | Constrains a number between a `min` and `max` bound. | `\| EVAL bounded = CLAMP(score, 0, 100)` |
| **`ROUND` / `CEIL` / `FLOOR`** | Rounds to `N` decimals, rounds up, or rounds down. | `\| EVAL pct = ROUND(ratio * 100, 2)` |
| **`ABS` / `POW` / `SQRT`** | Absolute value, exponentiation, and square root. | `\| EVAL dist = SQRT(POW(x, 2) + POW(y, 2))` |
| **`LOG` / `LOG10` / `EXP`** | Natural/base logarithm, base-10 log, and exponential ($e^x$). | `\| EVAL mag = LOG10(bytes)` |

---

## 8. Multivalue (`MV_*`) Functions
*ES|QL fields can hold arrays of values. Use `MV_*` functions to manipulate arrays without exploding rows.*

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`MV_COUNT`** | Counts the number of elements in a multivalue field. | `\| EVAL tag_count = MV_COUNT(tags)` |
| **`MV_CONCAT`** | Joins array elements into a single string with a delimiter. | `\| EVAL tag_str = MV_CONCAT(tags, ", ")` |
| **`MV_FIRST` / `MV_LAST`** | Extracts the first or last element from an array. | `\| EVAL first_ip = MV_FIRST(dns.resolved_ip)` |
| **`MV_SLICE`** | Extracts a sub-range of elements by start/end index. | `\| EVAL top2 = MV_SLICE(items, 0, 2)` |
| **`MV_SORT` / `MV_DEDUPE`** | Sorts elements (`"ASC"`/`"DESC"`) or removes duplicates. | `\| EVAL uniq = MV_DEDUPE(MV_SORT(roles))` |
| **`MV_MIN` / `MV_MAX` / `MV_AVG` / `MV_SUM` / `MV_MEDIAN`** | Computes numeric statistics across values inside a single row's array. | `\| EVAL max_val = MV_MAX(measurements)` |
| **`MV_APPEND` / `MV_UNION` / `MV_INTERSECT` / `MV_EXCEPT`** | Combines or performs set operations on two arrays. | `\| EVAL common = MV_INTERSECT(list_a, list_b)` |
| **`MV_ZIP`** | Pairs elements from two arrays element-by-element with a delimiter. | `\| EVAL pairs = MV_ZIP(keys, vals, "=")` |

---

## 9. Network, Spatial & Type Conversion Functions
*Use conversion functions (or the shorthand `::type` cast operator) to convert extracted strings into typed data.*

| Function | Usage | Example |
| :--- | :--- | :--- |
| **`CIDR_MATCH`** | Checks whether an `ip` field falls within one or more CIDR subnets. | `\| WHERE CIDR_MATCH(client.ip, "10.0.0.0/8", "192.168.0.0/16")` |
| **`TO_IP`** | Casts a string to an `ip` type. | `\| EVAL ip = TO_IP(raw_ip)` *(or `raw_ip::ip`)* |
| **`TO_INTEGER` / `TO_LONG` / `TO_DOUBLE`** | Casts strings or numbers to 32-bit int, 64-bit long, or float. | `\| EVAL port = TO_INTEGER(port_str)` |
| **`TO_STRING` / `TO_BOOLEAN`** | Casts values to text (`keyword`) or boolean. | `\| EVAL flag = TO_BOOLEAN(is_active)` |
| **`TO_DATETIME`** | Casts an ISO-8601 string or epoch millis to a `date`. | `\| EVAL dt = TO_DATETIME("2026-11-26T09:00:00Z")` |
| **`TO_VERSION`** | Casts a string to semantic version for accurate `<` / `>` comparisons. | `\| WHERE TO_VERSION(app.ver) < TO_VERSION("9.0.0")` |
| **`TO_GEOPOINT` / `TO_GEOSHAPE`** | Casts WKT or lat/lon strings to spatial types for `ST_DISTANCE` / `ST_INTERSECTS`. | `\| EVAL pt = TO_GEOPOINT("34.02, -6.84")` |

---

## 10. Operators

| Category | Operators | Example |
| :--- | :--- | :--- |
| **Comparison** | `==`, `!=`, `<`, `<=`, `>`, `>=` | `WHERE status >= 400 AND status != 404` |
| **Logical** | `AND`, `OR`, `NOT` | `WHERE NOT (env == "dev" OR env == "test")` |
| **Null & Membership** | `IS NULL`, `IS NOT NULL`, `IN (...)` | `WHERE user IS NOT NULL AND role IN ("admin", "ops")` |
| **Pattern Matching** | `LIKE` (wildcards `*`, `?`), `RLIKE` (regex) | `WHERE path LIKE "/api/*" AND ua RLIKE ".*(curl\|wget).*"` |
| **Search & Casting** | `:` (match), `::` (type cast) | `WHERE message : "timeout" AND code::integer == 504` |
| **Arithmetic** | `+`, `-`, `*`, `/`, `%` | `EVAL error_rate = (errors * 100.0) / total` |

---

## 11. End-to-End ES|QL Query Recipes

### Log Analysis Recipes

**1. Parse Raw Access Logs & Rank Top Error Endpoints (`DISSECT` + `EVAL` + `STATS`)**
Extracts structured fields from an unparsed `message` string, converts the HTTP status code to an integer, and ranks the noisiest failing endpoints.
```esql
FROM logs-nginx-*
| WHERE @timestamp >= NOW() - 1 hour
| DISSECT message "%{client_ip} - - [%{raw_ts}] \"%{method} %{path} %{proto}\" %{status_str} %{bytes_str}"
| EVAL status = TO_INTEGER(status_str), bytes = TO_LONG(bytes_str)
| WHERE status >= 400
| STATS error_count = COUNT(*), unique_clients = COUNT_DISTINCT(client_ip), total_kb = ROUND(SUM(bytes) / 1024.0, 2) BY method, path, status
| SORT error_count DESC
| LIMIT 20
```

**2. Auto-Group Unstructured Error Logs (`CATEGORIZE`)**
Automatically clusters messy, variable error messages into recurring log templates so you can see the root cause without writing regex.
```esql
FROM logs-app-*
| WHERE @timestamp >= NOW() - 24 hours AND log.level == "ERROR"
| STATS occurrences = COUNT(*), affected_hosts = COUNT_DISTINCT(host.name) BY pattern = CATEGORIZE(message)
| SORT occurrences DESC
| LIMIT 15
```

**3. Track 5xx Error Rate Over Time (`BUCKET` + `CASE`)**
Calculates total traffic, failed requests, and the percentage error rate in 5-minute intervals to pinpoint when an outage began.
```esql
FROM logs-http-*
| WHERE @timestamp >= NOW() - 2 hours
| EVAL is_5xx = CASE(http.response.status_code >= 500, 1, 0)
| STATS total_reqs = COUNT(*), failed_reqs = SUM(is_5xx) BY window = BUCKET(@timestamp, 5 minutes)
| EVAL error_rate_pct = ROUND((failed_reqs * 100.0) / total_reqs, 2)
| SORT window ASC
```

---


---

### Security & Threat Hunting Recipes

**1. Brute-Force & Password Spraying Detection (`STATS` + `COUNT_DISTINCT`)**
Catches both single-account brute force (high failure volume) and low-and-slow password spraying (one IP hitting many distinct usernames).
```esql
FROM logs-*
| WHERE @timestamp >= NOW() - 24 hours AND event.category == "authentication" AND event.outcome == "failure"
| STATS
    failed_attempts = COUNT(*),
    targeted_users = COUNT_DISTINCT(user.name),
    sample_users = VALUES(user.name)
  BY source.ip
| WHERE failed_attempts >= 20 OR targeted_users >= 5
| SORT targeted_users DESC, failed_attempts DESC
```

**2. Vertical Port Scan & Horizontal Host Sweep Detection (`BUCKET` + `COUNT_DISTINCT`)**
Flags any source IP touching 15+ distinct ports or 15+ distinct destination hosts within a 5-minute window.
```esql
FROM logs-network-*
| WHERE @timestamp >= NOW() - 6 hours AND source.ip IS NOT NULL AND destination.port IS NOT NULL
| STATS
    unique_ports = COUNT_DISTINCT(destination.port),
    unique_targets = COUNT_DISTINCT(destination.ip),
    total_conns = COUNT(*)
  BY source.ip, window = BUCKET(@timestamp, 5 minutes)
| WHERE unique_ports >= 15 OR unique_targets >= 15
| SORT unique_ports DESC, unique_targets DESC
```

**3. Suspicious Outbound Data Exfiltration (`CIDR_MATCH` + `SUM`)**
Filters out internal RFC1918 destinations and surfaces hosts transferring large volumes of data to external IPs.
```esql
FROM logs-network-*
| WHERE @timestamp >= NOW() - 24 hours
  AND destination.ip IS NOT NULL
  AND network.bytes IS NOT NULL
  AND NOT CIDR_MATCH(destination.ip, "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16")
| STATS
    total_mb = ROUND(SUM(network.bytes) / 1048576.0, 2),
    connections = COUNT(*),
    dest_ports = VALUES(destination.port)
  BY source.ip, destination.ip
| WHERE total_mb > 100
| SORT total_mb DESC
```

**4. DNS Tunneling & High-Volume Subdomain Enumeration (`LENGTH` + `STATS`)**
Identifies internal hosts querying unusually long hostnames or generating high unique-subdomain counts against a single root domain.
```esql
FROM logs-dns-*
| WHERE @timestamp >= NOW() - 6 hours AND dns.question.name IS NOT NULL
| EVAL query_len = LENGTH(dns.question.name)
| STATS
    total_queries = COUNT(*),
    unique_subdomains = COUNT_DISTINCT(dns.question.name),
    max_len = MAX(query_len)
  BY source.ip, dns.question.registered_domain
| WHERE max_len > 50 OR unique_subdomains > 100
| SORT unique_subdomains DESC, max_len DESC
```

**5. Rare LOLBin & Obfuscated Process Execution (`RLIKE` + `SORT ASC`)**
Hunts for encoded PowerShell, shell downloaders, or obfuscated command lines and sorts `ASC` so rare, one-off attacker commands float to the top.
```esql
FROM logs-endpoint-*
| WHERE @timestamp >= NOW() - 24 hours
  AND event.category == "process"
  AND (
    process.name IN ("powershell.exe", "pwsh.exe", "mshta.exe", "certutil.exe", "rundll32.exe", "curl", "wget")
    OR process.command_line RLIKE ".*(-enc|-EncodedCommand|FromBase64String|IEX|base64 -d|/dev/tcp/).*"
  )
| STATS exec_count = COUNT(*), hosts = VALUES(host.name) BY process.parent.name, process.name, process.command_line
| SORT exec_count ASC
| LIMIT 50
```