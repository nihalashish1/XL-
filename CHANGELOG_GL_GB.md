# CHANGELOG: GL / GB Rule Update (Minimal Diff)

## What changed

### GL strategy
- Added mandatory `RSI(3)_15m >= 50` gate for GL entries.
- Added post-09:30 pullback confirmation: previous slow candle close must be below GL buy level (`ENTRY`) before entry can pass.
- Added explicit GL block logging with reason tags (including RSI and pullback failures).
- Updated GL `T1` assignment to use GB buy level (`compute_buy_targets(...)["BUY"]`) per leg.
- Updated GL trailing activation to start from `T1`, lock SL to `T1` on activation, then continue with existing trailing mechanism.

### GB strategy
- Added mandatory `RSI(3)_15m >= 73` entry gate.
- Updated price/VWAP gate to use `vwap_15m` (`price > VWAP_15m`).
- Added explicit GB block logging with reason tags (including RSI and VWAP failures).
- Added GB stop-loss checks with explicit exit reasons:
  - `SL_GL_BUY` when price hits GL buy level.
  - `SL_RSI50` when `RSI(3)_15m < 50`.
- Kept GB target ladder/trailing structure unchanged.

## Replay validation (offline, CSV)
Command used:
- `python /tmp/replay_check.py`

Result:
- Replay script completed without runtime errors after code changes.
- Block logs are emitted for both GL and GB when entries are rejected.
- First signal / first attempt / first exit summary from replay run:
  - `GL`: first_signal=None, first_attempt=None, first_exit=None
  - `GB`: first_signal=None, first_attempt=None, first_exit=None

Notes:
- This replay harness executes extracted GL/GB strategy code from the notebook against `27TH. FEB CE.csv` and `27TH. FEB PE.csv` in paper-style flow.
- No True Open/login/token/WS logic was modified.
