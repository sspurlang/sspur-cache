# cache

Caching for SSPUR services, in three parts:

- an immutable LRU cache value, `Cache[K, V]`, with per-entry time to live, an optional cost
  bound, statistics, eviction callbacks (the evicted entries are returned) and `get_or_compute`,
  for one thread of work: a batch job, a worker loop, a single request;
- `Shared`, a cache in SQLite that the concurrent handlers of `http.serve` and several processes
  share, with time to live, LRU eviction at a bound, statistics, and `fetch`, a get-or-compute with
  single-flight: of the callers that miss one key at the same time, one computes and the others
  wait for its result, with leases so a crashed computer doesn't block anyone;
- HTTP caching after RFC 9110 and RFC 9111: entity tags (strong and weak, computed from bodies),
  HTTP dates in all three formats, conditional requests evaluated in the RFC's order (304 Not
  Modified and 412 Precondition Failed with the right fields), Cache-Control building and parsing,
  Vary, Age and freshness, and `conditional_get`, a handler wrapper.

## Install

```console
$ sspur add cache
```

## Example

```sspur
use cache.{Cache}

fn square(n: Int) -> Int
= n * n

fn page(req: Request) -> Response
= cache.with_cache_control(http.html(200, "<h1>Docs</h1>"), cache.revalidate())

fn main() -> Unit ! log, db, fail[DbErr], time, rand, div
= do
  var c: Cache[Int, Int] = cache.lru(2)
  for n in [3, 4, 3, 5, 4]
    (v, next) = cache.get_or_compute(c, n, 0, square)
    c := next
    log("{n}^2 = {v}")
  log("keys {cache.keys(c)}, {cache.stats(c)}")
  s = cache.shared(db.open(":memory:"), "pages", 1000)
  a = cache.fetch(s, "report", 60000, cache.flight(), () => "expensive report")
  b = cache.fetch(s, "report", 60000, cache.flight(), () => "computed again")
  log("{a} / {b} / {cache.shared_stats(s)}")
  req = Request{method: "GET", url: "/docs", headers: [], body: "", timeout_ms: 0, retries: 0, redirects: 0}
  first = cache.conditional_get(false, page)(req)
  tag = first.header("etag").or("")
  again = cache.conditional_get(false, page)(req with headers := [("if-none-match", tag)])
  log("{first.status} {tag} then {again.status} with {again.body.len} bytes")
  log(cache.format_http_date(time_ms(784111777000)))
```

```console
$ sspur run main.ssp
3^2 = 9
4^2 = 16
3^2 = 9
5^2 = 25
4^2 = 16
keys [4, 5], Stats{hits: 1, misses: 4, puts: 4, evictions: 2, expirations: 0}
expensive report / expensive report / Stats{hits: 1, misses: 1, puts: 1, evictions: 0, expirations: 0}
200 "nSW2vlLb9kmpJq8MMx2dmA" then 304 with 0 bytes
Sun, 06 Nov 1994 08:49:37 GMT
```

In a server, share the `Shared` value and build wrappers inside the handler (handlers may not
share function values):

```sspur
http.serve_with(http.server(8080), req => catch route(store, req)
  | _ => http.text(500, "error"))
```

where `route` calls `cache.fetch(store, ...)` and `cache.conditional_get(true, page)(req)`.
`tests/it/main.ssp` is such a server; the integration tests start it in both tiers and check the
304s, the cached values and single-flight across 8 concurrent clients.

## API

### Immutable cache

- `Config`, `config(capacity)`: entry bound, `max_cost`, default `ttl_ms`, `refresh_on_get`.
- `Cache[K, V]`, `Entry[V]`, `new(cfg)`, `lru(capacity)`, `ttl_lru(capacity, ttl_ms)`.
- `get(c, k, now_ms)`: the value and the new cache (recency, statistics, expiry).
- `peek`, `has`: read without touching anything.
- `put`, `put_ttl`, `insert(c, k, v, now_ms, ttl, cost)` (returns the entries that left).
- `remove`, `take`, `clear`, `touch`, `resize(c, capacity, max_cost)`, `purge_expired`.
- `keys` (most recent first), `items`, `lru_key`, `mru_key`, `ttl_left`, `size`, `total_cost`.
- `get_or_compute(c, k, now_ms, f)`.
- `Stats`, `stats(c)`, `hit_rate(s)`.

Times are passed in (`now_ms`), so the cache is deterministic and needs no effects. Expired entries
are dropped when read, when room is needed (before any live entry is evicted), or by
`purge_expired`. Operations are O(log n).

### Shared cache (SQLite)

- `Shared`, `shared(db, table, capacity)`, `valid_table_name(s)`.
- `shared_get`, `shared_peek`, `shared_put(s, key, value, now_ms, ttl_ms)`, `shared_delete`,
  `shared_clear`, `shared_purge`, `shared_count`, `shared_stats`.
- `shared_get_as(s, key, now_ms, decode)`, `shared_put_as(s, key, value, now_ms, ttl_ms)` for JSON-encodable values.
- `Flight`, `flight()`: lease length, how long waiters wait, how often they look.
- `fetch(s, key, ttl_ms, f, compute)`, `fetch_as(..., compute, decode)`: get or compute with single-flight.
- `shared_computing(s, key, now_ms)`: whether someone holds the lease.

Values are text: a library can't decode JSON into a type parameter (`json.decode[T]` needs a
concrete type), so typed helpers take a decoder from the caller, such as
`x => json.decode[User](x).map(some).or(none)`. Every operation is a transaction; each read records
its use for LRU, so reads are small writes.

### HTTP caching

- `Etag`, `strong_etag`, `weak_etag`, `etag_str`, `parse_etag`, `etag_text_ok`.
- `etag_for(body)` (strong, 128 bits of SHA-256), `weak_etag_for(body)`.
- `strong_match`, `weak_match` (RFC 9110 8.8.3.2); `Condition` (`AnyTag`, `Tags`), `parse_condition`.
- `format_http_date(t)`, `parse_http_date(s, now)` (IMF-fixdate, RFC 850, asctime).
- `Validators`, `validators_of(r, now)`, `Outcome` (`Proceed`, `NotModified`, `PreconditionFailed`).
- `evaluate(req, validators, exists, now)`: RFC 9110 13.2.2, for GET and for state-changing methods.
- `not_modified(r)`, `precondition_failed(v)`, `conditional(req, r, now)`, `with_etag(r, weak)`.
- `conditional_get(weak, next)`: the wrapper.
- `CacheControl`, `cache_control()`, `no_store()`, `revalidate()`, `public_for`, `private_for`, `immutable_for`.
- `render_cache_control`, `parse_cache_control`, `cache_control_header`, `cache_control_of(r)`, `with_cache_control`, `parse_delta`.
- `vary_of`, `add_vary`, `set_header`.
- `freshness_lifetime(cc, shared, date, expires)`, `current_age(...)` (RFC 9111 4.2.3), `is_fresh`, `age_header`.

## Notes

- Use weak ETags (`conditional_get(true, ...)`) when the server's `gzip` field is on: the bytes sent
  then differ from the body, and a strong tag promises byte equality.
- `conditional_get` runs the handler first and then decides, which saves bandwidth, not work, and is
  only right for GET and HEAD. For PUT, PATCH and DELETE call `evaluate` before changing anything.
- A malformed If-Match fails (412); malformed or repeated date conditions are ignored, as are
  If-Modified-Since dates in the future (RFC 9110 13.1).
- The shared cache's single-flight works across processes sharing the database file. A waiter waits
  at most `wait_ms` and then computes the value itself, so a stuck computer slows callers down but
  never blocks them.

## Out of scope

- A caching proxy (storing responses keyed by request, Vary matching, revalidation on behalf of
  clients); this package gives the building blocks.
- Range requests and If-Range.
- Distributed caches other than SQLite (Redis, memcached): sspur has no TCP client.
- Generic typed values in the shared cache without a caller-supplied decoder.

## License

MIT, see LICENSE.
