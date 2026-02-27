# Zerodha Options Bot — Architecture + 27 Feb Tick Post-Mortem

## Task A: How the bot works end-to-end

### 1) Architecture / modules
- **Block 1 (config/bootstrap):** initializes API/session, symbol/strike selection, lockout for paper mode (`_paper_mode_lockout`) and path constants (`TICK_LOG_PATH`, `TRADES_CSV`).
- **Block 2 (state + indicators):** maintains `global_state` (`CE`/`PE`) + `spot_state`, builds 1m/3m/15m candles, computes IMB, VWAP, ATR(7 points), RSI(3,15m), and publishes fields used by strategies.
- **Block 3 (strategies):** GL, GB, N19 strategy classes/handlers (`gl_on_tick`, `gb_on_tick`, `n19_on_tick`), order execution wrapper (`_place_or_revert`), and `log_trade()` writer for `trades.csv`.
- **Block 4 (signal bus):** emits strategy ENTRY/EXIT/PARTIAL events via `emit_signal` and keeps `LAST_SIGNALS` for dashboard.
- **Block 5 (runtime):** websocket `on_ticks` callback updates state, computes indicators, queues option ticks into `tick_q`; `engine_loop` pops from queue and calls GL/GB/N19 handlers.
- **Block 6 (dashboard):** HTML status panel reads `spot_state`, `global_state`, `LAST_SIGNALS`, and trade-state adapters.

### 2) Runtime loop and data flow
1. WS tick enters `on_ticks`.
2. Tick timestamp normalized (`exchange_timestamp` → IST), token mapped to SPOT/CE/PE.
3. For CE/PE: updates LTP/bid/ask/spread + `global_state[leg]`, attempts true-open lock (`open_locked`, `open_src`), runs indicator updater `_update_ind(...)`.
4. Same option tick is pushed to `tick_q` as `("OPT", ts, leg, prev_ltp, ltp, raw_tick)`.
5. `engine_loop` consumes queue and calls strategy handlers in order:
   - `gl_on_tick(ts, leg, prev_ltp, ltp)`
   - `gb_on_tick(ts, leg, prev_ltp, ltp)`
   - `n19_on_tick(ts, leg, ltp)`
6. On entry/exit, strategy wrappers call `_place_or_revert(...)` (paper-safe), then `log_trade(...)` appends to `trades.csv`.
7. Independently, tick rows are buffered (`tick_buffer`) and flushed by `_tick_writer_loop` into `ticks.csv`.

### 3) All trade blockers / gates even when chart “looks valid”
- **Strategy disabled / missing handler:** `_safe_call` skips or auto-disables after repeated exceptions.
- **Concurrent strategy block:** when `ALLOW_CONCURRENT_LAYERS=False`, one strategy can block another.
- **Per-strategy in-trade state:** `gl_strategy.trades[leg].in_trade`, `gb_strategy.trade.in_trade`, `n19_state.in_trade`.
- **Daily limits:** GL `day_state.trades_today < max_trades_per_day`; GB one trade/leg/day (`ce_traded`, `pe_traded`); N19 `trades_today < N19_MAX_TRADES_PER_DAY`.
- **Time windows / lunch windows:** GL/GB 09:15–14:30, GL lunch block 11:45–13:00, N19 09:20–14:20 + lunch block 11:30–13:00.
- **Re-entry rules:** GL `_leg_reentry_ok()` blocks same leg after 09:30 once traded.
- **Cooldowns:** N19 `cooldown_until` and post-loss timing gate.
- **Loss-throttle:** N19 `consecutive_losses >= N19_MAX_CONSECUTIVE_LOSSES`.
- **Spread gate:** `spread <= max_spread` (`MAX_SPREAD`/strategy copy).
- **IMB gates:** `imb_ratio >= IMB_MIN` (GL/GB), plus N19 impulse/reimpulse regimes.
- **VWAP/VWAP15 gates:** missing or unfavorable VWAP rejects entries.
- **ATR gate:** `atr` missing/<=0 blocks GL/GB/N19 entry.
- **Fresh-cross dependency:** GL/GB require `prev_ltp < ENTRY/BUY <= ltp`; if first cross occurs before indicators are ready, later ticks won’t re-enter unless price dips below and crosses again.
- **Bid/ask vs LTP execution fallback:** `get_marketable_entry/exit` prefers ask/bid, then ltp, then `_last_ltp`.
- **Open/true-open dependency:** missing `global_state[leg]["open"]` blocks Gann level generation.
- **Symbol/token mismatch:** in live path only `token == CE_TOKEN/PE_TOKEN` are considered option ticks.
- **Candle/close timing:** many indicators depend on 1m/15m candle roll and seeded history; stale/incomplete candles block conditions.
- **Swallowed exceptions:** many `try/except: return` paths silently stop a strategy branch.

### 4) Decision checklist (exact variable/function names)
- Global runtime:
  - `_safe_call(...)`
  - `globals()["_STRAT_DISABLED"]`
  - `ALLOW_CONCURRENT_LAYERS`
- GL:
  - `gl_strategy.check_entry(...)`
  - `gl_strategy.day_state.disabled`
  - `gl_strategy._daily_limit_ok()`
  - `gl_strategy._leg_reentry_ok(...)`
  - `gl_strategy._within_entry_window(...)`
  - `global_state[leg]["open"]`
  - `global_state[leg]["spread"]` + `gl_strategy._check_spread(...)`
  - `global_state[leg]["imb_ratio"]` vs `gl_strategy.imb_min`
  - `global_state[leg]["atr"]`
  - `global_state[leg]["vwap"]` / `global_state[leg]["vwap_1m"]`
  - `prev_ltp`, `ltp`, `entry_level`
  - extra filter in `gl_on_tick`: `_get_leg_vwap(leg)` and `_get_leg_rsi3(leg)`
- GB:
  - `gb_strategy.check_entry(...)`
  - `gb_strategy._within_entry_window(...)`
  - `gb_strategy._daily_limit_ok(leg)`
  - `gb_strategy._ensure_levels(leg)` from `global_state[leg]["open"]`
  - `global_state[leg]["atr"]`, `imb_ratio`, `spread`, `vwap_1m`
  - fresh-cross (`prev_ltp < BUY <= ltp`)
  - extra filters in `gb_on_tick`: `_get_leg_vwap`, `_get_leg_rsi3`, `rsi3_15m_prev`, `_get_leg_imb15`, `_get_leg_imb15_avg20`
- N19:
  - `n19_state.in_trade`, `n19_state.trades_today`, `n19_state.cooldown_until`
  - `n19_state.consecutive_losses`, `n19_state.last_loss_time`
  - `N19_TIME_START/END`, `N19_LUNCH_START/END`
  - `global_state[leg]["imb_ratio"]`, `imb_signal`, `vwap_1m`, `atr`, `bar_high/low/close`
  - `n19_state.leg_state[leg].phase` transitions (`IDLE`→`WAIT_PULLBACK`→`ARMED`)
  - reclaim + `_n19_trend_filter_ok(...)`

## Task B: 27 Feb post-mortem (tick replay + evidence)

### 5) Deterministic replay
- Added a new notebook replay cell that runs in paper mode and replays CE/PE CSV rows through the same strategy handlers (`gl_on_tick`, `gb_on_tick`, `n19_on_tick`) while restoring strategy state.
- It includes:
  - `replay_ticks(...)`
  - `first_cross_snapshot(...)`

### 6) First timestamp entry condition becomes TRUE
- **CE leg:** No fresh-cross above GL or GB entry level all day (0 crosses).
- **PE leg:** first (and only) fresh-cross above GB/GL entry happened at **2026-02-27 09:15:00**.

### 7) Signal TRUE but no trade — exact blockers at first PE cross
At `2026-02-27 09:15:00` (PE, first fresh-cross):
- `ltp = 714.65`
- `GB_BUY = 650.33`, `GL_ENTRY = 653.53`
- Blockers snapshot:
  - `spread_ok = False` (`spread = 5.30`, `MAX_SPREAD ~= 2.29`)
  - `imb_ok = False` (`imb_ratio = 1.00`, `IMB_MIN ~= 1.70`)
  - `atr_ok = False` (`opt_atr7` missing)
  - `imb15_x_ok = False` (`imb_avg20_15m` missing)
  - `vwap` field in CSV was empty (so any strict `vwap_1m`-style gate cannot pass from this feed snapshot)

Because GB/GL both require a **fresh** cross and this was the only fresh-cross, once indicators became available later the bot had no second crossing event to legally enter.

### 8) Why chart looked valid but bot didn’t enter
- **Fresh-cross timing mismatch:** price gapped/crossed at 09:15 before all quality filters were ready.
- **Indicator readiness mismatch:** ATR/VWAP/IMB gates not ready/valid at first cross tick.
- **Spread/IMB strictness at first cross:** even when price crossed level, quality gates failed.

### 9) Root cause(s) + minimal fix (without changing strategy rules)
Root causes:
1. One-shot fresh-cross happened before required filters were satisfied.
2. At that moment multiple quality gates (`spread`, `imb`, `atr`, rolling IMB avg) were false.
3. No later recross event occurred to re-trigger entry.

Minimal non-strategy-change fix:
- Add **explicit gate snapshot logging** whenever `prev_ltp < entry <= ltp` is seen but trade is rejected (log all gate variables to diagnostics CSV/console).
- Keep strategy thresholds unchanged.
- Keep paper-trade mode hard-locked during replay.

