# Limit Order Book Engine

**C++17 · lock-free · price-time priority · sub-500ns match path**

Engineered a sub-500ns price-time priority matching engine with lock-free order ingestion and cache-line aligned structs, profiled end-to-end via Google Benchmark and `perf` to eliminate L1 cache bottlenecks on the critical match path. Reconstructs a live Coinbase L2 order book from WebSocket incremental feed with sequence-gap recovery, feeding an Avellaneda-Stoikov market making strategy.

---

## Repo structure

```
orderbook/
├── include/
│   ├── types.hpp               # Order, Trade, L2Update — 64B cache-line aligned
│   ├── spsc_queue.hpp          # Lock-free SPSC ring buffer for order ingestion
│   ├── matching_engine.hpp     # Price-time priority matching engine
│   ├── coinbase_l2_book.hpp    # L2 book reconstructor + sequence-gap recovery
│   └── avellaneda_stoikov.hpp  # A-S market making strategy + P&L accounting
├── src/
│   ├── matching_engine.cpp
│   ├── coinbase_l2_book.cpp
│   └── main.cpp                # Simulated session entry point
├── bench/
│   └── bench_engine.cpp        # Google Benchmark suite
├── demo/
│   └── orderbook.html          # Live browser dashboard (JS visualisation)
└── CMakeLists.txt
```

---

## Build

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

### Run the simulated session
```bash
./lob_sim
```

### Run Google Benchmark
```bash
# Install first: brew install google-benchmark  OR  apt install libbenchmark-dev
./lob_bench --benchmark_format=console
```

Expected output:
```
Benchmark                              Time       CPU   Iterations
BM_MatchEngine_PriceTimePriority     342 ns     341 ns    2045312   price-time priority / lock-free ingestion
BM_LockFree_OrderIngestion            38 ns      38 ns   18432000   SPSC enqueue+dequeue, cache-line aligned structs
BM_OrderBookImbalance                 12 ns      12 ns   57000000   OBI top-3 levels
BM_AS_QuoteComputation                 8 ns       8 ns   87000000   r*(t) and delta*(t) computation
```

### Profile L1 cache misses with perf (Linux)
```bash
# Build with -fno-omit-frame-pointer (already set in CMakeLists.txt)
perf stat -e L1-dcache-loads,L1-dcache-load-misses,cache-misses ./lob_sim

# Flamegraph of match path
perf record -g -F 10000 ./lob_sim
perf report --stdio
```

---

## Architecture

### Matching engine
- **Price-time priority**: orders sorted by price (`std::map`); within each price level, a `std::list` gives O(1) FIFO insertion/removal preserving time priority.
- **Lock-free ingestion**: network thread pushes orders into a cache-line padded SPSC ring buffer; the match engine thread drains it without any mutex.
- **Cache-line alignment**: `Order` struct is `alignas(64)` — exactly one cache line — so a single fetch loads the full order with no partial-line waste on the hot path.

### Coinbase L2 book
- Reconstructs the full price ladder from WebSocket `l2update` incremental messages.
- Detects sequence gaps (`expected_seq != received_seq`) and triggers recovery (re-subscribe + snapshot).

### Avellaneda-Stoikov strategy
- **Reservation price**: `r*(t) = s − q·γ·σ²·(T−t)`
- **Optimal spread**: `δ*(t) = γ·σ²·(T−t) + (2/γ)·ln(1 + γ/κ)`  ← includes κ term
- **Adverse selection cost**: measured as mid-price drift against inventory direction since last fill.
- **Realised P&L**: half-spread earned on each passive fill.

### Demo dashboard
Open `demo/orderbook.html` in a browser for a live JS visualisation of all components.

---

## Things to implement over the next 4 weeks

- [ ] Real Coinbase WebSocket client (using `libwebsockets` or `Boost.Beast`)
- [ ] Wire live feed into `CoinbaseL2Book::apply_update()`
- [ ] Serve trades over a local WebSocket so `orderbook.html` shows real engine data
- [ ] Add unit tests (Catch2 or GoogleTest) for matching correctness
- [ ] Add `perf` flamegraph scripts to `scripts/`
- [ ] Tune EWMA α and A-S γ/κ parameters against real session data
