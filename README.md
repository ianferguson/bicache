[![GoDoc](https://godoc.org/github.com/jamiealquiza/bicache?status.svg)](https://godoc.org/github.com/jamiealquiza/bicache)

# bicache
Bicache is a sharded hybrid MFU/MRU, TTL optional, general purpose cache for Go. It combines exact LRU ordering and MFU accounting per shard (read: accurate but possibly expensive).

Bicache's two tiers of cache are individually size configurable (in key count). A global lookup table is used to limit read ops to a max of one cache miss, even with two tiers of sharded cache levels. Bicache handles MRU to MFU promotions and overflow evictions at write time or on automatic interval as a background task.

Bicached is built for highly concurrent, read heavy workloads.

# API

See [GoDoc](https://godoc.org/github.com/jamiealquiza/bicache) for additional reference.

See code [Example](https://github.com/jamiealquiza/bicache#example) section at bottom.

### Set(string, interface{}) bool
```go
ok := c.Set("key", "value")
```

Sets `key` to `value` (if exists, updates). Set can be used to update an existing TTL'd key without affecting the TTL. A status bool is returned to signal whether or not the set was successful. A `false` is returned when Bicache is configured with `NoOverflow` enabled and the cache is full.

### SetTTL(string, interface{}, int32) bool
```go
ok := c.SetTTL("key", "value", 3600)
```

Sets `key` to `value` (if exists, updates) with a TTL expiration (in seconds). SetTTL can be used to add a TTL to an existing non-TTL'd key, or, updating an existing TTL. A status bool is returned to signal whether or not the set was successful. A `false` is returned when Bicache is configured with `NoOverflow` enabled and the cache is full.

### Get(string) interface{}
```go
value := c.Get("key")
```

Returns `value` for `key`. Increments the key score by 1. Get returns `nil` if the key doesn't exist.

### Del(string)
```go
c.Del("key")
```

Removes `key` from the cache.

### List(int) ListResults
```go
c.List(10)
```

Returns a \*bicache.ListResults that includes the top n keys by score, formatted as `key:state:score` (state: 0 = MRU cache, 1 = MFU cache).

```go
type ListResults []*KeyInfo

type KeyInfo struct {
    Key   string
    State uint8
    Score uint64
}
```

### FlushMRU() error, FlushMFU() error, FlushAll() error
```go
err := c.FlushMRU()
err := c.FlushMFU()
err := c.FlushAll()
```

Flush commands flush all keys from the respective cache. `FlushAll` is faster than combining `FlushMRU` and `FlushMFU`.

### Pause() error, Resume() error
```go
c.Pause()
c.Resume()
```

Pause and Resume allow auto evictions to be suspended and resumed, respectively. If eviction logging is enabled and evictions are paused, bicache will log accordingly.

### Close()
```go
c.Close()
```

Close should be called when a \*Bicache is done being used, before removing any references to it, to ensure any background tasks have returned and that it can be cleanly garbage collected.

### Stats() \*Stats
```go
stats := c.Stats()
```

Returns a \*bicache.Stats.

```go
type Stats struct {
    MFUSize   uint   // Number of acive MFU keys.
    MRUSize   uint   // Number of active MRU keys.
    MFUUsedP  uint   // MFU used in percent.
    MRUUsedP  uint   // MRU used in percent.
    Hits      uint64 // Cache hits.
    Misses    uint64 // Cache misses.
    Evictions uint64 // Cache evictions.
    Overflows uint64 // Failed sets on full caches.
}
```

Stats structs can be formatted as a json string:

```go
j, _ := json.Marshal(stats)
fmt.Prinln(string(j))
```
```
{"MFUSize":0,"MRUSize":3,"MFUUsedP":0,"MRUUsedP":4,"Hits":3,"Misses":0,"Evictions":0,"Overflows":0}
```

# Design

In a pure MRU cache, both fetching and setting a key moves it to the front of the list. When the list is full, keys are evicted from the tail when space for a new key is needed. Bicache isolates MRU thrashing by promoting the most frequently used keys to an MFU cache when the MRU cache is full. At MRU eviction time, Bicache gathers the highest score MRU keys and promotes only those that have scores exceeding keys in the MFU. Any remainder key count that must be evicted is accomplished with MFU to MRU demotion followed by MRU tail eviction.

![img_0836](https://cloud.githubusercontent.com/assets/4108044/26748074/cf5e7858-47b7-11e7-8063-b9e95bfa3fdc.PNG)

New keys are always set to the head of the MRU list; MFU keys are only ever populated through promotion from the MRU list.

Internally, bicache shards the two cache tiers into many sub-caches (sized through configuration in powers of 2). This is done for two primary reasons: 1) to reduce lock contention in highly concurrent workloads 2) minimize the maximum runtime of expensive maintenance tasks (e.g. many MRU to MFU promotions followed by many MRU evictions). Otherwise, shards are invisible from the perspective of the API.

![img_0849](https://user-images.githubusercontent.com/4108044/27234682-1dfcac78-527b-11e7-9d4f-5908ab1cbfef.PNG)
> *color key denotes shard lock exclusivity; blue represents a read lock, orange is a full rw lock*

Get, Set and Delete requests are routed to the appropriate cache shard with a hash-routing on the key name. Bicache's internal accounting, cache promotion, evictions and usage stats are all isolated per shard. Promotions and evictions are handled sequentially across shards in a dedicated background task at the configured `AutoEvict` interval (promotion/eviction timings are emitted if configured; these metrics represet the most performance influencing aspect of bicache). When calling the `Stat()` method on bicache, shard statistics (hits, misses, usage) are aggregated and returned.

# Installation
Tested with Go 1.7+.

- `go get -u github.com/jamiealquiza/bicache`
- Import package (or `go install github.com/jamiealquiza/bicache/...` for examples)

# Configuration

### Shard counts

Shards must be sized in powers of 2. Shards are relatively inexpensive to manage but should not be arbitrarily high. Shard sizing should be relative to desired cache sizes and workload; more key space and greater write concurrency/rates are better suited with more shards. Typical sizes might be 8 shards for simple testing and 1024 shards for production workloads that experience tens of thousands (or more) of cache lookups a second.

### Cache sizes

Bicache can be configured with arbitrary sizes for each cache, allowing a ratio of MFU to MRU for different usage patterns. While the example shows very low cache sizes, this is purely to demonstrate functionality when the MRU is overflowed. A real world configuration might be a 10,000 key MFU and 30,000 key MRU capacity.

The `Config.NoOverflow` setting specifies whether or not `Set` and `SetTTL` methods are allowed to add additional keys when the cache is full. If NoOverflow is enabled, a set will return `false` if the cache is full. Allowing overflow will allow caches to run over 100% utilization until a promovtion/eviction cycle is performed to evict overflow keys. No Overflow may be interesting for strict cache size controls with extremely high set volumes, where the caches could reach several times their capacity between eviction cycles.

The MFU can also be set to 0, causing Bicache to behave like a typical MRU/LRU cache.

Also take note that the actual cache capacity may vary slightly from what's configured, once incorporating the shard count setting. MFU and MRU sizes are divided over the number of configured shards, rounded up for even distribution. For example, settings the MRU capacity to 9 and the shard count to 6 would result in an actual MRU capacity of 12 (minimum of 2 MRU keys per shard to deliver the requested 9). In practice, this would go mostly unnoticed as most typical shard counts will be upwards of 1024 and cache sizes in the tens of thousands.

### Auto Eviction

TTL expirations, MRU to MFU promotions, and MRU overflow evictions only occur automatically if the `AutoEvict` configuration parameter is set. This is a background task that only runs if a non-zero parameter is set. If unset or explicitly configured to 0, TTL expirations never run and MRU promotions and evictions will be performed at each Set operation.

The Bicache `EvictLog` configuration specifies whether or not eviction timing logs are emitted:
<pre>
2017/02/22 11:01:47 [PromoteEvict] cumulative: 61.023µs | min: 52ns | max: 434ns
</pre>

This reports the total time spent on the previous eviction cycle across all shards, along with the min and max time experienced for any individual shard.

# Example

test.go:
```go
package main

import (
        "encoding/json"
        "fmt"
        "time"

        "github.com/jamiealquiza/bicache"
)

func main() {
        c, _ := bicache.New(&bicache.Config{
                MFUSize:    24,    // MFU capacity in keys
                MRUSize:    64,    // MRU capacity in keys
                ShardCount: 16,    // Shard count. Defaults to 512 if unset.
                AutoEvict:  30000, // Run TTL evictions + MRU->MFU promotions / evictions automatically every 30s.
                EvictLog:   true,  // Emit eviction timing logs.
                NoOverflow: true,  // Disallow Set ops when the MRU cache is full.
        })

        // Keys must be strings and values
        // can be essentially anything (value is an interface{}).
        // Key and value types can be mixed
        // in a single cache object.
        c.Set("name", "john")
        c.Set("1", 5535)
        c.Set("myKey", []byte("my value"))

        time.Sleep(time.Second)

        fmt.Println(c.Get("name"))
        fmt.Println(c.Get("1"))
        fmt.Println(c.Get("myKey"))

        stats := c.Stats()
        j, _ := json.Marshal(stats)
        fmt.Printf("\n%s\n", string(j))
}
```

Output:
```
% go run test.go
john
5535
[109 121 32 118 97 108 117 101]

{"MFUSize":0,"MRUSize":3,"MFUUsedP":0,"MRUUsedP":4,"Hits":3,"Misses":0,"Evictions":0,"Overflows":0}
```
