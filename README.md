This repository contains the results and analysis after benchmarking the rustls library in different
configurations. It also includes benchmarks of OpenSSL, which are interesting because rustls is
aiming to be an OpenSSL replacement. The benchmarks are based on previous work by
[@ctz](https://github.com/ctz), who originally benchmarked rustls against OpenSSL in 2019 and
published the results [online](https://jbp.io/2019/07/01/rustls-vs-openssl-performance.html).

# 1. Scenarios

The benchmarks focus on the following aspects of performance:

1. Bulk data transfer throughput in MB/s;
2. Handshake throughput (full, session id, tickets) in handshakes per second.

For _bulk transfers_, we benchmark the following cipher suites: `AES128-GCM_SHA256`,
`AES256-GCM_SHA384` and `CHACHA20_POLY1305`. We test them all using TLS 1.2, and additionally test
`AES256-GCM_SHA384` using TLS 1.3 as a sanity check (performance should be a [tiny bit
better](https://discord.com/channels/976380008299917365/1015156984007381033/1182311617904517270)
than TLS 1.2).

For _handshakes_, we benchmark ECDHE + RSA using a 2048-bit key. We use both TLS 1.2 and TLS 1.3.

As an example, see below a comparison between OpenSSL and the aws-lc rustls configuration, showing
the speedup / slowdown factor in the rightmost column:

|Scenario|OpenSSL (3.2.0)|Rustls (0.22.0, aws-lc)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|6503.58|6712.29|1.03x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|7234.11|6017.82|0.83x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|6162.57|6193.18|1.00x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|6609.27|5669.77|0.86x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|2998.06|1750.11|0.58x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|3107.17|1731.27|0.56x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|6041.82|6256.94|1.04x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_received|6406.44|5945.02|0.93x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|2581.16|4991.73|1.93x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|2120.95|1485.95|0.70x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|21514.4|49893.99|2.32x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|23183.8|41846.34|1.80x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|21491.3|47419.84|2.21x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|20190.1|36518.87|1.81x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|2273.57|4683.46|2.06x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|1874.1|1362.79|0.73x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|3415.35|11272.98|3.30x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|3060.17|7434.41|2.43x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|5182.51|11244.79|2.17x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|4718.39|7347.91|1.56x|

# 2. Methodology

### System configuration

We ran the benchmarks on a bare-metal server with the following characteristics:

- OS: Debian 12 (Bookworm).
- CPU: Xeon E-2386G (supporting AVX-512).
- Memory: 32GB.
- Extra configuration: hyper-threading disabled, dynamic frequency scaling disabled, cpu scaling
  governor set to performance for all cores.

### OpenSSL

The most recent version of OpenSSL at the time of this research was 3.2.0. We benchmarked it using
ctz's
[openssl-bench](https://github.com/ctz/openssl-bench/tree/7bc3277b062c598463d60e6d821198ec5c7a4763)
repository, which expects a built OpenSSL tree in `../openssl/` and the rustls repository in
`../rustls`. We ran the benchmarks using `BENCH_MULTIPLIER=8 setarch -R make measure` (setarch is
used here and elsewhere to disable ASLR and thereby reduce noise).

### rustls

The most recent version of rustls at the time of this research was 0.22.0. We benchmarked it using
`bench.rs`, included in the
[repository](https://github.com/rustls/rustls/blob/8b054591f0a64cfec6dee7d4e0f86302bacba7b2/rustls/examples/internal/bench.rs).
We ran it using `BENCH_MULTIPLIER=8 setarch -R make -f admin/bench-measure.mk measure` (the makefile
assumes that the `bench.rs` binary has already been built, see below for details).

When building rustls, we tested a matrix of configurations involving 3 variables:

- Cryptography backend: [ring 0.17.7](https://github.com/briansmith/ring/) vs [aws-lc-rs
  1.5.2](https://github.com/aws/aws-lc-rs).
- C/C++ toolchain: GCC 12.2.0 vs Clang 14.0.6.
- Allocator: glibc malloc vs jemalloc.

The **cryptography backend** was specified through cargo features:

- Build with ring: `cargo build -p rustls --release --example bench`.
- Build with aws-lc-rs: `cargo build -p rustls --release --example bench --no-default-features --features
  tls12,aws_lc_rs`.

The **C/C++ toolchain** was specified through Debian's `update-alternatives` utility. We manually
checked the build artifacts of ring and aws-lc-rs to ensure they were indeed built with the right
toolchain (using `objdump -s --section .comment target/release/deps/artifact.rlib`).

The **allocator** was specified by manually modifying `bench.rs` to use `Jemallocator` through the
`#[global_allocator]` attribute (see jemallocator's
[repository](https://github.com/tikv/jemallocator/tree/417d62bc976aec34ec729bd391cb66f837bd0eb8) for
details).

### Result collection and reporting

For each OpenSSL and rustls configuration, we ran the benchmarks 30 times and stored their stdout
output in `data`. We experimentally settled on the median as the value used for reporting due to
its stability (taking the maximum or the average was more prone to noise).

Check out our Python notebook in `notebook.ipynb`, which has code to generate comparison tables in
markdown format (and could be modified to generate CSV or something else that suits your needs).
Hint: use `poetry install --no-root` to generate a virtual environment and Visual Studio Code to
play with the notebook.

# 3. Conclusions

Based on our own benchmarks:

- For rustls, jemalloc offers significantly higher throughput over glibc's malloc in the sending
  part of the bulk transfer scenario (35% to 136% higher, depending on the crypto backend and cipher
  suite). The cause seems to be a [massive difference in page
  faults](https://github.com/rustls/rustls/issues/1626#issuecomment-1842733970). See
  [ring](#jemallocs-effect-on-rustls--ring-bulk-transfers) and
  [aws-lc](#jemallocs-effect-on-rustls--aws-lc-bulk-transfers) tables for details.
- For rustls + ring, Clang offers significantly better handshake throughput over GCC (up to 15% in
  the full handshake scenarios, and up to 27% in the resumed handshake scenarios). See
  [table](#clangs-effect-on-rustls--ring-handshakes) for details.
- There is no significant performance difference for rustls + aws-lc between GCC and Clang.
- There is no significant performance difference for OpenSSL between GCC and Clang, except in the
  `CHACHA20_POLY1305` bulk transfer, which takes a 45% hit when Clang is used. See
  [table](#clangs-effect-on-openssl-bulk-transfers) for details.
- Overall, rustls + aws-lc beats rustls + ring by a wide margin in bulk transfers (up to 67%), but
  it lags behind a bit in TLS 1.3 handshakes (up to 16%). See [table](#ring-vs-aws-lc) for details.
- Overall, OpenSSL beats rustls + aws-lc in bulk transfers and server-side full-handshakes.
  Conversely, rustls + aws-lc blows OpenSSL out of the water in all other handshake scenarios. See
  [table](#openssl-vs-rustls--aws-lc) for details.

Additional thoughts:

- `aws-lc` has its own benchmarking suite which reports that performance is on par with OpenSSL for
  `AES-128-GCM` and `AES-256-GCM`, suggesting that the current bottleneck is elsewhere (in the
  `aws-lc-rs` wrapper or in `rustls` itself). See `data/aws-lc-bench-results` for the raw output.
- According to the same `aws-lc` benchmarks, the `ChaCha20-Poly1305` is in `aws-lc` itself.

Open questions:

- Where does the performance difference in handshakes per second come from?

# 4. Appendix

### jemalloc's effect on rustls + ring bulk transfers

|Scenario|Rustls (0.22.0, ring_clang, malloc)|Rustls (0.22.0, ring_clang, jemalloc)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|2281.58|4150.53|1.82x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|4161.15|4104.5|0.99x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|2144.81|3700.62|1.73x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|3710.15|3660.96|0.99x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|1291.56|1739.36|1.35x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|1739.0|1732.48|1.00x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|2141.69|3738.28|1.75x|

### jemalloc's effect on rustls + aws-lc bulk transfers

|Scenario|Rustls (0.22.0, aws-lc_gcc, malloc)|Rustls (0.22.0, aws-lc_gcc, jemalloc)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|2847.44|6712.29|2.36x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|6135.09|6017.82|0.98x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|2788.35|6193.18|2.22x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|5776.34|5669.77|0.98x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|1287.94|1750.11|1.36x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|1738.97|1731.27|1.00x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|2791.74|6256.94|2.24x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_received|6093.48|5945.02|0.98x|

### clang's effect on rustls + ring handshakes

|Scenario|Rustls (0.22.0, ring_gcc, jemalloc)|Rustls (0.22.0, ring_clang, jemalloc)|Factor|
|-|-:|-:|-:|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|3671.06|4208.9|1.15x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|1427.56|1494.01|1.05x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|47940.59|47471.95|0.99x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|42929.45|41969.72|0.98x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|45656.71|45306.78|0.99x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|39121.89|38510.33|0.98x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|4060.86|4216.28|1.04x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|1373.62|1433.71|1.04x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|12008.75|12616.7|1.05x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|7067.54|8944.5|1.27x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|12031.75|12615.56|1.05x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|7036.61|8883.55|1.26x|

### clang's effect on OpenSSL bulk transfers

|Scenario|OpenSSL (3.2.0_gcc)|OpenSSL (3.2.0_clang)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|6503.58|6376.77|0.98x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|7234.11|7055.49|0.98x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|6162.57|6140.83|1.00x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|6609.27|6671.44|1.01x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|2998.06|1676.11|0.56x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|3107.17|1708.9|0.55x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|6041.82|6064.64|1.00x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_received|6406.44|6405.74|1.00x|

### ring vs aws-lc

|Scenario|Rustls (0.22.0, ring_clang, jemalloc)|Rustls (0.22.0, aws-lc_clang, jemalloc)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|4150.53|6573.1|1.58x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|4104.5|6036.7|1.47x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|3700.62|6171.02|1.67x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|3660.96|5680.81|1.55x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|1739.36|1740.76|1.00x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|1732.48|1731.35|1.00x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|3738.28|6225.5|1.67x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_received|3657.53|5987.83|1.64x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|4208.9|4959.08|1.18x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|1494.01|1481.99|0.99x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|47471.95|50580.45|1.07x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|41969.72|42275.27|1.01x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|45306.78|48077.08|1.06x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|38510.33|36683.26|0.95x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|4216.28|4680.92|1.11x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|1433.71|1363.29|0.95x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|12616.7|11522.16|0.91x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|8944.5|7529.76|0.84x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|12615.56|11519.54|0.91x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|8883.55|7482.64|0.84x|

### OpenSSL vs rustls + aws-lc

|Scenario|OpenSSL (3.2.0)|Rustls (0.22.0, aws-lc_gcc, jemalloc)|Factor|
|-|-:|-:|-:|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_sent|6503.58|6712.29|1.03x|
|bulk_1.2_ECDHE_RSA_AES128-GCM_SHA256_received|7234.11|6017.82|0.83x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_sent|6162.57|6193.18|1.00x|
|bulk_1.2_ECDHE_RSA_AES256-GCM_SHA384_received|6609.27|5669.77|0.86x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_sent|2998.06|1750.11|0.58x|
|bulk_1.2_ECDHE_RSA_CHACHA20_POLY1305_received|3107.17|1731.27|0.56x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_sent|6041.82|6256.94|1.04x|
|bulk_1.3_ECDHE_RSA_AES256-GCM_SHA384_received|6406.44|5945.02|0.93x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|2581.16|4991.73|1.93x|
|handshake-full_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|2120.95|1485.95|0.70x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|21514.4|49893.99|2.32x|
|handshake-session-id_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|23183.8|41846.34|1.80x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_client|21491.3|47419.84|2.21x|
|handshake-ticket_1.2_ECDHE_RSA_AES256-GCM_SHA384_server|20190.1|36518.87|1.81x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|2273.57|4683.46|2.06x|
|handshake-full_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|1874.1|1362.79|0.73x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|3415.35|11272.98|3.30x|
|handshake-session-id_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|3060.17|7434.41|2.43x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_client|5182.51|11244.79|2.17x|
|handshake-ticket_1.3_ECDHE_RSA_AES256-GCM_SHA384_server|4718.39|7347.91|1.56x|