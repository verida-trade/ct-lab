# FAQ — Known Limitations

## What's NOT possible

| Limitation | Why |
|---|---|
| Edit product source code | Repo is educational, not source code |
| Access microstructure data without premium | Collectors are premium-gated |
| Persist collection tasks after closing app | Tasks live in server stdio memory |
| More than 100 series in cache (premium) | Design limit — LRU eviction |
| More than 1 series in cache (free) | Free tier limit |
| Exact price prediction | By axiom: the future is unknowable |

## `uv` environment limits

- Not a security sandbox (doesn't block FS/network from malicious script)
- Strong isolation (microsandbox) is a future direction
- First run may download Python/libs (network)
