# License Errors

> Common issues with premium license.

## "Tool not visible in list"

**Cause:** Invalid, expired, or unconfigured license.

**Solution:**

1. Check with `ct://license/info`
2. If no license: use `comprar_premium`
3. If expired: renew via `comprar_premium`

## "Fingerprint mismatch"

The machine fingerprint doesn't match the license.

**Cause:** License is hardware-bound. Hardware change, cloned VM, or container can cause mismatch.

**Solution:** Reactivate the license on the new hardware.

## Series cap (1 free, 100 premium)

```
Error: series limit reached (1/1)
```

**Solution:** Remove unused series (`remover_serie`) or upgrade to premium (100 series).

---

> Back to: [README](./README.en.md)
