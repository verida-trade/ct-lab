# Adaptive Layer of the `grupo` Lib

> **Premium.** Risk corrects to market; opportunity waits at rest. The adaptive layer improves the distribution shape — it doesn't create EV.

The `grupo` lib supports adaptive channels that allow the policy to re-decide per bar, adjusting risk to market.

---

## `alvo_pos` — market risk adjustment

Absolute position target (signed), idempotent. Only with open position. Never inverts side (clamp to 0):

```rhai
let cfg = #{
    entradas: [...],
    saidas: [...],
    ciclos: 1.0,
    alvo_pos: 2.0,
};
```

- `alvo_pos: 0` closes the position immediately.
- `alvo_pos: 3.0` pyramid (increase from 1.0 to 3.0).
- `alvo_pos: 0.5` reduce (decrease from 1.0 to 0.5).
- Hysteresis (adjust in steps) is the policy's responsibility.

---

## `saidas_vivas` — updatable exit plan

```rhai
let cfg = #{
    entradas: [...],
    saidas: [...],
    ciclos: 1.0,
    saidas_vivas: true,
};
```

Current cfg EXITS replace the frozen ones every bar (move stop to breakeven, rescale takes/trailing with vol). Entries remain frozen at arming (fill tracking). Trailing state is preserved by index.

---

## Adaptive rules (in the manager)

| Rule | What it does | Effect |
|---|---|---|
| Breakeven | Move stop to entry when position favors | Cuts left tail |
| Vol-rescale | Adjust stops/takes with local volatility | Adapts to regime |
| Pyramid out of high vol | Add position when vol drops | Increases right capture |

> **Important:** Adaptation does NOT create EV (theorem) — it improves the **shape** of the distribution: left tail −50%, right capture +30-40%.

---

> Next: [Survival test](./08-teste-sobrevivencia.en.md)
