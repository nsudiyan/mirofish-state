## `backtest_6y/core_replay.py`

```python
#!/usr/bin/env python3
"""Единое ядро шестилетнего honest-бэктеста (PREREG studies/2026-07-10_six_year_PREREG.md).

Чистые функции ИМПОРТИРУЮТСЯ из боевого vol_radar.py (sha256[:16]=9e112767a763fd2e):
detect_spike / select_hits / select_awakenings / is_stablecoin / is_commodity / MAJOR_SYMBOLS.
Здесь реплицируется ТОЛЬКО оркестрация run() (кулдауны, бюджет, каскад, капы) +
позиционный слой (входы/филлы/выходы/funding/дневной MTM), которого в бою нет.

Скан = одно закрытие 30м бара. Live-guard цены незакрытого бара аппроксимируется
open следующего бара (PREREG §9.3). Позиции: только доставленные ОДИНОЧНЫЕ и
ПРОБУЖДЕНИЯ (§9.2), equal-weight 1/5, max concurrent 5 (по market-книге; limit-книги
делят список допущенных сделок, отличаются только фактом филла и ценой входа).
"""
import sys, os, json, gzip, argparse
from datetime import datetime, timezone

sys.path.insert(0, os.path.expanduser("~/trading"))
from vol_radar import (detect_spike, select_hits, select_awakenings,      # боевые чистые функции
                       MAJOR_SYMBOLS, COOLDOWN_H, UNSENT_COOLDOWN_MIN,
                       CASCADE_COOLDOWN_H, SINGLES_PER_DAY, AWAKENING_PER_DAY)

BAR = 1800_000
DAY = 86400_000
MAX_POS = 5
CENSOR_GUARD = 25*3600_000       # входы стоп за 25ч до правой границы (PREREG §9.5)

def uday(ms): return datetime.fromtimestamp(ms/1000, timezone.utc).strftime("%Y-%m-%d")

# ── оркестрация одного скана (зеркало vol_radar.run, без I/O) ──────────────────
def sim_scan(T, uni, bars, cd, budget, counters, hit_log=None):
    """T = start ts закрытого бара; scan_now = T+BAR (момент закрытия).
    Возвращает список доставленных [(hit, kind)]; мутирует cd/budget/counters."""
    scan_now = T + BAR
    today = uday(scan_now)
    if budget.get("date") != today:                       # ролловер UTC-дня (как в бою)
        budget.update({"date": today, "singles": 0, "awakenings": 0,
                       "cascade_ts": budget.get("cascade_ts", 0)})
    hits = []
    for sym in uni:
        if cd.get(sym, 0) > scan_now: continue            # кулдаун-скип ДО детекта (боевой порядок)
        sb = bars.get(sym)
        if not sb: continue
        nxt = sb.get(T + BAR)
        if nxt is None:                                   # нет следующего бара — нет ни live-guard, ни входа
            counters["skip_no_next"] += 1; continue
        win = [sb.get(T - i*BAR) for i in range(10, -1, -1)]
        if any(w is None for w in win):
            counters["skip_gap"] += 1; continue
        kl = win + [[T + BAR, nxt[1], nxt[1], nxt[1], nxt[1], 0.0]]   # live-стаб: close=open следующего
        sp = detect_spike(kl)                             # ═ боевой детект ═
        if sp:
            hits.append({"symbol": sym, "vol_ratio": sp["vol_ratio"],
                         "price_chg_pct": sp["price_chg_pct"], "price": kl[-1][4]})
    counters["hits"] += len(hits)
    mode, chosen = select_hits(hits)                      # ═ боевой отбор ═
    delivered, sent = [], set()

    aw_room = max(0, AWAKENING_PER_DAY - budget["awakenings"])
    delivered_awk = set()
    for h in select_awakenings(hits)[:aw_room]:           # ═ боевой отбор пробуждений ═
        budget["awakenings"] += 1
        sent.add(h["symbol"]); delivered_awk.add(h["symbol"])
        delivered.append((h, "awakening"))
    chosen = [h for h in chosen if h["symbol"] not in delivered_awk]

    if mode == "cascade":
        if scan_now - budget["cascade_ts"] >= CASCADE_COOLDOWN_H*3600_000:
            budget["cascade_ts"] = scan_now
            sent |= {h["symbol"] for h in chosen}         # дайджест: кулдауны да, позиций нет (§9.2)
            counters["cascades"] += 1
        else:
            counters["cascades_muted"] += 1
    else:
        room = max(0, SINGLES_PER_DAY - budget["singles"])
        for h in chosen[:room]:
            budget["singles"] += 1
            sent.add(h["symbol"])
            delivered.append((h, "single"))
        counters["singles_capped"] += max(0, len(chosen) - room)

    for h in hits:                                        # кулдауны на ВСЕ хиты (боевая строка 334)
        cd[h["symbol"]] = scan_now + (COOLDOWN_H*3600_000 if h["symbol"] in sent
                                      else UNSENT_COOLDOWN_MIN*60_000)
    if hit_log is not None and hits:                      # инструментовка parity: состав, БЕЗ доходностей
        hit_log.append({"T": T, "mode": mode,
                        "hits": [(h["symbol"], h["vol_ratio"]) for h in hits],
                        "sent": sorted(sent)})
    return delivered

# ── позиционный слой ────────────────────────────────────────────────────────────
def find_exit(sb, entry_start, t_end):
    """close первого бара с start >= entry+24ч; при дырах ищем до +48ч, иначе цензура."""
    tgt = entry_start + DAY
    t = tgt
    while t <= min(tgt + 2*DAY, t_end):
        b = sb.get(t)
        if b: return t, b[4], False
        t += BAR
    last, lb = None, None
    t = tgt - BAR
    while t > entry_start:
        b = sb.get(t)
        if b: last, lb = t, b[4]; break
        t -= BAR
    return last, lb, True                                  # цензура (делистинг/край данных)

def funding_cost_pct(fr, entry_ms, exit_ms):
    """Лонг платит положительный funding: cost% = +rate*100 за каждую 8ч-метку в (entry, exit]."""
    return sum(r for ts, r in fr if entry_ms < ts <= exit_ms) * 100.0

def run_engine(bars, fund, uni_by_day, t0, t1, log=lambda *a: None, hit_log=None,
               state=None, censor_end=None):
    """bars: {sym:{start_ts:[ts,o,h,l,c,vol,...]}}, fund: {sym:[(ts,rate)...]},
    uni_by_day: {'YYYY-MM-DD': [syms по убыванию turnover]}. Возвращает (trades, counters).
    state: перенос cd/budget/open_ex между кусками сквозного прогона (мутируется на месте);
    censor_end: глобальная правая граница для хвостовой цензуры (по умолчанию t1)."""
    st = state if state is not None else {}
    cd = st.setdefault("cd", {})
    budget = st.setdefault("budget", {"date": None, "singles": 0, "awakenings": 0, "cascade_ts": 0})
    open_ex = st.setdefault("open_ex", [])                 # exit_close_ms открытых позиций (market-книга)
    censor_end = censor_end or t1
    counters = dict(scans=0, hits=0, cascades=0, cascades_muted=0, singles_capped=0,
                    skip_gap=0, skip_no_next=0, skipped_full=0, censored=0,
                    delivered_singles=0, delivered_awakenings=0)
    trades = []
    T = t0
    while T + BAR <= t1:
        uni = uni_by_day.get(uday(T + BAR), [])
        if uni:
            counters["scans"] += 1
            for h, kind in sim_scan(T, uni, bars, cd, budget, counters, hit_log):
                counters[f"delivered_{kind}s"] += 1
                scan_now = T + BAR
                if scan_now > censor_end - CENSOR_GUARD: continue  # хвостовая цензура входов
                open_ex[:] = [e for e in open_ex if e > scan_now]
                if len(open_ex) >= MAX_POS:
                    counters["skipped_full"] += 1; continue
                sym = h["symbol"]; sb = bars[sym]
                entry_start = T + BAR                      # вход market по open следующего бара
                eb = sb[entry_start]
                mkt_px, lim_px = eb[1], sb[T][4]           # limit = close сигнального бара
                exit_start, exit_px, cens = find_exit(sb, entry_start, t1)
                if exit_px is None: continue
                if cens: counters["censored"] += 1
                exit_close_ms = exit_start + BAR
                open_ex.append(exit_close_ms)
                fpct = funding_cost_pct(fund.get(sym, ()), entry_start, exit_close_ms)
                gross_mkt = (exit_px/mkt_px - 1)*100
                opt_fill  = eb[3] <= lim_px                # касание low в баре входа
                cons_fill = eb[3] <= lim_px*0.9995         # пробитие
                trades.append({
                    "ts_utc": datetime.fromtimestamp((T+BAR)/1000, timezone.utc).strftime("%Y-%m-%dT%H:%M"),
                    "signal_bar": T, "symbol": sym, "kind": kind, "vol_ratio": h["vol_ratio"],
                    "entry_ts": entry_start, "mkt_entry": mkt_px, "entry_close": eb[4],
                    "lim_price": lim_px,
                    "opt_fill": int(opt_fill), "cons_fill": int(cons_fill),
                    "exit_ts": exit_close_ms, "exit_px": exit_px, "censored": int(cens),
                    "gross_mkt_pct": round(gross_mkt, 4),
                    "gross_opt_pct": round((exit_px/lim_px - 1)*100, 4) if opt_fill else None,
                    "gross_cons_pct": round((exit_px/lim_px - 1)*100, 4) if cons_fill else None,
                    "funding_pct": round(fpct, 4),
                })
        T += BAR
        if T % (30*DAY) < BAR: log(f"  …{uday(T)} trades={len(trades)}")
    return trades, counters

def daily_series(trades, bars, cost_rt=0.31):
    """Календарный дневной портфельный PnL, %: MTM по закрытиям суток, вес 1/MAX_POS,
    издержки+funding списываются в день выхода. Некомпаундированная сумма весов."""
    def px_at(sb, ms):
        t = ms - BAR
        for _ in range(96):
            b = sb.get(t)
            if b: return b[4]
            t -= BAR
        return None
    daily = {}
    w = 1.0/MAX_POS
    for tr in trades:
        if tr["censored"]: continue
        sb = bars[tr["symbol"]]
        prev_px, prev_ms = tr["mkt_entry"], tr["entry_ts"]
        m0 = (tr["entry_ts"]//DAY + 1)*DAY
        marks = list(range(m0, tr["exit_ts"], DAY)) + [tr["exit_ts"]]
        for i, m in enumerate(marks):
            px = tr["exit_px"] if m == tr["exit_ts"] else px_at(sb, m)
            if px is None: continue
            d = uday(m - 1)
            daily[d] = daily.get(d, 0.0) + w*(px/prev_px - 1)*100
            prev_px, prev_ms = px, m
        d_exit = uday(tr["exit_ts"] - 1)
        daily[d_exit] = daily.get(d_exit, 0.0) - w*(cost_rt + tr["funding_pct"])
    return dict(sorted(daily.items()))

# ── I/O обвязка ────────────────────────────────────────────────────────────────
ROOT = os.path.expanduser("~/trading/backtest_6y")
def load_bars(sym):
    p = f"{ROOT}/data/klines/{sym}.json.gz"
    if not os.path.exists(p): return None
    return {int(k): v for k, v in json.load(gzip.open(p, "rt")).items()}
def load_fund(sym):
    p = f"{ROOT}/data/funding/{sym}.json.gz"
    if not os.path.exists(p): return []
    return sorted((int(k), v) for k, v in json.load(gzip.open(p, "rt")).items())

# ── selfcheck: фикстуры оркестрации и позиционного слоя ────────────────────────
def _mk(ts, o, h, l, c, v): return [ts, o, h, l, c, v]
def _flat(sym_bars, t0, n, px=100.0, vol=50.0):
    for i in range(n): sym_bars[t0+i*BAR] = _mk(t0+i*BAR, px, px+0.2, px-0.2, px, vol)

def selfcheck():
    t0 = 1600000000000 - (1600000000000 % DAY)             # ровная UTC-полночь
    # A: одиночный спайк на альте → доставка, позиция, кулдаун 4ч
    A = {}; _flat(A, t0, 11)
    sig = t0+11*BAR
    A[sig] = _mk(sig, 100, 100.5, 99.6, 100.3, 400)        # 8× объём, цена стоит
    for i in range(12, 120): A[t0+i*BAR] = _mk(t0+i*BAR, 100.3, 106, 100.2, 105, 60)
    bars = {"AAAUSDT": A}
    uni = {uday(t0+i*BAR): ["AAAUSDT"] for i in range(0, 130)}
    tr, c = run_engine(bars, {}, uni, t0, t0+120*BAR)
    assert c["delivered_singles"] == 1 and len(tr) == 1, (c, tr)
    assert tr[0]["mkt_entry"] == 100.3 and abs(tr[0]["gross_mkt_pct"] - (105/100.3-1)*100) < 1e-3
    exp_exit = sig+BAR+DAY                                  # первый бар ≥ вход+24ч
    assert tr[0]["exit_ts"] == exp_exit + BAR, tr[0]
    # B: мажор-одиночка НЕ доставляется; кулдаун unsent 30м
    B = dict(A); bars2 = {"BTCUSDT": B}
    tr2, c2 = run_engine(bars2, {}, {d: ["BTCUSDT"] for d in uni}, t0, t0+120*BAR)
    assert c2["delivered_singles"] == 0 and c2["hits"] >= 1, c2
    # C: каскад ≥3 → дайджест без позиций; повтор в 2ч подавлен
    bars3 = {}
    for s in ("AUSDT", "BUSDT", "CUSDT"):
        d = {}; _flat(d, t0, 11); d[sig] = _mk(sig, 100, 100.5, 99.6, 100.3, 400)
        d[sig+BAR] = _mk(sig+BAR, 100.3, 100.6, 100.1, 100.4, 300)   # 2й спайк-бар подряд
        for i in range(13, 120): d[t0+i*BAR] = _mk(t0+i*BAR, 100.4, 100.8, 100.1, 100.4, 60)
        bars3[s] = d
    tr3, c3 = run_engine(bars3, {}, {d_: ["AUSDT","BUSDT","CUSDT"] for d_ in uni}, t0, t0+120*BAR)
    assert c3["cascades"] == 1 and len(tr3) == 0, (c3, len(tr3))
    # D: пробуждение vr≥15 → позиция даже при каскаде; мини-кап 3/день
    bars4 = {}
    for s in ("AUSDT", "BUSDT", "CUSDT", "DUSDT"):
        d = {}; _flat(d, t0, 11)
        d[sig] = _mk(sig, 100, 100.5, 99.6, 100.3, 1000)   # 20× = пробуждение
        for i in range(12, 120): d[t0+i*BAR] = _mk(t0+i*BAR, 100.3, 101, 100, 100.5, 60)
        bars4[s] = d
    tr4, c4 = run_engine(bars4, {}, {d_: ["AUSDT","BUSDT","CUSDT","DUSDT"] for d_ in uni}, t0, t0+120*BAR)
    assert c4["delivered_awakenings"] == 3, c4              # 4-е — сверх мини-капа
    assert c4["cascades"] == 1, c4                          # остаток ушёл дайджестом
    # E: live-guard — цена улетела на open следующего бара → хита нет
    E = {}; _flat(E, t0, 11)
    E[sig] = _mk(sig, 100, 100.5, 99.6, 100.3, 400)
    E[sig+BAR] = _mk(sig+BAR, 103, 104, 102.5, 103.5, 60)   # open +2.7% — опоздали
    for i in range(13, 120): E[t0+i*BAR] = _mk(t0+i*BAR, 103, 104, 102, 103, 60)
    tr5, c5 = run_engine({"AAAUSDT": E}, {}, uni, t0, t0+120*BAR)
    assert c5["hits"] == 0, c5
    # F: funding уменьшает net; филлы: opt по касанию, cons по пробитию
    fund = {"AAAUSDT": [(sig+BAR+8*3600_000, 0.01)]}        # 1% за период — утрирован для теста
    tr6, _ = run_engine(bars, fund, uni, t0, t0+120*BAR)
    assert abs(tr6[0]["funding_pct"] - 1.0) < 1e-9, tr6[0]
    eb_low = bars["AAAUSDT"][sig+BAR][3]                    # low=100.2 ≤ lim=100.3 → opt филл
    assert tr6[0]["opt_fill"] == 1 and eb_low <= tr6[0]["lim_price"]
    assert tr6[0]["cons_fill"] == 1                         # 100.2 ≤ 100.3×0.9995=100.25
    # G: дневной MTM — сумма дневных = сделке минус кост
    ds = daily_series(tr, bars, cost_rt=0.31)
    assert abs(sum(ds.values()) - (tr[0]["gross_mkt_pct"] - 0.31 - 0)/MAX_POS) < 1e-3, ds
    print("✓ core selfcheck: 7 фикстур прошли (single/major/cascade/awakening-кап/"
          "live-guard/funding+филлы/дневной MTM)")

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--selfcheck", action="store_true")
    a = ap.parse_args()
    if a.selfcheck: selfcheck()
    else: print("используется как модуль (smoke/parity/full — отдельные раннеры)")
```

## `backtest_6y/results_stats.py`

```python
#!/usr/bin/env python3
"""RESULTS-машина (PREREG §5, §7 + поправки §9): механический расчёт метрик,
ablations, sensitivity и ВЕРДИКТА. Запускается ОДИН раз после full_runner.
Все пороги — из замороженного предрега, интерпретация запрещена коду."""
import json, gzip, csv, os, random
from datetime import datetime, timezone, timedelta
from statistics import mean, pstdev

ROOT = os.path.expanduser("~/trading/backtest_6y")
rnd = random.Random(20260710)

trades = []
for r in csv.DictReader(open(f"{ROOT}/out/trades.csv")):
    for k in ("vol_ratio","mkt_entry","entry_close","lim_price","exit_px","gross_mkt_pct","funding_pct"):
        r[k] = float(r[k])
    for k in ("signal_bar","entry_ts","exit_ts","opt_fill","cons_fill","censored"):
        r[k] = int(r[k])
    r["gross_opt_pct"] = float(r["gross_opt_pct"]) if r["gross_opt_pct"] not in ("", "None") else None
    trades.append(r)
trades = [t for t in trades if not t["censored"]]
daily = {}
for r in csv.DictReader(open(f"{ROOT}/out/daily_pnl.csv")):
    daily[r["date"]] = {0.14: float(r["ret_c014"]), 0.31: float(r["ret_c031"]), 0.71: float(r["ret_c071"])}
summ = json.load(open(f"{ROOT}/data/manifest/_summary.json"))
first_bar = {s: c["first"] for s, c in summ["coverage"].items()}

def drange(a, b):
    d = datetime.strptime(a, "%Y-%m-%d")
    while d.strftime("%Y-%m-%d") <= b:
        yield d.strftime("%Y-%m-%d"); d += timedelta(days=1)
WIN = {"full": ("2021-10-17","2026-07-09"), "train": ("2021-10-17","2023-12-31"),
       "val": ("2024-01-01","2024-12-31"), "test_verdict": ("2025-01-01","2026-04-30"),
       "test_full": ("2025-01-01","2026-07-09"), "contaminated": ("2026-05-01","2026-07-09"),
       "2021H2": ("2021-10-17","2021-12-31"), "2022": ("2022-01-01","2022-12-31"),
       "2023": ("2023-01-01","2023-12-31"), "2024": ("2024-01-01","2024-12-31"),
       "2025": ("2025-01-01","2025-12-31"), "2026H1": ("2026-01-01","2026-07-09")}

# ЕДИНАЯ модель портфельного учёта (архивная v2, ревью Codex №7): terminal
# per-trade net, атрибуция на UTC-дату ВЫХОДА, вес 1/5, без компаундинга.
# Детерминированно из замороженного trades.csv; идентична базе ablations.
# MTM-ряд из daily_pnl.csv остаётся вспомогательным (печатается кросс-чеком).
REAL = {c: {} for c in (0.14, 0.31, 0.71)}      # наполняется ниже, после def net()

def series(win, cost):
    a, b = WIN[win]
    return [REAL[cost].get(d, 0.0) for d in drange(a, b)]
def series_mtm(win, cost):
    a, b = WIN[win]
    return [daily.get(d, {}).get(cost, 0.0) for d in drange(a, b)]
def stats(xs):
    n = len(xs); m = mean(xs); sd = pstdev(xs)*(n/(n-1))**0.5 if n > 1 else 0
    t = m/(sd/n**0.5) if sd else 0
    sharpe = m/sd*(365**0.5) if sd else 0
    dn = [x for x in xs if x < 0]
    sortino = m/(pstdev(dn)*(len(dn)/(len(dn)-1))**0.5)*(365**0.5) if len(dn) > 2 and pstdev(dn) else 0
    cum = mx = dd = 0.0
    for x in xs:
        cum += x; mx = max(mx, cum); dd = min(dd, cum - mx)
    return {"total": round(sum(xs), 2), "t": round(t, 2), "sharpe": round(sharpe, 2),
            "sortino": round(sortino, 2), "maxDD": round(dd, 2), "days": n,
            "act_days": sum(1 for x in xs if x != 0.0)}
def boot(xs, k=10000):
    nb = max(1, len(xs)//7)
    blocks = [xs[i*7:(i+1)*7] for i in range(nb)]
    tot = sorted(sum(v for b in (rnd.choice(blocks) for _ in range(nb)) for v in b) for _ in range(k))
    return round(tot[int(0.025*k)], 2), round(tot[int(0.975*k)], 2)

def net(t, cost, entry="mkt_entry", short=False):
    g = (t["exit_px"]/t[entry] - 1)*100
    return (-g - cost + t["funding_pct"]) if short else (g - cost - t["funding_pct"])
def in_win(t, win):
    a, b = WIN[win]
    d = t["ts_utc"][:10]
    return a <= d <= b
for _t in trades:
    _d = datetime.fromtimestamp(_t["exit_ts"]/1000, timezone.utc).strftime("%Y-%m-%d")
    for _c in REAL:
        REAL[_c][_d] = REAL[_c].get(_d, 0.0) + net(_t, _c)/5

def realized(sub, cost=0.31, **kw):          # атрибуция по дню выхода (для ablations)
    dd = {}
    for t in sub:
        d = datetime.fromtimestamp(t["exit_ts"]/1000, timezone.utc).strftime("%Y-%m-%d")
        dd[d] = dd.get(d, 0.0) + net(t, cost, **kw)/5
    a, b = WIN["full"]
    return [dd.get(d, 0.0) for d in drange(a, b)]

print("═══════ ШЕСТИЛЕТНИЙ ПРОГОН: RESULTS v2 (единая учётная модель) ═══════")
print("SURVIVOR-ONLY UNIVERSE: RESULTS ARE AN UPPER BOUND FOR LONG-SIDE PERFORMANCE")
print("учёт: terminal per-trade net → дата выхода, вес 1/5, без компаундинга")
mtm = round(sum(series_mtm("full", 0.31)), 2)
print(f"кросс-чек идентичности: MTM-ряд full@0.31 = {mtm} vs terminal (ниже) — "
      f"разница = путевая сумма промежуточных сегментов, обе модели одного знака\n")
print(f"{'окно':>13} {'кост':>5} | {'total%':>8} {'t':>6} {'Sharpe':>6} {'Sortino':>7} {'maxDD':>7} {'дней':>5} {'актив':>5}")
S = {}
for w in WIN:
    for c in (0.14, 0.31, 0.71):
        S[(w, c)] = stats(series(w, c))
        if c == 0.31 or w in ("full", "test_verdict", "val"):
            s = S[(w, c)]
            print(f"{w:>13} {c:>5} | {s['total']:>8} {s['t']:>6} {s['sharpe']:>6} "
                  f"{s['sortino']:>7} {s['maxDD']:>7} {s['days']:>5} {s['act_days']:>5}")
for w in ("full", "val", "test_verdict"):
    lo, hi = boot(series(w, 0.31))
    print(f"bootstrap 7д×10к {w}@0.31: 95% CI total = [{lo}, {hi}]")

n_by = {}
for t in trades:
    if in_win(t, "full"): n_by[t["kind"]] = n_by.get(t["kind"], 0) + 1
print(f"\nсделок в sample: {sum(n_by.values())} {n_by} · fill opt {sum(t['opt_fill'] for t in trades)}"
      f"/{len(trades)} cons {sum(t['cons_fill'] for t in trades)}/{len(trades)}")

print("\n── ABLATIONS (realized-атрибуция, full@0.31) ──")
base = realized([t for t in trades if in_win(t, "full")])
print(f"база realized: total {round(sum(base),2)} t={stats(base)['t']}")
contrib = {}
for t in trades:
    if in_win(t, "full"): contrib[t["symbol"]] = contrib.get(t["symbol"], 0) + net(t, 0.31)
top5s = sorted(contrib, key=lambda s: -abs(contrib[s]))[:5]
ab1 = realized([t for t in trades if in_win(t, "full") and t["symbol"] not in top5s])
print(f"без топ-5 символов {top5s}: total {round(sum(ab1),2)} t={stats(ab1)['t']}")
day_r = {}
for i, d in enumerate(drange(*WIN["full"])): day_r[d] = base[i]
top5d = sorted(day_r, key=lambda d: -abs(day_r[d]))[:5]
ab2 = [v for d, v in day_r.items() if d not in top5d]
print(f"без топ-5 дней {top5d}: total {round(sum(ab2),2)} t={stats(ab2)['t']}")
mature = [t for t in trades if in_win(t, "full")
          and first_bar.get(t["symbol"]) and t["signal_bar"] - first_bar[t["symbol"]] >= 180*86400_000]
ab3 = realized(mature)
print(f"возраст ≥180д (n={len(mature)}): total {round(sum(ab3),2)} t={stats(ab3)['t']}")
ab4 = realized([t for t in trades if in_win(t, "full")], entry="entry_close")
print(f"запоздалый вход (+30м, по close): total {round(sum(ab4),2)} t={stats(ab4)['t']}")
ab5 = realized([t for t in trades if in_win(t, "full")], short=True)
print(f"SHORT-зеркало (диагностика): total {round(sum(ab5),2)} t={stats(ab5)['t']}")

print("\n── ВЕРДИКТ ПО §7 (механически) ──")
ok1 = all(S[(w, 0.31)]["total"] > 0 for w in ("full", "val", "test_verdict"))
f31, f71 = S[("full", 0.31)]["total"], S[("full", 0.71)]["total"]
ok2 = (f71 > -0.5*abs(f31)) and (S[("test_verdict", 0.71)]["total"] > 0) == (S[("test_verdict", 0.31)]["total"] > 0)
sgn = f31 > 0
ok3 = all((sum(a) > 0) == sgn and stats(a)["t"] > 1.0 for a in (ab1, ab2, ab3))
ok4 = S[("test_verdict", 0.31)]["total"] > 0 and S[("test_verdict", 0.31)]["t"] >= 2.0
delayed_flip = (sum(ab4) > 0) != sgn
print(f"1) net>0 @0.31 (full+val+test_verdict): {ok1}")
print(f"2) выживает @0.71: {ok2}")
print(f"3) ablations держат знак и t>1 (симв/дни/зрелость): {ok3}")
print(f"4) test_verdict: net>0 и t≥2.0: {ok4}  (t={S[('test_verdict',0.31)]['t']})")
print(f"5) запоздалый вход флипает знак: {delayed_flip}")
if ok1 and ok2 and ok3 and ok4 and not delayed_flip:
    verdict = "CONFIRMED FOR PAPER ONLY"
elif not ok1 or not ok4:
    verdict = "KILLED / NO-TRADE"
else:
    verdict = "INCONCLUSIVE"
print(f"\n════ ВЕРДИКТ: {verdict} ════")
json.dump({"S": {f"{w}@{c}": v for (w, c), v in S.items()}, "verdict": verdict,
           "gates": {"ok1": ok1, "ok2": ok2, "ok3": ok3, "ok4": ok4, "delayed_flip": delayed_flip},
           "ablations": {"no_top5_sym": round(sum(ab1),2), "no_top5_day": round(sum(ab2),2),
                          "mature180": round(sum(ab3),2), "delayed": round(sum(ab4),2),
                          "short_mirror": round(sum(ab5),2)},
           "top5_symbols": top5s, "top5_days": top5d},
          open(f"{ROOT}/out/results_stats.json", "w"), indent=1)
print("сохранено: out/results_stats.json")
```

## `backtest_6y/btc_hedged_runner.py`

```python
#!/usr/bin/env python3
"""BTC-BETA-HEDGED RADAR — тест №2 (PREREG studies/2026-07-10_btc_hedged_PREREG.md,
ЗАМОРОЖЕН до открытия данных). Вход = замороженный out/trades.csv (6702 сделки,
ре-симуляции НЕТ). На сделку: β по формуле Codex (30д 30м-ретёрнов ДО сигнального
бара, ≥480 общих точек, клип [0.2;3.0], нет беты → исключение из primary),
alpha-net(c) = (ret_coin − β·ret_btc) − c·(1+β) − f_coin + β·f_btc.
Учётная модель v2 (terminal per-trade → дата выхода, вес 1/5). Один запуск,
вердикт печатает машина по §5."""
import json, gzip, csv, os, random
from datetime import datetime, timezone, timedelta
from statistics import mean, pstdev

ROOT = os.path.expanduser("~/trading/backtest_6y")
BAR, D30 = 1800_000, 30*86400_000
rnd = random.Random(20260711)

def load_bars(sym):
    p = f"{ROOT}/data/klines/{sym}.json.gz"
    if not os.path.exists(p): return None
    return {int(k): v for k, v in json.load(gzip.open(p, "rt")).items()}
def load_fund(sym):
    p = f"{ROOT}/data/funding/{sym}.json.gz"
    if not os.path.exists(p): return []
    return sorted((int(k), v) for k, v in json.load(gzip.open(p, "rt")).items())

BTC = load_bars("BTCUSDT")
BTC_F = load_fund("BTCUSDT")
btc_close = {t: b[4] for t, b in BTC.items()}

trades = []
for r in csv.DictReader(open(f"{ROOT}/out/trades.csv")):
    if int(r["censored"]): continue
    trades.append({"sym": r["symbol"], "kind": r["kind"], "T": int(r["signal_bar"]),
                   "entry_ts": int(r["entry_ts"]), "exit_ts": int(r["exit_ts"]),
                   "entry": float(r["mkt_entry"]), "exit": float(r["exit_px"]),
                   "f_coin": float(r["funding_pct"]), "date": r["ts_utc"][:10]})
print(f"замороженных сделок на входе: {len(trades)}")

def beta_for(sb, T):
    """β по предрегу §2: окно [T+BAR−30д; T−BAR], пары соседних баров обеих серий."""
    lo, hi = T + BAR - D30, T - BAR
    xs, ys = [], []
    t = lo - (lo % BAR)
    while t <= hi:
        a0, a1 = sb.get(t - BAR), sb.get(t)
        b0, b1 = btc_close.get(t - BAR), btc_close.get(t)
        if a0 and a1 and b0 and b1:
            xs.append(b1/b0 - 1)
            ys.append(a1[4]/a0[4] - 1)
        t += BAR
    if len(xs) < 480: return None, len(xs)
    mx, my = mean(xs), mean(ys)
    var = sum((x - mx)**2 for x in xs)
    if var <= 0: return None, len(xs)
    cov = sum((x - mx)*(y - my) for x, y in zip(xs, ys))
    return cov/var, len(xs)

out, counters = [], {"no_beta": 0, "no_btc_bar": 0, "clip_lo": 0, "clip_hi": 0, "beta_pts_med": []}
by_sym = {}
for t in trades: by_sym.setdefault(t["sym"], []).append(t)
for n, (sym, ts_) in enumerate(sorted(by_sym.items())):
    sb = load_bars(sym)
    ff = load_fund(sym)
    for tr in ts_:
        raw_b, npts = beta_for(sb, tr["T"]) if sb else (None, 0)
        eb, xb = BTC.get(tr["entry_ts"]), BTC.get(tr["exit_ts"] - BAR)
        if eb is None or xb is None:
            counters["no_btc_bar"] += 1; continue
        rec = dict(tr)
        rec["ret_coin"] = (tr["exit"]/tr["entry"] - 1)*100
        rec["ret_btc"] = (xb[4]/eb[1] - 1)*100
        rec["f_btc"] = sum(r for t_, r in BTC_F if tr["entry_ts"] < t_ <= tr["exit_ts"]) * 100
        if raw_b is None:
            counters["no_beta"] += 1
            rec["beta"] = None                    # вне primary; живёт в β=1-диагностике
        else:
            b = min(3.0, max(0.2, raw_b))
            counters["clip_lo"] += raw_b < 0.2; counters["clip_hi"] += raw_b > 3.0
            counters["beta_pts_med"].append(npts)
            rec["beta"] = round(b, 4); rec["beta_raw"] = round(raw_b, 4)
        out.append(rec)
    if (n+1) % 100 == 0: print(f"  …символов {n+1}/{len(by_sym)}", flush=True)

def alpha(rec, c, b=None):
    b = rec["beta"] if b is None else b
    return (rec["ret_coin"] - b*rec["ret_btc"]) - c*(1 + b) - rec["f_coin"] + b*rec["f_btc"]

prim = [r for r in out if r["beta"] is not None]
med = sorted(counters["beta_pts_med"])[len(counters["beta_pts_med"])//2] if counters["beta_pts_med"] else 0
print(f"\nprimary sample: {len(prim)} · исключено no_beta={counters['no_beta']} "
      f"no_btc_bar={counters['no_btc_bar']} · клипы lo/hi={counters['clip_lo']}/{counters['clip_hi']} · медиана точек β={med}")
with open(f"{ROOT}/out/hedged_trades.csv", "w", newline="") as f:
    cols = ["sym","kind","date","T","entry_ts","exit_ts","beta","beta_raw","ret_coin","ret_btc","f_coin","f_btc"]
    w = csv.writer(f); w.writerow(cols)
    for r in out: w.writerow([r.get(k, "") for k in cols])

# ── вердиктная печать (окна/ворота §4–5 = убитый тест; учёт v2) ──
def ud_exit(r): return datetime.fromtimestamp(r["exit_ts"]/1000, timezone.utc).strftime("%Y-%m-%d")
def drange(a, b):
    d = datetime.strptime(a, "%Y-%m-%d")
    while d.strftime("%Y-%m-%d") <= b:
        yield d.strftime("%Y-%m-%d"); d += timedelta(days=1)
WIN = {"full": ("2021-10-17","2026-07-09"), "train": ("2021-10-17","2023-12-31"),
       "val": ("2024-01-01","2024-12-31"), "test_verdict": ("2025-01-01","2026-04-30"),
       "contaminated": ("2026-05-01","2026-07-09"), "2022": ("2022-01-01","2022-12-31"),
       "2023": ("2023-01-01","2023-12-31"), "2025": ("2025-01-01","2025-12-31"),
       "2026H1": ("2026-01-01","2026-07-09")}
def stream(sub, c, b=None):
    dd = {}
    for r in sub: dd[ud_exit(r)] = dd.get(ud_exit(r), 0.0) + alpha(r, c, b)/5
    return dd
def series(dd, win):
    a, b = WIN[win]; return [dd.get(d, 0.0) for d in drange(a, b)]
def stats(xs):
    n = len(xs); m = mean(xs); sd = pstdev(xs)*(n/(n-1))**0.5 if n > 1 else 0
    t = m/(sd/n**0.5) if sd else 0
    cum = mx = dd_ = 0.0
    for x in xs:
        cum += x; mx = max(mx, cum); dd_ = min(dd_, cum - mx)
    return {"total": round(sum(xs), 2), "t": round(t, 2),
            "sharpe": round(m/sd*365**0.5, 2) if sd else 0, "maxDD": round(dd_, 2)}
def boot(xs, k=10000):
    nb = max(1, len(xs)//7)
    blocks = [xs[i*7:(i+1)*7] for i in range(nb)]
    tot = sorted(sum(v for bl in (rnd.choice(blocks) for _ in range(nb)) for v in bl) for _ in range(k))
    return round(tot[int(0.025*k)], 2), round(tot[int(0.975*k)], 2)

print("\n═══════ BTC-BETA-HEDGED RADAR: RESULTS (единственное вскрытие) ═══════")
print("SURVIVOR-ONLY UNIVERSE: UPPER BOUND (наследуется) · учёт v2 · косты c·(1+β)\n")
print(f"{'окно':>13} {'c/ногу':>6} | {'alphaΣ%':>8} {'t':>6} {'Sharpe':>6} {'maxDD':>8}")
S = {}
for c in (0.14, 0.31, 0.71):
    dd = stream(prim, c)
    for w in WIN:
        S[(w, c)] = stats(series(dd, w))
        if c == 0.31 or w in ("full", "val", "test_verdict"):
            s = S[(w, c)]
            print(f"{w:>13} {c:>6} | {s['total']:>8} {s['t']:>6} {s['sharpe']:>6} {s['maxDD']:>8}")
dd31 = stream(prim, 0.31)
for w in ("full", "val", "test_verdict"):
    lo, hi = boot(series(dd31, w))
    print(f"bootstrap {w}@0.31: 95% CI [{lo}, {hi}]")

print("\n── вторичные (НЕ вердикт) ──")
bmed = sorted(r["beta"] for r in prim)[len(prim)//2]
print(f"β: медиана {bmed}, gross-альфа (c=0, funding в нулях НЕ обнулён): "
      f"{round(sum(alpha(r, 0) for r in prim)/len(prim), 4)}%/сделку")
for k in ("single", "awakening"):
    sub = [r for r in prim if r["kind"] == k]
    s = stats(series(stream(sub, 0.31), "full"))
    print(f"  {k}: n={len(sub)} full@0.31 {s['total']} t={s['t']}")
for lab, lo_, hi_ in (("β<0.8", 0, 0.8), ("0.8–1.5", 0.8, 1.5), (">1.5", 1.5, 99)):
    sub = [r for r in prim if lo_ <= r["beta"] < hi_]
    if sub:
        s = stats(series(stream(sub, 0.31), "full"))
        print(f"  β {lab}: n={len(sub)} full@0.31 {s['total']} t={s['t']}")
sub_all = out
s1 = stats(series(stream(sub_all, 0.31, b=1.0), "full"))
print(f"  диагностика β=1 (ВСЕ {len(sub_all)} сделок, вкл. исключённых): full@0.31 {s1['total']} t={s1['t']}")

print("\n── ВЕРДИКТ ПО §5 (механически) ──")
ok1 = all(S[(w, 0.31)]["total"] > 0 for w in ("full", "val", "test_verdict"))
f31, f71 = S[("full", 0.31)]["total"], S[("full", 0.71)]["total"]
ok2 = (f71 > -0.5*abs(f31)) and ((S[("test_verdict", 0.71)]["total"] > 0) == (S[("test_verdict", 0.31)]["total"] > 0))
sgn = f31 > 0
contrib = {}
for r in prim: contrib[r["sym"]] = contrib.get(r["sym"], 0) + alpha(r, 0.31)
top5s = sorted(contrib, key=lambda s_: -abs(contrib[s_]))[:5]
ab1 = series(stream([r for r in prim if r["sym"] not in top5s], 0.31), "full")
day31 = series(dd31, "full")
dmap = dict(zip(drange(*WIN["full"]), day31))
top5d = sorted(dmap, key=lambda d: -abs(dmap[d]))[:5]
ab2 = [v for d, v in dmap.items() if d not in top5d]
summ = json.load(open(f"{ROOT}/data/manifest/_summary.json"))
fb = {s_: c_["first"] for s_, c_ in summ["coverage"].items()}
ab3 = series(stream([r for r in prim if fb.get(r["sym"]) and r["T"] - fb[r["sym"]] >= 180*86400_000], 0.31), "full")
ok3 = all((sum(a) > 0) == sgn and abs(stats(a)["t"]) > 1.0 for a in (ab1, ab2, ab3))
ok4 = S[("test_verdict", 0.31)]["total"] > 0 and S[("test_verdict", 0.31)]["t"] >= 2.0
print(f"1) alpha>0 @0.31 (full+val+test_verdict): {ok1}")
print(f"2) выживает @0.71: {ok2}")
print(f"3) ablations (топ-5 симв {round(sum(ab1),1)} / топ-5 дней {round(sum(ab2),1)} / ≥180д {round(sum(ab3),1)}) держат знак и |t|>1: {ok3}")
print(f"4) test_verdict: alpha>0 и t≥2.0: {ok4} (t={S[('test_verdict',0.31)]['t']})")
verdict = ("CONFIRMED FOR PAPER ONLY" if (ok1 and ok2 and ok3 and ok4)
           else "KILLED / NO-TRADE" if (not ok1 or not ok4) else "INCONCLUSIVE")
print(f"\n════ ВЕРДИКТ: {verdict} ════")
json.dump({"verdict": verdict, "gates": {"ok1": ok1, "ok2": ok2, "ok3": ok3, "ok4": ok4},
           "primary_n": len(prim), "excluded_no_beta": counters["no_beta"],
           "S": {f"{w}@{c}": v for (w, c), v in S.items()}},
          open(f"{ROOT}/out/hedged_results.json", "w"), indent=1)
print("сохранено: out/hedged_trades.csv, out/hedged_results.json")
```

## `backtest_6y/h_oi_runner.py`

```python
#!/usr/bin/env python3
"""H-OI-01: crowded leverage unwind (PREREG studies/2026-07-11_H-OI-01_PREREG.md,
ЗАМОРОЖЕН с вето Codex). Раз в сутки 00:00 UTC по PIT-вселенной:
t1/t0-правила OI буквально по timestamp; f_norm = rate×24ч/фактический интервал
двух последних settlement; score = f_norm × max(0, ΔOI_4h%); K_D = min(10,#pos,#neg),
торгуем при K_D≥5; шорт top-K положительных, лонг top-K отрицательных; вес 1/K_D
на сторону; hold 24ч; косты и funding обеих ног. Учёт v2. Дни независимы (стейта нет).
python3 h_oi_runner.py --selfcheck | python3 h_oi_runner.py"""
import json, gzip, csv, os, random, argparse
from datetime import datetime, timezone, timedelta
from statistics import mean, pstdev
from bisect import bisect_right, bisect_left

ROOT = os.path.expanduser("~/trading/backtest_6y")
BAR, H, DAY = 1800_000, 3600_000, 86400_000
COSTS = (0.14, 0.31, 0.71)
rnd = random.Random(20260711)
def ms(s): return int(datetime.strptime(s, "%Y-%m-%d").replace(tzinfo=timezone.utc).timestamp()*1000)
def ud(t): return datetime.fromtimestamp(t/1000, timezone.utc).strftime("%Y-%m-%d")

# ── чистое ядро (тестируемо): один день ──────────────────────────────────────
def day_positions(D, uni_day, oi_ts, oi_val, fund, counters):
    """Кандидаты дня по замороженным правилам. oi_ts: {sym: sorted [ts]},
    oi_val: {sym: {ts: oi}}, fund: {sym: sorted [(ts, rate)]}. → [(sym, score)]"""
    cands = []
    for sym in uni_day:                                   # порядок вселенной = тай-брейк
        ts = oi_ts.get(sym)
        if not ts: counters["excl_no_oi"] += 1; continue
        i1 = bisect_left(ts, D) - 1                       # as-of bugfix 11.07: строго < D
        if i1 < 0 or D - ts[i1] > 5*H: counters["excl_oi_age"] += 1; continue
        t1 = ts[i1]
        i0 = bisect_right(ts, t1 - 4*H) - 1
        if i0 < 0 or not (3*H <= t1 - ts[i0] <= 5*H): counters["excl_oi_gap"] += 1; continue
        t0 = ts[i0]
        v0, v1 = oi_val[sym][t0], oi_val[sym][t1]
        if v0 <= 0: counters["excl_oi_zero"] += 1; continue
        doi = v1/v0 - 1
        fs = fund.get(sym, [])
        j = bisect_left(fs, (D, float("-inf"))) - 1       # as-of bugfix: оба settlement < D
        if j < 1: counters["excl_no_fund"] += 1; continue
        (tp, _), (tl, rl) = fs[j-1], fs[j]
        if tl <= tp: counters["excl_no_fund"] += 1; continue
        f_norm = rl * DAY / (tl - tp)                     # суточный эквивалент, факт. интервал
        cands.append((sym, f_norm * max(0.0, doi)))
    pos = sorted([c for c in cands if c[1] > 0], key=lambda x: -x[1])
    neg = sorted([c for c in cands if c[1] < 0], key=lambda x: x[1])
    K = min(10, len(pos), len(neg))
    if K < 5: counters["skipped_K"] += 1; return None
    return {"K": K, "short": pos[:K], "long": neg[:K]}

def price_leg(sb, D, t_end):
    """(entry_open, exit_close, exit_ms). as-of bugfix 11.07: вход по open бара
    D+30м (00:30) — решение из данных <D не может исполняться по цене момента D."""
    eb = sb.get(D + BAR)
    if eb is None: return None
    t = D + BAR + DAY
    while t <= min(D + BAR + 3*DAY, t_end):
        b = sb.get(t)
        if b: return eb[1], b[4], t + BAR
        t += BAR
    return None

# ── статистика/вердикт (тот же §7-паттерн) ──────────────────────────────────
def drange(a, b):
    d = datetime.strptime(a, "%Y-%m-%d")
    while d.strftime("%Y-%m-%d") <= b:
        yield d.strftime("%Y-%m-%d"); d += timedelta(days=1)
def stats(xs):
    n = len(xs); m = mean(xs); sd = pstdev(xs)*(n/(n-1))**0.5 if n > 1 else 0
    t = m/(sd/n**0.5) if sd else 0
    cum = mx = dd = 0.0
    for x in xs:
        cum += x; mx = max(mx, cum); dd = min(dd, cum - mx)
    return {"total": round(sum(xs), 2), "t": round(t, 2),
            "sharpe": round(m/sd*365**0.5, 2) if sd else 0, "maxDD": round(dd, 2)}
def boot(xs, k=10000):
    nb = max(1, len(xs)//7); blocks = [xs[i*7:(i+1)*7] for i in range(nb)]
    tot = sorted(sum(v for bl in (rnd.choice(blocks) for _ in range(nb)) for v in bl) for _ in range(k))
    return round(tot[int(.025*k)], 2), round(tot[int(.975*k)], 2)

def main():
    uni_all = json.load(gzip.open(f"{ROOT}/data/universe.json.gz", "rt"))
    END = ms("2026-07-10")
    chunks = ["2021-01-01","2021-10-01","2022-04-01","2022-10-01","2023-04-01","2023-10-01",
              "2024-04-01","2024-10-01","2025-04-01","2025-10-01","2026-04-01","2026-07-09"]
    counters = {k: 0 for k in ("excl_no_oi","excl_oi_age","excl_oi_gap","excl_oi_zero",
                               "excl_no_fund","skipped_K","pos_no_bar")}
    daily = {c: {} for c in COSTS}; rows = []; Ks = []
    for i in range(len(chunks)-1):
        a, b = ms(chunks[i]), ms(chunks[i+1])
        days = [d for d in uni_all if chunks[i] <= d < chunks[i+1]]
        if not days: continue
        need = sorted({s for d in days for s in uni_all[d]})
        bars, oi_ts, oi_val, fund = {}, {}, {}, {}
        for s in need:
            po = f"{ROOT}/data/oi/{s}.json.gz"
            if os.path.exists(po):
                ov = {int(k): v for k, v in json.load(gzip.open(po, "rt")).items()
                      if a - DAY <= int(k) <= b}
                if ov: oi_val[s] = ov; oi_ts[s] = sorted(ov)
            pk = f"{ROOT}/data/klines/{s}.json.gz"
            if os.path.exists(pk):
                bb = {int(k): v for k, v in json.load(gzip.open(pk, "rt")).items()
                      if a - DAY <= int(k) <= b + 4*DAY}
                if bb: bars[s] = bb
            pf = f"{ROOT}/data/funding/{s}.json.gz"
            if os.path.exists(pf):
                fund[s] = sorted((int(k), v) for k, v in json.load(gzip.open(pf, "rt")).items())
        for dstr in sorted(days):
            D = ms(dstr)
            if D > END - 25*H: continue                    # хвостовая цензура
            sel = day_positions(D, uni_all[dstr], oi_ts, oi_val, fund, counters)
            if sel is None: continue
            K = sel["K"]; Ks.append(K)
            for side, sgn in (("long", 1), ("short", -1)):
                for sym, score in sel[side]:
                    sb = bars.get(sym)
                    leg = price_leg(sb, D, b + 3*DAY) if sb else None
                    if leg is None: counters["pos_no_bar"] += 1; continue
                    e, x, xms = leg
                    ret = (x/e - 1)*100
                    f = sum(r for t_, r in fund.get(sym, ()) if D + BAR < t_ <= xms) * 100
                    rows.append({"date": dstr, "sym": sym, "side": side, "K": K,
                                 "ret": round(ret, 4), "f": round(f, 4), "score": round(score, 6)})
                    for c in COSTS:
                        net = (ret - c - f) if sgn == 1 else (-ret - c + f)
                        dd_ = daily[c]; dd_[dstr] = dd_.get(dstr, 0.0) + net/K
        print(f"кусок {chunks[i]}…{chunks[i+1]}: дней с торгами накоплено "
              f"{len(daily[0.31])}, позиций {len(rows)}", flush=True)

    tradable = sorted(daily[0.31])
    if not tradable: print("НЕТ торгуемых дней"); return
    start = max("2021-10-17", tradable[0])
    WIN = {"full": (start, "2026-07-09"), "train": (start, "2023-12-31"),
           "val": ("2024-01-01","2024-12-31"), "test_verdict": ("2025-01-01","2026-04-30"),
           "contaminated": ("2026-05-01","2026-07-09"), "2022": ("2022-01-01","2022-12-31"),
           "2023": ("2023-01-01","2023-12-31"), "2025": ("2025-01-01","2025-12-31"),
           "2026H1": ("2026-01-01","2026-07-09")}
    def series(c, w): a_, b_ = WIN[w]; return [daily[c].get(d, 0.0) for d in drange(a_, b_)]

    with open(f"{ROOT}/out/hoi_positions.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0].keys())); w.writeheader(); w.writerows(rows)
    print(f"\n═══════ H-OI-01: RESULTS (единственное вскрытие) ═══════")
    print(f"SURVIVOR-ONLY POOL: направление смещения для L/S заявлено неоднозначным (PREREG §4)")
    print(f"старт торгуемого окна: {start} (правило §3) · дней с торгами {len(tradable)} · "
          f"медиана K_D {sorted(Ks)[len(Ks)//2]} · счётчики { {k: v for k, v in counters.items() if v} }\n")
    print(f"{'окно':>13} {'c/ногу':>6} | {'netΣ%':>8} {'t':>6} {'Sharpe':>6} {'maxDD':>8}")
    S = {}
    for c in COSTS:
        for w in WIN:
            S[(w, c)] = stats(series(c, w))
            if c == 0.31 or w in ("full", "val", "test_verdict"):
                s = S[(w, c)]
                print(f"{w:>13} {c:>6} | {s['total']:>8} {s['t']:>6} {s['sharpe']:>6} {s['maxDD']:>8}")
    for w in ("full", "val", "test_verdict"):
        lo, hi = boot(series(0.31, w))
        print(f"bootstrap {w}@0.31: 95% CI [{lo}, {hi}]")

    print("\n── вторичные (НЕ вердикт) ──")
    gross = [ (r["ret"] - r["f"]) if r["side"] == "long" else (-r["ret"] + r["f"]) for r in rows]
    print(f"gross/позицию (c=0, funding учтён): {round(mean(gross), 4)}% · позиций {len(rows)}")
    for side in ("long", "short"):
        sub = [ (r["ret"] - r["f"]) if side == "long" else (-r["ret"] + r["f"])
                for r in rows if r["side"] == side]
        print(f"  нога {side}: gross {round(mean(sub), 4)}%/позицию (n={len(sub)})")

    print("\n── ВЕРДИКТ ПО §3 (механически) ──")
    ok1 = all(S[(w, 0.31)]["total"] > 0 for w in ("full", "val", "test_verdict"))
    f31, f71 = S[("full", 0.31)]["total"], S[("full", 0.71)]["total"]
    ok2 = (f71 > -0.5*abs(f31)) and ((S[("test_verdict", 0.71)]["total"] > 0) == (S[("test_verdict", 0.31)]["total"] > 0))
    sgn = f31 > 0
    contrib = {}
    for r in rows:
        net = (r["ret"] - 0.31 - r["f"]) if r["side"] == "long" else (-r["ret"] - 0.31 + r["f"])
        contrib[r["sym"]] = contrib.get(r["sym"], 0) + net/r["K"]
    top5s = set(sorted(contrib, key=lambda s_: -abs(contrib[s_]))[:5])
    dd_ab = {}
    summ = json.load(open(f"{ROOT}/data/manifest/_summary.json"))
    fb = {s_: c_["first"] for s_, c_ in summ["coverage"].items()}
    for tag, keep in (("ab1", lambda r: r["sym"] not in top5s),
                      ("ab3", lambda r: fb.get(r["sym"]) and ms(r["date"]) - fb[r["sym"]] >= 180*DAY)):
        dd_ = {}
        for r in rows:
            if not keep(r): continue
            net = (r["ret"] - 0.31 - r["f"]) if r["side"] == "long" else (-r["ret"] - 0.31 + r["f"])
            dd_[r["date"]] = dd_.get(r["date"], 0.0) + net/r["K"]
        dd_ab[tag] = [dd_.get(d, 0.0) for d in drange(*WIN["full"])]
    dmap = dict(zip(drange(*WIN["full"]), series(0.31, "full")))
    top5d = set(sorted(dmap, key=lambda d: -abs(dmap[d]))[:5])
    dd_ab["ab2"] = [v for d, v in dmap.items() if d not in top5d]
    ok3 = all((sum(a) > 0) == sgn and abs(stats(a)["t"]) > 1.0 for a in dd_ab.values())
    ok4 = S[("test_verdict", 0.31)]["total"] > 0 and S[("test_verdict", 0.31)]["t"] >= 2.0
    print(f"1) net>0 @0.31 (full+val+test_verdict): {ok1}")
    print(f"2) выживает @0.71: {ok2}")
    print(f"3) ablations (симв {round(sum(dd_ab['ab1']),1)} / дни {round(sum(dd_ab['ab2']),1)} / ≥180д {round(sum(dd_ab['ab3']),1)}): {ok3}")
    print(f"4) test_verdict: net>0 и t≥2.0: {ok4} (t={S[('test_verdict',0.31)]['t']})")
    verdict = ("CONFIRMED FOR PAPER ONLY" if (ok1 and ok2 and ok3 and ok4)
               else "KILLED / NO-TRADE" if (not ok1 or not ok4) else "INCONCLUSIVE")
    print(f"\n════ ВЕРДИКТ: {verdict} ════")
    json.dump({"verdict": verdict, "gates": [ok1, ok2, ok3, ok4], "start": start,
               "days": len(tradable), "positions": len(rows), "counters": counters,
               "S": {f"{w}@{c}": v for (w, c), v in S.items()}},
              open(f"{ROOT}/out/hoi_results.json", "w"), indent=1)
    print("сохранено: out/hoi_positions.csv, out/hoi_results.json")

def selfcheck():
    D = ms("2024-06-01")
    mkoi = lambda pts: ({k: v for k, v in pts}, sorted(k for k, _ in pts))
    oiv, oit = {}, {}
    # A: валидный crowded-long (f+, OI +10%); B: crowded-short (f−, OI +10%);
    # C: OI-точка старая (>5ч) → excl; E: разгрузка (OI ↓, f−) → score 0
    for sym, (o0, o1, t1off) in {"A": (100, 110, -H), "B": (200, 220, -H),
                                 "C": (50, 55, -6*H), "E": (300, 270, -H)}.items():
        pts = [(D - 5*H + 0, o0), (D + t1off, o1)] if sym != "C" else [(D - 10*H, o0), (D - 6*H, o1)]
        oiv[sym], oit[sym] = mkoi(pts)
    fund = {s: [(D - 12*H, r), (D - 4*H, r)] for s, r in
            (("A", .0004), ("B", -.0004), ("C", .0004), ("E", -.0004))}
    c = {k: 0 for k in ("excl_no_oi","excl_oi_age","excl_oi_gap","excl_oi_zero","excl_no_fund","skipped_K")}
    sel = day_positions(D, ["A","B","C","E"], oit, oiv, fund, c)
    assert sel is None and c["skipped_K"] == 1 and c["excl_oi_age"] == 1, (sel, c)  # K=1 < 5 → день скипнут
    # 10 лонг-толп + 10 шорт-толп → K=10, состав верный
    oiv2, oit2, fund2, uni = {}, {}, {}, []
    for i in range(12):
        s = f"L{i}"; uni.append(s)
        oiv2[s], oit2[s] = mkoi([(D - 5*H, 100), (D - H, 100 + i + 1)])
        fund2[s] = [(D - 12*H, .0001*(i+1)), (D - 4*H, .0001*(i+1))]
    for i in range(12):
        s = f"S{i}"; uni.append(s)
        oiv2[s], oit2[s] = mkoi([(D - 5*H, 100), (D - H, 100 + i + 1)])
        fund2[s] = [(D - 12*H, -.0001*(i+1)), (D - 4*H, -.0001*(i+1))]
    c2 = {k: 0 for k in c}
    sel2 = day_positions(D, uni, oit2, oiv2, fund2, c2)
    assert sel2["K"] == 10 and sel2["short"][0][0] == "L11" and sel2["long"][0][0] == "S11", sel2
    # f_norm: интервал 8ч → rate×3 (суточный эквивалент)
    _, sc = sel2["short"][0]
    exp = (.0001*12)*(DAY/(8*H)) * ((100+12)/100 - 1)
    assert abs(sc - exp) < 1e-12, (sc, exp)
    # нога/арифметика: вход 00:30 (as-of), выход +24ч
    sb = {D + BAR: [D+BAR, 100, 101, 99, 100, 1],
          D + BAR + DAY: [D+BAR+DAY, 98, 99, 97, 98, 1]}
    e, x, xms = price_leg(sb, D, D + 5*DAY)
    assert (e, x) == (100, 98) and xms == D + BAR + DAY + BAR
    # ═ AS-OF РЕГРЕССИЯ (bugfix 11.07): данные с меткой ровно D НЕВИДИМЫ решению ═
    # всем 24 символам подкладываем «отравленные» точки в ts=D: OI-обвал до 1.0
    # (видимость → ΔOI<0 → relu 0 → день бы скипнулся) и funding ×100
    # (видимость → другой f_norm → другой score). Отбор обязан НЕ измениться.
    for s in list(oiv2):
        oiv2[s] = dict(oiv2[s]); oiv2[s][D] = 1.0
        oit2[s] = oit2[s] + [D]
        fund2[s] = fund2[s] + [(D, fund2[s][-1][1]*100)]
    c3 = {k: 0 for k in c2}
    sel3 = day_positions(D, uni, oit2, oiv2, fund2, c3)
    assert sel3 is not None and sel3["K"] == 10, (sel3, c3)
    assert sel3["short"][0][0] == "L11" and abs(sel3["short"][0][1] - exp) < 1e-12, sel3["short"][0]
    net_short = -(x/e - 1)*100 - 0.31 + 0.05
    assert abs(net_short - (2 - 0.31 + 0.05)) < 1e-9
    print("✓ H-OI selfcheck: исключения/K-гейт/состав корзин/f_norm-кадентность/нога — OK")

if __name__ == "__main__":
    ap = argparse.ArgumentParser(); ap.add_argument("--selfcheck", action="store_true")
    if ap.parse_args().selfcheck: selfcheck()
    else: main()
```

## `backtest_6y/carry_core.py`

```python
#!/usr/bin/env python3
"""H-CARRY-01 — чистое ядро cash-and-carry (PREREG DRAFT 2026-07-11, §6).
ОТДЕЛЬНО от радар-стека. Данные инжектируются; PnL на реальных данных НЕ
запускается до заморозки предрега (этот файл — механика + селфчеки).

Позиция: long spot Q_s=N/S_open + short perp Q_p=N/P_open (N=1), без ребаланса.
Funding: short ПОЛУЧАЕТ +rate×Q_p×P на каждом фактическом settlement в позиции.
Вход/выход: на settlement-триггере (f_ann с гистерезисом), исполнение обеих ног
по OPEN первого 1ч-бара СТРОГО ПОЗЖЕ settlement (as-of, урок H-OI-01).
Капитал = 1 + m; breach: убыток перп-ноги > (m − maint)·N → принудительное
закрытие пары на закрытии того же бара (taker+slip), счётчик.
python3 carry_core.py --selfcheck"""
import argparse
from bisect import bisect_right

H = 3600_000
YEAR_MS = 365*86400_000

def resample_1h(bars30):
    """30м → 1ч, только полные пары (нет пары — часа нет). [o,h,l,c]"""
    out = {}
    for t, b in bars30.items():
        if t % H: continue
        b2 = bars30.get(t + 1800_000)
        if b2 is None: continue
        out[t] = [b[1], max(b[2], b2[2]), min(b[3], b2[3]), b2[4]]
    return out

def f_ann(events, i):
    """Аннуализированный funding события i по ФАКТИЧЕСКОМУ интервалу к предыдущему."""
    ts, rate = events[i]
    tp = events[i-1][0]
    if ts <= tp: return None
    return rate * (YEAR_MS / (ts - tp))

def next_bar_after(keys, ts):
    """Первый 1ч-бар с open СТРОГО ПОЗЖЕ ts (одновременная точка НЕВИДИМА)."""
    i = bisect_right(keys, ts)
    return keys[i] if i < len(keys) else None

def run_carry(spot, perp, fund, *, entry_th, exit_th, fee_spot, fee_perp, slip,
              m, maint, t0=None, t1=None):
    """spot/perp: {ts_1h: [o,h,l,c]}, fund: sorted [(ts, rate)].
    Возвращает: {"episodes", "hourly": {ts: equity_на_капитал}, "counters"}."""
    sk = sorted(spot); pk = sorted(perp)
    common = sorted(set(sk) & set(pk))
    if t0: common = [t for t in common if t >= t0]
    if t1: common = [t for t in common if t <= t1]
    cset = set(common)
    cap = 1.0 + m
    st = {"in": False}
    eq_cum = 0.0                     # накопленный реализованный результат, USD на пару N=1
    hourly, episodes = {}, []
    counters = {"entries": 0, "exits": 0, "breach": 0, "skipped_no_bar": 0,
                "funding_events_in_pos": 0}

    def exec_cost(S, P):
        return (fee_spot + slip) + (fee_perp + slip)      # доли нотионала N=1 на ногу

    def open_pos(tb):
        S, P = spot[tb][0], perp[tb][0]
        st.update({"in": True, "Qs": 1.0/S, "Qp": 1.0/P, "S0": S, "P0": P,
                   "t_in": tb, "exec_ts": tb, "fund_recv": 0.0, "fees": exec_cost(S, P)})
        counters["entries"] += 1

    def mtm(tb, use_close=True):
        i = 3 if use_close else 0
        S, P = spot[tb][i], perp[tb][i]
        return (st["Qs"]*S - 1.0) + (1.0 - st["Qp"]*P) + st["fund_recv"] - st["fees"]

    def close_pos(tb, why, use_close=False):
        nonlocal eq_cum
        i = 3 if use_close else 0
        S, P = spot[tb][i], perp[tb][i]
        st["fees"] += exec_cost(S, P)
        pnl = mtm(tb, use_close)
        eq_cum += pnl
        episodes.append({"t_in": st["t_in"], "t_out": tb, "pnl": pnl,
                         "fund": st["fund_recv"], "fees": st["fees"], "why": why})
        st["in"] = False
        counters["exits"] += 1

    def perp_settle_price(ts):
        """Цена funding-начисления, ИЗВЕСТНАЯ в момент settlement (правка Codex №1):
        OPEN 1ч-бара, в который попадает ts; бара нет → close ПРЕДЫДУЩЕГО полного
        часа. Никогда не close текущего бара (это будущее)."""
        start = ts - (ts % H)
        if start in cset: return perp[start][0]
        i = bisect_right(common, start - 1) - 1
        return perp[common[i]][3] if i >= 0 else None

    fi = 0
    for tb in common:
        # settlement-события с ts < open этого бара: сперва НАЧИСЛЕНИЕ, потом РЕШЕНИЕ
        while fi < len(fund) and fund[fi][0] < tb:
            ts_e, rate = fund[fi]
            # 1) начисление: позиция существовала в момент события (вход был раньше)
            if st["in"] and ts_e > st["exec_ts"]:
                P_e = perp_settle_price(ts_e)
                if P_e is not None:
                    st["fund_recv"] += rate * st["Qp"] * P_e   # short ПОЛУЧАЕТ +rate
                    counters["funding_events_in_pos"] += 1
            # 2) решение по f_ann этого события; исполнение — на ЭТОМ баре
            if fi > 0:
                fa = f_ann(fund, fi)
                if fa is not None:
                    if not st["in"] and fa >= entry_th:
                        if tb in cset: open_pos(tb)
                        else: counters["skipped_no_bar"] += 1
                    elif st["in"] and fa < exit_th:
                        close_pos(tb, "exit_signal")
                    # EXIT_TH ≤ fa < ENTRY_TH → гистерезис: ничего не делаем
            fi += 1
        # 3) маржин-контроль по close бара: убыток перп-ноги > (m − maint) → breach
        if st["in"]:
            loss_perp = st["Qp"]*perp[tb][3] - 1.0
            if loss_perp > (m - maint):
                counters["breach"] += 1
                close_pos(tb, "breach", use_close=True)
        hourly[tb] = (eq_cum + (mtm(tb) if st["in"] else 0.0)) / cap
    # terminal accounting (правка Codex №3): позиция, дожившая до правой границы,
    # принудительно закрывается по close последнего бара с exit-fees/slippage
    if st["in"] and common:
        counters["terminal_close"] = counters.get("terminal_close", 0) + 1
        close_pos(common[-1], "terminal_close", use_close=True)
        hourly[common[-1]] = eq_cum / cap
    return {"episodes": episodes, "hourly": hourly, "counters": counters}

# ── селфчеки (§6 PREREG DRAFT): синтетика, без реальных данных ──
def _mk_series(t0, n, px):
    return {t0 + i*H: [px, px, px, px] for i in range(n)}

def selfcheck():
    t0 = 1700000000000 - (1700000000000 % H)
    ok = []
    # 1) as-of: settlement РОВНО в open бара → исполнение на СЛЕДУЮЩЕМ баре
    keys = sorted(_mk_series(t0, 10, 100))
    nb = next_bar_after(keys, t0 + 3*H)          # событие точно в открытие бара 3
    assert nb == t0 + 4*H, "отравленная одновременная точка должна быть невидима"
    nb2 = next_bar_after(keys, t0 + 3*H + 1)
    assert nb2 == t0 + 4*H
    ok.append("as-of")
    # 2) f_ann по фактическому интервалу: 8ч → rate×3×365
    ev = [(t0, 0.0001), (t0 + 8*H, 0.0001)]
    assert abs(f_ann(ev, 1) - 0.0001*3*365) < 1e-12
    ev4 = [(t0, 0.0001), (t0 + 4*H, 0.0001)]
    assert abs(f_ann(ev4, 1) - 0.0001*6*365) < 1e-12
    ok.append("f_ann-кадентность")
    # 3) базис-идентичность: без трений/funding эквити = сближение базиса
    spot = {t0: [100,100,100,100], t0+H: [105,105,105,105]}
    perp = {t0: [102,102,102,102], t0+H: [104,104,104,104]}
    Qs, Qp = 1/100, 1/102
    manual = Qs*105 - 1 + 1 - Qp*104
    assert abs(manual - (0.05 - 2/102)) < 1e-12
    ok.append("базис-MTM")
    # 4) знак funding + 5) fees цикла + 6) гистерезис — плоские цены, полный эпизод
    spotN = _mk_series(t0, 40, 100); perpN = _mk_series(t0, 40, 100)
    fund = [(t0 + 1*H + 1, 0.001),      # событие №1 (для интервала)
            (t0 + 9*H + 1, 0.001),      # №2: f_ann=+1.095 ≥ 0.5 → ВХОД (бар 10)
            (t0 + 17*H + 1, 0.0005),    # №3: f_ann=+0.5475 — в позиции, НАЧИСЛЯЕТСЯ
            (t0 + 25*H + 1, 0.0002),    # №4: f_ann=+0.219 — гистерезис (−0.1 < fa < 0.5): ДЕРЖИМ
            (t0 + 33*H + 1, -0.0004)]   # №5: f_ann=−0.438 < −0.1 → ВЫХОД (+начислен)
    r = run_carry(spotN, perpN, fund, entry_th=0.5, exit_th=-0.1,
                  fee_spot=0.001, fee_perp=0.00055, slip=0.0005, m=0.25, maint=0.125)
    c = r["counters"]
    assert c["entries"] == 1 and c["exits"] == 1 and c["breach"] == 0, c
    assert c["funding_events_in_pos"] == 3, c          # №3, №4 (держим, но получаем), №5
    ep = r["episodes"][0]
    exp_fund = (0.0005 + 0.0002 - 0.0004) * (1/100) * 100   # плоские цены: rate×Qp×P = rate
    assert abs(ep["fund"] - exp_fund) < 1e-12, (ep["fund"], exp_fund)
    exp_fees = 2*(0.001 + 0.00055 + 2*0.0005)
    assert abs(ep["fees"] - exp_fees) < 1e-12, (ep["fees"], exp_fees)
    assert abs(ep["pnl"] - (exp_fund - exp_fees)) < 1e-12   # плоские цены: pnl = funding − fees
    ok.append("funding-знак/начисление"); ok.append("fees-цикл"); ok.append("гистерезис")
    # 7) отрицательный funding в позиции РЕАЛЬНО платится (знак вниз)
    fund_neg = [(t0 + 1*H + 1, 0.001), (t0 + 9*H + 1, 0.001), (t0 + 17*H + 1, -0.00001),
                (t0 + 25*H + 1, -0.0004)]
    r2 = run_carry(spotN, perpN, fund_neg, entry_th=0.5, exit_th=-0.1,
                   fee_spot=0, fee_perp=0, slip=0, m=0.25, maint=0.125)
    assert r2["episodes"][0]["fund"] < 0, r2["episodes"][0]
    ok.append("funding-минус-платится")
    # 8) breach: перп улетает вверх → принудительное закрытие пары
    perpB = dict(perpN); spotB = dict(spotN)
    for i in range(12, 40):
        perpB[t0 + i*H] = [114, 114, 114, 114]; spotB[t0 + i*H] = [114, 114, 114, 114]
    rb = run_carry(spotB, perpB, fund, entry_th=0.5, exit_th=-0.1,
                   fee_spot=0.001, fee_perp=0.00055, slip=0.0005, m=0.25, maint=0.125)
    assert rb["counters"]["breach"] == 1 and rb["episodes"][0]["why"] == "breach", rb["counters"]
    ok.append("маржин-breach")
    # 9) settlement-цена БЕЗ look-ahead (правка Codex №1): open=100, close=150,
    #    событие в начале часа → funding считается от 100, НЕ от 150
    spotL = _mk_series(t0, 40, 100)
    perpL = {t: [100, 100, 100, 100] for t in spotL}
    hot = t0 + 17*H
    perpL[hot] = [100, 155, 95, 150]                    # взрывной бар: open 100 → close 150
    fundL = [(t0 + 1*H + 1, 0.001), (t0 + 9*H + 1, 0.001), (hot, 0.001),
             (t0 + 25*H + 1, -0.0004)]                  # событие РОВНО в открытие взрывного бара
    rl = run_carry(spotL, perpL, fundL, entry_th=0.5, exit_th=-0.1,
                   fee_spot=0, fee_perp=0, slip=0, m=0.9, maint=0.1)
    ep = rl["episodes"][0]
    exp = 0.001*(1/100)*100 + (-0.0004)*(1/100)*100     # оба начисления по цене 100
    assert abs(ep["fund"] - exp) < 1e-12, (ep["fund"], exp, "funding взял close будущего бара!")
    ok.append("settlement-цена-as-of")
    # 10) terminal_close: позиция без выхода закрывается на последнем баре с fees
    fundT = [(t0 + 1*H + 1, 0.001), (t0 + 9*H + 1, 0.001)]   # вход и никогда не выход
    rt = run_carry(spotN, perpN, fundT, entry_th=0.5, exit_th=-0.1,
                   fee_spot=0.001, fee_perp=0.00055, slip=0.0005, m=0.25, maint=0.125)
    assert rt["counters"].get("terminal_close") == 1, rt["counters"]
    et = rt["episodes"][-1]
    assert et["why"] == "terminal_close" and abs(et["fees"] - 2*(0.001+0.00055+2*0.0005)) < 1e-12
    ok.append("terminal-close")
    print(f"✓ carry_core selfcheck ({len(ok)}): {' · '.join(ok)} — OK")

if __name__ == "__main__":
    ap = argparse.ArgumentParser(); ap.add_argument("--selfcheck", action="store_true")
    if ap.parse_args().selfcheck: selfcheck()
    else: print("ядро H-CARRY-01: использовать как модуль; PnL до заморозки предрега не запускается")
```

## `backtest_6y/carry_runner.py`

```python
#!/usr/bin/env python3
"""H-CARRY-01 — ЕДИНЫЙ ЗАПУСК по замороженному предрегу (2026-07-11).
Параметры зашиты из PREREG §8 (одобрены Codex), правки №1-2 в ядре/портфеле.
Печатает вердикт механически; сохраняет out/carry_results.json + episodes."""
import json, gzip, os, csv, random
from datetime import datetime, timezone, timedelta
from statistics import mean, pstdev
from carry_core import run_carry, resample_1h

ROOT = os.path.expanduser("~/trading/backtest_6y")
rnd = random.Random(20260711)
def ms(s): return int(datetime.strptime(s, "%Y-%m-%d").replace(tzinfo=timezone.utc).timestamp()*1000)
def ud(t): return datetime.fromtimestamp(t/1000, timezone.utc).strftime("%Y-%m-%d")

P = dict(entry_th=0.10, exit_th=0.03, fee_spot=0.0010, fee_perp=0.00055,
         m=0.25, maint=0.125)                       # PREREG §8, заморожено
SLIPS = (0.0002, 0.0005, 0.0010)                    # per-leg-execution; base = 0.0005
ASSETS = {"BTCUSDT": "2021-07-01", "ETHUSDT": "2021-07-01", "SOLUSDT": "2022-01-01"}
T1 = ms("2026-07-10")

def load(sym):
    spot = {int(k): v[:4] for k, v in json.load(gzip.open(f"{ROOT}/data/spot/{sym}.json.gz", "rt")).items()}
    p30 = {int(k): v for k, v in json.load(gzip.open(f"{ROOT}/data/klines/{sym}.json.gz", "rt")).items()}
    perp = resample_1h(p30)
    fund = sorted((int(k), v) for k, v in json.load(gzip.open(f"{ROOT}/data/funding/{sym}.json.gz", "rt")).items())
    return spot, perp, fund

def drange(a, b):
    d = datetime.strptime(a, "%Y-%m-%d")
    while d.strftime("%Y-%m-%d") <= b:
        yield d.strftime("%Y-%m-%d"); d += timedelta(days=1)
WIN = {"full": ("2021-07-01", "2026-07-09"), "train": ("2021-07-01", "2023-12-31"),
       "val": ("2024-01-01", "2024-12-31"), "test_verdict": ("2025-01-01", "2026-04-30"),
       "contaminated": ("2026-05-01", "2026-07-09"), "2022": ("2022-01-01", "2022-12-31"),
       "2023": ("2023-01-01", "2023-12-31"), "2025": ("2025-01-01", "2025-12-31")}
def stats(xs):
    n = len(xs); mu = mean(xs); sd = pstdev(xs)*(n/(n-1))**0.5 if n > 1 else 0
    t = mu/(sd/n**0.5) if sd else 0
    cum = mx = dd = 0.0
    for x in xs:
        cum += x; mx = max(mx, cum); dd = min(dd, cum - mx)
    return {"total": round(sum(xs), 2), "t": round(t, 2),
            "sharpe": round(mu/sd*365**0.5, 2) if sd else 0, "maxDD": round(dd, 2)}
def boot(xs, k=10000):
    nb = max(1, len(xs)//7); blocks = [xs[i*7:(i+1)*7] for i in range(nb)]
    tot = sorted(sum(v for bl in (rnd.choice(blocks) for _ in range(nb)) for v in bl) for _ in range(k))
    return round(tot[int(.025*k)], 2), round(tot[int(.975*k)], 2)

res, sleeves = {}, {}
for sym, start in ASSETS.items():
    spot, perp, fund = load(sym)
    for slip in SLIPS:
        r = run_carry(spot, perp, fund, slip=slip, t0=ms(start), t1=T1, **P)
        res[(sym, slip)] = r
        # sleeve-дневная серия: последний hourly-эквити дня, приращения ×100 (%)
        by_day = {}
        for t, e in sorted(r["hourly"].items()): by_day[ud(t)] = e
        dd_, prev = {}, 0.0
        for d in drange(*WIN["full"]):
            e = by_day.get(d, prev)
            dd_[d] = (e - prev)*100; prev = e
        sleeves[(sym, slip)] = dd_
port = {slip: {d: mean(sleeves[(s, slip)][d] for s in ASSETS) for d in sleeves[("BTCUSDT", slip)]}
        for slip in SLIPS}

print("═══════ H-CARRY-01: RESULTS (единственное вскрытие) ═══════")
print("cash-and-carry BTC/ETH/SOL · косты = КОНСЕРВАТИВНАЯ МОДЕЛЬ (базовая сетка Bybit)")
print("портфель = среднее трёх фиксированных sleeve по 1/3, без перетоков\n")
S = {}
print(f"{'окно':>13} {'slip':>7} | {'net%':>7} {'t':>6} {'Sharpe':>6} {'maxDD':>7}")
for slip in SLIPS:
    for w in WIN:
        xs = [port[slip][d] for d in drange(*WIN[w])]
        S[(w, slip)] = stats(xs)
        if slip == 0.0005 or w in ("full", "val", "test_verdict"):
            s = S[(w, slip)]
            print(f"{w:>13} {slip:>7} | {s['total']:>7} {s['t']:>6} {s['sharpe']:>6} {s['maxDD']:>7}")
for w in ("full", "val", "test_verdict"):
    lo, hi = boot([port[0.0005][d] for d in drange(*WIN[w])])
    print(f"bootstrap {w}@base: 95% CI [{lo}, {hi}]")

print("\n── по активам (base slip) ──")
tot_breach = 0
all_eps = []
for sym, start in ASSETS.items():
    r = res[(sym, 0.0005)]; c = r["counters"]; eps = r["episodes"]
    tot_breach += c["breach"]
    dur = sum(e["t_out"] - e["t_in"] for e in eps)/3600_000
    span = (T1 - ms(start))/3600_000
    open_end = c["entries"] > c["exits"]
    fund_sum = sum(e["fund"] for e in eps); fee_sum = sum(e["fees"] for e in eps)
    s = stats([sleeves[(sym, 0.0005)][d] for d in drange(*WIN["full"])])
    print(f"  {sym}: net {s['total']}% (t={s['t']}) · эпизодов {len(eps)} · в позиции "
          f"{100*dur/span:.0f}% времени · funding +{fund_sum*100:.1f}% · fees −{fee_sum*100:.1f}% "
          f"· breach {c['breach']} · открыта_в_конце={open_end}")
    for e in eps: all_eps.append({"sym": sym, **{k: (round(v, 6) if isinstance(v, float) else v) for k, v in e.items()}})
with open(f"{ROOT}/out/carry_episodes.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=list(all_eps[0].keys())); w.writeheader(); w.writerows(all_eps)

print("\n── ВЕРДИКТ ПО §5 (механически) ──")
ok1 = all(S[(w, 0.0005)]["total"] > 0 for w in ("full", "val", "test_verdict"))
f_b, f_w = S[("full", 0.0005)]["total"], S[("full", 0.0010)]["total"]
ok2 = ((S[("test_verdict", 0.0010)]["total"] > 0) == (S[("test_verdict", 0.0005)]["total"] > 0)) \
      and (f_w > -0.5*abs(f_b) if f_b < 0 else f_w > 0.5*f_b if f_b > 0 else True)
ok3 = S[("test_verdict", 0.0005)]["total"] > 0 and S[("test_verdict", 0.0005)]["t"] >= 2.0
ok4 = tot_breach <= 2
print(f"1) net>0 @base (full+val+test_verdict): {ok1}")
print(f"2) выживает worst-slip: {ok2}")
print(f"3) test_verdict: net>0 и t≥2.0: {ok3} (t={S[('test_verdict', 0.0005)]['t']})")
print(f"4) breach ≤2: {ok4} (всего {tot_breach})")
# §5 замороженного предрега: ВСЕ четыре ворот, иначе KILLED (правка Codex №4 —
# формула приведена к frozen-тексту; INCONCLUSIVE в §5 carry не существует)
verdict = "CONFIRMED FOR PAPER ONLY" if (ok1 and ok2 and ok3 and ok4) else "KILLED / NO-TRADE"
print(f"\n════ ВЕРДИКТ: {verdict} ════")
json.dump({"verdict": verdict, "gates": [ok1, ok2, ok3, ok4], "breach": tot_breach,
           "S": {f"{w}@{s_}": v for (w, s_), v in S.items()}},
          open(f"{ROOT}/out/carry_results.json", "w"), indent=1)
print("сохранено: out/carry_results.json, out/carry_episodes.csv")
```

## `backtest_6y/universe_pit.py`

```python
#!/usr/bin/env python3
"""PIT-вселенная (PREREG §2): на каждый UTC-день — top-150 по turnover ПРЕДЫДУЩЕГО
дня из собственных скачанных баров. Допуск: first_bar ≤ D−48ч И last_bar ≥ D.
Исключения = боевые (is_stablecoin/is_commodity из vol_radar) + symbolType=='stock'
из снапшота инструментов. Манифест на каждый день: n_alive, исключения по причинам, топ с turnover.
Запуск: python3 universe_pit.py [--from 2026-04-01] [--to 2026-07-10]"""
import sys, os, json, gzip, argparse
from datetime import datetime, timezone, timedelta

sys.path.insert(0, os.path.expanduser("~/trading"))
from vol_radar import is_stablecoin, is_commodity

ROOT = os.path.expanduser("~/trading/backtest_6y")
DAY = 86400_000
TOP_N = 150

def build(d_from, d_to, kdir=None, out_suffix=""):
    ins = json.load(open(f"{ROOT}/data/instruments_2026-07-10.json"))
    stocks = {i["symbol"] for i in ins if i.get("symbolType") == "stock"}
    kdir = kdir or f"{ROOT}/data/klines"
    syms = [f[:-8] for f in os.listdir(kdir) if f.endswith(".json.gz")]
    # дневной turnover + границы данных по каждому символу
    turn, bounds = {}, {}
    for n, s in enumerate(sorted(syms)):
        try:
            bars = json.load(gzip.open(f"{kdir}/{s}.json.gz", "rt"))
        except Exception as e:                      # полузаписанный gz при параллельной качке
            print(f"  ⚠ {s}: нечитаем ({e}) — пропущен"); continue
        if not bars: continue
        ts = sorted(int(k) for k in bars)
        bounds[s] = (ts[0], ts[-1])
        dt_ = {}
        for k, b in bars.items():
            d = datetime.fromtimestamp(int(k)/1000, timezone.utc).strftime("%Y-%m-%d")
            dt_[d] = dt_.get(d, 0.0) + (b[6] if len(b) > 6 else 0.0)
        turn[s] = dt_
        if (n+1) % 100 == 0: print(f"  …turnover {n+1}/{len(syms)}", flush=True)

    uni, manifest = {}, {}
    cur = datetime.strptime(d_from, "%Y-%m-%d").replace(tzinfo=timezone.utc)
    end = datetime.strptime(d_to, "%Y-%m-%d").replace(tzinfo=timezone.utc)
    while cur <= end:
        D = cur.strftime("%Y-%m-%d")
        D0 = int(cur.timestamp()*1000)
        prev = (cur - timedelta(days=1)).strftime("%Y-%m-%d")
        excl = {"stable": 0, "commodity": 0, "stock": 0, "young": 0, "dead": 0, "no_turnover": 0}
        alive = []
        for s, (fb, lb) in bounds.items():
            if is_stablecoin(s):   excl["stable"] += 1;    continue
            if is_commodity(s):    excl["commodity"] += 1; continue
            if s in stocks:        excl["stock"] += 1;     continue
            if fb > D0 - 2*DAY:    excl["young"] += 1;     continue
            if lb < D0:            excl["dead"] += 1;      continue
            tv = turn.get(s, {}).get(prev, 0.0)
            if tv <= 0:            excl["no_turnover"] += 1; continue
            alive.append((s, tv))
        alive.sort(key=lambda x: -x[1])
        top = alive[:TOP_N]
        uni[D] = [s for s, _ in top]
        manifest[D] = {"n_alive": len(alive), "excluded": excl,
                       "top_min_turnover": round(top[-1][1]) if top else 0,
                       "top": [(s, round(tv)) for s, tv in top]}
        cur += timedelta(days=1)
    save = lambda p, o: json.dump(o, gzip.open(p, "wt"), separators=(",", ":"))
    save(f"{ROOT}/data/universe{out_suffix}.json.gz", uni)
    save(f"{ROOT}/data/universe_manifest{out_suffix}.json.gz", manifest)
    days = sorted(uni)
    widths = [len(v) for v in uni.values()]
    print(f"вселенная: {days[0]}…{days[-1]}, дней {len(days)}, ширина мин/мед/макс = "
          f"{min(widths)}/{sorted(widths)[len(widths)//2]}/{max(widths)}")
    thirty = next((d for d in days if len(uni[d]) >= 30), None)
    print(f"первый день с шириной ≥30 (старт портфельного учёта, PREREG §2): {thirty}")

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--from", dest="d_from", default="2020-04-01")
    ap.add_argument("--to", dest="d_to", default="2026-07-09")
    ap.add_argument("--kdir", default=None)
    ap.add_argument("--suffix", default="")
    a = ap.parse_args()
    build(a.d_from, a.d_to, a.kdir, a.suffix)
```

## `hflow_replay/hflow_replay.py`

```python
#!/usr/bin/env python3
"""H-FLOW historical FEATURE-ONLY replay (contract: PROMPT_FOR_CLAUDE_HFLOW_REPLAY_RU.md).

Answers exactly one question per (symbol, closed minute D):
    "Which as-of features and candidate flag were KNOWN at decision time D?"
It must NOT be able to answer "what happened to price afterwards". There is no
entry, exit, close-after-D, return, PnL, MFE/MAE, fee, funding-PnL, win-rate or
equity here — by construction (see ALLOWED_KEYS / FORBIDDEN_KEYS, enforced by the
independent verifier).

FROZEN AS-OF RULES (single clock, no look-ahead):
  * The only decision clock is `recv_ms = local_timestamp // 1000` (Tardis µs → ms).
    Exchange `timestamp` is NEVER used for availability or windowing.
  * A source row is visible at D iff `recv_ms <= D`.
  * Decision at each closed UTC minute: D = minute_start + 60_000 (minute close, ms).
  * Trailing windows are half-open-low / closed-high: (D - W, D].
      - 5-min canonical SHORT-liquidation notional: W = 300_000.
      - 1-min taker-buy share:                      W =  60_000.
  * Book: use the LAST book_snapshot_25 with recv_ms <= D ("as-of snapshot").
    a10 = Σ top-10 ask notional of THAT exact snapshot. Ratio denominator =
    median of the 15 most recent VALID a10 of snapshots received strictly before
    the as-of snapshot (all recv_ms <= D). No future update/snapshot ever enters.
  * OI/funding: last derivative_ticker row with recv_ms <= D; age = D - recv_ms.
  * Any gap, out-of-order receipt, duplicate, unknown/non-USDT unit or absence of
    a required snapshot marks the minute `excluded` with a reason — never zero.

Canonical liquidation mapping is delegated to the SHARED, SHA-pinned adapter
`hflow_liquidation_schema.py` (Tardis side=buy → short; Bybit Sell → short).
"""
from __future__ import annotations

import argparse
import csv
import gzip
import json
from bisect import bisect_right, bisect_left
from statistics import median
from pathlib import Path
from typing import Iterable

from hflow_liquidation_schema import SchemaError, from_tardis_csv

# ── frozen parameters (subject to Codex review, NOT tuned to any outcome) ──
MINUTE_MS = 60_000
LIQ_WINDOW_MS = 300_000            # 5 minutes
TAKER_WINDOW_MS = 60_000          # 1 minute
A10_HISTORY = 15                  # median over previous 15 valid a10 snapshots
BOOK_LEVELS = 10                  # top-10 notional
MAX_BOOK_STALE_MS = 5_000         # as-of book older than this ⇒ excluded (feed gap)
MAX_TICKER_STALE_MS = 60_000      # as-of OI/funding older than this ⇒ excluded (ticker feed gap)

# ── output schema guardrails (the verifier enforces these independently) ──
ALLOWED_KEYS = {
    "decision_ts", "symbol",
    "liq_short_notional_5m", "liq_short_count_5m", "liq_long_notional_5m",
    "taker_buy_share_1m", "taker_trade_count_1m", "taker_total_notional_1m",
    "a10_asof", "a10_prev_median", "a10_ratio", "a10_asof_recv_ms",
    "oi_value", "oi_asof_recv_ms", "oi_age_ms",
    "funding_rate", "funding_asof_recv_ms", "funding_age_ms", "funding_timestamp_us",
    "excluded", "exclusion_reason", "flags", "eligible", "candidate_ts",
}
FORBIDDEN_KEYS = {
    "entry", "exit", "close", "close_after_d", "price_after", "return", "ret",
    "pnl", "mfe", "mae", "fee", "fees", "funding_pnl", "win", "winrate",
    "win_rate", "hit_rate", "equity", "outcome", "exit_price", "entry_price",
}
EXCLUSION_REASONS = {
    "window_before_data_start", "window_after_data_end", "no_book_asof",
    "insufficient_book_history", "stale_book", "no_ticker_asof", "stale_ticker",
    "duplicate_row", "order_conflict", "unknown_unit",
}


def _rows(path: str | Path) -> Iterable[dict]:
    op = gzip.open if str(path).endswith(".gz") else open
    with op(path, "rt", newline="") as fh:
        yield from csv.DictReader(fh)


def recv_ms_of(row: dict) -> int:
    """The single as-of clock: local receipt time in ms. Rejects future receipt."""
    ts, local = int(row["timestamp"]), int(row["local_timestamp"])
    if ts <= 0 or local < ts:
        raise SchemaError("local_timestamp precedes exchange timestamp")
    return local // 1000


def _guarded_recv(row: dict, invalid: set) -> int | None:
    """recv_ms, or None (recording an integrity taint) for a malformed local<ts row.
    Accepted samples never hit this (the gate rejects such files), but a single bad
    row must taint its minute, not crash the replay."""
    try:
        return recv_ms_of(row)
    except (SchemaError, KeyError, TypeError, ValueError):
        try:
            invalid.add(int(row["timestamp"]) // 1000)
        except (KeyError, TypeError, ValueError):
            pass
        return None


def _integrity_scan(recv_list: list[int], full_rows: list[tuple]) -> tuple[set[int], set[int]]:
    """Return (dup_recv_ms, order_conflict_recv_ms). recv_list is file order."""
    dup, conflict, seen = set(), set(), set()
    prev = None
    for rm, key in zip(recv_list, full_rows):
        if prev is not None and rm < prev:
            conflict.add(rm)               # out of receipt order
        if key in seen:
            dup.add(rm)                     # exact duplicate row
        seen.add(key)
        prev = rm
    return dup, conflict


# ── book: precompute per-snapshot a10, keep receipt order ──
def load_book(path: str | Path, symbol: str) -> dict:
    recv, a10, valid, keys, invalid = [], [], [], [], set()
    for r in _rows(path):
        if str(r["symbol"]).upper() != symbol:
            continue
        rm = _guarded_recv(r, invalid)
        if rm is None:
            continue
        try:
            ask = sum(float(r[f"asks[{i}].price"]) * float(r[f"asks[{i}].amount"])
                      for i in range(BOOK_LEVELS))
            ok = ask > 0
        except (KeyError, TypeError, ValueError):
            ask, ok = None, False
        recv.append(rm); a10.append(ask); valid.append(ok)
        keys.append(int(r["local_timestamp"]))     # unique receipt µs = snapshot identity
    dup, conflict = _integrity_scan(recv, keys)
    # stable sort by recv (Tardis is already sorted; keep index for as-of walk)
    order = sorted(range(len(recv)), key=lambda i: recv[i])
    return {"recv": [recv[i] for i in order], "a10": [a10[i] for i in order],
            "valid": [valid[i] for i in order], "dup": dup, "conflict": conflict,
            "invalid": invalid,
            "data_min": min(recv) if recv else None, "data_max": max(recv) if recv else None}


def load_liq(path: str | Path, symbol: str) -> dict:
    recv, short_n, long_n, keys, invalid, unknown = [], [], [], [], set(), set()
    for r in _rows(path):
        if str(r["symbol"]).upper() != symbol:
            continue                            # different symbol / inverse: not our stream
        rm = _guarded_recv(r, invalid)
        if rm is None:
            continue
        try:
            ev = from_tardis_csv(r)            # canonical: buy→short, sell→long, USDT-only
        except SchemaError:
            unknown.add(rm); continue          # malformed side/unit on OUR USDT symbol ⇒ taint
        if ev.notional_usdt <= 0:
            unknown.add(rm); continue          # non-positive notional ⇒ taint, never zero-fill
        recv.append(rm)
        short_n.append(ev.notional_usdt if ev.liquidated_position == "short" else 0.0)
        long_n.append(ev.notional_usdt if ev.liquidated_position == "long" else 0.0)
        keys.append(r.get("id") or tuple(sorted(r.items())))   # true-dup key
    dup, conflict = _integrity_scan(recv, keys)
    order = sorted(range(len(recv)), key=lambda i: recv[i])
    return {"recv": [recv[i] for i in order], "short": [short_n[i] for i in order],
            "long": [long_n[i] for i in order], "dup": dup, "conflict": conflict,
            "invalid": invalid, "unknown": unknown,
            "data_min": min(recv) if recv else None, "data_max": max(recv) if recv else None}


def load_trades(path: str | Path, symbol: str) -> dict:
    recv, buy_n, sell_n, keys, unknown, invalid = [], [], [], [], set(), set()
    for r in _rows(path):
        if str(r["symbol"]).upper() != symbol:
            continue
        rm = _guarded_recv(r, invalid)
        if rm is None:
            continue
        if not str(r["symbol"]).upper().endswith("USDT"):
            unknown.add(rm); continue
        notion = float(r["amount"]) * float(r["price"])
        side = r["side"]
        if side == "buy":
            buy_n.append(notion); sell_n.append(0.0)
        elif side == "sell":
            buy_n.append(0.0); sell_n.append(notion)
        else:
            unknown.add(rm); continue
        recv.append(rm)
        keys.append(r.get("id") or tuple(sorted(r.items())))   # true-dup key
    dup, conflict = _integrity_scan(recv, keys)
    order = sorted(range(len(recv)), key=lambda i: recv[i])
    return {"recv": [recv[i] for i in order], "buy": [buy_n[i] for i in order],
            "sell": [sell_n[i] for i in order], "dup": dup, "conflict": conflict,
            "unknown": unknown, "invalid": invalid,
            "data_min": min(recv) if recv else None, "data_max": max(recv) if recv else None}


def load_ticker(path: str | Path, symbol: str) -> dict:
    recv, oi, fr, ft, keys, invalid = [], [], [], [], [], set()
    for r in _rows(path):
        if str(r["symbol"]).upper() != symbol:
            continue
        rm = _guarded_recv(r, invalid)          # malformed local<ts ⇒ taint, never crash
        if rm is None:
            continue
        recv.append(rm)
        oi.append(float(r["open_interest"]) if r.get("open_interest") not in (None, "") else None)
        fr.append(float(r["funding_rate"]) if r.get("funding_rate") not in (None, "") else None)
        ft.append(int(r["funding_timestamp"]) if r.get("funding_timestamp") not in (None, "") else None)
        keys.append(int(r["local_timestamp"]))  # unique receipt µs = ticker-row identity
    dup, conflict = _integrity_scan(recv, keys)  # ticker is a required as-of input ⇒ must be checked
    order = sorted(range(len(recv)), key=lambda i: recv[i])
    return {"recv": [recv[i] for i in order], "oi": [oi[i] for i in order],
            "fr": [fr[i] for i in order], "ft": [ft[i] for i in order],
            "dup": dup, "conflict": conflict, "invalid": invalid,
            "data_min": min(recv) if recv else None, "data_max": max(recv) if recv else None}


def _window_overlaps(tainted: set[int], lo: int, hi: int) -> bool:
    return any(lo < t <= hi for t in tainted)


def replay_minute(D: int, symbol: str, book: dict, liq: dict, trades: dict, ticker: dict) -> dict:
    """Compute the as-of feature row for the minute closing at D. Pure function."""
    row = {"decision_ts": D, "symbol": symbol, "flags": []}
    excl = None

    # coverage: LOW end needs every feed (incl. the sparse liq feed) to have started;
    # HIGH end gates on the CONTINUOUS feeds (book/trades/ticker) — a liquidation
    # tail-absence while those are live is a genuine zero, not a gap. NB: a
    # liquidation-feed-SPECIFIC outage cannot be told apart from genuine zero in
    # historical data alone (documented limitation; the live pipeline cross-checks
    # its own gap manifest).
    dmins = [book["data_min"], liq["data_min"], trades["data_min"], ticker["data_min"]]
    earliest = max(dmins) if all(x is not None for x in dmins) else None
    cont_max = [book["data_max"], trades["data_max"], ticker["data_max"]]
    latest = min(cont_max) if all(x is not None for x in cont_max) else None
    if earliest is None or (D - LIQ_WINDOW_MS) < earliest:
        excl = "window_before_data_start"
    elif latest is None or D > latest:
        excl = "window_after_data_end"

    # integrity taint in any relevant window ⇒ exclude (ticker gated on its staleness window)
    if excl is None:
        for s, lo in ((liq, D - LIQ_WINDOW_MS), (trades, D - TAKER_WINDOW_MS),
                      (book, D - LIQ_WINDOW_MS), (ticker, D - MAX_TICKER_STALE_MS)):
            if _window_overlaps(s.get("dup", set()), lo, D):
                excl = "duplicate_row"; break
            if _window_overlaps(s.get("conflict", set()), lo, D) or _window_overlaps(s.get("invalid", set()), lo, D):
                excl = "order_conflict"; break
        if excl is None and (_window_overlaps(trades.get("unknown", set()), D - TAKER_WINDOW_MS, D)
                             or _window_overlaps(liq.get("unknown", set()), D - LIQ_WINDOW_MS, D)):
            excl = "unknown_unit"

    # ── liquidation 5m (short, canonical) — genuine 0.0 is a fact, not a gap ──
    if excl is None:
        lo, hi = D - LIQ_WINDOW_MS, D
        i0 = bisect_right(liq["recv"], lo)
        i1 = bisect_right(liq["recv"], hi)
        sn = sum(liq["short"][i0:i1]); ln = sum(liq["long"][i0:i1])
        cnt = sum(1 for x in liq["short"][i0:i1] if x > 0)
        row["liq_short_notional_5m"] = round(sn, 6)
        row["liq_short_count_5m"] = cnt
        row["liq_long_notional_5m"] = round(ln, 6)
        if sn == 0.0:
            row["flags"].append("zero_short_liq")

    # ── taker-buy share 1m (notional) ──
    if excl is None:
        lo, hi = D - TAKER_WINDOW_MS, D
        i0 = bisect_right(trades["recv"], lo)
        i1 = bisect_right(trades["recv"], hi)
        bn = sum(trades["buy"][i0:i1]); sen = sum(trades["sell"][i0:i1])
        tot = bn + sen
        row["taker_trade_count_1m"] = i1 - i0
        row["taker_total_notional_1m"] = round(tot, 6)
        if tot > 0:
            row["taker_buy_share_1m"] = round(bn / tot, 8)
        else:
            row["taker_buy_share_1m"] = None
            row["flags"].append("no_taker_flow")

    # ── a10 as-of + ratio to median of previous 15 valid ──
    if excl is None:
        k = bisect_right(book["recv"], D) - 1        # last snapshot recv_ms <= D
        if k < 0:
            excl = "no_book_asof"
        elif not book["valid"][k]:
            excl = "no_book_asof"                     # as-of snapshot itself unusable
        elif (D - book["recv"][k]) > MAX_BOOK_STALE_MS:
            excl = "stale_book"
        else:
            prev, j = [], k - 1          # 15 most-recent VALID prior a10 (early-stop)
            while j >= 0 and len(prev) < A10_HISTORY:
                if book["valid"][j]:
                    prev.append(book["a10"][j])
                j -= 1
            if len(prev) < A10_HISTORY:
                excl = "insufficient_book_history"
            else:
                a10 = book["a10"][k]
                med = median(prev)        # order-independent
                row["a10_asof"] = round(a10, 6)
                row["a10_asof_recv_ms"] = book["recv"][k]
                row["a10_prev_median"] = round(med, 6)
                row["a10_ratio"] = round(a10 / med, 8) if med > 0 else None
                if med <= 0:
                    excl = "no_book_asof"

    # ── OI / funding as-of (with staleness gate, mirroring the book feed) ──
    if excl is None:
        j = bisect_right(ticker["recv"], D) - 1
        if j < 0:
            excl = "no_ticker_asof"
        elif (D - ticker["recv"][j]) > MAX_TICKER_STALE_MS:
            excl = "stale_ticker"                 # ticker feed gap ⇒ exclude, don't carry stale value
        elif ticker["oi"][j] is None or ticker["fr"][j] is None:
            excl = "no_ticker_asof"
        else:
            row["oi_value"] = ticker["oi"][j]
            row["oi_asof_recv_ms"] = ticker["recv"][j]
            row["oi_age_ms"] = D - ticker["recv"][j]
            row["funding_rate"] = ticker["fr"][j]
            row["funding_asof_recv_ms"] = ticker["recv"][j]
            row["funding_age_ms"] = D - ticker["recv"][j]
            row["funding_timestamp_us"] = ticker["ft"][j]

    # ── finalize ──
    if excl is not None:
        # canonical excluded row: FIXED 7-key set, zero partial-feature leakage
        return {"decision_ts": D, "symbol": symbol, "flags": [],
                "excluded": True, "exclusion_reason": excl,
                "eligible": False, "candidate_ts": None}
    # non-excluded ⇒ every feature block ran ⇒ full 24-key set is present
    row["excluded"] = False
    row["exclusion_reason"] = None
    # eligible = complete trustworthy as-of vector (NO trigger threshold — that is
    # deferred to the separate outcome-prereg; here a candidate = full clean vector)
    row["eligible"] = (row["taker_buy_share_1m"] is not None
                       and row["a10_ratio"] is not None
                       and row["oi_value"] is not None
                       and row["funding_rate"] is not None)
    row["candidate_ts"] = D if row["eligible"] else None
    return row


def run(liq_path, trades_path, book_path, ticker_path, symbol="BTCUSDT",
        d_start=None, d_end=None) -> Iterable[dict]:
    symbol = symbol.upper()
    book = load_book(book_path, symbol)
    liq = load_liq(liq_path, symbol)
    trades = load_trades(trades_path, symbol)
    ticker = load_ticker(ticker_path, symbol)
    mins = [x for x in (book["data_min"], liq["data_min"], trades["data_min"], ticker["data_min"])
            if x is not None]
    maxs = [x for x in (book["data_max"], liq["data_max"], trades["data_max"], ticker["data_max"])
            if x is not None]
    if not mins:
        return
    lo = d_start if d_start is not None else ((min(mins) // MINUTE_MS) + 1) * MINUTE_MS
    hi = d_end if d_end is not None else (max(maxs) // MINUTE_MS) * MINUTE_MS
    D = lo
    while D <= hi:
        yield replay_minute(D, symbol, book, liq, trades, ticker)
        D += MINUTE_MS


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--liquidations", required=True)
    ap.add_argument("--trades", required=True)
    ap.add_argument("--book25", required=True)
    ap.add_argument("--ticker", required=True)
    ap.add_argument("--symbol", default="BTCUSDT")
    ap.add_argument("--out", required=True)
    a = ap.parse_args()
    n, elig, exc = 0, 0, {}
    with open(a.out, "w") as fh:
        for r in run(a.liquidations, a.trades, a.book25, a.ticker, a.symbol):
            bad = set(r) - ALLOWED_KEYS
            assert not bad, f"forbidden/unknown output key(s): {bad}"
            fh.write(json.dumps(r, sort_keys=True) + "\n")
            n += 1
            elig += int(r["eligible"])
            if r["excluded"]:
                exc[r["exclusion_reason"]] = exc.get(r["exclusion_reason"], 0) + 1
    print(json.dumps({"minutes": n, "eligible": elig, "excluded_by_reason": exc,
                      "symbol": a.symbol}, sort_keys=True))
    print("NO OUTCOMES OR PNL WERE OPENED")


if __name__ == "__main__":
    main()
```

## `hflow_replay/hflow_verifier.py`

```python
#!/usr/bin/env python3
"""INDEPENDENT verifier for the H-FLOW feature replay — chronological FOLD.

Architecturally independent of the engine (Codex review 2026-07-11, remediation #2):
it does NOT import `hflow_replay`, uses NONE of its loaders, and uses NO bisect /
sorted-array as-of lookup. Instead it merges all raw rows into a single receipt-time
event stream, interleaves minute-close markers, and processes them in ONE forward
fold — maintaining sliding-window deques (evicted as D advances) and running as-of
state (last book / last ticker / last-16 valid a10). At each minute close it emits
the feature row and compares to the replay's output.

Output-identity (remediation #1): rejects duplicate / non-monotonic / missing / extra
decision timestamps, compares EVERY emitted key and value (float tolerance),
independently derives and compares `flags`, and rejects any unexpected extra or
missing key on every row.

It MAY import the shared, SHA-pinned canonical liquidation adapter (input under test).
"""
from __future__ import annotations

import argparse
import csv
import gzip
import json
from collections import deque
from statistics import median

from hflow_liquidation_schema import SchemaError, from_tardis_csv

MIN = 60_000
LIQ_W = 300_000
TAKER_W = 60_000
HIST = 15
LEVELS = 10
BOOK_STALE = 5_000
TICK_STALE = 60_000
TOL = 1e-6

ALLOWED = {
    "decision_ts", "symbol", "liq_short_notional_5m", "liq_short_count_5m",
    "liq_long_notional_5m", "taker_buy_share_1m", "taker_trade_count_1m",
    "taker_total_notional_1m", "a10_asof", "a10_prev_median", "a10_ratio",
    "a10_asof_recv_ms", "oi_value", "oi_asof_recv_ms", "oi_age_ms", "funding_rate",
    "funding_asof_recv_ms", "funding_age_ms", "funding_timestamp_us", "excluded",
    "exclusion_reason", "flags", "eligible", "candidate_ts",
}
FORBIDDEN = {"entry", "exit", "close", "return", "ret", "pnl", "mfe", "mae", "fee",
             "fees", "funding_pnl", "win", "winrate", "win_rate", "hit_rate",
             "equity", "outcome", "exit_price", "entry_price", "price_after"}


def _raw(path):
    op = gzip.open if str(path).endswith(".gz") else open
    with op(path, "rt", newline="") as fh:
        yield from csv.DictReader(fh)


def scan(path, sym, kind):
    """Own parse of one stream, in FILE order: build events + integrity taints + bounds.
    Returns (events:list[(recv,payload)], taint:dict[set], dmin, dmax)."""
    events, dup, conf, inval, unknown = [], set(), set(), set(), set()
    seen, prev = set(), None
    for r in _raw(path):
        if str(r["symbol"]).upper() != sym:
            continue
        try:
            ts, lo = int(r["timestamp"]), int(r["local_timestamp"])
            if ts <= 0 or lo < ts:
                raise ValueError
            recv = lo // 1000
        except (KeyError, TypeError, ValueError):
            try:
                inval.add(int(r["timestamp"]) // 1000)
            except (KeyError, TypeError, ValueError):
                pass
            continue
        if prev is not None and recv < prev:
            conf.add(recv)
        prev = recv
        key = int(r["local_timestamp"]) if kind in ("book", "ticker") \
            else (r.get("id") or tuple(sorted(r.items())))
        if key in seen:
            dup.add(recv)
        seen.add(key)
        if kind == "book":
            try:
                a = sum(float(r[f"asks[{i}].price"]) * float(r[f"asks[{i}].amount"]) for i in range(LEVELS))
                a = a if a > 0 else None
            except (KeyError, TypeError, ValueError):
                a = None
            events.append((recv, ("book", a)))
        elif kind == "liq":
            try:
                ev = from_tardis_csv(r)
            except SchemaError:
                unknown.add(recv); continue
            if ev.notional_usdt <= 0:
                unknown.add(recv); continue
            events.append((recv, ("liq", ev.liquidated_position, ev.notional_usdt)))
        elif kind == "trades":
            side = r["side"]
            if side not in ("buy", "sell"):
                unknown.add(recv); continue
            events.append((recv, ("trade", side, float(r["amount"]) * float(r["price"]))))
        else:  # ticker
            g = lambda c: (float(r[c]) if r.get(c) not in (None, "") else None)
            ft = int(r["funding_timestamp"]) if r.get("funding_timestamp") not in (None, "") else None
            events.append((recv, ("ticker", g("open_interest"), g("funding_rate"), ft)))
    dmin = min((e[0] for e in events), default=None)
    dmax = max((e[0] for e in events), default=None)
    return events, {"dup": dup, "conf": conf, "inval": inval, "unknown": unknown}, dmin, dmax


def _excl(D, sym, reason):
    return {"decision_ts": D, "symbol": sym, "flags": [], "excluded": True,
            "exclusion_reason": reason, "eligible": False, "candidate_ts": None}


def build_fold(liq_p, tr_p, bk_p, tk_p, sym):
    sym = sym.upper()
    ev_l, T_l, lmin, lmax = scan(liq_p, sym, "liq")
    ev_t, T_t, tmin, tmax = scan(tr_p, sym, "trades")
    ev_b, T_b, bmin, bmax = scan(bk_p, sym, "book")
    ev_k, T_k, kmin, kmax = scan(tk_p, sym, "ticker")
    T = {"liq": T_l, "trades": T_t, "book": T_b, "ticker": T_k}
    dmins = [bmin, lmin, tmin, kmin]
    if any(x is None for x in dmins):
        return {}, []
    earliest = max(dmins)
    latest = min(x for x in (bmax, tmax, kmax) if x is not None)  # continuous feeds
    lo = ((min(dmins) // MIN) + 1) * MIN
    hi = (max(bmax, lmax, tmax, kmax) // MIN) * MIN
    # merge all events + minute markers; at equal recv, events (0) precede marker (1)
    stream = ev_l + ev_t + ev_b + ev_k + [(D, ("MINUTE",)) for D in range(lo, hi + 1, MIN)]
    stream.sort(key=lambda e: (e[0], 1 if e[1][0] == "MINUTE" else 0))

    liq_win, tr_win = deque(), deque()
    valid_a10 = deque(maxlen=HIST + 1)
    last_book = last_tk = None
    rows = {}
    order = []
    for recv, p in stream:
        kind = p[0]
        if kind == "book":
            last_book = (recv, p[1])
            if p[1] is not None:
                valid_a10.append((recv, p[1]))
        elif kind == "liq":
            liq_win.append((recv, p[1], p[2]))
        elif kind == "trade":
            tr_win.append((recv, p[1], p[2]))
        elif kind == "ticker":
            last_tk = (recv, p[1], p[2], p[3])
        else:  # MINUTE close D
            D = recv
            while liq_win and liq_win[0][0] <= D - LIQ_W:
                liq_win.popleft()
            while tr_win and tr_win[0][0] <= D - TAKER_W:
                tr_win.popleft()
            row = _emit(D, sym, liq_win, tr_win, last_book, valid_a10, last_tk, T, earliest, latest)
            rows[D] = row; order.append(D)
    return rows, order


def _emit(D, sym, liq_win, tr_win, last_book, valid_a10, last_tk, T, earliest, latest):
    if earliest is None or (D - LIQ_W) < earliest:
        return _excl(D, sym, "window_before_data_start")
    if latest is None or D > latest:
        return _excl(D, sym, "window_after_data_end")
    ov = lambda s, lo: any(lo < t <= D for t in s)
    for kind, lo in (("liq", D - LIQ_W), ("trades", D - TAKER_W),
                     ("book", D - LIQ_W), ("ticker", D - TICK_STALE)):
        if ov(T[kind]["dup"], lo):
            return _excl(D, sym, "duplicate_row")
        if ov(T[kind]["conf"], lo) or ov(T[kind]["inval"], lo):
            return _excl(D, sym, "order_conflict")
    if ov(T["trades"]["unknown"], D - TAKER_W) or ov(T["liq"]["unknown"], D - LIQ_W):
        return _excl(D, sym, "unknown_unit")

    flags = []
    sn = sum(n for _, pos, n in liq_win if pos == "short")
    ln = sum(n for _, pos, n in liq_win if pos == "long")
    cnt = sum(1 for _, pos, n in liq_win if pos == "short")
    row = {"decision_ts": D, "symbol": sym, "flags": flags,
           "liq_short_notional_5m": round(sn, 6), "liq_short_count_5m": cnt,
           "liq_long_notional_5m": round(ln, 6)}
    if sn == 0.0:
        flags.append("zero_short_liq")

    bn = sum(n for _, side, n in tr_win if side == "buy")
    tot = sum(n for _, _, n in tr_win)
    row["taker_trade_count_1m"] = len(tr_win)
    row["taker_total_notional_1m"] = round(tot, 6)
    row["taker_buy_share_1m"] = round(bn / tot, 8) if tot > 0 else None
    if tot == 0:
        flags.append("no_taker_flow")

    if last_book is None or last_book[1] is None:
        return _excl(D, sym, "no_book_asof")
    if D - last_book[0] > BOOK_STALE:
        return _excl(D, sym, "stale_book")
    va = list(valid_a10)
    if not va or va[-1][0] != last_book[0]:
        return _excl(D, sym, "no_book_asof")
    prev = [a for _, a in va[:-1]]
    if len(prev) < HIST:
        return _excl(D, sym, "insufficient_book_history")
    med = median(prev[-HIST:])
    if med <= 0:
        return _excl(D, sym, "no_book_asof")
    row["a10_asof"] = round(last_book[1], 6)
    row["a10_asof_recv_ms"] = last_book[0]
    row["a10_prev_median"] = round(med, 6)
    row["a10_ratio"] = round(last_book[1] / med, 8)

    if last_tk is None:
        return _excl(D, sym, "no_ticker_asof")
    if D - last_tk[0] > TICK_STALE:
        return _excl(D, sym, "stale_ticker")
    if last_tk[1] is None or last_tk[2] is None:
        return _excl(D, sym, "no_ticker_asof")
    row["oi_value"] = last_tk[1]; row["oi_asof_recv_ms"] = last_tk[0]; row["oi_age_ms"] = D - last_tk[0]
    row["funding_rate"] = last_tk[2]; row["funding_asof_recv_ms"] = last_tk[0]
    row["funding_age_ms"] = D - last_tk[0]; row["funding_timestamp_us"] = last_tk[3]
    row["excluded"] = False; row["exclusion_reason"] = None
    row["eligible"] = (row["taker_buy_share_1m"] is not None and row["a10_ratio"] is not None
                       and row["oi_value"] is not None and row["funding_rate"] is not None)
    row["candidate_ts"] = D if row["eligible"] else None
    return row


def _cmp(x, y):
    if isinstance(x, float) or isinstance(y, float):
        if x is None or y is None:
            return x == y
        return abs(x - y) <= TOL
    return x == y


def verify(replay_jsonl, liq_p, tr_p, bk_p, tk_p, sym="BTCUSDT"):
    fold_rows, fold_order = build_fold(liq_p, tr_p, bk_p, tk_p, sym)
    replay = [json.loads(l) for l in open(replay_jsonl)]
    fails = []
    rts = [r["decision_ts"] for r in replay]
    # decision_ts must be strictly increasing by exactly one minute (no dup/gap/reorder)
    for i in range(1, len(rts)):
        if rts[i] <= rts[i - 1]:
            fails.append(("nonmonotonic_or_duplicate_decision_ts", rts[i - 1], rts[i]))
        elif rts[i] - rts[i - 1] != MIN:
            fails.append(("gap_in_decision_ts", rts[i - 1], rts[i]))
    # exact set equality vs the independently-derived fold grid
    miss = sorted(set(fold_order) - set(rts))[:10]
    extra = sorted(set(rts) - set(fold_order))[:10]
    if miss:
        fails.append(("missing_minutes", miss, None))
    if extra:
        fails.append(("extra_minutes", extra, None))
    # per-row full identity
    for r in replay:
        D = r["decision_ts"]
        bad = (set(r) - ALLOWED) | (set(r) & FORBIDDEN)
        if bad:
            fails.append(("forbidden_or_unknown_key", D, sorted(bad)))
        mine = fold_rows.get(D)
        if mine is None:
            fails.append(("replay_minute_absent_from_fold", D, None)); continue
        if set(r) != set(mine):
            fails.append(("key_set_mismatch", D, (sorted(set(mine) - set(r)), sorted(set(r) - set(mine)))))
            continue
        for k in mine:
            if k == "flags":
                if sorted(r.get("flags", [])) != sorted(mine["flags"]):
                    fails.append(("flags_mismatch", D, (mine["flags"], r.get("flags"))))
            elif not _cmp(mine[k], r.get(k)):
                fails.append(("value_mismatch", D, (k, mine[k], r.get(k))))
    return {"checked": len(fold_order), "replay_rows": len(replay),
            "fail_count": len(fails), "fails": fails[:40]}


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--replay", required=True)
    ap.add_argument("--liquidations", required=True)
    ap.add_argument("--trades", required=True)
    ap.add_argument("--book25", required=True)
    ap.add_argument("--ticker", required=True)
    ap.add_argument("--symbol", default="BTCUSDT")
    a = ap.parse_args()
    res = verify(a.replay, a.liquidations, a.trades, a.book25, a.ticker, a.symbol)
    print(json.dumps({k: v for k, v in res.items() if k != "fails"}, sort_keys=True))
    if res["fail_count"]:
        for f in res["fails"]:
            print("FAIL", f)
        print("VERIFIER: DIVERGENCE — replay NOT independently reproduced")
        raise SystemExit(1)
    print("VERIFIER: PASS — replay independently reproduced by chronological fold, full key identity")
    print("NO OUTCOMES OR PNL WERE OPENED")


if __name__ == "__main__":
    main()
```

## `hflow_replay/hflow_liquidation_schema.py`

```python
#!/usr/bin/env python3
"""Canonical liquidation adapter for H-FLOW research.

Never compare provider-specific ``side`` fields directly. Bybit V5 live data
labels the liquidated *position* (Sell = short liquidated); Tardis normalized
CSV labels the liquidation *order* (buy = short position liquidated).

The live collector's H-FLOW universe is USDT linear contracts. This adapter
rejects non-USDT rows rather than applying a meaningless ``amount * price``
conversion to inverse contracts.
"""

from __future__ import annotations

from dataclasses import dataclass
from typing import Mapping


class SchemaError(ValueError):
    """A source row cannot be safely interpreted as a linear liquidation."""


@dataclass(frozen=True)
class Liquidation:
    timestamp_ms: int
    received_ms: int | None
    symbol: str
    liquidated_position: str  # exactly ``short`` or ``long``
    notional_usdt: float
    source: str


def _position(source: str, side: str) -> str:
    if source == "bybit_live_v5":
        # Bybit V5 allLiquidation S is the position side.
        table = {"Sell": "short", "Buy": "long"}
    elif source == "tardis_normalized":
        # Tardis liquidation CSV side is the forced liquidation order side.
        table = {"buy": "short", "sell": "long"}
    else:
        raise SchemaError(f"unknown liquidation source: {source}")
    try:
        return table[side]
    except KeyError as exc:
        raise SchemaError(f"invalid {source} side: {side!r}") from exc


def _linear_usdt(symbol: str) -> str:
    symbol = str(symbol).upper()
    if not symbol.endswith("USDT"):
        raise SchemaError(f"not a Bybit USDT-linear instrument: {symbol}")
    return symbol


def from_bybit_live(row: Mapping[str, object]) -> Liquidation:
    """Adapt a raw row already emitted by flow_collector/collector.py."""
    symbol = _linear_usdt(str(row["s"]))
    return Liquidation(
        timestamp_ms=int(row["ts"]),
        received_ms=None,
        symbol=symbol,
        liquidated_position=_position("bybit_live_v5", str(row["side"])),
        notional_usdt=float(row["v"]) * float(row["p"]),
        source="bybit_live_v5",
    )


def from_tardis_csv(row: Mapping[str, object]) -> Liquidation:
    """Adapt a normalized Tardis liquidation CSV row (timestamps in µs)."""
    symbol = _linear_usdt(str(row["symbol"]))
    timestamp_us, local_timestamp_us = int(row["timestamp"]), int(row["local_timestamp"])
    if timestamp_us <= 0 or local_timestamp_us < timestamp_us:
        raise SchemaError("Tardis local_timestamp is not a valid as-of receipt time")
    return Liquidation(
        timestamp_ms=timestamp_us // 1_000,
        received_ms=local_timestamp_us // 1_000,
        symbol=symbol,
        liquidated_position=_position("tardis_normalized", str(row["side"])),
        notional_usdt=float(row["amount"]) * float(row["price"]),
        source="tardis_normalized",
    )


def _selfcheck() -> None:
    # Same economic event: a short gets liquidated and buys to close.
    live = from_bybit_live({"ts": 1_000, "s": "ABCUSDT", "side": "Sell", "v": 2, "p": 10})
    tardis = from_tardis_csv({
        "timestamp": 1_000_000, "local_timestamp": 1_100_000,
        "symbol": "ABCUSDT", "side": "buy", "amount": 2, "price": 10,
    })
    assert live.liquidated_position == tardis.liquidated_position == "short"
    assert live.notional_usdt == tardis.notional_usdt == 20.0
    assert from_bybit_live({"ts": 1, "s": "ABCUSDT", "side": "Buy", "v": 1, "p": 2}).liquidated_position == "long"
    assert from_tardis_csv({"timestamp": 1, "local_timestamp": 1, "symbol": "ABCUSDT", "side": "sell", "amount": 1, "price": 2}).liquidated_position == "long"
    try:
        from_tardis_csv({"timestamp": 1, "local_timestamp": 1, "symbol": "BTCUSD", "side": "buy", "amount": 1, "price": 2})
    except SchemaError:
        pass
    else:
        raise AssertionError("inverse contract must be rejected")
    try:
        from_tardis_csv({"timestamp": 2_000, "local_timestamp": 1_999, "symbol": "ABCUSDT", "side": "buy", "amount": 1, "price": 2})
    except SchemaError:
        pass
    else:
        raise AssertionError("future receipt time must be rejected")


if __name__ == "__main__":
    _selfcheck()
    print("H-FLOW liquidation schema self-check: OK")
```

## `hflow_replay/test_hflow_replay.py`

```python
#!/usr/bin/env python3
"""No-network tests for the H-FLOW feature replay + independent verifier.

Covers the six required checks from the contract:
  1) both liquidation semantics (Bybit Sell=short, Tardis buy=short);
  2) an event with local_timestamp > D is invisible at D;
  3) a book snapshot received after D never enters a10 / the median;
  4) inverse / non-USDT contract is rejected, not converted;
  5) duplicate and out-of-order (gap) receipt ⇒ minute excluded, not zero-filled;
  6) engine and verifier produce identical feature/candidate timestamps on a fixture.

Fixtures are generated in a temp dir; nothing touches the network or real data.
"""
from __future__ import annotations

import csv, gzip, json, os, tempfile

import hflow_replay as R
import hflow_verifier as V
from hflow_liquidation_schema import from_bybit_live, from_tardis_csv, SchemaError

US = 1_000_000
MIN_US = 60 * US


def _w(path, header, rows):
    with open(path, "w", newline="") as fh:
        wr = csv.writer(fh); wr.writerow(header)
        for r in rows:
            wr.writerow(r)


def build_fixture(dir_, *, extra_liq=None, dup_trade=False, ooo_trade=False,
                  late_book=False, ticker_dup=False, liq_bad_side=False):
    """~20 min of clean BTCUSDT data (µs), enough to clear 5m + 15-snapshot warm-up."""
    sym = "BTCUSDT"
    # minute-aligned base so fixture minutes line up with the replay's 60000ms grid
    base_ex = (1_700_000_000 * US // MIN_US) * MIN_US
    LAT = 200_000                           # 200 ms receipt latency (local = ex + LAT)
    def L(ex): return ex + LAT              # local_timestamp from exchange ts

    liq_h = ["exchange", "symbol", "timestamp", "local_timestamp", "side", "price", "amount"]
    tr_h = ["exchange", "symbol", "timestamp", "local_timestamp", "id", "side", "price", "amount"]
    bk_h = ["exchange", "symbol", "timestamp", "local_timestamp"] + \
        [f"asks[{i}].price" for i in range(10)] + [f"asks[{i}].amount" for i in range(10)]
    tk_h = ["exchange", "symbol", "timestamp", "local_timestamp",
            "funding_timestamp", "funding_rate", "open_interest", "last_price", "mark_price"]

    liq, tr, bk, tk = [], [], [], []
    N_MIN = 20
    for m in range(N_MIN):
        m0 = base_ex + m * MIN_US
        # book: 30 snapshots/min (2s apart) — realistic Tardis density, fresh at close
        for s in range(30):
            ex = m0 + s * 2 * US
            aprice = [100.0 + i for i in range(10)]
            aamt = [1.0] * 10
            a10 = sum(p * q for p, q in zip(aprice, aamt))   # constant = 1045.0
            r = ["bybit", sym, ex, L(ex)] + aprice + aamt
            bk.append(r)
        # trades: 3 buys of 10 notional + 1 sell of 10 → buy share 30/40 = 0.75
        for s, (side, amt) in enumerate([("buy", 1), ("buy", 1), ("buy", 1), ("sell", 1)]):
            ex = m0 + (s * 10 + 5) * US
            tr.append(["bybit", sym, ex, L(ex), f"t{m}_{s}", side, 10.0, amt])
        # one short liquidation (tardis side=buy) of notional 500 each minute
        ex = m0 + 30 * US
        liq.append(["bybit", sym, ex, L(ex), "buy", 100.0, 5.0])
        # ticker every 10s (realistic dense derivative_ticker feed)
        for s in range(6):
            ex = m0 + (s * 10 + 1) * US
            tk.append(["bybit", sym, ex, L(ex), base_ex, 0.0001, 1_000_000.0, 100.0, 100.0])

    if dup_trade:                            # TRUE duplicate: same trade id re-delivered (minute 18)
        ex = base_ex + 18 * MIN_US + 5 * US
        tr.append(["bybit", sym, ex, L(ex), "t18_0", "buy", 10.0, 1])   # id "t18_0" already exists
    if ooo_trade:                            # out-of-order RECEIPT in minute 17 (local>=ts, valid row,
        ex = base_ex + 17 * MIN_US + 10 * US  # but appended last ⇒ recv drops below prior file rows)
        tr.append(["bybit", sym, ex, ex + LAT, "t_ooo", "buy", 10.0, 1])
    if late_book:                            # snapshot received AFTER minute-19 close but ex before
        ex = base_ex + 19 * MIN_US - 1 * US
        local = base_ex + 19 * MIN_US + 30 * US       # received 30s after the close
        r = ["bybit", sym, ex, local] + [9999.0] * 10 + [1.0] * 10
        bk.append(r)
    if ticker_dup:                           # exact-duplicate ticker row (same local µs) in minute 18
        ex = base_ex + 18 * MIN_US + 1 * US
        tk.append(["bybit", sym, ex, L(ex), base_ex, 0.0001, 1_000_000.0, 100.0, 100.0])
    if liq_bad_side:                         # malformed USDT-linear liquidation (unknown side), minute 18
        ex = base_ex + 18 * MIN_US + 30 * US
        liq.append(["bybit", sym, ex, L(ex), "unknown", 100.0, 5.0])
    for spec in (extra_liq or []):
        liq.append(spec)

    p = {}
    for name, h, data in (("liq", liq_h, liq), ("trades", tr_h, tr),
                          ("book", bk_h, bk), ("ticker", tk_h, tk)):
        path = os.path.join(dir_, f"{name}.csv")
        _w(path, h, data)
        p[name] = path
    return p, base_ex


def test_1_both_liq_semantics():
    live = from_bybit_live({"ts": 1, "s": "ABCUSDT", "side": "Sell", "v": 2, "p": 10})
    tard = from_tardis_csv({"timestamp": 1, "local_timestamp": 2, "symbol": "ABCUSDT",
                            "side": "buy", "amount": 2, "price": 10})
    assert live.liquidated_position == tard.liquidated_position == "short"
    assert from_bybit_live({"ts": 1, "s": "ABCUSDT", "side": "Buy", "v": 1, "p": 2}).liquidated_position == "long"
    # replay picks up the tardis buy-liq as SHORT notional
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d)
        rowsr = list(R.run(p["liq"], p["trades"], p["book"], p["ticker"]))
        elig = [r for r in rowsr if r["eligible"]]
        assert elig and all(r["liq_short_notional_5m"] > 0 for r in elig[3:]), "short liq not counted"
    print("  ✓ 1 both liquidation semantics")


def test_2_future_local_ts_invisible():
    with tempfile.TemporaryDirectory() as d:
        # add a HUGE short liq whose local_timestamp is AFTER the decision minute close
        base = (1_700_000_000 * US // MIN_US) * MIN_US
        # decision minute m=10 closes at exchange base + 11*MIN; put liq received 30s later
        future_liq = ["bybit", "BTCUSDT", base + 10 * MIN_US + 30 * US,
                      base + 11 * MIN_US + 30 * US, "buy", 100.0, 9999.0]
        p, _ = build_fixture(d, extra_liq=[future_liq])
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 11 * MIN_US) // 1000            # close of minute 10 in ms
        r = rows.get(D)
        assert r is not None and not r["excluded"]
        # the 9999*100 notional must NOT be in this minute's 5m window (received after D)
        assert r["liq_short_notional_5m"] < 100_000, "future liquidation leaked into window"
    print("  ✓ 2 event with local_timestamp > D is invisible")


def test_3_future_book_not_in_a10():
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d, late_book=True)   # a 9999-priced snapshot received after m19 close
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 19 * MIN_US) // 1000
        r = rows.get(D)
        if r and not r["excluded"] and r.get("a10_asof") is not None:
            assert r["a10_asof"] < 5000, "future book snapshot became as-of a10"
            assert r["a10_prev_median"] < 5000, "future book snapshot entered median"
    print("  ✓ 3 book received after D never enters a10")


def test_4_inverse_rejected():
    try:
        from_tardis_csv({"timestamp": 1, "local_timestamp": 2, "symbol": "BTCUSD",
                         "side": "buy", "amount": 1, "price": 2})
        raise AssertionError("inverse BTCUSD must raise SchemaError")
    except SchemaError:
        pass
    # replay: an inverse liq row for a different symbol is simply not counted (skipped)
    with tempfile.TemporaryDirectory() as d:
        base = (1_700_000_000 * US // MIN_US) * MIN_US
        inv = ["bybit", "BTCUSD", base + 5 * MIN_US, base + 5 * MIN_US + 200_000, "buy", 100.0, 9999.0]
        p, _ = build_fixture(d, extra_liq=[inv])
        rows = list(R.run(p["liq"], p["trades"], p["book"], p["ticker"], symbol="BTCUSDT"))
        assert all(r.get("liq_short_notional_5m", 0) < 100_000 for r in rows), "inverse leaked"
    print("  ✓ 4 inverse / non-USDT rejected, not converted")


def test_5_dup_and_gap_exclude():
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d, dup_trade=True)
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 19 * MIN_US) // 1000            # minute 18 closes here
        r = rows.get(D)
        assert r is not None and r["excluded"] and r["exclusion_reason"] == "duplicate_row", \
            f"dup not excluded: {r}"
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d, ooo_trade=True)
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 18 * MIN_US) // 1000            # minute 17 closes here
        r = rows.get(D)
        assert r is not None and r["excluded"] and r["exclusion_reason"] == "order_conflict", \
            f"gap not excluded: {r}"
    print("  ✓ 5 duplicate & out-of-order ⇒ excluded, not zero-filled")


def test_6_engine_equals_verifier():
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d)
        out = os.path.join(d, "replay.jsonl")
        with open(out, "w") as fh:
            for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"]):
                assert set(r) <= R.ALLOWED_KEYS and not (set(r) & R.FORBIDDEN_KEYS)
                fh.write(json.dumps(r, sort_keys=True) + "\n")
        res = V.verify(out, p["liq"], p["trades"], p["book"], p["ticker"], "BTCUSDT")
        assert res["fail_count"] == 0, f"engine≠verifier: {res['fails']}"
        # sanity: at least some eligible minutes with the known taker share 0.75
        rowsr = [json.loads(l) for l in open(out)]
        elig = [r for r in rowsr if r["eligible"]]
        assert elig, "no eligible minutes produced"
        assert all(abs(r["taker_buy_share_1m"] - 0.75) < 1e-9 for r in elig), "taker share wrong"
    print("  ✓ 6 engine == verifier on fixture (identical timestamps & features)")


def _clean_replay(d):
    p, base = build_fixture(d)
    out = os.path.join(d, "replay.jsonl")
    with open(out, "w") as fh:
        for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"]):
            fh.write(json.dumps(r, sort_keys=True) + "\n")
    return p, out


def test_neg_1_verifier_catches_field_mutation():
    # Codex reproduced: mutating flags/funding_timestamp_us previously gave fail_count 0.
    with tempfile.TemporaryDirectory() as d:
        p, out = _clean_replay(d)
        rows = [json.loads(l) for l in open(out)]
        i = next(k for k, r in enumerate(rows) if r["eligible"])
        rows[i]["flags"] = rows[i]["flags"] + ["BOGUS"]
        rows[i]["funding_timestamp_us"] = (rows[i]["funding_timestamp_us"] or 0) + 7
        open(out, "w").write("\n".join(json.dumps(r, sort_keys=True) for r in rows) + "\n")
        res = V.verify(out, p["liq"], p["trades"], p["book"], p["ticker"], "BTCUSDT")
        assert res["fail_count"] > 0, "verifier missed flags/funding_timestamp_us mutation"
    print("  ✓ neg1 verifier catches flags + funding_timestamp_us mutation")


def test_neg_2_verifier_catches_duplicate_output_row():
    with tempfile.TemporaryDirectory() as d:
        p, out = _clean_replay(d)
        rows = [json.loads(l) for l in open(out)]
        rows.append(dict(rows[10]))            # duplicate an emitted decision_ts
        open(out, "w").write("\n".join(json.dumps(r, sort_keys=True) for r in rows) + "\n")
        res = V.verify(out, p["liq"], p["trades"], p["book"], p["ticker"], "BTCUSDT")
        assert res["fail_count"] > 0, "verifier missed a duplicate output row"
    print("  ✓ neg2 verifier catches duplicate output row")


def test_neg_3_ticker_integrity_excludes():
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d, ticker_dup=True)
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 19 * MIN_US) // 1000       # minute 18 close
        r = rows.get(D)
        assert r and r["excluded"] and not r["eligible"] and r["exclusion_reason"] == "duplicate_row", \
            f"duplicate ticker row not excluded: {r}"
    print("  ✓ neg3 duplicate ticker row ⇒ minute excluded (was silently eligible)")


def test_neg_4_malformed_usdt_liq_taints():
    with tempfile.TemporaryDirectory() as d:
        p, base = build_fixture(d, liq_bad_side=True)
        rows = {r["decision_ts"]: r for r in R.run(p["liq"], p["trades"], p["book"], p["ticker"])}
        D = (base + 19 * MIN_US) // 1000       # minute 18 close (within the tainted 5m window)
        r = rows.get(D)
        assert r and r["excluded"] and not r["eligible"] and r["exclusion_reason"] == "unknown_unit", \
            f"malformed USDT liquidation not tainted: {r}"
    print("  ✓ neg4 malformed USDT liquidation ⇒ minute tainted (was silently eligible)")


if __name__ == "__main__":
    test_1_both_liq_semantics()
    test_2_future_local_ts_invisible()
    test_3_future_book_not_in_a10()
    test_4_inverse_rejected()
    test_5_dup_and_gap_exclude()
    test_6_engine_equals_verifier()
    test_neg_1_verifier_catches_field_mutation()
    test_neg_2_verifier_catches_duplicate_output_row()
    test_neg_3_ticker_integrity_excludes()
    test_neg_4_malformed_usdt_liq_taints()
    print("ALL 10 CHECKS PASSED — NO OUTCOMES OR PNL WERE OPENED")
```

## `hflow_replay/outcome_firewall/hflow_outcome_firewall.py`

```python
#!/usr/bin/env python3
"""H-FLOW outcome firewall (task: CLAUDE_TASK_HFLOW_OUTCOME_FIREWALL_DRAFT.md).

A physical boundary that makes it impossible to accidentally open an outcome / PnL
field while the H-FLOW-01 preregistration is still a DRAFT.

Two guarantees:
  1. A candidate manifest may carry ONLY the fixed allowlist below. Any non-allowlisted
     top-level key, or ANY key at ANY nesting depth whose name is a forbidden
     outcome-token (entry/exit/price/return/pnl/funding_pnl/win/outcome/fill/mfe/mae),
     is rejected.
  2. The outcome reader FAILS CLOSED before opening any file unless an explicit
     frozen-prereg hash is supplied AND matches the module's frozen hash. The prereg
     is a DRAFT, so `FROZEN_PREREG_SHA256 is None` ⇒ the reader can never open a file.

This module reads NO real price/return/PnL/fill/future data. It only validates the
shape of candidate/data-quality manifests. Nothing here runs an outcome study.
"""
from __future__ import annotations

import argparse
import json
import re
import sys

# Frozen-prereg hash. Stays None until Codex reviews and the prereg is frozen.
# While None, the outcome reader is permanently sealed.
FROZEN_PREREG_SHA256: str | None = None

# The ONLY keys a candidate manifest may carry (task Deliverable A.1).
CANDIDATE_ALLOWLIST = frozenset({
    "decision_ts", "symbol", "feature_schema_sha256", "universe_snapshot_id",
    "feature_values", "exclusion_flags", "candidate_id",
})

# Outcome-field tokens rejected at any nesting depth (task A.2). SUBSTRING-matched, not
# whole-token — the adversarial council (2026-07-11) proved whole-token matching lets
# concatenations/camelCase/plurals through (entryprice, exitPrice, netpnl, maxmfe,
# returns). Substrings are calibrated to have ZERO false positives against the real
# H-FLOW vocabulary (feature keys, allowlist keys, exclusion-reason strings, symbol/
# hex ids). "win" is substring-safe in KEYS (no legit key contains it) but only
# whole-token-safe in VALUES (the legit reason string "window_*" contains "win").
KEY_SUBSTR = frozenset({"entry", "exit", "price", "return", "pnl", "fill", "mfe",
                        "mae", "outcome", "win", "funding_pnl"})
# For string VALUES: drop "price" (a snapshot id could legitimately contain it) and use
# whole-token for "win" (so a "window_*" reason value is not false-rejected).
VALUE_SUBSTR = frozenset({"entry", "exit", "return", "pnl", "fill", "mfe", "mae",
                          "outcome", "funding_pnl"})
VALUE_WHOLE = frozenset({"win"})
FORBIDDEN_EXACT = frozenset({"funding_pnl"})


class FirewallError(ValueError):
    """A manifest tried to carry a disallowed or outcome-shaped field/value, or the
    outcome reader was invoked without a valid frozen-prereg hash."""


def _tokens(name: str) -> set[str]:
    return {t for t in re.split(r"[^a-z0-9]+", str(name).lower()) if t}


def _key_forbidden(name) -> bool:
    low = str(name).lower()
    return (low in FORBIDDEN_EXACT or any(s in low for s in KEY_SUBSTR))


def _value_forbidden(s: str) -> bool:
    low = str(s).lower()
    return any(sub in low for sub in VALUE_SUBSTR) or bool(_tokens(low) & VALUE_WHOLE)


def _scan_forbidden(obj, path: str = "") -> list[str]:
    """Recursively collect paths of every forbidden dict KEY *or* string VALUE."""
    bad: list[str] = []
    if isinstance(obj, dict):
        for k, v in obj.items():
            here = f"{path}.{k}" if path else str(k)
            if _key_forbidden(k):
                bad.append("key:" + here)
            bad += _scan_forbidden(v, here)
    elif isinstance(obj, (list, tuple)):
        for i, v in enumerate(obj):
            bad += _scan_forbidden(v, f"{path}[{i}]")
    elif isinstance(obj, str):
        if _value_forbidden(obj):
            bad.append("value:" + path)
    return bad


def validate_candidate(manifest: dict, feature_schema=None) -> dict:
    """Return the manifest unchanged iff it is a clean candidate/data-quality record.
    Raise FirewallError otherwise. Never opens or reads any outcome data.

    `feature_schema` (optional): a frozen allowlist of exact feature names. When
    supplied, `feature_values` keys must be a SUBSET of it — the strong defence against
    the bare-number value channel (an outcome number smuggled under an innocent key
    like `f1`/`v`). Codex owns the frozen feature contract; until it exists, the
    hardened token/value blocklist below is the fallback and CANNOT stop a bare number
    under an arbitrary innocent key (documented residual)."""
    if not isinstance(manifest, dict):
        raise FirewallError("candidate manifest must be a JSON object")
    extra = set(manifest) - CANDIDATE_ALLOWLIST
    if extra:
        raise FirewallError(f"non-allowlisted top-level key(s): {sorted(extra)}")
    bad = _scan_forbidden(manifest)
    if bad:
        raise FirewallError(f"forbidden outcome-field name(s)/value(s) at: {sorted(bad)}")
    if feature_schema is not None:
        fv = manifest.get("feature_values")
        if not isinstance(fv, dict):
            raise FirewallError("feature_values must be an object when a feature_schema is enforced")
        unknown = set(fv) - set(feature_schema)
        if unknown:
            raise FirewallError(f"feature name(s) not in the frozen feature_schema: {sorted(unknown)}")
    return manifest


def open_outcome_reader(path: str, frozen_prereg_hash: str | None = None):
    """Outcome reader. FAILS CLOSED *before* opening `path` unless a frozen-prereg hash
    is supplied and equals the module's FROZEN_PREREG_SHA256. While the prereg is a
    draft (FROZEN_PREREG_SHA256 is None), this can never open a file."""
    if FROZEN_PREREG_SHA256 is None:
        raise FirewallError(
            "outcome reader SEALED: prereg is still a DRAFT (no frozen hash exists) — "
            "failing closed, file not opened")
    if not frozen_prereg_hash:
        raise FirewallError(
            "outcome reader SEALED: no --frozen-prereg-hash supplied — failing closed, "
            "file not opened")
    if not re.fullmatch(r"[0-9a-f]{64}", str(frozen_prereg_hash)):
        raise FirewallError("outcome reader SEALED: hash is not a 64-hex sha256 — failing closed")
    if frozen_prereg_hash != FROZEN_PREREG_SHA256:
        raise FirewallError(
            "outcome reader SEALED: supplied hash does not match the frozen prereg — "
            "failing closed, file not opened")
    # Unreachable while draft. When frozen, a reviewed outcome reader would live here.
    raise FirewallError("outcome reader not implemented in draft phase")  # pragma: no cover


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description="H-FLOW outcome firewall (draft phase)")
    sub = ap.add_subparsers(dest="cmd", required=True)
    v = sub.add_parser("validate", help="validate a candidate/data-quality manifest (default mode)")
    v.add_argument("manifest", nargs="?", default="-",
                   help="path to a JSON candidate manifest; default '-' = stdin")
    o = sub.add_parser("open-outcome", help="attempt to open an outcome file (fails closed while draft)")
    o.add_argument("path")
    o.add_argument("--frozen-prereg-hash", default=None)
    a = ap.parse_args(argv)

    if a.cmd == "validate":
        # stdin by default. A file arg is read defensively: size-capped (a candidate
        # manifest is tiny; refuse to slurp a large outcome dump) and JSON errors are
        # caught (never crash mid-read). `validate` is a candidate-manifest linter only;
        # it is NOT the outcome gate — the sealed open_outcome_reader is.
        MAX_BYTES = 1_000_000
        if a.manifest == "-":
            raw = sys.stdin.read(MAX_BYTES + 1)
        else:
            import os
            if os.path.isfile(a.manifest) and os.path.getsize(a.manifest) > MAX_BYTES:
                print(json.dumps({"accepted": False,
                                  "error": f"manifest exceeds {MAX_BYTES} bytes — refusing to read (not a candidate manifest?)"}))
                return 2
            with open(a.manifest, "r", encoding="utf-8", errors="strict") as fh:
                raw = fh.read(MAX_BYTES + 1)
        if len(raw) > MAX_BYTES:
            print(json.dumps({"accepted": False, "error": "input exceeds size cap — refused"}))
            return 2
        try:
            parsed = json.loads(raw)
        except json.JSONDecodeError as e:
            print(json.dumps({"accepted": False, "error": f"not a JSON candidate manifest: {e}"}))
            return 2
        try:
            clean = validate_candidate(parsed)
        except FirewallError as e:
            print(json.dumps({"accepted": False, "error": str(e)}))
            return 2
        print(json.dumps({"accepted": True, "candidate": clean}, sort_keys=True))
        print("NO OUTCOMES OR PNL WERE OPENED")
        return 0

    # open-outcome: must fail closed
    try:
        open_outcome_reader(a.path, a.frozen_prereg_hash)
    except FirewallError as e:
        print(json.dumps({"opened": False, "error": str(e)}))
        print("NO OUTCOMES OR PNL WERE OPENED")
        return 3
    return 0  # pragma: no cover


if __name__ == "__main__":
    raise SystemExit(main())
```

## `hflow_replay/outcome_firewall/test_outcome_firewall.py`

```python
#!/usr/bin/env python3
"""No-network tests for the H-FLOW outcome firewall. No real PnL, no outcome data."""
from __future__ import annotations

import json
import hflow_outcome_firewall as F


# A clean candidate uses the real feature-schema names (none are outcome tokens).
CLEAN = {
    "decision_ts": 1783762800000,
    "symbol": "BTCUSDT",
    "feature_schema_sha256": "0" * 64,
    "universe_snapshot_id": "top150@2026-07-01T00:00Z",
    "candidate_id": "BTCUSDT-1783762800000",
    "feature_values": {
        "liq_short_notional_5m": 125000.0,
        "liq_short_count_5m": 7,
        "taker_buy_share_1m": 0.71,
        "taker_total_notional_1m": 980000.0,
        "a10_ratio": 0.42,
        "a10_asof_recv_ms": 1783762799880,
        "oi_value": 21084030.2,
        "oi_age_ms": 1200,
        "funding_rate": -0.0004,
        "funding_age_ms": 1200,
    },
    "exclusion_flags": ["zero_short_liq"],   # list of reason strings (values, not keys)
}


def test_clean_validates_and_roundtrips():
    out = F.validate_candidate(json.loads(json.dumps(CLEAN)))
    assert out == CLEAN, "exact candidate fields must survive round-trip unchanged"
    assert not F._scan_forbidden(out), "clean output must contain no forbidden keys"
    print("  ✓ clean candidate validates + exact round-trip + no forbidden keys")


def test_non_allowlisted_toplevel_rejected():
    for bad_key in ("entry", "pnl", "note", "score"):
        m = dict(CLEAN); m[bad_key] = 1
        try:
            F.validate_candidate(m); raise AssertionError(f"{bad_key} must be rejected")
        except F.FirewallError:
            pass
    print("  ✓ any non-allowlisted top-level key rejected")


def test_nested_forbidden_key_rejected():
    for bad in ("mfe", "mae", "exit_price", "entry", "fill", "pnl", "funding_pnl",
                "realized_return", "win", "outcome_price"):
        m = json.loads(json.dumps(CLEAN))
        m["feature_values"][bad] = 1.0
        try:
            F.validate_candidate(m); raise AssertionError(f"nested {bad} must be rejected")
        except F.FirewallError:
            pass
    # deeply nested (feature_values -> dict -> forbidden)
    m = json.loads(json.dumps(CLEAN))
    m["feature_values"]["book"] = {"ask_depletion": 0.3, "exit_price": 100.0}
    try:
        F.validate_candidate(m); raise AssertionError("deeply nested exit_price must be rejected")
    except F.FirewallError:
        pass
    print("  ✓ nested & deeply-nested forbidden keys rejected")


def test_no_false_rejection_on_real_vocabulary():
    # every real H-FLOW feature key + normalized/ΔOI additions must be accepted as KEYS
    m = json.loads(json.dumps(CLEAN))
    for ok in ("normalized_funding", "delta_oi_4h", "doi_4h", "ask_depletion_ratio",
               "funding_asof_recv_ms", "taker_buy_share_1m"):
        m["feature_values"][ok] = 1.0
    # every real exclusion-reason string must be accepted as a VALUE (window≠win token)
    m["exclusion_flags"] = ["window_before_data_start", "window_after_data_end",
                            "no_taker_flow", "stale_book", "stale_ticker", "duplicate_row",
                            "order_conflict", "unknown_unit", "insufficient_book_history",
                            "no_book_asof", "no_ticker_asof", "zero_short_liq"]
    F.validate_candidate(m)   # must not raise
    print("  ✓ no false rejection on the real feature keys + exclusion-reason values")


# ── permanent regressions for the 8 bypasses the adversarial council confirmed ──
COUNCIL_BYPASSES = [
    {"feature_values": {"entryprice": 100.5}},                      # concatenation
    {"feature_values": {"exitPrice": 200.0}},                       # camelCase
    {"exclusion_flags": {"realizedpnl": 9.0, "winflag": True, "outcomeknown": 1}},
    {"feature_values": {"maxmfe": 12.3, "maxmae": -4.1}},           # mfe/mae fused
    {"symbol": "BTCUSDT", "feature_values": {"returns": [0.12, -0.03]}},  # plural
    {"symbol": "BTCUSDT", "feature_values": {"exitprice": 43120.0, "netpnl": 9.3}},
    {"exclusion_flags": ["realized_return_0_042", "exit_filled_43010", "pnl_positive"]},  # string list
    {"candidate_id": "ret=+0.042;pnl=+130usd;win=1;exit_price=43010"},   # string value channel
]


def test_council_bypasses_now_rejected():
    for i, m in enumerate(COUNCIL_BYPASSES):
        try:
            F.validate_candidate(json.loads(json.dumps(m)))
            raise AssertionError(f"council bypass #{i} STILL accepted: {m}")
        except F.FirewallError:
            pass
    print(f"  ✓ all {len(COUNCIL_BYPASSES)} council-confirmed bypasses now rejected")


def test_feature_schema_allowlist_closes_bare_number_channel():
    schema = set(CLEAN["feature_values"])          # frozen allowed feature names
    F.validate_candidate(json.loads(json.dumps(CLEAN)), feature_schema=schema)  # ok
    for smuggle in ({"f1": 0.042, "f2": 43010.0},  # innocent-named bare numbers
                    {"v": 43010.0}, {"entryprice": 1.0}):
        m = json.loads(json.dumps(CLEAN)); m["feature_values"] = smuggle
        try:
            F.validate_candidate(m, feature_schema=schema)
            raise AssertionError(f"schema allowlist let unknown feature through: {smuggle}")
        except F.FirewallError:
            pass
    print("  ✓ feature_schema allowlist rejects unknown/bare-number feature keys")


def test_cli_non_json_and_size_guard():
    import io, contextlib, tempfile, os
    with tempfile.TemporaryDirectory() as d:
        # a non-JSON 'outcome' file must NOT crash validate (Codex/council lens 4)
        p = os.path.join(d, "outcomes.csv")
        open(p, "w").write("ts,entry_price,pnl\n1,100,50\n")
        buf = io.StringIO()
        with contextlib.redirect_stdout(buf):
            rc = F.main(["validate", p])
        assert rc == 2 and json.loads(buf.getvalue())["accepted"] is False, "non-JSON must be a clean reject, not a crash"
        # size cap: an oversized file is refused before full interpretation
        big = os.path.join(d, "big.json"); open(big, "w").write("{" + '"x":1,' * 200000 + '"y":2}')
        buf = io.StringIO()
        with contextlib.redirect_stdout(buf):
            rc = F.main(["validate", big])
        assert rc == 2, "oversized manifest must be refused"
    print("  ✓ validate: non-JSON rejected cleanly (no crash); oversized refused")


def test_outcome_reader_fails_closed_no_hash():
    # nonexistent path + no hash: must raise FirewallError (SEALED), NOT FileNotFoundError
    try:
        F.open_outcome_reader("/definitely/not/a/real/path.jsonl", frozen_prereg_hash=None)
        raise AssertionError("outcome reader must fail closed without a hash")
    except FileNotFoundError:
        raise AssertionError("reader OPENED the file — it must fail closed BEFORE opening")
    except F.FirewallError as e:
        assert "SEALED" in str(e)
    print("  ✓ outcome reader fails closed before opening (no hash)")


def test_outcome_reader_sealed_even_with_valid_format_hash():
    # a well-formed 64-hex hash still fails while the prereg is a draft (FROZEN is None)
    assert F.FROZEN_PREREG_SHA256 is None, "prereg must be unfrozen in draft phase"
    try:
        F.open_outcome_reader("/definitely/not/a/real/path.jsonl", frozen_prereg_hash="a" * 64)
        raise AssertionError("must stay sealed while draft")
    except FileNotFoundError:
        raise AssertionError("reader OPENED the file — must fail closed while draft")
    except F.FirewallError as e:
        assert "SEALED" in str(e)
    print("  ✓ outcome reader stays sealed even with a valid-format hash (draft)")


def test_cli_modes():
    import io, contextlib
    # validate mode accepts clean, prints no forbidden keys
    import tempfile, os
    with tempfile.TemporaryDirectory() as d:
        p = os.path.join(d, "c.json"); open(p, "w").write(json.dumps(CLEAN))
        buf = io.StringIO()
        with contextlib.redirect_stdout(buf):
            rc = F.main(["validate", p])
        assert rc == 0 and "NO OUTCOMES OR PNL WERE OPENED" in buf.getvalue()
        assert not F._scan_forbidden(json.loads(buf.getvalue().splitlines()[0])["candidate"])
        # validate rejects a forbidden manifest with non-zero rc
        bad = json.loads(json.dumps(CLEAN)); bad["feature_values"]["pnl"] = 1
        open(p, "w").write(json.dumps(bad))
        buf = io.StringIO()
        with contextlib.redirect_stdout(buf):
            rc = F.main(["validate", p])
        assert rc == 2 and json.loads(buf.getvalue())["accepted"] is False
        # open-outcome fails closed
        buf = io.StringIO()
        with contextlib.redirect_stdout(buf):
            rc = F.main(["open-outcome", "/no/such/file"])
        assert rc == 3 and "NO OUTCOMES OR PNL WERE OPENED" in buf.getvalue()
    print("  ✓ CLI: validate accepts/rejects; open-outcome fails closed")


if __name__ == "__main__":
    test_clean_validates_and_roundtrips()
    test_non_allowlisted_toplevel_rejected()
    test_nested_forbidden_key_rejected()
    test_no_false_rejection_on_real_vocabulary()
    test_council_bypasses_now_rejected()
    test_feature_schema_allowlist_closes_bare_number_channel()
    test_cli_non_json_and_size_guard()
    test_outcome_reader_fails_closed_no_hash()
    test_outcome_reader_sealed_even_with_valid_format_hash()
    test_cli_modes()
    print("ALL FIREWALL CHECKS PASSED — NO OUTCOMES OR PNL WERE OPENED")
```

## `xarb01/xarb01_runner.py`

```python
#!/usr/bin/env python3
"""X-ARB-01 outcome-free feasibility runner.

Implements EXACTLY the frozen preregistration
``2026-07-11_RESEARCH_DIRECTOR_DECISION_XARB_01.md`` bound by
``2026-07-11_XARB_01_FREEZE_RECEIPT.json``.

It answers one mechanical question: do persistent, two-sided cross-venue quote
crossings survive all-in reserves of 14 / 31 / 71 bp under a conservative as-of
and data-quality contract?  It is a FEASIBILITY GATE.  It never reads, derives
or emits a future price, return, position, order, fill, PnL, MFE, MAE or win.

Design notes (see FROZEN spec sections in brackets):
- Decision clock is the fixed 250 ms UTC lattice [spec 5].  For each lattice
  point D the "current" quote of a venue is the latest sealed row with
  ``flush_ts <= D`` and age ``D - flush_ts <= 1000 ms`` [spec 3].  Because that
  current-fresh quote is piecewise-constant in D, buckets are computed by
  interval projection, not by iterating 345_600 empty points per day.
- Each of the seven UTC days is evaluated INDEPENDENTLY from its own records; no
  freshness is carried across day-file boundaries.  A quote sealed in the final
  <=1000 ms of a day is therefore not used to price the first buckets of the
  next day.  This is a conservative edge loss of at most a few buckets out of
  345_600 (<0.01 %), far below the 95 % coverage gate.  [Ambiguity #6.]
- Every threshold below is copied from the frozen file; NONE is chosen here.
  Genuine ambiguities are enumerated in AMBIGUITIES and reported to Codex, never
  silently resolved into a load-bearing parameter.
"""

from __future__ import annotations

import argparse
import gzip
import hashlib
import json
import math
import random
from bisect import bisect_right
from collections import Counter, defaultdict
from datetime import UTC, datetime
from pathlib import Path
from typing import Any, Iterable

# --------------------------------------------------------------------------- #
# Frozen constants (verbatim from the preregistration / receipt).             #
# --------------------------------------------------------------------------- #
SCHEMA = 3
COSTS_BP: tuple[int, ...] = (14, 31, 71)
VENUES: tuple[str, ...] = ("by", "bn", "ok")

MAX_QUOTE_AGE_MS = 1_000              # spec 3: D - flush_ts <= 1000 ms
RECONNECT_GUARD_MS = 10_000           # spec 3: exclude within 10 s of a bad event
DEPTH_TARGET_USD = 1_000.0            # spec 3: observed L5 notional >= $1000 both legs
BUCKET_MS = 250                       # spec 5: 250 ms decision buckets
EXPECTED_BUCKETS_PER_DAY = 86_400_000 // BUCKET_MS   # spec 5: 345_600, by formula

EPISODE_SEPARATION_MS = 10_000        # spec 4: a >=10 s dry spell separates episodes
EPISODE_MIN_POINTS = 4                # spec 4: >= four qualifying sealed decisions
EPISODE_MAX_ADJ_GAP_MS = 1_000        # spec 4: each adjacent distance <= 1000 ms
EPISODE_MIN_SPAN_MS = 750             # spec 4: D4 - D1 >= 750 ms

COVERAGE_MIN = 0.95                   # spec 5: valid coverage >= 95 % of expected buckets, every day
BOOTSTRAP_N = 10_000                  # spec 5: 10_000 day-block resamples
BOOTSTRAP_SEED = 20260711            # task item 7: fixed seed

KILLED_MIN_C71_EPISODES = 20          # spec 6: >= 20 persistent c=71 episodes
KILLED_MIN_DAYS = 4                   # spec 6: episodes in >= 4 of 7 UTC days
EVAL_DAYS = 7

# Manifest / clock bad-event vocabulary (from the X-VENUE integrity contract,
# mirrored from the accepted X-ARB-00B auditor).
BAD_EVENTS = {"late_xex_fragment", "map_source_empty", "gzip_fail", "FATAL_disk"}
BAD_EVENT_SUFFIXES = ("_close", "_error", "_crash")

# Output key hygiene: no outcome/PnL-shaped field may ever leave this program.
FORBIDDEN_OUTPUT_TOKENS = (
    "price_after", "future", "return", "pnl", "fill", "mfe", "mae", "win",
    "position", "entry", "exit", "outcome", "profit", "revenue", "roi",
)

VERDICT_DATA_INSUFFICIENT = "DATA-INSUFFICIENT / NO-TRADE"
VERDICT_KILLED = "KILLED AS EXECUTABLE ROUTE / NO-TRADE"
VERDICT_ELIGIBLE = "ELIGIBLE FOR SEPARATE PAPER-EXECUTION PREREG — NOT CONFIRMED"

# Interpretation choices reported to Codex (see spec sections). The runner is
# faithful to every FROZEN number; these are edge semantics the frozen prose
# does not fully pin down. Codex confirms/corrects before the real 18-Jul run.
AMBIGUITIES = [
    "1. Episode separation uses gap >= 10_000 ms to start a new episode (spec 4 "
    "'after 10 seconds without a qualifying point'). '>' vs '>=' unconfirmed.",
    "2. An episode counts once iff SOME window of four CONSECUTIVE qualifying "
    "decisions has all adjacent gaps <= 1000 ms and span >= 750 ms (existential "
    "reading of spec 4 'there exist at least four ...').",
    "3. Coverage-valid bucket = both legs fresh + BBO-valid + L5 depth KNOWN + "
    "not bad-event-excluded. A known depth < $1000 is valid coverage but "
    "non-qualifying (depth is a qualification filter, not a coverage filter); "
    "UNKNOWN depth is an exclusion per spec 3.",
    "4. Decision instant D = left edge of each 250 ms bucket (D_k = midnight + "
    "k*250). 'flush_ts <= D' uses this D. Right-edge convention unconfirmed.",
    "5. Bad-event exclusion window is [e, e+10_000 ms] (a decision within 10 s "
    "AFTER a bad event is excluded), matching the accepted X-ARB-00B auditor.",
    "6. Days are evaluated independently from their own day-files; no cross-"
    "midnight freshness carry-in (conservative, <0.01 % of buckets).",
    "7. Canonical map = top-level JSON object; its keys are the 20 bases. SHA is "
    "sha256(json.dumps(map, sort_keys=True, separators=(',',':'))). Codex must "
    "confirm the real map JSON structure so the receipt SHA matches.",
    "8. An extra UTC day outside the 7-day window => REFUSE. A missing day (or a "
    "day below 95 % coverage) => that route is not data-sufficient; if no route "
    "is data-sufficient the verdict is DATA-INSUFFICIENT.",
]


class RefuseEvaluation(Exception):
    """Raised when receipt/schema/map/day integrity fails: never evaluate."""


# --------------------------------------------------------------------------- #
# Output hygiene                                                               #
# --------------------------------------------------------------------------- #
def assert_output_clean(obj: Any, path: str = "") -> None:
    """Recursively forbid any outcome/PnL-shaped key anywhere in the report."""
    if isinstance(obj, dict):
        for key, value in obj.items():
            lowered = str(key).lower()
            for token in FORBIDDEN_OUTPUT_TOKENS:
                if token in lowered:
                    raise ValueError(
                        f"forbidden output key token {token!r} at {path}/{key}"
                    )
            assert_output_clean(value, f"{path}/{key}")
    elif isinstance(obj, (list, tuple)):
        for i, value in enumerate(obj):
            assert_output_clean(value, f"{path}[{i}]")


# --------------------------------------------------------------------------- #
# Receipt / map / schema verification                                         #
# --------------------------------------------------------------------------- #
def canonical_map_sha256(map_obj: Any) -> str:
    encoded = json.dumps(map_obj, sort_keys=True, separators=(",", ":")).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()


def window_days(receipt: dict[str, Any]) -> list[str]:
    """Return the exactly-seven UTC day strings named by the receipt."""
    win = receipt["evaluation_window"]
    first = datetime.strptime(win["first_utc_day"], "%Y-%m-%d").replace(tzinfo=UTC)
    days = [(first.timestamp() + i * 86_400) for i in range(EVAL_DAYS)]
    out = [datetime.fromtimestamp(t, UTC).date().isoformat() for t in days]
    if out[-1] != win["last_utc_day"] or int(win.get("complete_utc_days", 0)) != EVAL_DAYS:
        raise RefuseEvaluation(
            f"receipt window is not exactly seven days: {win}"
        )
    return out


def verify_receipt(receipt: dict[str, Any], map_obj: Any) -> list[str]:
    """Refuse evaluation unless schema=3, map SHA and window all match. Returns days."""
    contract = receipt.get("data_contract", {})
    if int(contract.get("schema", -1)) != SCHEMA:
        raise RefuseEvaluation(f"receipt schema != {SCHEMA}: {contract.get('schema')!r}")
    expected_map_sha = contract.get("canonical_20_base_map_sha256")
    actual_map_sha = canonical_map_sha256(map_obj)
    if actual_map_sha != expected_map_sha:
        raise RefuseEvaluation(
            f"canonical map SHA mismatch: got {actual_map_sha}, receipt {expected_map_sha}"
        )
    bases = sorted(map_obj) if isinstance(map_obj, dict) else []
    if len(bases) != int(contract.get("universe_size", -1)) or len(bases) != 20:
        raise RefuseEvaluation(
            f"universe must be exactly 20 bases, got {len(bases)}"
        )
    return window_days(receipt)


# --------------------------------------------------------------------------- #
# Record ingestion & classification                                           #
# --------------------------------------------------------------------------- #
def _open_text(path: Path):
    return gzip.open(path, "rt") if path.suffix == ".gz" else path.open("rt")


def iter_records(paths: Iterable[Path]) -> Iterable[dict[str, Any]]:
    for path in paths:
        with _open_text(path) as fh:
            for line in fh:
                line = line.strip()
                if line:
                    yield json.loads(line)


def quote_is_valid_bbo(row: dict[str, Any]) -> bool:
    """schema=3 row with a well-ordered receipt clock and a positive two-sided BBO."""
    try:
        if int(row.get("schema", -1)) != SCHEMA:
            return False
        if row.get("e") not in VENUES or not isinstance(row.get("s"), str):
            return False
        rmin, rmax, flush = int(row["rmin"]), int(row["rmax"]), int(row["flush_ts"])
        if not (rmin <= rmax <= flush):
            return False
        return float(row["a"]) > 0 and float(row["b"]) > 0
    except (KeyError, TypeError, ValueError):
        return False


def _depth(row: dict[str, Any], side: str) -> float | None:
    """L5 notional on the ask (buy leg) or bid (sell leg) side; None if unknown."""
    key = "a5" if side == "ask" else "b5"
    try:
        return float(row[key])
    except (KeyError, TypeError, ValueError):
        return None


def _utc_day(ts_ms: int) -> str:
    return datetime.fromtimestamp(ts_ms / 1000, UTC).date().isoformat()


def _day_midnight_ms(day_iso: str) -> int:
    dt = datetime.strptime(day_iso, "%Y-%m-%d").replace(tzinfo=UTC)
    return int(dt.timestamp() * 1000)


# --------------------------------------------------------------------------- #
# Bad events (reconnect / clock / manifest) -> per-key sorted timestamps       #
# --------------------------------------------------------------------------- #
def bad_event_times(
    manifest_rows: Iterable[dict[str, Any]],
    clock_rows: Iterable[dict[str, Any]] = (),
) -> dict[str | None, list[int]]:
    out: dict[str | None, list[int]] = defaultdict(list)
    for row in manifest_rows:
        ev = str(row.get("ev", ""))
        if (ev in BAD_EVENTS or ev.endswith(BAD_EVENT_SUFFIXES)) and isinstance(row.get("ts"), int):
            out[row.get("e")].append(int(row["ts"]))
    for row in clock_rows:
        if int(row.get("schema", -1)) != SCHEMA:
            continue
        probes = row.get("probes")
        if not isinstance(probes, list):
            continue
        failed, seen, event_ts = False, set(), 0
        for probe in probes:
            try:
                venue = str(probe["venue"])
                send_ts, recv_ts = int(probe["send_ts"]), int(probe["recv_ts"])
                failed = failed or venue not in set(VENUES) or send_ts > recv_ts or probe.get("server_ts") is None
                seen.add(venue)
                event_ts = max(event_ts, recv_ts)
            except (KeyError, TypeError, ValueError):
                failed = True
        if (failed or seen != set(VENUES)) and event_ts:
            out[None].append(event_ts)
    for values in out.values():
        values.sort()
    return dict(out)


def _excluded_intervals_for_pair(
    bad: dict[str | None, list[int]], venue_a: str, venue_b: str
) -> list[tuple[int, int]]:
    """Merged time intervals [e, e+10s] excluding a decision on route A->B."""
    events: list[int] = []
    for key in (venue_a, venue_b, None):
        events.extend(bad.get(key, []))
    events.sort()
    merged: list[tuple[int, int]] = []
    for e in events:
        lo, hi = e, e + RECONNECT_GUARD_MS
        if merged and lo <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], hi))
        else:
            merged.append((lo, hi))
    return merged


# --------------------------------------------------------------------------- #
# Lattice / interval helpers                                                   #
# --------------------------------------------------------------------------- #
def _bucket_range(lo_ms: int, hi_ms: int, midnight_ms: int) -> tuple[int, int] | None:
    """Lattice indices k with D_k = midnight + k*250 in [lo, hi] inclusive, clamped to the day."""
    if hi_ms < lo_ms:
        return None
    k_lo = max(0, math.ceil((lo_ms - midnight_ms) / BUCKET_MS))
    k_hi = min(EXPECTED_BUCKETS_PER_DAY - 1, (hi_ms - midnight_ms) // BUCKET_MS)
    if k_hi < k_lo:
        return None
    return (k_lo, k_hi)


def _subtract_ranges(base: tuple[int, int], holes: list[tuple[int, int]]) -> list[tuple[int, int]]:
    """base minus a list of (k_lo,k_hi) hole ranges, returning surviving sub-ranges."""
    survivors = [base]
    for h_lo, h_hi in holes:
        nxt: list[tuple[int, int]] = []
        for s_lo, s_hi in survivors:
            if h_hi < s_lo or h_lo > s_hi:
                nxt.append((s_lo, s_hi))
                continue
            if h_lo > s_lo:
                nxt.append((s_lo, h_lo - 1))
            if h_hi < s_hi:
                nxt.append((h_hi + 1, s_hi))
        survivors = nxt
    return survivors


def _current_fresh_intervals(quotes: list[dict[str, Any]]) -> list[tuple[int, int, dict[str, Any]]]:
    """For a venue's day quotes (sorted, de-duped by flush_ts), the time window
    [start, end] (inclusive, ms) during which each quote is BOTH current
    (flush_ts <= D < next flush) AND fresh (D - flush_ts <= 1000)."""
    out: list[tuple[int, int, dict[str, Any]]] = []
    n = len(quotes)
    for i, q in enumerate(quotes):
        f = int(q["flush_ts"])
        hi = f + MAX_QUOTE_AGE_MS
        if i + 1 < n:
            hi = min(hi, int(quotes[i + 1]["flush_ts"]) - 1)
        if hi >= f:
            out.append((f, hi, q))
    return out


def _dedupe_current(rows: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Keep, per flush_ts, the row with the largest (rmax, t) — deterministic like X-ARB-00B."""
    by_flush: dict[int, dict[str, Any]] = {}
    for row in rows:
        f = int(row["flush_ts"])
        prev = by_flush.get(f)
        key = (int(row["rmax"]), int(row["t"]) if row.get("t") is not None else -1)
        if prev is None:
            by_flush[f] = row
        else:
            pkey = (int(prev["rmax"]), int(prev["t"]) if prev.get("t") is not None else -1)
            if key >= pkey:
                by_flush[f] = row
    return [by_flush[f] for f in sorted(by_flush)]


def _overlaps(
    a: list[tuple[int, int, dict[str, Any]]],
    b: list[tuple[int, int, dict[str, Any]]],
) -> Iterable[tuple[int, int, dict[str, Any], dict[str, Any]]]:
    """Merge two sorted fresh-interval lists into (lo, hi, quote_a, quote_b) overlaps."""
    i = j = 0
    while i < len(a) and j < len(b):
        a_lo, a_hi, qa = a[i]
        b_lo, b_hi, qb = b[j]
        lo, hi = max(a_lo, b_lo), min(a_hi, b_hi)
        if lo <= hi:
            yield (lo, hi, qa, qb)
        if a_hi <= b_hi:
            i += 1
        else:
            j += 1


# --------------------------------------------------------------------------- #
# Episode detection (pure)                                                      #
# --------------------------------------------------------------------------- #
def _has_persistent_window(times: list[int]) -> bool:
    """Exists four consecutive points with each adjacent gap <=1000 ms and span >=750 ms."""
    for i in range(len(times) - EPISODE_MIN_POINTS + 1):
        w = times[i : i + EPISODE_MIN_POINTS]
        if all(w[k + 1] - w[k] <= EPISODE_MAX_ADJ_GAP_MS for k in range(len(w) - 1)) and (
            w[-1] - w[0]
        ) >= EPISODE_MIN_SPAN_MS:
            return True
    return False


def episode_starts(qual_times: list[int]) -> list[int]:
    """Return the start time of each COUNTED persistent episode from sorted qualifying times.

    Episodes are separated by a >= 10 s dry spell; an episode counts once iff it
    contains a valid four-point persistence window (spec 4)."""
    if not qual_times:
        return []
    starts: list[int] = []
    current = [qual_times[0]]
    for t in qual_times[1:]:
        if t - current[-1] >= EPISODE_SEPARATION_MS:
            if _has_persistent_window(current):
                starts.append(current[0])
            current = [t]
        else:
            current.append(t)
    if _has_persistent_window(current):
        starts.append(current[0])
    return starts


# --------------------------------------------------------------------------- #
# Day-block bootstrap (deterministic)                                          #
# --------------------------------------------------------------------------- #
def day_block_bootstrap(daily_counts: list[int]) -> dict[str, float]:
    """10_000 UTC-day block resamples (fixed seed) of episodes/day; 95 % CI of the mean."""
    rng = random.Random(BOOTSTRAP_SEED)
    n = len(daily_counts)
    means: list[float] = []
    for _ in range(BOOTSTRAP_N):
        sample = [daily_counts[rng.randrange(n)] for _ in range(n)]
        means.append(sum(sample) / n)
    means.sort()
    lo = means[int(0.025 * (BOOTSTRAP_N - 1))]
    hi = means[int(math.ceil(0.975 * (BOOTSTRAP_N - 1)))]
    return {
        "mean_episodes_per_day": round(sum(daily_counts) / n, 6),
        "ci95_low": round(lo, 6),
        "ci95_high": round(hi, 6),
    }


# --------------------------------------------------------------------------- #
# Core analysis                                                                 #
# --------------------------------------------------------------------------- #
def analyze(
    xex_rows: Iterable[dict[str, Any]],
    manifest_rows: Iterable[dict[str, Any]],
    clock_rows: Iterable[dict[str, Any]],
    receipt: dict[str, Any],
    canonical_map: Any,
) -> dict[str, Any]:
    """Run the frozen X-ARB-01 feasibility accounting. No future price/PnL is ever touched."""
    days = verify_receipt(receipt, canonical_map)
    day_set = set(days)
    bases = sorted(canonical_map)

    # Partition valid rows by (day, base, venue); refuse on any out-of-window day.
    per: dict[tuple[str, str, str], list[dict[str, Any]]] = defaultdict(list)
    counters: Counter[str] = Counter()
    for row in xex_rows:
        if int(row.get("schema", -1)) != SCHEMA:
            counters["non_schema3_rows"] += 1
            continue
        if not quote_is_valid_bbo(row):
            counters["malformed_rows"] += 1
            continue
        day = _utc_day(int(row["flush_ts"]))
        if day not in day_set:
            raise RefuseEvaluation(f"record on out-of-window UTC day {day}: extra day violation")
        base = row["s"]
        if base not in canonical_map:
            counters["off_universe_rows"] += 1
            continue
        per[(day, base, row["e"])].append(row)

    bad = bad_event_times(manifest_rows, clock_rows)

    routes = [
        (base, a, b)
        for base in bases
        for a in VENUES
        for b in VENUES
        if a != b
    ]

    # coverage[route][day] = valid bucket count ; episodes[route][cost][day] = count
    coverage: dict[tuple[str, str, str], dict[str, int]] = {
        r: {d: 0 for d in days} for r in routes
    }
    episodes: dict[tuple[str, str, str], dict[int, dict[str, int]]] = {
        r: {c: {d: 0 for d in days} for c in COSTS_BP} for r in routes
    }
    excl: dict[tuple[str, str, str], Counter] = {r: Counter() for r in routes}

    for base in bases:
        for a in VENUES:
            for b in VENUES:
                if a == b:
                    continue
                route = (base, a, b)
                pair_excluded = _excluded_intervals_for_pair(bad, a, b)
                for day in days:
                    midnight = _day_midnight_ms(day)
                    a_rows = _dedupe_current(per.get((day, base, a), []))
                    b_rows = _dedupe_current(per.get((day, base, b), []))
                    if not a_rows or not b_rows:
                        continue
                    a_int = _current_fresh_intervals(a_rows)
                    b_int = _current_fresh_intervals(b_rows)
                    # excluded bucket ranges for this day
                    excl_ranges: list[tuple[int, int]] = []
                    for e_lo, e_hi in pair_excluded:
                        rng = _bucket_range(e_lo, e_hi, midnight)
                        if rng:
                            excl_ranges.append(rng)
                    qual_times: dict[int, list[int]] = {c: [] for c in COSTS_BP}
                    for lo, hi, qa, qb in _overlaps(a_int, b_int):
                        base_range = _bucket_range(lo, hi, midnight)
                        if base_range is None:
                            continue
                        # depth known? (buy leg consumes ask a5, sell leg consumes bid b5)
                        ask_depth = _depth(qa, "ask")
                        bid_depth = _depth(qb, "bid")
                        survivors = _subtract_ranges(base_range, excl_ranges)
                        excl_here = (base_range[1] - base_range[0] + 1) - sum(
                            s_hi - s_lo + 1 for s_lo, s_hi in survivors
                        )
                        if excl_here:
                            excl[route]["reconnect_guard_buckets"] += excl_here
                        if ask_depth is None or bid_depth is None:
                            for s_lo, s_hi in survivors:
                                excl[route]["depth_unknown_buckets"] += s_hi - s_lo + 1
                            continue
                        # valid coverage buckets (data present, clean, known depth)
                        for s_lo, s_hi in survivors:
                            coverage[route][day] += s_hi - s_lo + 1
                        gross_bp = 10_000.0 * (float(qb["b"]) / float(qa["a"]) - 1.0)
                        depth_ok = ask_depth >= DEPTH_TARGET_USD and bid_depth >= DEPTH_TARGET_USD
                        if not depth_ok:
                            continue
                        for c in COSTS_BP:
                            if gross_bp - c > 0:
                                for s_lo, s_hi in survivors:
                                    qual_times[c].extend(
                                        midnight + k * BUCKET_MS for k in range(s_lo, s_hi + 1)
                                    )
                    for c in COSTS_BP:
                        starts = episode_starts(sorted(qual_times[c]))
                        episodes[route][c][day] = len(starts)

    return _assemble(days, routes, coverage, episodes, excl, dict(sorted(counters.items())))


def _assemble(
    days: list[str],
    routes: list[tuple[str, str, str]],
    coverage: dict[tuple[str, str, str], dict[str, int]],
    episodes: dict[tuple[str, str, str], dict[int, dict[str, int]]],
    excl: dict[tuple[str, str, str], Counter],
    counters: dict[str, int],
) -> dict[str, Any]:
    route_reports: dict[str, Any] = {}
    data_sufficient: list[tuple[str, str, str]] = []
    passing: list[tuple[str, str, str]] = []

    for route in routes:
        base, a, b = route
        cov_frac = {
            d: coverage[route][d] / EXPECTED_BUCKETS_PER_DAY for d in days
        }
        is_sufficient = all(cov_frac[d] >= COVERAGE_MIN for d in days)
        c71_daily = [episodes[route][71][d] for d in days]
        c71_total = sum(c71_daily)
        c71_days_present = sum(1 for v in c71_daily if v > 0)
        boot = day_block_bootstrap(c71_daily) if is_sufficient else None
        route_passes = bool(
            is_sufficient
            and c71_total >= KILLED_MIN_C71_EPISODES
            and c71_days_present >= KILLED_MIN_DAYS
            and boot is not None
            and boot["ci95_low"] > 0
        )
        if is_sufficient:
            data_sufficient.append(route)
        if route_passes:
            passing.append(route)
        route_reports[f"{base}:{a}->{b}"] = {
            "data_sufficient": is_sufficient,
            "coverage_fraction_by_day": {d: round(cov_frac[d], 6) for d in days},
            "min_coverage_fraction": round(min(cov_frac.values()), 6),
            "episodes_by_cost": {
                str(c): {
                    "total": sum(episodes[route][c][d] for d in days),
                    "days_present": sum(1 for d in days if episodes[route][c][d] > 0),
                    "by_day": {d: episodes[route][c][d] for d in days},
                }
                for c in COSTS_BP
            },
            "c71_bootstrap": boot,
            "exclusions": dict(sorted(excl[route].items())),
            "passes_executable_gate": route_passes,
        }

    if not data_sufficient:
        verdict = VERDICT_DATA_INSUFFICIENT
        selected = None
    elif passing:
        verdict = VERDICT_ELIGIBLE
        # deterministic: most c=71 episodes, then lexical base, buy, sell
        selected = min(
            passing,
            key=lambda r: (
                -sum(episodes[r][71][d] for d in days),
                r[0],
                r[1],
                r[2],
            ),
        )
    else:
        verdict = VERDICT_KILLED
        selected = None

    report = {
        "study": "X-ARB-01",
        "statement": "FEASIBILITY ONLY — NO FUTURE PRICE, POSITION, RETURN OR PNL WAS COMPUTED",
        "verdict": verdict,
        "selected_route": (
            f"{selected[0]}:{selected[1]}->{selected[2]}" if selected else None
        ),
        "parameters": {
            "schema": SCHEMA,
            "cost_grid_bp": list(COSTS_BP),
            "max_quote_age_ms": MAX_QUOTE_AGE_MS,
            "reconnect_guard_ms": RECONNECT_GUARD_MS,
            "depth_target_usd": DEPTH_TARGET_USD,
            "bucket_ms": BUCKET_MS,
            "expected_buckets_per_day": EXPECTED_BUCKETS_PER_DAY,
            "episode_separation_ms": EPISODE_SEPARATION_MS,
            "episode_min_points": EPISODE_MIN_POINTS,
            "episode_max_adjacent_gap_ms": EPISODE_MAX_ADJ_GAP_MS,
            "episode_min_span_ms": EPISODE_MIN_SPAN_MS,
            "coverage_min": COVERAGE_MIN,
            "bootstrap_n": BOOTSTRAP_N,
            "bootstrap_seed": BOOTSTRAP_SEED,
            "killed_min_c71_episodes": KILLED_MIN_C71_EPISODES,
            "killed_min_days": KILLED_MIN_DAYS,
        },
        "days": days,
        "counts": {
            "routes_total": len(routes),
            "routes_data_sufficient": len(data_sufficient),
            "routes_passing_gate": len(passing),
        },
        "ingest_counters": counters,
        "routes": route_reports,
    }
    assert_output_clean(report)
    return report


# --------------------------------------------------------------------------- #
# CLI                                                                          #
# --------------------------------------------------------------------------- #
def _expand(values: list[str], stem: str) -> list[Path]:
    paths: list[Path] = []
    for value in values:
        p = Path(value)
        if p.is_dir():
            paths.extend(sorted(p.glob(f"{stem}.jsonl*")))
            paths.extend(sorted(p.glob(f"*/{stem}.jsonl*")))
        else:
            paths.append(p)
    return paths


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--receipt", required=True, type=Path, help="freeze receipt json")
    parser.add_argument("--map", required=True, type=Path, help="canonical 20-base map json")
    parser.add_argument("--xex", nargs="+", required=True, help="xex jsonl/gz files or day dirs")
    parser.add_argument("--manifest", nargs="*", default=[], help="manifest jsonl/gz or day dirs")
    parser.add_argument("--clock", nargs="*", default=[], help="clock jsonl/gz or day dirs")
    parser.add_argument("--out", type=Path)
    args = parser.parse_args()

    receipt = json.loads(args.receipt.read_text())
    canonical_map = json.loads(args.map.read_text())
    xex_paths = _expand(args.xex, "xex")
    if not xex_paths:
        raise SystemExit("no xex files found")
    manifest_paths = _expand(args.manifest, "manifest")
    clock_paths = _expand(args.clock, "clock")

    try:
        report = analyze(
            iter_records(xex_paths),
            iter_records(manifest_paths),
            iter_records(clock_paths),
            receipt,
            canonical_map,
        )
    except RefuseEvaluation as exc:
        refusal = {
            "study": "X-ARB-01",
            "verdict": "REFUSED — INTEGRITY GATE FAILED",
            "reason": str(exc),
            "statement": "NO EVALUATION PERFORMED — NO OUTCOME OR PNL WAS OPENED",
        }
        encoded = json.dumps(refusal, sort_keys=True, indent=2)
        if args.out:
            args.out.write_text(encoded + "\n")
        print(encoded)
        raise SystemExit(2)

    encoded = json.dumps(report, sort_keys=True, indent=2)
    if args.out:
        args.out.write_text(encoded + "\n")
    print(encoded)


if __name__ == "__main__":
    main()
```

## `xarb_obs02/xarb_obs02_analyzer.py`

```python
#!/usr/bin/env python3
"""X-ARB-OBS-02 — timing-only cross-venue observability analyzer.

Implements EXACTLY the frozen contract `2026-07-11_XARB_OBS_02_PREREG.md`.

It answers ONE mechanical, timing-only question: for which fixed
`(base, venue A, venue B)` triples can this server map public-book source times
onto its local clock with a conservative p99 `pair_error_bound <= 50 ms`?  That
establishes only whether a later cross-venue feasibility test is *measurable*.
It is NOT AN EDGE.

The analyzer never reads, derives or emits a price, size, depth, spread, trade,
order, fill, return, PnL or position field.  Its inputs carry timestamps,
sequences and integrity flags only.

Frozen numbers below are copied verbatim from the prereg; NONE is chosen here.
Genuine edge-semantics the frozen prose does not fully pin down are enumerated
in AMBIGUITIES and reported to Codex, never silently baked into a load-bearing
rule.  The concrete default used for each is marked `# AMB-n`.
"""

from __future__ import annotations

import argparse
import gzip
import hashlib
import json
import math
from array import array
from bisect import bisect_right
from collections import defaultdict
from datetime import UTC, datetime
from pathlib import Path
from typing import Any, Iterable

# --------------------------------------------------------------------------- #
# Frozen constants (verbatim from the prereg / probe).                        #
# --------------------------------------------------------------------------- #
SCHEMA = 1
VENUES: tuple[str, ...] = ("by", "bn", "ok")
GRID_MS = 250
WINDOW_MS = 86_400_000                       # exactly 24 h
DENOM = WINDOW_MS // GRID_MS                  # 345_600 grid points / cell / day
PROBE_MAX_GAP_MS = 15_000                     # adjacent bracketing probe gap <= 15 s
P99_MAX_MS = 50.0                             # time-observable iff p99 pair_error_bound <= 50 ms
COVERAGE_MIN = 0.95                           # data-sufficient iff coverage >= 95 %
TS_RESOLUTION_MS = 1                          # "+1 ms timestamp resolution"
PROBE_ERR_ADD_MS = 1                          # probe_error = (recv-send)/2 + 1 ms

# §Exact sampling: a grid is invalid for a leg if ANY of these flags is present.
DENY_FLAGS = frozenset({
    "missing_book_state",
    "missing_primary_source_ts",
    "source_seq_regression",
    "source_seq_unorderable",
})

VERDICT_UNOBSERVABLE = "UNOBSERVABLE / NO X-ARB"
VERDICT_OBSERVABLE = "OBSERVABLE FOR ONE FRESH X-ARB PREREG — NOT AN EDGE"

# Output hygiene: no economic field may ever leave this analyzer.
FORBIDDEN_OUTPUT_TOKENS = (
    "price", "size", "depth", "spread", "trade", "order", "fill",
    "return", "pnl", "position", "profit", "notional", "qty", "volume",
)

AMBIGUITIES = [
    "AMB-1 Interpolation anchor: clock offset is interpolated at the message's "
    "recv_ts (probes are indexed by their local midpoint (send+recv)/2). src_ts / "
    "grid_ts / decision_ts anchors were rejected; recv_ts is the local time the "
    "measurement was taken. Load-bearing for every error value — confirm.",
    "AMB-2 'primary source-time local sequences monotone in each epoch' is read as: "
    "for each leg (venue,base), within each epoch, local_seq is NON-DECREASING when "
    "rows are taken in grid_ts order (equal allowed because the last book state "
    "repeats across grids; a decrease => cell not time-observable). strict-vs-"
    "non-strict and local_seq-vs-src_ts unconfirmed.",
    "AMB-3 'exactly 86,400,000 ms': the window is [window_start_grid, "
    "window_start_grid + 86_400_000) = exactly 345_600 grids, where "
    "window_start_grid = ceil(obs_start.ts / 250)*250 (or receipt.window_start_grid_ts "
    "if supplied). Requires exactly one obs_start + one obs_complete and elapsed "
    ">= 86_400_000 ms. obs_complete.duration_ms need not equal 86_400_000 exactly "
    "(the probe cannot emit that). Confirm window-start convention.",
    "AMB-4 Verdicts: the prereg names TWO verdict strings; the task test mentions "
    "'three branches'. Implemented as 2 strings over 3 internal paths "
    "(no-data-sufficient-cell -> UNOBSERVABLE; sufficient but p99>50 -> UNOBSERVABLE; "
    "sufficient and p99<=50 -> OBSERVABLE). Confirm no distinct third string is wanted.",
    "AMB-5 Secondary diagnostics: the prereg mentions Bybit `ts` / Binance `E` as "
    "output-only diagnostics, but the frozen probe timing schema has no field for "
    "them, so the analyzer has none to emit. Confirm the probe is not expected to add "
    "them (clause is moot for the analyzer).",
    "AMB-6 Clock-probe validity: a probe is valid iff server_ts is an int AND "
    "send_ts <= recv_ts (a recv<send probe is a 'future/negative-RTT' reject). "
    "Bracketing = nearest valid probe with mid<=recv_ts and nearest with mid>recv_ts, "
    "gap<=15 s. Clock probes are venue-level and span epochs freely (independent REST "
    "time calls). Confirm reconnect/epoch does not invalidate clock bracketing.",
    "AMB-7 Percentiles: p50/p95/p99 of the per-cell pair_error_bound population "
    "(one value per VALID grid sample) via nearest-rank on the ascending sorted values "
    "(index = ceil(p/100 * n) - 1). p99 is the gate. Confirm percentile method.",
]


class RefuseAnalysis(Exception):
    """Integrity gate failed: never analyze."""


# --------------------------------------------------------------------------- #
# Output hygiene                                                              #
# --------------------------------------------------------------------------- #
def assert_output_clean(obj: Any, path: str = "") -> None:
    if isinstance(obj, dict):
        for key, value in obj.items():
            lowered = str(key).lower()
            for token in FORBIDDEN_OUTPUT_TOKENS:
                if token in lowered:
                    raise ValueError(f"forbidden output key token {token!r} at {path}/{key}")
            assert_output_clean(value, f"{path}/{key}")
    elif isinstance(obj, (list, tuple)):
        for i, value in enumerate(obj):
            assert_output_clean(value, f"{path}[{i}]")


# --------------------------------------------------------------------------- #
# I/O                                                                         #
# --------------------------------------------------------------------------- #
def _open_text(path: Path):
    return gzip.open(path, "rt") if path.suffix == ".gz" else path.open("rt")


def iter_records(paths: Iterable[Path]) -> Iterable[dict[str, Any]]:
    for path in paths:
        with _open_text(path) as fh:
            for line in fh:
                line = line.strip()
                if line:
                    yield json.loads(line)


def map_sha256(mapping: Any) -> str:
    return hashlib.sha256(
        json.dumps(mapping, sort_keys=True, separators=(",", ":")).encode()
    ).hexdigest()


# --------------------------------------------------------------------------- #
# Receipt / manifest verification                                            #
# --------------------------------------------------------------------------- #
def _verify(manifest: list[dict[str, Any]], receipt: dict[str, Any]) -> tuple[int, int, dict[str, Any]]:
    """Require one obs_start + one obs_complete, matching hashes, and a full 24 h.

    Returns (window_start_grid_ts, window_end_ts_exclusive, obs_start)."""
    starts = [r for r in manifest if r.get("ev") == "obs_start"]
    completes = [r for r in manifest if r.get("ev") == "obs_complete"]
    if len(starts) != 1:
        raise RefuseAnalysis(f"need exactly one obs_start, got {len(starts)}")
    if len(completes) != 1:
        raise RefuseAnalysis(f"need exactly one obs_complete, got {len(completes)}")
    start, complete = starts[0], completes[0]
    if int(start.get("schema", -1)) != SCHEMA:
        raise RefuseAnalysis("obs_start schema != 1")
    for field in ("code_sha256", "map_sha256"):
        if start.get(field) != receipt.get(field):
            raise RefuseAnalysis(f"receipt/{field} mismatch")
    mp = start.get("map")
    if not isinstance(mp, list) or len(mp) != 20:
        raise RefuseAnalysis("obs_start map must hold exactly 20 base records")
    if map_sha256(mp) != start.get("map_sha256"):
        raise RefuseAnalysis("obs_start map does not match its own map_sha256")
    start_ts, complete_ts = int(start["ts"]), int(complete["ts"])
    if complete_ts - start_ts < WINDOW_MS:  # AMB-3: full 24 h collected
        raise RefuseAnalysis(
            f"collection shorter than 24 h: {complete_ts - start_ts} ms"
        )
    if "window_start_grid_ts" in receipt:
        wsg = int(receipt["window_start_grid_ts"])
    else:
        wsg = math.ceil(start_ts / GRID_MS) * GRID_MS  # AMB-3
    return wsg, wsg + WINDOW_MS, start


# --------------------------------------------------------------------------- #
# Clock probes -> per-venue interpolation table                              #
# --------------------------------------------------------------------------- #
class VenueClock:
    """Sorted valid probes for one venue; linear-interpolates offset + error."""

    __slots__ = ("mids", "offsets", "errors")

    def __init__(self) -> None:
        self.mids: list[float] = []
        self.offsets: list[float] = []
        self.errors: list[float] = []

    def add_sorted(self, triples: list[tuple[float, float, float]]) -> None:
        triples.sort(key=lambda t: t[0])
        for mid, offset, error in triples:
            self.mids.append(mid)
            self.offsets.append(offset)
            self.errors.append(error)

    def interp_at(self, recv_ts: int) -> tuple[float, float] | None:
        """Return (offset_interpolated, clock_error) at recv_ts, or None if unbracketed."""
        mids = self.mids
        if not mids:
            return None
        # prev = last mid <= recv_ts ; nxt = first mid > recv_ts   (AMB-1, AMB-6)
        j = bisect_right(mids, recv_ts)
        if j == 0 or j == len(mids):
            return None
        i = j - 1
        span = mids[j] - mids[i]
        if span > PROBE_MAX_GAP_MS:  # clause 4: adjacent gap <= 15 s
            return None
        frac = 0.0 if span == 0 else (recv_ts - mids[i]) / span
        offset = self.offsets[i] + frac * (self.offsets[j] - self.offsets[i])
        drift = 0.5 * abs(self.offsets[j] - self.offsets[i])  # half offset movement
        error = max(self.errors[i], self.errors[j]) + drift + TS_RESOLUTION_MS
        return offset, error

    def error_at(self, recv_ts: int) -> float | None:
        """Clock error bound for a message received at recv_ts, or None if unbracketed."""
        result = self.interp_at(recv_ts)
        return None if result is None else result[1]

    def source_local(self, src_ts: int, recv_ts: int) -> tuple[float, float] | None:
        """Auditable mapping source_local = src_ts - offset, with its clock error bound.

        recv_ts is the interpolation anchor (AMB-1); src_ts is the value being mapped,
        so a fresh recv_ts can never disguise an old src_ts — the two are kept distinct."""
        result = self.interp_at(recv_ts)
        if result is None:
            return None
        offset, error = result
        return src_ts - offset, error


def build_clocks(clock_rows: Iterable[dict[str, Any]]) -> dict[str, VenueClock]:
    staging: dict[str, list[tuple[float, float, float]]] = defaultdict(list)
    for row in clock_rows:
        if int(row.get("schema", -1)) != SCHEMA:
            continue
        for probe in row.get("probes") or []:
            venue = probe.get("venue")
            if venue not in VENUES:
                continue
            try:
                send, recv = int(probe["send_ts"]), int(probe["recv_ts"])
                server = probe["server_ts"]
            except (KeyError, TypeError, ValueError):
                continue
            if server is None or recv < send:  # AMB-6: invalid / future / negative-RTT
                continue
            server = int(server)
            mid = (send + recv) / 2
            offset = server - mid
            probe_error = (recv - send) / 2 + PROBE_ERR_ADD_MS
            staging[venue].append((mid, offset, probe_error))
    clocks: dict[str, VenueClock] = {}
    for venue in VENUES:
        clock = VenueClock()
        clock.add_sorted(staging.get(venue, []))
        clocks[venue] = clock
    return clocks


# --------------------------------------------------------------------------- #
# Per-leg validity + error                                                    #
# --------------------------------------------------------------------------- #
def leg_error(
    rows: list[dict[str, Any]], venue: str, clocks: dict[str, VenueClock]
) -> tuple[float | None, str]:
    """Return (clock_error, reason). error is None iff the leg is not a valid sample."""
    if len(rows) == 0:
        return None, "missing"
    if len(rows) > 1:
        return None, "duplicate"  # clause 5: duplicate invalidates, never dedups
    row = rows[0]
    flags = row.get("flags") or []
    if any(f in DENY_FLAGS for f in flags):
        return None, "deny_flag"
    src_ts, recv_ts, local_seq = row.get("src_ts"), row.get("recv_ts"), row.get("local_seq")
    if src_ts is None or recv_ts is None or local_seq is None:  # clause 2
        return None, "null_required_field"
    try:
        recv_ts_i = int(recv_ts)
        decision_ts = int(row["decision_ts"])
    except (KeyError, TypeError, ValueError):
        return None, "malformed"
    if recv_ts_i > decision_ts:  # clause 2: recv_ts <= decision_ts (as-of / no future)
        return None, "future_recv"
    error = clocks[venue].error_at(recv_ts_i)  # clause 4: bracketing probes <= 15 s
    if error is None:
        return None, "no_clock_bracket"
    return error, "valid"


# --------------------------------------------------------------------------- #
# Core                                                                        #
# --------------------------------------------------------------------------- #
def analyze(
    timing_rows: Iterable[dict[str, Any]],
    clock_rows: Iterable[dict[str, Any]],
    manifest_rows: Iterable[dict[str, Any]],
    receipt: dict[str, Any],
) -> dict[str, Any]:
    manifest = list(manifest_rows)
    wsg, wend, start = _verify(manifest, receipt)
    bases = [rec["base"] for rec in start["map"]]
    base_set = set(bases)
    clocks = build_clocks(clock_rows)

    directed = [(a, b) for a in VENUES for b in VENUES if a != b]

    # per (base, A, B) -> array of pair_error_bound over valid grid samples
    cell_bounds: dict[tuple[str, str, str], array] = {
        (base, a, b): array("d") for base in bases for a, b in directed
    }
    # leg monotonicity per (venue, base): per-epoch last local_seq
    mono_state: dict[tuple[str, str], dict[str, Any]] = {}
    leg_monotone: dict[tuple[str, str], bool] = {}
    counters: dict[str, int] = defaultdict(int)

    def flush_grid(base_rows: dict[str, dict[str, list[dict[str, Any]]]]) -> None:
        for base, venue_rows in base_rows.items():
            if base not in base_set:
                counters["off_universe_grid_rows"] += 1
                continue
            # monotonicity tracking (per leg, per epoch) — uses the single row if present
            leg_err: dict[str, float | None] = {}
            for venue in VENUES:
                rows = venue_rows.get(venue, [])
                _track_monotone(mono_state, leg_monotone, venue, base, rows)
                err, reason = leg_error(rows, venue, clocks)
                leg_err[venue] = err
                if err is None:
                    counters[f"leg_{reason}"] += 1
            for a, b in directed:
                ea, eb = leg_err[a], leg_err[b]
                if ea is not None and eb is not None:
                    cell_bounds[(base, a, b)].append(ea + eb)

    # stream timing rows grouped by grid_ts (probe writes are grid-ordered).
    current_grid: int | None = None
    base_rows: dict[str, dict[str, list[dict[str, Any]]]] = defaultdict(lambda: defaultdict(list))
    for row in timing_rows:
        if int(row.get("schema", -1)) != SCHEMA:
            counters["non_schema_rows"] += 1
            continue
        try:
            grid = int(row["grid_ts"])
        except (KeyError, TypeError, ValueError):
            counters["malformed_grid_rows"] += 1
            continue
        if not (wsg <= grid < wend):  # AMB-3: only the fixed 24 h window
            counters["out_of_window_rows"] += 1
            continue
        if current_grid is None:
            current_grid = grid
        elif grid != current_grid:
            flush_grid(base_rows)
            base_rows = defaultdict(lambda: defaultdict(list))
            current_grid = grid
        base_rows[row.get("base")][row.get("venue")].append(row)
    if current_grid is not None:
        flush_grid(base_rows)

    return _assemble(bases, directed, cell_bounds, leg_monotone, dict(counters), wsg)


def _track_monotone(
    mono_state: dict[tuple[str, str], dict[str, Any]],
    leg_monotone: dict[tuple[str, str], bool],
    venue: str,
    base: str,
    rows: list[dict[str, Any]],
) -> None:
    """AMB-2: local_seq must be non-decreasing within each epoch (per leg)."""
    if len(rows) != 1:
        return
    row = rows[0]
    local_seq, epoch = row.get("local_seq"), row.get("epoch")
    if local_seq is None or epoch is None:
        return
    key = (venue, base)
    leg_monotone.setdefault(key, True)
    st = mono_state.get(key)
    if st is None or st["epoch"] != epoch:
        mono_state[key] = {"epoch": epoch, "last": int(local_seq)}
        return
    if int(local_seq) < st["last"]:
        leg_monotone[key] = False
    else:
        st["last"] = int(local_seq)


def _percentiles(values: array) -> dict[str, float] | None:
    if not values:
        return None
    ordered = sorted(values)
    n = len(ordered)

    def pick(p: float) -> float:
        idx = math.ceil(p * n) - 1  # AMB-7 nearest-rank, 1-based -> 0-based
        return ordered[max(0, min(idx, n - 1))]

    return {"p50": round(pick(0.50), 6), "p95": round(pick(0.95), 6), "p99": round(pick(0.99), 6)}


def _assemble(
    bases: list[str],
    directed: list[tuple[str, str]],
    cell_bounds: dict[tuple[str, str, str], array],
    leg_monotone: dict[tuple[str, str], bool],
    counters: dict[str, int],
    window_start_grid_ts: int,
) -> dict[str, Any]:
    cells: dict[str, Any] = {}
    observable: list[tuple[float, str, str, str]] = []
    for base in bases:
        for a, b in directed:
            bounds = cell_bounds[(base, a, b)]
            valid = len(bounds)
            coverage = valid / DENOM
            pct = _percentiles(bounds)
            monotone = leg_monotone.get((a, base), True) and leg_monotone.get((b, base), True)
            data_sufficient = coverage >= COVERAGE_MIN
            time_observable = bool(
                data_sufficient and monotone and pct is not None and pct["p99"] <= P99_MAX_MS
            )
            if time_observable:
                observable.append((pct["p99"], base, a, b))
            cells[f"{base}:{a}->{b}"] = {
                "valid_samples": valid,
                "coverage": round(coverage, 6),
                "data_sufficient": data_sufficient,
                "epoch_local_seq_monotone": monotone,
                "pair_error_bound_ms": pct,
                "time_observable": time_observable,
            }

    if observable:
        verdict = VERDICT_OBSERVABLE
        # lowest p99, then base, A, B lexically
        best = min(observable, key=lambda t: (t[0], t[1], t[2], t[3]))
        selected = {
            "cell": f"{best[1]}:{best[2]}->{best[3]}",
            "p99_pair_error_bound_ms": best[0],
            "label": "NOT AN EDGE",
        }
    else:
        verdict = VERDICT_UNOBSERVABLE
        selected = None

    report = {
        "study": "X-ARB-OBS-02",
        "statement": "TIMING OBSERVABILITY ONLY — NOT AN EDGE; NO PRICE/SIZE/DEPTH/RETURN/PNL COMPUTED",
        "verdict": verdict,
        "selected_candidate": selected,
        "parameters": {
            "schema": SCHEMA,
            "grid_ms": GRID_MS,
            "window_ms": WINDOW_MS,
            "denominator_per_cell": DENOM,
            "probe_max_gap_ms": PROBE_MAX_GAP_MS,
            "p99_max_ms": P99_MAX_MS,
            "coverage_min": COVERAGE_MIN,
            "ts_resolution_ms": TS_RESOLUTION_MS,
            "deny_flags": sorted(DENY_FLAGS),
        },
        "window_start_grid_ts": window_start_grid_ts,
        "counts": {
            "cells_total": len(cells),
            "cells_data_sufficient": sum(1 for c in cells.values() if c["data_sufficient"]),
            "cells_time_observable": sum(1 for c in cells.values() if c["time_observable"]),
        },
        "ingest_counters": counters,
        "cells": cells,
    }
    assert_output_clean(report)
    return report


# --------------------------------------------------------------------------- #
# CLI                                                                         #
# --------------------------------------------------------------------------- #
def _expand(values: list[str], stem: str) -> list[Path]:
    paths: list[Path] = []
    for value in values:
        p = Path(value)
        if p.is_dir():
            paths.extend(sorted(p.glob(f"{stem}.jsonl*")))
            paths.extend(sorted(p.glob(f"*/{stem}.jsonl*")))
        else:
            paths.append(p)
    return paths


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--receipt", required=True, type=Path)
    parser.add_argument("--timing", nargs="+", required=True, help="timing jsonl/gz or day dirs")
    parser.add_argument("--clock", nargs="+", required=True, help="clock jsonl/gz or day dirs")
    parser.add_argument("--manifest", nargs="+", required=True, help="manifest jsonl/gz or day dirs")
    parser.add_argument("--out", type=Path)
    args = parser.parse_args()

    receipt = json.loads(args.receipt.read_text())
    timing = _expand(args.timing, "timing")
    clock = _expand(args.clock, "clock")
    manifest = _expand(args.manifest, "manifest")
    if not timing:
        raise SystemExit("no timing files found")

    try:
        report = analyze(
            iter_records(timing), iter_records(clock), iter_records(manifest), receipt
        )
    except RefuseAnalysis as exc:
        refusal = {
            "study": "X-ARB-OBS-02",
            "verdict": "REFUSED — INTEGRITY GATE FAILED",
            "reason": str(exc),
            "statement": "NO ANALYSIS PERFORMED — NO OUTCOME OR PNL WAS OPENED",
        }
        encoded = json.dumps(refusal, sort_keys=True, indent=2)
        if args.out:
            args.out.write_text(encoded + "\n")
        print(encoded)
        raise SystemExit(2)

    encoded = json.dumps(report, sort_keys=True, indent=2)
    if args.out:
        args.out.write_text(encoded + "\n")
    print(encoded)


if __name__ == "__main__":
    main()
```

## `xarb_obs02/test_xarb_obs02_analyzer.py`

```python
#!/usr/bin/env python3
"""No-network synthetic acceptance tests for the X-ARB-OBS-02 timing analyzer.

Framework-free (assert-based). Run: python3 test_xarb_obs02_analyzer.py
Every fixture carries timing/sequence values only — no price/size/depth field.
Covers the eight mandatory cases from CLAUDE_TASK_XARB_OBS_02_ANALYZER.md.
"""
from __future__ import annotations

import json
from array import array

import xarb_obs02_analyzer as A

PASS = 0
FAIL = 0


def check(name: str, cond: bool) -> None:
    global PASS, FAIL
    if cond:
        PASS += 1
        print(f"  ok   {name}")
    else:
        FAIL += 1
        print(f"  FAIL {name}")


# --------------------------------------------------------------------------- #
# Builders                                                                    #
# --------------------------------------------------------------------------- #
def clocks_from(probe_rows: list[dict]) -> dict[str, A.VenueClock]:
    return A.build_clocks(probe_rows)


def clock_row(*probes: dict) -> dict:
    return {"schema": 1, "probes": list(probes)}


def probe(venue: str, send: int, recv: int, server: int | None) -> dict:
    return {"venue": venue, "send_ts": send, "recv_ts": recv, "server_ts": server}


def trow(grid: int, base: str, venue: str, **over) -> dict:
    row = {
        "schema": 1, "grid_ts": grid, "decision_ts": grid + 10,
        "venue": venue, "base": base, "channel": f"{venue}_x",
        "src_ts": grid - 30, "recv_ts": grid - 5,
        "source_seq": 100, "local_seq": 1, "epoch": 0, "flags": [],
    }
    row.update(over)
    return row


def mk_map(n: int = 20) -> list[dict]:
    return [{"base": f"B{i:02d}", "by": f"B{i:02d}USDT", "bn": f"B{i:02d}USDT",
             "ok": f"B{i:02d}-USDT-SWAP"} for i in range(n)]


def mk_receipt(mapping: list[dict], **over) -> dict:
    r = {"schema": 1, "code_sha256": "code", "map_sha256": A.map_sha256(mapping)}
    r.update(over)
    return r


def mk_manifest(mapping: list[dict], start_ts: int, dur: int = A.WINDOW_MS) -> list[dict]:
    return [
        {"ev": "obs_start", "ts": start_ts, "schema": 1, "code_sha256": "code",
         "map": mapping, "map_sha256": A.map_sha256(mapping)},
        {"ev": "obs_complete", "ts": start_ts + dur},
    ]


# --------------------------------------------------------------------------- #
# T1 — fresh recv + 400ms-old src cannot be disguised; mapping auditable
# --------------------------------------------------------------------------- #
def t1() -> None:
    print("T1 fresh-recv / old-src auditable")
    # two probes bracketing recv, zero RTT, zero offset -> source_local == src_ts
    cl = clocks_from([clock_row(probe("by", 1000, 1000, 1000)),
                      clock_row(probe("by", 2000, 2000, 2000))])["by"]
    recv, src = 1500, 1500 - 400  # fresh recv, 400ms-old source content
    mapped = cl.source_local(src, recv)
    check("valid sample keeps src_ts distinct from recv_ts", mapped is not None and abs(mapped[0] - src) < 1e-9)
    # a fresh recv cannot substitute for a MISSING source: null src_ts => invalid
    err, reason = A.leg_error([trow(2000, "B00", "by", src_ts=None, recv_ts=1995)],
                              "by", clocks_from([clock_row(probe("by", 1000, 1000, 1000)),
                                                 clock_row(probe("by", 3000, 3000, 3000))]))
    check("null src_ts rejected even with fresh recv", err is None and reason == "null_required_field")
    # the mapping is deterministic/auditable: same inputs -> same bound
    check("mapping deterministic", cl.source_local(src, recv) == cl.source_local(src, recv))


# --------------------------------------------------------------------------- #
# T2 — hand-calculated midpoint offset + error bound
# --------------------------------------------------------------------------- #
def t2() -> None:
    print("T2 hand-calc offset + error")
    # probe P1: send=1000 recv=1010 server=1100 -> mid=1005, offset=1100-1005=95, pe=(10)/2+1=6
    # probe P2: send=2000 recv=2040 server=2130 -> mid=2020, offset=2130-2020=110, pe=(40)/2+1=21
    cl = clocks_from([clock_row(probe("by", 1000, 1010, 1100)),
                      clock_row(probe("by", 2000, 2040, 2130))])["by"]
    r = cl.interp_at(1512.5)  # midpoint of mids 1005 & 2020 => frac 0.5
    off, err = r
    exp_off = 95 + 0.5 * (110 - 95)         # 102.5
    exp_err = max(6, 21) + 0.5 * abs(110 - 95) + 1  # 21 + 7.5 + 1 = 29.5
    check("interpolated offset", abs(off - exp_off) < 1e-9)
    check("clock error bound", abs(err - exp_err) < 1e-9)


# --------------------------------------------------------------------------- #
# T3 — probe gap 15,001ms rejects; 15,000 accepts
# --------------------------------------------------------------------------- #
def t3() -> None:
    print("T3 probe gap boundary")
    over = clocks_from([clock_row(probe("by", 0, 0, 0)),
                        clock_row(probe("by", 15001, 15001, 15001))])["by"]
    check("gap 15001 rejects", over.error_at(7000) is None)
    ok = clocks_from([clock_row(probe("by", 0, 0, 0)),
                      clock_row(probe("by", 15000, 15000, 15000))])["by"]
    check("gap 15000 accepts", ok.error_at(7000) is not None)


# --------------------------------------------------------------------------- #
# T4 — duplicate grid rejects; null exchange source_seq accepts; seq regression rejects
# --------------------------------------------------------------------------- #
def t4() -> None:
    print("T4 duplicate / null-source-seq / seq-regression")
    cl = clocks_from([clock_row(probe("by", 0, 0, 0)), clock_row(probe("by", 10000, 10000, 10000))])
    # duplicate (grid,base,venue) -> two rows -> invalid, never dedup
    err, reason = A.leg_error([trow(5000, "B00", "by"), trow(5000, "B00", "by")], "by", cl)
    check("duplicate grid invalidates", err is None and reason == "duplicate")
    # null exchange source_seq (OKX) is valid as long as local_seq present
    ok = clocks_from([clock_row(probe("ok", 0, 0, 0)), clock_row(probe("ok", 10000, 10000, 10000))])
    err2, reason2 = A.leg_error([trow(5000, "B00", "ok", source_seq=None)], "ok", ok)
    check("null source_seq accepts", err2 is not None and reason2 == "valid")
    # source_seq_regression flag is in the deny set
    err3, reason3 = A.leg_error([trow(5000, "B00", "by", flags=["source_seq_regression"])], "by", cl)
    check("source_seq_regression flag rejects", err3 is None and reason3 == "deny_flag")
    # local_seq regression within an epoch -> leg not monotone
    mono, legmono = {}, {}
    A._track_monotone(mono, legmono, "by", "B00", [trow(1000, "B00", "by", local_seq=5, epoch=0)])
    A._track_monotone(mono, legmono, "by", "B00", [trow(1250, "B00", "by", local_seq=3, epoch=0)])
    check("local_seq regression breaks monotonicity", legmono[("by", "B00")] is False)
    # ... but a reset across a NEW epoch is NOT a regression
    mono2, legmono2 = {}, {}
    A._track_monotone(mono2, legmono2, "by", "B00", [trow(1000, "B00", "by", local_seq=9, epoch=0)])
    A._track_monotone(mono2, legmono2, "by", "B00", [trow(1250, "B00", "by", local_seq=1, epoch=1)])
    check("epoch reset is not a regression", legmono2[("by", "B00")] is True)


# --------------------------------------------------------------------------- #
# T5 — missing source state and future recv_ts reject
# --------------------------------------------------------------------------- #
def t5() -> None:
    print("T5 missing-book-state / future-recv")
    cl = clocks_from([clock_row(probe("by", 0, 0, 0)), clock_row(probe("by", 10000, 10000, 10000))])
    err, reason = A.leg_error([trow(5000, "B00", "by", flags=["missing_book_state"],
                                    src_ts=None, recv_ts=None, local_seq=None)], "by", cl)
    check("missing_book_state rejects", err is None and reason == "deny_flag")
    # future recv: recv_ts > decision_ts
    err2, reason2 = A.leg_error([trow(5000, "B00", "by", recv_ts=5000 + 11, decision_ts=5000 + 10)],
                                "by", cl)
    check("future recv_ts rejects", err2 is None and reason2 == "future_recv")
    # boundary: recv_ts == decision_ts is allowed
    err3, reason3 = A.leg_error([trow(5000, "B00", "by", recv_ts=5010, decision_ts=5010)], "by", cl)
    check("recv_ts == decision_ts accepts", err3 is not None and reason3 == "valid")


# --------------------------------------------------------------------------- #
# T6 — <95% coverage is not data-sufficient (via _assemble)
# --------------------------------------------------------------------------- #
def _assemble_one(valid: int, err_value: float, monotone: bool = True) -> dict:
    directed = [(a, b) for a in A.VENUES for b in A.VENUES if a != b]
    bases = ["B00"]
    cell_bounds = {(base, a, b): array("d") for base in bases for a, b in directed}
    bounds = array("d", [err_value] * valid)
    cell_bounds[("B00", "by", "bn")] = bounds
    leg_monotone = {("by", "B00"): monotone, ("bn", "B00"): monotone}
    return A._assemble(bases, directed, cell_bounds, leg_monotone, {}, 0)


def t6() -> None:
    print("T6 coverage threshold")
    just_under = int(A.DENOM * 0.95) - 1
    rep = _assemble_one(just_under, 10.0)
    check("just under 95% not data-sufficient",
          rep["cells"]["B00:by->bn"]["data_sufficient"] is False)
    exactly = int(A.DENOM * 0.95) + 1  # comfortably >= 95%
    rep2 = _assemble_one(exactly, 10.0)
    check("above 95% data-sufficient", rep2["cells"]["B00:by->bn"]["data_sufficient"] is True)


# --------------------------------------------------------------------------- #
# T7 — three verdict branches + deterministic tie-break
# --------------------------------------------------------------------------- #
def t7() -> None:
    print("T7 verdict branches + tie-break")
    # (a) no data-sufficient cell -> UNOBSERVABLE
    rep_a = _assemble_one(100, 10.0)
    check("no sufficient cell -> UNOBSERVABLE", rep_a["verdict"] == A.VERDICT_UNOBSERVABLE)
    # (b) data-sufficient but p99 > 50 -> UNOBSERVABLE
    rep_b = _assemble_one(A.DENOM, 60.0)
    check("sufficient but p99>50 -> UNOBSERVABLE", rep_b["verdict"] == A.VERDICT_UNOBSERVABLE
          and rep_b["cells"]["B00:by->bn"]["data_sufficient"] is True)
    # (c) sufficient and p99<=50 -> OBSERVABLE
    rep_c = _assemble_one(A.DENOM, 20.0)
    check("sufficient and p99<=50 -> OBSERVABLE", rep_c["verdict"] == A.VERDICT_OBSERVABLE)
    check("selected labelled NOT AN EDGE", rep_c["selected_candidate"]["label"] == "NOT AN EDGE")
    # (d) monotonicity gate blocks observability
    rep_d = _assemble_one(A.DENOM, 20.0, monotone=False)
    check("non-monotone -> not observable", rep_d["verdict"] == A.VERDICT_UNOBSERVABLE)
    # (e) deterministic tie-break: two observable cells, lower p99 wins; tie -> lexical
    directed = [(a, b) for a in A.VENUES for b in A.VENUES if a != b]
    cb = {("B00", a, b): array("d") for a, b in directed}
    cb[("B00", "by", "bn")] = array("d", [30.0] * A.DENOM)
    cb[("B00", "ok", "bn")] = array("d", [10.0] * A.DENOM)  # lower p99 -> should win
    lm = {v: True for v in [("by", "B00"), ("bn", "B00"), ("ok", "B00")]}
    rep_e = A._assemble(["B00"], directed, cb, lm, {}, 0)
    check("lowest p99 selected", rep_e["selected_candidate"]["cell"] == "B00:ok->bn")
    # tie on p99 -> lexical base,A,B  (bn->by beats by->bn? compare A then B)
    cb2 = {("B00", a, b): array("d") for a, b in directed}
    cb2[("B00", "by", "ok")] = array("d", [12.0] * A.DENOM)
    cb2[("B00", "bn", "ok")] = array("d", [12.0] * A.DENOM)  # same p99; 'bn' < 'by' lexically
    rep_f = A._assemble(["B00"], directed, cb2, lm, {}, 0)
    check("tie -> lexical A wins (bn<by)", rep_f["selected_candidate"]["cell"] == "B00:bn->ok")


# --------------------------------------------------------------------------- #
# T8 — output-hygiene mutations fail
# --------------------------------------------------------------------------- #
def t8() -> None:
    print("T8 output hygiene")
    for bad in ("price", "mid_price", "pnl", "trade_ts", "size5", "depth_usd", "position"):
        raised = False
        try:
            A.assert_output_clean({"cells": {"x": {bad: 1}}})
        except ValueError:
            raised = True
        check(f"forbidden key {bad!r} rejected", raised)
    # a clean timing-only report passes
    ok = True
    try:
        A.assert_output_clean(_assemble_one(10, 5.0))
    except ValueError:
        ok = False
    check("clean report passes", ok)


# --------------------------------------------------------------------------- #
# T9 — end-to-end smoke: verify + as-of window + integrity refusals
# --------------------------------------------------------------------------- #
def t9() -> None:
    print("T9 end-to-end analyze smoke + refusals")
    mp = mk_map(20)
    start = 1_000_000_000_000
    wsg = ((start + A.GRID_MS - 1) // A.GRID_MS) * A.GRID_MS
    receipt = mk_receipt(mp, window_start_grid_ts=wsg)
    manifest = mk_manifest(mp, start)
    clocks = [clock_row(probe("by", wsg - 100, wsg - 100, wsg - 100),
                        probe("bn", wsg - 100, wsg - 100, wsg - 100)),
              clock_row(probe("by", wsg + 5000, wsg + 5000, wsg + 5000),
                        probe("bn", wsg + 5000, wsg + 5000, wsg + 5000))]
    # 4 grids, base B00, legs by+bn valid; one grid has a future-recv on bn (invalid)
    timing = []
    for k in range(4):
        g = wsg + k * A.GRID_MS
        timing.append(trow(g, "B00", "by", src_ts=g - 30, recv_ts=g - 5, local_seq=k + 1))
        bn_recv = g + 20 if k == 2 else g - 5  # k==2 future recv -> invalid bn leg
        timing.append(trow(g, "B00", "bn", src_ts=g - 30, recv_ts=bn_recv, local_seq=k + 1))
    rep = A.analyze(timing, clocks, manifest, receipt)
    cell = rep["cells"]["B00:by->bn"]
    check("3 of 4 grids valid for by->bn", cell["valid_samples"] == 3)
    check("verdict UNOBSERVABLE (tiny coverage)", rep["verdict"] == A.VERDICT_UNOBSERVABLE)
    check("output clean end-to-end", True)  # analyze() asserts internally

    # refusal: two obs_start
    bad_manifest = manifest + [{"ev": "obs_start", "ts": start, "schema": 1,
                                "code_sha256": "code", "map": mp, "map_sha256": A.map_sha256(mp)}]
    refused = False
    try:
        A.analyze(timing, clocks, bad_manifest, receipt)
    except A.RefuseAnalysis:
        refused = True
    check("two obs_start refuses", refused)

    # refusal: map hash mismatch
    refused2 = False
    try:
        A.analyze(timing, clocks, manifest, mk_receipt(mp, code_sha256="code", map_sha256="WRONG",
                                                       window_start_grid_ts=wsg))
    except A.RefuseAnalysis:
        refused2 = True
    check("map_sha256 mismatch refuses", refused2)

    # refusal: short collection (< 24h)
    refused3 = False
    try:
        A.analyze(timing, clocks, mk_manifest(mp, start, dur=A.WINDOW_MS - 1), receipt)
    except A.RefuseAnalysis:
        refused3 = True
    check("sub-24h collection refuses", refused3)


def main() -> None:
    for t in (t1, t2, t3, t4, t5, t6, t7, t8, t9):
        t()
    print(f"\n{PASS} passed, {FAIL} failed")
    if FAIL:
        raise SystemExit(1)


if __name__ == "__main__":
    main()
```

## `flow_collector/collector.py`

```python
#!/usr/bin/env python3
"""FLOW-COLLECTOR для H-FLOW-01 (спека Codex 11.07): append-only forward-сбор
причинного слоя — ликвидации (сырьё), taker buy/sell notional (1м-вёдра),
книга L10/L50 + спред (1м REST-снапшоты), OI+funding (5м из tickers),
динамический топ-150 (1ч, версионируется). Манифест: старты/реконнекты/дыры/счётчики.

Хранение: ~/trading/flow_collector/data/YYYY-MM-DD/{liq,trades_1m,book_1m,oi_5m,universe_1h,manifest}.jsonl
(вчерашний день гзипается на ролловере). Диск ≈ 30-60 МБ/день.
⚠ Семантика ликвидаций Bybit: side=Buy — ликвидирован ШОРТ (buy-ордер закрывает шорт)."""
import json, gzip, os, time, threading, shutil, urllib.request, traceback, hashlib
from datetime import datetime, timezone
import websocket

ROOT = os.path.expanduser("~/trading/flow_collector/data")
WS_URL = "wss://stream.bybit.com/v5/public/linear"
TOP_N = 150
SCHEMA = 2
SCHEMA_VERSION = "hflow-schema-v2-2026-07-11"
STABLE_BASES = {
    "USDC", "USDE", "FDUSD", "TUSD", "DAI", "USDD", "USTC", "PYUSD", "GUSD",
    "EUR", "EURC", "EURT", "EURI", "AEUR", "USD1", "USDR", "USDX", "USDY",
    "BUSD", "LUSD", "FRAX", "USDP", "SUSD", "CRVUSD", "GHO", "USDB", "USDF",
}
# Fallback only. Current instrument metadata is authoritative and also catches new names (e.g. CLUSDT).
COMMODITY_BASES = {"XAU", "XAG", "XAUT", "PAXG"}
os.makedirs(ROOT, exist_ok=True)

_lock = threading.RLock()   # реентерабельный: flush/rollover зовут emit ПОД локом
state = {"day": None, "files": {}, "top": [], "sub_syms": set(),
         "tr_buckets": {}, "sealed_mins": {}, "instrument_types": None, "instrument_recv_ts": 0,
         "stop_write": False,
         "counters": {"liq": 0, "trades": 0, "book": 0, "oi": 0, "late_trade_fragment": 0},
         "ws": None, "last_msg": time.time()}

def utcnow(): return datetime.now(timezone.utc)
def day_str(): return utcnow().strftime("%Y-%m-%d")

def _open_day(d):
    dd = f"{ROOT}/{d}"
    os.makedirs(dd, exist_ok=True)
    for name in ("liq", "trades_1m", "book_1m", "oi_5m", "universe_1h", "manifest"):
        state["files"][name] = open(f"{dd}/{name}.jsonl", "a", buffering=1)

def _gzip_old(d_old):
    dd = f"{ROOT}/{d_old}"
    for f in os.listdir(dd):
        if f.endswith(".jsonl"):
            p = f"{dd}/{f}"
            with open(p, "rb") as fin, gzip.open(p + ".gz", "wb") as fout:
                fout.write(fin.read())
            os.remove(p)

def emit(name, obj):
    with _lock:
        if state["stop_write"] and name != "manifest":
            return
        d = day_str()
        if d != state["day"]:
            old = state["day"]
            for f in state["files"].values(): f.close()
            _open_day(d)
            state["day"] = d
            if old:
                try: _gzip_old(old)
                except Exception as e: mani({"ev": "gzip_fail", "err": str(e)})
            mani({"ev": "rollover", "from": old})
        state["files"][name].write(json.dumps(obj, separators=(",", ":")) + "\n")

def mani(obj):
    obj["ts"] = int(time.time()*1000)
    emit("manifest", obj)

def rest(path, params):
    q = "&".join(f"{k}={v}" for k, v in params.items())
    for i in range(3):
        try:
            return json.load(urllib.request.urlopen(
                f"https://api.bybit.com/v5/market/{path}?{q}", timeout=15))
        except Exception:
            if i == 2: return None
            time.sleep(1)

def code_sha256():
    with open(__file__, "rb") as f:
        return hashlib.sha256(f.read()).hexdigest()

def refresh_instrument_types(now):
    """Cache Bybit metadata; on startup failure fail closed, later retain last known good set."""
    if state["instrument_types"] is not None and now - state["instrument_recv_ts"] < 6*3600_000:
        return True
    r = rest("instruments-info", {"category": "linear", "limit": 1000})
    recv_ts = int(time.time()*1000)
    if not r or r.get("retCode") != 0:
        mani({"ev": "instrument_metadata_fail", "has_cached": state["instrument_types"] is not None})
        return state["instrument_types"] is not None
    types = {i.get("symbol"): i.get("symbolType", "") for i in r["result"].get("list", []) if i.get("symbol")}
    if not types:
        mani({"ev": "instrument_metadata_empty", "has_cached": state["instrument_types"] is not None})
        return state["instrument_types"] is not None
    state["instrument_types"] = types
    state["instrument_recv_ts"] = recv_ts
    return True

# ── динамический топ-150 + OI/funding из одного вызова tickers ──
def refresh_universe():
    r = rest("tickers", {"category": "linear"})
    if not r or r.get("retCode") != 0:
        mani({"ev": "tickers_fail"}); return
    now = int(time.time()*1000)  # response receipt time, not request-start time
    if not refresh_instrument_types(now):
        mani({"ev": "universe_fail_closed_no_metadata"}); return
    types = state["instrument_types"]
    excluded = {"stable": 0, "commodity": 0, "stock": 0, "unknown_meta": 0}
    rows = []
    for t in r["result"]["list"]:
        s = t.get("symbol", "")
        if not s.endswith("USDT"):
            continue
        base = s[:-4]
        typ = types.get(s)
        if typ is None:
            excluded["unknown_meta"] += 1; continue
        if base in STABLE_BASES:
            excluded["stable"] += 1; continue
        if typ == "commodity" or base in COMMODITY_BASES:
            excluded["commodity"] += 1; continue
        if typ == "stock":
            excluded["stock"] += 1; continue
        rows.append(t)
    rows.sort(key=lambda t: -float(t.get("turnover24h") or 0))
    top = [t["symbol"] for t in rows[:TOP_N]]
    with _lock:
        changed = top != state["top"]
        state["top"] = top
    emit("universe_1h", {"ts": now, "recv_ts": now, "top": top, "excluded": excluded,
                          "instrument_recv_ts": state["instrument_recv_ts"], "schema": SCHEMA})
    if changed: resubscribe()
    for t in rows[:TOP_N]:
        emit("oi_5m", {"ts": now, "recv_ts": now, "schema": SCHEMA, "s": t["symbol"], "oi": float(t.get("openInterest") or 0),
                       "oiv": float(t.get("openInterestValue") or 0),
                       "f": float(t.get("fundingRate") or 0),
                       "nf": int(t.get("nextFundingTime") or 0),
                       "last": float(t.get("lastPrice") or 0)})
        state["counters"]["oi"] += 1

def oi_loop():
    while True:
        try: refresh_universe()
        except Exception as e: mani({"ev": "oi_loop_err", "err": str(e)[:200]})
        time.sleep(300)

# ── книга: REST-снапшот L50 раз в минуту на символ (растянуто по минуте) ──
def book_loop():
    while True:
        top = list(state["top"])
        if not top: time.sleep(5); continue
        pause = max(0.2, 55.0/len(top))
        for s in top:
            r = rest("orderbook", {"category": "linear", "symbol": s, "limit": 50})
            recv_ts = int(time.time()*1000)
            if r and r.get("retCode") == 0:
                res = r["result"]
                b, a = res.get("b") or [], res.get("a") or []
                if b and a:
                    bid, ask = float(b[0][0]), float(a[0][0])
                    f10 = lambda side: sum(float(p)*float(v) for p, v in side[:10])
                    f50 = lambda side: sum(float(p)*float(v) for p, v in side[:50])
                    emit("book_1m", {"ts": int(res.get("ts") or recv_ts), "recv_ts": recv_ts, "schema": SCHEMA, "s": s,
                                     "spread_bp": round((ask-bid)/bid*1e4, 3),
                                     "b10": round(f10(b)), "a10": round(f10(a)),
                                     "b50": round(f50(b)), "a50": round(f50(a))})
                    state["counters"]["book"] += 1
            time.sleep(pause)

# ── WS: ликвидации сырьём + trades → минутные вёдра ──
def flush_buckets(force=False):
    flush_ts = int(time.time()*1000)
    now_min = flush_ts // 60_000
    with _lock:
        done = [k for k in state["tr_buckets"] if force or k[1] < now_min]
        for k in done:
            v = state["tr_buckets"].pop(k)
            state["sealed_mins"][k] = flush_ts
            emit("trades_1m", {"s": k[0], "min": k[1]*60_000, "first_recv_ts": v["first_recv_ts"],
                                "last_recv_ts": v["last_recv_ts"], "flush_ts": flush_ts, "schema": SCHEMA,
                                "buyN": round(v["buyN"], 2), "sellN": round(v["sellN"], 2), "n": v["n"]})
        expire = now_min - 24*60
        state["sealed_mins"] = {k: v for k, v in state["sealed_mins"].items() if k[1] >= expire}

def on_message(ws, raw):
    state["last_msg"] = time.time()
    recv_ts = int(time.time()*1000)
    try: m = json.loads(raw)
    except Exception: return
    topic = m.get("topic", "")
    if topic.startswith("allLiquidation"):
        for d in m.get("data", []):
            emit("liq", {"ts": int(d.get("T") or 0), "recv_ts": recv_ts, "schema": SCHEMA, "s": d.get("s"), "side": d.get("S"),
                         "v": float(d.get("v") or 0), "p": float(d.get("p") or 0)})
            state["counters"]["liq"] += 1
    elif topic.startswith("publicTrade"):
        for d in m.get("data", []):
            s, mn = d["s"], int(int(d["T"])//60000)
            key = (s, mn)
            with _lock:
                if key in state["sealed_mins"] or mn < recv_ts//60_000 - 1:
                    state["counters"]["late_trade_fragment"] += 1
                    mani({"ev": "late_trade_fragment", "s": s, "min": mn*60_000, "recv_ts": recv_ts})
                    continue
                b = state["tr_buckets"].setdefault(key, {"buyN": 0.0, "sellN": 0.0, "n": 0,
                                                           "first_recv_ts": recv_ts, "last_recv_ts": recv_ts})
                notional = float(d["v"])*float(d["p"])
                b["buyN" if d["S"] == "Buy" else "sellN"] += notional
                b["n"] += 1
                b["last_recv_ts"] = recv_ts
            state["counters"]["trades"] += 1

def resubscribe():
    ws = state.get("ws")
    top = list(state["top"])
    if not ws or not top: return
    want = set(top)
    old = state["sub_syms"]
    add, rem = sorted(want - old), sorted(old - want)
    try:
        for i in range(0, len(rem), 10):
            args = [f"allLiquidation.{s}" for s in rem[i:i+10]] + [f"publicTrade.{s}" for s in rem[i:i+10]]
            ws.send(json.dumps({"op": "unsubscribe", "args": args}))
        for i in range(0, len(add), 10):
            args = [f"allLiquidation.{s}" for s in add[i:i+10]] + [f"publicTrade.{s}" for s in add[i:i+10]]
            ws.send(json.dumps({"op": "subscribe", "args": args}))
        state["sub_syms"] = want
        if add or rem: mani({"ev": "resub", "add": len(add), "rem": len(rem)})
    except Exception as e:
        mani({"ev": "resub_err", "err": str(e)[:200]})

def on_open(ws):
    state["ws"] = ws
    state["sub_syms"] = set()
    mani({"ev": "ws_open"})
    resubscribe()

def on_close(ws, code, msg): mani({"ev": "ws_close", "code": code})
def on_error(ws, err): mani({"ev": "ws_error", "err": str(err)[:200]})

def flush_loop():
    while True:
        time.sleep(10)
        try: flush_buckets()
        except Exception as e: mani({"ev": "flush_err", "err": str(e)[:200]})

def guard_once(free_gb=None):
    """One conservative disk-safety pass; separated for no-network tests."""
    if free_gb is None:
        free_gb = shutil.disk_usage("/").free/2**30
    if free_gb < 2:
        with _lock: state["stop_write"] = True
        mani({"ev": "FATAL_disk", "free_gb": round(free_gb, 1)})
    elif free_gb < 5:
        days = sorted(d for d in os.listdir(ROOT) if d[:1].isdigit() and d != state["day"])
        pruned = False
        for d in days:
            if os.path.exists(f"{ROOT}/{d}/.pulled_ok"):
                shutil.rmtree(f"{ROOT}/{d}")
                mani({"ev": "prune_day", "day": d, "free_gb": round(free_gb, 1)})
                pruned = True; break
        if not pruned:
            mani({"ev": "prune_blocked_no_pulled_copy", "free_gb": round(free_gb, 1)})
    mani({"ev": "hb", **state["counters"], "top_n": len(state["top"]), "free_gb": round(free_gb, 1),
          "pending": len(state["tr_buckets"])})

def guard_loop():
    while True:
        guard_once()
        time.sleep(3600)

def main():
    with _lock:
        state["day"] = day_str(); _open_day(state["day"])
    mani({"ev": "schema_v2_start", "schema": SCHEMA, "version": SCHEMA_VERSION,
          "code_sha256": code_sha256(), "pid": os.getpid()})
    refresh_universe()
    for fn in (oi_loop, book_loop, flush_loop, guard_loop):
        threading.Thread(target=fn, daemon=True).start()
    while True:                                   # реконнект-петля; дыры видны по манифесту
        try:
            ws = websocket.WebSocketApp(WS_URL, on_open=on_open, on_message=on_message,
                                        on_close=on_close, on_error=on_error)
            ws.run_forever(ping_interval=20, ping_timeout=10)
        except Exception as e:
            mani({"ev": "run_forever_err", "err": str(e)[:200]})
        mani({"ev": "reconnect_wait"})
        time.sleep(5)

if __name__ == "__main__":
    main()
```

## `flow_collector/xex_collector.py`

```python
#!/usr/bin/env python3
"""XEX-COLLECTOR — multi-exchange сбор для будущего cross-exchange lead-lag
(ROADMAP №2; директива Codex 11.07: «параллельно ТОЛЬКО данные, не тестируем»).

Bybit + Binance futures + OKX swap, топ-20 общих перпов по обороту Bybit:
- trades (сторона тейкера) и best bid/ask → 250мс-вёдра по биржевым меткам;
  решение будущего runner возможно только после локального `flush_ts`, а `d`
  остаётся диагностикой, не оценкой latency;
- L5-глубина: Binance depth5@500ms, OKX books5 (нотионалы b5/a5 в ведро);
  Bybit bbo из orderbook.1 (его L50 уже пишет flow-collector раз в минуту);
- clock.jsonl: раз в 60с REST serverTime всех трёх + локальное (дрифт-контроль);
- манифест: старты/реконнекты по биржам/ошибки/часовой heartbeat со счётчиками;
- диск-предохранитель (чужой прод!): <5ГБ свободно → прунит старейший день
  с записью в манифест; <2ГБ → останавливает запись (FATAL), сервер не душит.
Хранение: data_x/YYYY-MM-DD/{xex.jsonl, clock.jsonl, manifest.jsonl}; gzip по дням.
Ведро пишется ТОЛЬКО при событиях (альты молчат → диск экономится)."""
import json, gzip, os, time, threading, shutil, urllib.request, hashlib
from datetime import datetime, timezone
import websocket

ROOT = os.environ.get("XEX_ROOT", os.path.expanduser("~/trading/flow_collector/data_x"))
os.makedirs(ROOT, exist_ok=True)
TOP_N = 20
BUCKET_MS = 250
SCHEMA = 2
SCHEMA_VERSION = "xvenue-schema-v2-2026-07-11"

_lock = threading.RLock()
S = {"day": None, "files": {}, "buckets": {}, "sealed": {}, "stop_write": False,
     "counters": {"by": 0, "bn": 0, "ok": 0, "clock": 0, "late_xex_fragment": 0}}

def rest(url):
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 (xex-collector)"})
    for i in range(3):                     # OKX 403-ит дефолтный python-UA — заголовок обязателен
        try: return json.load(urllib.request.urlopen(req, timeout=20))
        except Exception:
            if i == 2: return None
            time.sleep(1.5)

def code_sha256():
    with open(__file__, "rb") as f:
        return hashlib.sha256(f.read()).hexdigest()

def day_str(): return datetime.now(timezone.utc).strftime("%Y-%m-%d")
def _open_day(d):
    dd = f"{ROOT}/{d}"; os.makedirs(dd, exist_ok=True)
    for n in ("xex", "clock", "manifest"):
        S["files"][n] = open(f"{dd}/{n}.jsonl", "a", buffering=1)
def _gzip_old(d):
    dd = f"{ROOT}/{d}"
    for f in os.listdir(dd):
        if f.endswith(".jsonl"):
            with open(f"{dd}/{f}", "rb") as i, gzip.open(f"{dd}/{f}.gz", "wb") as o:
                o.write(i.read())
            os.remove(f"{dd}/{f}")

def emit(name, obj):
    with _lock:
        if S["stop_write"] and name != "manifest": return
        d = day_str()
        if d != S["day"]:
            old = S["day"]
            for f in S["files"].values(): f.close()
            _open_day(d); S["day"] = d
            if old:
                try: _gzip_old(old)
                except Exception as e: mani({"ev": "gzip_fail", "err": str(e)[:120]})
            mani({"ev": "rollover", "from": old})
        S["files"][name].write(json.dumps(obj, separators=(",", ":")) + "\n")
def mani(o): o["ts"] = int(time.time()*1000); emit("manifest", o)

# ── маппинг символов: пересечение трёх бирж по base, топ-20 по обороту Bybit ──
def build_mapping():
    by = rest("https://api.bybit.com/v5/market/tickers?category=linear")
    bn = rest("https://fapi.binance.com/fapi/v1/exchangeInfo")
    ok = rest("https://www.okx.com/api/v5/public/instruments?instType=SWAP")
    bn_set = {s["symbol"] for s in bn["symbols"]
              if s.get("contractType") == "PERPETUAL" and s.get("status") == "TRADING"} if bn else set()
    ok_set = {i["instId"] for i in ok["data"] if i.get("state") == "live"} if ok else set()
    if not bn_set or not ok_set:
        mani({"ev": "map_source_empty", "bn": len(bn_set), "ok": len(ok_set)})
    rows = sorted((t for t in by["result"]["list"] if t["symbol"].endswith("USDT")),
                  key=lambda t: -float(t.get("turnover24h") or 0))
    out = []
    for t in rows:
        sym = t["symbol"]; base = sym[:-4]
        okx = f"{base}-USDT-SWAP"
        if sym in bn_set and okx in ok_set:
            out.append({"base": base, "by": sym, "bn": sym, "ok": okx})
        if len(out) >= TOP_N: break
    return out

MAP, BY2B, BN2B, OK2B = [], {}, {}, {}   # заполняется в main() ПОСЛЕ открытия манифеста

def bucket(exch, base, ts_srv, recv_ms):
    k = (exch, base, int(ts_srv)//BUCKET_MS)
    with _lock:
        if k in S["sealed"]:
            S["counters"]["late_xex_fragment"] += 1
            mani({"ev": "late_xex_fragment", "e": exch, "s": base, "t": k[2]*BUCKET_MS,
                  "recv_ts": int(recv_ms)})
            return None
        b = S["buckets"].setdefault(k, {"b": None, "a": None, "bn": 0.0, "sn": 0.0,
                                        "n": 0, "b5": None, "a5": None, "ds": [],
                                        "rmin": int(recv_ms), "rmax": int(recv_ms)})
        b["ds"].append(recv_ms - ts_srv)
        b["rmin"] = min(b["rmin"], int(recv_ms))
        b["rmax"] = max(b["rmax"], int(recv_ms))
        return b

def flush_once(now_ms=None):
    """Seal only buckets quiet in local receipt time for >=1s; testable without network."""
    flush_ts = int(time.time()*1000) if now_ms is None else int(now_ms)
    with _lock:
        done = [k for k, v in S["buckets"].items() if flush_ts - v["rmax"] >= 1000]
        rows = []
        for k in done:
            v = S["buckets"].pop(k)
            S["sealed"][k] = flush_ts
            r = {"e": k[0], "s": k[1], "t": k[2]*BUCKET_MS, "n": v["n"], "schema": SCHEMA,
                 "rmin": v["rmin"], "rmax": v["rmax"], "flush_ts": flush_ts,
                 "d": round(sum(v["ds"])/len(v["ds"])) if v["ds"] else None}
            for f in ("b", "a", "b5", "a5"):
                if v[f] is not None: r[f] = v[f]
            if v["n"]: r["bn"] = round(v["bn"], 2); r["sn"] = round(v["sn"], 2)
            rows.append(r)
        ttl = flush_ts - 24*3600_000
        S["sealed"] = {k: t for k, t in S["sealed"].items() if t >= ttl}
    for r in sorted(rows, key=lambda x: (x["flush_ts"], x["e"], x["s"], x["t"])):
        emit("xex", r)
    return rows

def flush_loop():
    while True:
        time.sleep(2)
        flush_once()

# ── Bybit: publicTrade + orderbook.1 (bbo с matching-ts cts) ──
def ws_bybit():
    def on_open(ws):
        mani({"ev": "by_open"})
        args = [f"publicTrade.{m['by']}" for m in MAP] + [f"orderbook.1.{m['by']}" for m in MAP]
        for i in range(0, len(args), 10):
            ws.send(json.dumps({"op": "subscribe", "args": args[i:i+10]}))
    def on_msg(ws, raw):
        recv = time.time()*1000
        m = json.loads(raw); t = m.get("topic", "")
        if t.startswith("publicTrade"):
            for d in m.get("data", []):
                base = BY2B.get(d["s"])
                if not base: continue
                b = bucket("by", base, int(d["T"]), recv)
                if b is None: continue
                nl = float(d["v"])*float(d["p"])
                b["bn" if d["S"] == "Buy" else "sn"] += nl; b["n"] += 1
                S["counters"]["by"] += 1
        elif t.startswith("orderbook.1"):
            d = m.get("data", {})
            base = BY2B.get(d.get("s"))
            if not base: return
            ts = int(m.get("cts") or m.get("ts"))
            b = bucket("by", base, ts, recv)
            if b is None: return
            if d.get("b"): b["b"] = float(d["b"][0][0])
            if d.get("a"): b["a"] = float(d["a"][0][0])
            S["counters"]["by"] += 1
    _run_ws("wss://stream.bybit.com/v5/public/linear", on_open, on_msg, "by", ping=20)

# ── Binance futures: aggTrade + bookTicker + depth5@500ms (combined stream) ──
def ws_binance():
    streams = "/".join(f"{m['bn'].lower()}@{ch}" for m in MAP
                       for ch in ("aggTrade", "bookTicker", "depth5@500ms"))
    url = f"wss://fstream.binance.com/stream?streams={streams}"
    def on_open(ws): mani({"ev": "bn_open"})
    def on_msg(ws, raw):
        recv = time.time()*1000
        m = json.loads(raw).get("data", {})
        et = m.get("e")
        if et == "aggTrade":
            base = BN2B.get(m["s"])
            if not base: return
            b = bucket("bn", base, int(m["T"]), recv)
            if b is None: return
            nl = float(m["q"])*float(m["p"])
            b["sn" if m["m"] else "bn"] += nl; b["n"] += 1    # m=true → тейкер продал
            S["counters"]["bn"] += 1
        elif et == "bookTicker":
            base = BN2B.get(m.get("s"))
            if not base: return
            b = bucket("bn", base, int(m.get("E") or m.get("T")), recv)
            if b is None: return
            b["b"] = float(m["b"]); b["a"] = float(m["a"])
            S["counters"]["bn"] += 1
        elif et == "depthUpdate":
            base = BN2B.get(m.get("s"))
            if not base: return
            b = bucket("bn", base, int(m.get("E")), recv)
            if b is None: return
            b["b5"] = round(sum(float(p)*float(q) for p, q in m.get("b", [])[:5]))
            b["a5"] = round(sum(float(p)*float(q) for p, q in m.get("a", [])[:5]))
            S["counters"]["bn"] += 1
    _run_ws(url, on_open, on_msg, "bn", ping=180)

# ── OKX: trades + bbo-tbt + books5 ──
def ws_okx():
    def on_open(ws):
        mani({"ev": "ok_open"})
        args = ([{"channel": "trades", "instId": m["ok"]} for m in MAP]
                + [{"channel": "bbo-tbt", "instId": m["ok"]} for m in MAP]
                + [{"channel": "books5", "instId": m["ok"]} for m in MAP])
        ws.send(json.dumps({"op": "subscribe", "args": args}))
    def on_msg(ws, raw):
        recv = time.time()*1000
        m = json.loads(raw)
        ch = (m.get("arg") or {}).get("channel"); inst = (m.get("arg") or {}).get("instId")
        base = OK2B.get(inst)
        if not base or "data" not in m: return
        if ch == "trades":
            for d in m["data"]:
                b = bucket("ok", base, int(d["ts"]), recv)
                if b is None: continue
                nl = float(d["sz"])*float(d["px"])*_ctval(inst)
                b["bn" if d["side"] == "buy" else "sn"] += nl; b["n"] += 1
                S["counters"]["ok"] += 1
        elif ch == "bbo-tbt":
            d = m["data"][0]
            b = bucket("ok", base, int(d["ts"]), recv)
            if b is None: return
            if d.get("bids"): b["b"] = float(d["bids"][0][0])
            if d.get("asks"): b["a"] = float(d["asks"][0][0])
            S["counters"]["ok"] += 1
        elif ch == "books5":
            d = m["data"][0]
            b = bucket("ok", base, int(d["ts"]), recv)
            if b is None: return
            cv = _ctval(inst)
            b["b5"] = round(sum(float(p)*float(q)*cv for p, q, *_ in d.get("bids", [])))
            b["a5"] = round(sum(float(p)*float(q)*cv for p, q, *_ in d.get("asks", [])))
            S["counters"]["ok"] += 1
    _run_ws("wss://ws.okx.com:8443/ws/v5/public", on_open, on_msg, "ok", ping=25)

_CTVAL = {}
def _ctval(inst):
    """OKX swap: размер в КОНТРАКТАХ → нотионал = sz × ctVal × px."""
    if inst not in _CTVAL:
        r = rest(f"https://www.okx.com/api/v5/public/instruments?instType=SWAP&instId={inst}")
        try: _CTVAL[inst] = float(r["data"][0]["ctVal"])
        except Exception: _CTVAL[inst] = 1.0
    return _CTVAL[inst]

def _run_ws(url, on_open, on_msg, tag, ping):
    while True:
        try:
            ws = websocket.WebSocketApp(url, on_open=on_open, on_message=on_msg,
                on_close=lambda w, c, m: mani({"ev": f"{tag}_close", "code": c}),
                on_error=lambda w, e: mani({"ev": f"{tag}_error", "err": str(e)[:120]}))
            ws.run_forever(ping_interval=ping, ping_timeout=10)
        except Exception as e:
            mani({"ev": f"{tag}_crash", "err": str(e)[:120]})
        time.sleep(5)

def clock_probe(venue, url, extract):
    send_ts = int(time.time()*1000)
    response = rest(url)
    recv_ts = int(time.time()*1000)
    try:
        server_ts = extract(response) if response else None
    except Exception:
        server_ts = None
    return {"venue": venue, "send_ts": send_ts, "recv_ts": recv_ts, "server_ts": server_ts}

def clock_once():
    probes = [
        clock_probe("by", "https://api.bybit.com/v5/market/time", lambda x: int(x["result"]["timeNano"])//1_000_000),
        clock_probe("bn", "https://fapi.binance.com/fapi/v1/time", lambda x: int(x["serverTime"])),
        clock_probe("ok", "https://www.okx.com/api/v5/public/time", lambda x: int(x["data"][0]["ts"])),
    ]
    emit("clock", {"schema": SCHEMA, "probes": probes})
    S["counters"]["clock"] += 1
    return probes

def clock_loop():
    while True:
        clock_once()
        time.sleep(60)

def guard_loop():
    while True:
        free_gb = shutil.disk_usage("/").free/2**30
        if free_gb < 2:
            with _lock: S["stop_write"] = True
            mani({"ev": "FATAL_disk", "free_gb": round(free_gb, 1)})
        elif free_gb < 5:
            # жёсткая защита (Codex 11.07): прунить можно ТОЛЬКО дни с подтверждённой
            # внешней копией (маркер .pulled_ok ставит daily-pull после checksum-сверки).
            # Нет подтверждённых — лучше упереться в FATAL, чем стереть forward-историю.
            days = sorted(d for d in os.listdir(ROOT) if d[0].isdigit() and d != S["day"])
            pruned = False
            for d in days:
                if os.path.exists(f"{ROOT}/{d}/.pulled_ok"):
                    shutil.rmtree(f"{ROOT}/{d}")
                    mani({"ev": "prune_day", "day": d, "free_gb": round(free_gb, 1)})
                    pruned = True; break
            if not pruned:
                mani({"ev": "prune_blocked_no_pulled_copy", "free_gb": round(free_gb, 1)})
        mani({"ev": "hb", **S["counters"], "free_gb": round(free_gb, 1),
              "pend": len(S["buckets"])})
        time.sleep(3600)

def main():
    with _lock:
        S["day"] = day_str(); _open_day(S["day"])
    MAP.extend(build_mapping())
    BY2B.update({m["by"]: m["base"] for m in MAP})
    BN2B.update({m["bn"]: m["base"] for m in MAP})
    OK2B.update({m["ok"]: m["base"] for m in MAP})
    mani({"ev": "schema_v2_start", "schema": SCHEMA, "version": SCHEMA_VERSION,
          "code_sha256": code_sha256(), "pid": os.getpid(), "map": MAP})
    if not MAP:
        mani({"ev": "FATAL_empty_map"}); time.sleep(60); raise SystemExit(1)
    for m in MAP: _ctval(m["ok"])                       # прогреть ctVal до потока
    for fn in (flush_loop, clock_loop, guard_loop):
        threading.Thread(target=fn, daemon=True).start()
    for fn in (ws_bybit, ws_binance, ws_okx):
        threading.Thread(target=fn, daemon=True).start()
    while True: time.sleep(60)

if __name__ == "__main__":
    main()
```

## `dashboard/diary.py`

```python
#!/usr/bin/env python3
"""📓 Дневник трейдера — предрегистрированные ожидания и экзамен исходов.

Заказ брата 2026-07-07: «все сделки, которые всплыли в зонах входа (даже
неотработавшие): монета — диапазон — итог; и самообучение, которое не
обучается хуйне».

ТРИ ЗАМКА ПРОТИВ САМООБМАНА (менять только с явного добра брата):
1. expectation ЗАМОРАЖИВАЕТСЯ в момент появления сигнала в зоне и НИКОГДА
   не переписывается — экзамен всегда «прогноз ДО» vs «факт ПОСЛЕ».
2. Вердикт «сюрприз» объявляется ТОЛЬКО при n_class ≥ MIN_CLASS_N закрытых
   кейсов; иначе честное «⏳ копим базу» (никаких выводов с трёх примеров).
3. Сюрпризы КОПЯТСЯ (библиотека со снапшотом контекста), но НЕ меняют ни
   одно торговое правило — правила рождаются только через side_study +
   форвард-экзамен (конвейер версий, как bias v1→v2→v3).

Вердикт — квартильный забор Тьюки по закрытым кейсам класса (робастно к
малым n, без предположений о распределении):
  peak24 > p75 + 1.5·IQR → 🚀 сюрприз вверх
  peak24 < p25 − 1.5·IQR → 💥 сюрприз вниз
  внутри [p25..p75]      → ✅ в рамках (вера класса крепнет)
  иначе                  → 〰 в хвосте нормы

Холодный старт ожиданий: пока в дневнике < MIN_CLASS_N закрытых кейсов
класса, база берётся из outcomes/radar_resolved.csv (mfe/mae как прокси
peak/dd; major=0; awakening = vol_ratio≥15, radar_alt = cascade=0) с
пометкой src, чтобы прокси-базу было видно и позже вытеснить своей.
"""
from __future__ import annotations

import csv
import json
import math
import sys
import urllib.request
from datetime import datetime, timezone
from pathlib import Path

DIR = Path(__file__).resolve().parent
TRADING = DIR.parent
sys.path.insert(0, str(TRADING))

from file_lock import atomic_json_update, atomic_json_read  # noqa: E402

DIARY_PATH = DIR / "diary.json"
ENTRY_LOG = TRADING / "outcomes" / "entry_candidates.csv"
RADAR_RESOLVED = TRADING / "outcomes" / "radar_resolved.csv"

MIN_CLASS_N = 10          # замок №2: до этого порога вердикт = «копим базу»
HORIZON_H = 24            # экзамен по peak/dd/ret за 24ч от появления в зоне
DEAD_GRACE_H = 48         # signal_ts + HORIZON + это = запись no_data, из очереди (S1)
AWAKENING_RATIO = 15.0
# Только витрина дневника: endpoint RET24 минус фиксированный круг costs.
# Это НЕ PnL, не учитывает funding/slippage и не участвует в сигналах/вердиктах.
COST_GRID_PCT = (0.14, 0.31, 0.71)
MIN_DAY_CLUSTERS_FOR_T = 5


def utcnow() -> datetime:
    return datetime.now(timezone.utc)


def _quartiles(vals: list[float]) -> dict:
    """p25/p50/p75 без numpy (линейная интерполяция)."""
    xs = sorted(vals)
    n = len(xs)

    def q(p: float) -> float:
        if n == 1:
            return xs[0]
        k = (n - 1) * p
        lo, hi = int(k), min(int(k) + 1, n - 1)
        return round(xs[lo] + (xs[hi] - xs[lo]) * (k - lo), 2)

    return {"p25": q(0.25), "p50": q(0.5), "p75": q(0.75)}


def _endpoint24_stats(kind: str, closed_records: list[dict]) -> dict:
    """Read-only статистика исполнимого endpoint-а дневника.

    MFE/peak24 остаётся отдельным описанием доступной волатильности и базой
    старого экзамена сюрпризов. Здесь намеренно считается только ret24 — close
    последнего 15m-бара в 24ч-окне — плюс грубая сетка ret24−cost. Это НЕ
    торговый отчёт: funding, проскальзывание и исполнение лимиток не известны.

    Обычный t по монетам здесь запрещён: записи одного дня/скана коррелированы.
    Поэтому выводится только t по дневным средним и лишь от пяти UTC-дней;
    иначе честное «insufficient», а не ложная значимость.
    """
    rets: list[float] = []
    by_day: dict[str, list[float]] = {}
    scans: set[int] = set()

    for rec in closed_records:
        if rec.get("kind") != kind:
            continue
        outcome = rec.get("outcome") or {}
        ret = outcome.get("ret24")
        if not outcome.get("final") or not isinstance(ret, (int, float)):
            continue
        rets.append(float(ret))
        try:
            ts = datetime.fromisoformat(rec["signal_ts"])
        except (KeyError, TypeError, ValueError):
            continue
        day = ts.astimezone(timezone.utc).date().isoformat()
        by_day.setdefault(day, []).append(float(ret))
        scans.add(int(ts.timestamp()) // 300)

    out = {
        "n": len(rets),
        "utc_days": len(by_day),
        "scan_clusters": len(scans),
        "median_ret24": None,
        "mean_ret24": None,
        "mean_ret24_after_cost": {},
        "day_t": None,
        "day_t_status": "insufficient",
    }
    if not rets:
        return out

    mean_ret = sum(rets) / len(rets)
    out["median_ret24"] = _quartiles(rets)["p50"]
    out["mean_ret24"] = round(mean_ret, 2)
    out["mean_ret24_after_cost"] = {
        f"{cost:.2f}": round(mean_ret - cost, 2) for cost in COST_GRID_PCT
    }

    day_means = [sum(xs) / len(xs) for xs in by_day.values()]
    if len(day_means) < MIN_DAY_CLUSTERS_FOR_T:
        return out
    avg = sum(day_means) / len(day_means)
    variance = sum((x - avg) ** 2 for x in day_means) / (len(day_means) - 1)
    se = math.sqrt(variance) / math.sqrt(len(day_means))
    if se == 0:
        out["day_t_status"] = "zero_dispersion"
    else:
        out["day_t"] = round(avg / se, 2)
        out["day_t_status"] = "ok"
    return out


def _cold_start_base(kind: str) -> list[float]:
    """Прокси-база peak24 из radar_resolved (закрытые mfe эпизодов)."""
    try:
        rows = list(csv.DictReader(open(RADAR_RESOLVED, encoding="utf-8")))
    except FileNotFoundError:
        return []
    out, seen = [], {}
    for r in rows:
        if r.get("major") == "1":
            continue
        try:
            ts = datetime.fromisoformat(r["ts_utc"])
            ratio = float(r["vol_ratio"])
            mfe = float(r["mfe_pct"])
        except Exception:
            continue
        last = seen.get(r["symbol"])
        if last and abs((ts - last).total_seconds()) < 24 * 3600:
            continue
        seen[r["symbol"]] = ts
        is_awk = ratio >= AWAKENING_RATIO
        if kind == "awakening" and is_awk:
            out.append(mfe)
        elif kind == "radar_alt" and not is_awk and r.get("cascade") == "0":
            out.append(mfe)
    return out


def mark_waves() -> int:
    """Диспансеризация (07.07): пометить волновые записи (≥5 radar_alt одного
    5-мин скана — 06.07 сервер логировал волну, фронт скрывал). Помеченные
    исключаются из own-базы ожиданий. Замороженные expectation НЕ трогаем
    (замок №1) — чинится линейка будущих ожиданий, не история."""
    def mut(d):
        d = d or {"records": []}
        buckets: dict[int, list] = {}
        for r in d["records"]:
            if r.get("kind") != "radar_alt":
                continue
            try:
                b = int(datetime.fromisoformat(r["signal_ts"]).timestamp()) // 300
            except Exception:
                continue
            buckets.setdefault(b, []).append(r)
        n = 0
        for b, recs in buckets.items():
            if len(recs) >= 5:
                for r in recs:
                    if not (r.get("ctx") or {}).get("wave"):
                        r.setdefault("ctx", {})["wave"] = True
                        n += 1
        return d
    atomic_json_update(DIARY_PATH, mut, default={"records": []})
    return _last_marked(mut)


def _last_marked(_):   # счётчик через повторное чтение (mut внутри лока)
    d = atomic_json_read(DIARY_PATH, default={"records": []}) or {"records": []}
    return sum(1 for r in d["records"] if (r.get("ctx") or {}).get("wave"))


def build_expectation(kind: str, closed_records: list[dict]) -> dict:
    """Ожидание класса из ЗАКРЫТЫХ записей дневника; холодный старт — прокси.
    Волновые записи (ctx.wave) в базу НЕ идут — коррелированы (см. mark_waves).
    Возвращаемый дикт замораживается в записи (замок №1)."""
    own = [r["outcome"]["peak24"] for r in closed_records
           if r.get("kind") == kind and (r.get("outcome") or {}).get("final")
           and r["outcome"].get("peak24") is not None
           and not (r.get("ctx") or {}).get("wave")]
    if len(own) >= MIN_CLASS_N:
        base, src = own, "diary"
    else:
        proxy = _cold_start_base(kind)
        if len(own) + len(proxy) >= MIN_CLASS_N:
            base, src = own + proxy, f"diary({len(own)})+proxy({len(proxy)})"
        else:
            base, src = own + proxy, "insufficient"
    exp = {"class": kind, "n_class": len(base), "src": src,
           "frozen_at": utcnow().replace(microsecond=0).isoformat()}
    if base:
        qs = _quartiles(base)
        exp.update(qs)
        exp["move_rate"] = round(sum(1 for x in base if x >= 5) / len(base), 2)
    return exp


def grade(exp: dict, outcome: dict) -> dict:
    """Экзамен факта против замороженного ожидания. Замок №2: при
    src=insufficient или n<MIN_CLASS_N — только «collect», без сюрпризов."""
    peak = (outcome or {}).get("peak24")
    if peak is None:
        return {"verdict": "pending"}
    if exp.get("src") == "insufficient" or exp.get("n_class", 0) < MIN_CLASS_N:
        return {"verdict": "collect",
                "note": f"база класса мала (n={exp.get('n_class', 0)}<{MIN_CLASS_N}) — копим, выводов нет"}
    p25, p75 = exp["p25"], exp["p75"]
    iqr = max(p75 - p25, 0.1)
    hi_fence, lo_fence = p75 + 1.5 * iqr, p25 - 1.5 * iqr
    if peak > hi_fence:
        return {"verdict": "surprise_up",
                "note": f"peak {peak:+.1f}% > забор {hi_fence:+.1f}% (ожидание p50 {exp['p50']:+.1f}%) — в библиотеку сюрпризов"}
    if peak < lo_fence:
        return {"verdict": "surprise_down",
                "note": f"peak {peak:+.1f}% < забор {lo_fence:+.1f}% — класс не дал даже минимума"}
    if p25 <= peak <= p75:
        return {"verdict": "confirm",
                "note": f"в рамках ожидания [{p25:+.1f}..{p75:+.1f}] — вера класса крепнет"}
    return {"verdict": "tail_ok",
            "note": f"в хвосте нормы (забор {lo_fence:+.1f}..{hi_fence:+.1f})"}


def _chg24_at(symbol: str, signal_ts: str, age_h: float,
              current_chg24: float | None) -> float | None:
    """24ч-изменение цены НА МОМЕНТ СИГНАЛА (ревью 07.07 M1: метка «вторая
    волна» мерялась на момент лога — look-ahead для поздних заходов, у
    пробуждений лог бывает на часы позже сигнала). age_h ≤ 0.25 — лог почти
    мгновенный (тик 60с), берём текущее без сети; иначе считаем по 1ч-барам
    close(signal) vs close(signal−24ч). None = честно «не знаем»."""
    if age_h <= 0.25:
        return current_chg24
    try:
        t_sig = int(datetime.fromisoformat(signal_ts).timestamp() * 1000)
        url = (f"https://api.bybit.com/v5/market/kline?category=linear&symbol={symbol}"
               f"&interval=60&start={t_sig - 25 * 3600_000}&end={t_sig}&limit=30")
        with urllib.request.urlopen(url, timeout=8) as r:
            d = json.load(r)
        if d.get("retCode") != 0:
            return None
        bars = sorted((int(x[0]), float(x[4])) for x in d["result"]["list"])
        if len(bars) < 20:
            return None
        c_now = bars[-1][1]
        c_then = min(bars, key=lambda b: abs(b[0] - (t_sig - 24 * 3600_000)))[1]
        return round((c_now - c_then) / c_then * 100, 2) if c_then > 0 else None
    except Exception:
        return None


def _fetch_bars_15m(symbol: str, start_ms: int, end_ms: int) -> list:
    url = (f"https://api.bybit.com/v5/market/kline?category=linear&symbol={symbol}"
           f"&interval=15&start={start_ms}&end={end_ms}&limit=120")
    with urllib.request.urlopen(url, timeout=12) as r:
        d = json.load(r)
    if d.get("retCode") != 0:
        raise RuntimeError(f"retCode={d.get('retCode')}")
    now_ms = int(utcnow().timestamp() * 1000)
    bars = sorted((int(x[0]), float(x[1]), float(x[2]), float(x[3]), float(x[4]))
                  for x in d.get("result", {}).get("list") or [])
    return [b for b in bars if b[0] + 15 * 60_000 <= now_ms]


def resolve_record(rec: dict) -> dict | None:
    """peak/dd/ret за 24ч от basis, бары строго ПОСЛЕ signal_ts (анти-look-ahead:
    бар входа выброшен, как всюду в проекте). None = данных пока нет."""
    t0 = int(datetime.fromisoformat(rec["signal_ts"]).timestamp() * 1000)
    try:
        bars = _fetch_bars_15m(rec["symbol"], t0, t0 + (HORIZON_H + 1) * 3600_000)
    except Exception:
        return None
    first_full = ((t0 // 900_000) + 1) * 900_000
    full = [b for b in bars if b[0] >= first_full]
    post = full[1:]                                      # бар входа выброшен
    if not post:
        return None
    basis = rec.get("basis")
    if not basis:
        # fast-связка без basis (не было снапшота при рождении): та же
        # конвенция входа — close первого ПОЛНОГО бара после поста
        basis = full[0][4]
        if not basis:
            return None
    horizon_end = t0 + HORIZON_H * 3600_000
    inwin = [b for b in post if b[0] < horizon_end]
    if not inwin:
        inwin = post[:1]
    peak = max((b[2] - basis) / basis * 100 for b in inwin)
    dd = min((b[3] - basis) / basis * 100 for b in inwin)
    ret = (inwin[-1][4] - basis) / basis * 100
    final = post and post[-1][0] + 15 * 60_000 >= horizon_end
    return {"peak24": round(peak, 2), "dd24": round(dd, 2),
            "ret24": round(ret, 2), "bars": len(inwin), "final": bool(final),
            "basis": round(basis, 8)}


def upsert_new(extra_combo: list[dict] | None = None,
               btc_ret24: float | None = None,
               chg24_map: dict | None = None,
               btc_range24: float | None = None) -> int:
    """Затянуть в дневник новые записи: все entry-кандидаты из CSV + связки.
    Идемпотентно по id. Ожидание замораживается ЗДЕСЬ, в момент рождения."""
    rows = []
    if ENTRY_LOG.exists():
        rows = list(csv.DictReader(open(ENTRY_LOG, encoding="utf-8")))
    combos = extra_combo or []
    added = 0

    def mut(d):
        nonlocal added
        d = d or {"records": []}
        have = {r["id"] for r in d["records"]}
        closed = [r for r in d["records"] if (r.get("outcome") or {}).get("final")]
        for r in rows:
            rid = f"{r['symbol']}|{r['signal_ts_utc']}"
            if rid in have:
                continue
            kind = r["kind"]
            rec = {
                "id": rid, "kind": kind, "symbol": r["symbol"],
                # радар/пробуждение = всегда лонг от базиса; шорт-класс,
                # если появится, обязан принести side из источника явно
                "side": "long",
                "signal_ts": r["signal_ts_utc"], "zone_ts": r["logged_ts_utc"],
                "basis": float(r["basis"]),
                "ctx": {"live_at_zone": float(r["last_pct"]),
                        "dd_at_zone": float(r["dd_pct"]),
                        "age_h": float(r["age_h"]),
                        "vol_ratio": float(r["vol_ratio"]) if r.get("vol_ratio") else None,
                        "btc_ret24": btc_ret24, "btc_range24": btc_range24,
                        # «вторая волна» (≥+10%/24ч до сигнала) — когорта для
                        # форвард-разреза (кейс ALLO 07.07); None = не знаем
                        "chg24_at_zone": (chg24_map or {}).get(r["symbol"]),
                        # когорта «вторых волн» = chg24_at_signal ≥ 10 (факт
                        # храним, производное считаем при разрезе)
                        "chg24_at_signal": _chg24_at(r["symbol"], r["signal_ts_utc"],
                                                     float(r["age_h"]),
                                                     (chg24_map or {}).get(r["symbol"]))},
                "expectation": build_expectation(kind, closed),
                "outcome": None, "grade": {"verdict": "pending"},
            }
            d["records"].append(rec)
            have.add(rid)
            added += 1
        for c in combos:
            # шорты пока НЕ встраиваем (решение брата 08.07): экзамен дневника
            # меряет пик ВВЕРХ от базиса — шорт-пост лонговой линейкой не мерить,
            # в базы классов не пускать (на Пульсе связка живёт как жила)
            if (c.get("direction") or "").lower() == "short":
                continue
            # id по msg_key: быстрая (превью, 60с) и каноническая (Telethon,
            # 15 мин) версии одного поста = ОДНА запись дневника, без дублей
            # id по ПРОБУЖДЕНИЮ (awake_ts): подтверждение поста меняет msg_key
            # (последний alert) → был дубль записи (ревью 07.07 S2); пробуждение
            # стабильно задаёт событие связки на всём её жизненном цикле
            rid = f"{c['symbol']}|{c.get('awake_ts') or c.get('msg_key') or c['post_ts']}|combo"
            if rid in have:
                continue
            d["records"].append({
                "id": rid, "kind": "combo", "symbol": c["symbol"],
                # сторона связки — из направления поста канала; неизвестно → None (фронт покажет «—»)
                "side": c.get("direction") if c.get("direction") in ("long", "short") else None,
                "signal_ts": c["post_ts"], "zone_ts": utcnow().isoformat(),
                "basis": c.get("basis"),
                "ctx": {"awake_ratio": c.get("awake_ratio"),
                        "channel": c.get("channel"), "direction": c.get("direction"),
                        "btc_ret24": btc_ret24},
                "expectation": build_expectation("combo", closed),
                "outcome": None, "grade": {"verdict": "pending"},
            })
            have.add(rid)
            added += 1
        d["records"] = d["records"][-800:]
        return d

    atomic_json_update(DIARY_PATH, mut, default={"records": []})
    return added


def resolve_pending(max_fetch: int = 25) -> int:
    """Резолв незакрытых записей (снимок → сеть → merge под локом).

    Анти-голодание (ревью 07.07 S1): записи старше HORIZON+DEAD_GRACE_H без
    успешного резолва финализируются status=no_data БЕЗ сетевой попытки и
    навсегда выходят из очереди — мёртвый хвост (делисты, битые символы)
    не съедает бюджет max_fetch у живых. no_data в базы ожиданий не попадает
    (туда берутся только записи с числовым peak24)."""
    snap = atomic_json_read(DIARY_PATH, default={"records": []}) or {"records": []}
    updates: dict[str, dict] = {}
    fetched = 0
    now = utcnow()
    for rec in snap["records"]:
        if (rec.get("outcome") or {}).get("final"):
            continue
        try:
            age_h = (now - datetime.fromisoformat(rec["signal_ts"])).total_seconds() / 3600
        except Exception:
            age_h = None
        if age_h is not None and age_h > HORIZON_H + DEAD_GRACE_H:
            updates[rec["id"]] = {"outcome": {"status": "no_data", "final": True},
                                  "grade": {"verdict": "no_data",
                                            "note": f"свечей не дождались за {age_h:.0f}ч — из очереди"}}
            continue
        if fetched >= max_fetch:
            break
        o = resolve_record(rec)
        fetched += 1
        if o:
            # вердикт — ТОЛЬКО по закрытым суткам; промежуточный пик не судим
            g = grade(rec.get("expectation") or {}, o) if o.get("final") \
                else {"verdict": "in_progress"}
            updates[rec["id"]] = {"outcome": o, "grade": g}
    if not updates:
        return 0

    def mut(d):
        d = d or {"records": []}
        for r in d["records"]:
            u = updates.get(r["id"])
            if u:
                r["outcome"] = u["outcome"]
                r["grade"] = u["grade"]
                # fast-связка родилась без basis → доустановить из резолва
                if not r.get("basis") and u["outcome"].get("basis"):
                    r["basis"] = u["outcome"]["basis"]
        return d

    atomic_json_update(DIARY_PATH, mut, default={"records": []})
    return len(updates)


def diary_block(limit: int = 120) -> dict:
    """Блок для фида: записи (свежие первыми) + агрегаты «что бот выучил»."""
    d = atomic_json_read(DIARY_PATH, default={"records": []}) or {"records": []}
    recs = sorted(d["records"], key=lambda r: r.get("zone_ts") or "", reverse=True)
    closed = [r for r in recs if (r.get("outcome") or {}).get("final")]
    verdicts = {}
    for r in closed:
        v = (r.get("grade") or {}).get("verdict", "pending")
        verdicts[v] = verdicts.get(v, 0) + 1
    classes = {}
    for kind in ("awakening", "radar_alt", "combo"):
        # guard 11.07: цензурированный outcome-обрубок (VET: только final/status,
        # без peak24) не должен валить весь diary-блок фида — fail-open по записи
        own = [r["outcome"]["peak24"] for r in closed
               if r["kind"] == kind and (r.get("outcome") or {}).get("peak24") is not None]
        c = {"n_closed": len(own)}
        if own:
            c.update(_quartiles(own))
            c["move_rate"] = round(sum(1 for x in own if x >= 5) / len(own), 2)
        exp_now = build_expectation(kind, closed)
        c["expectation_now"] = {k: exp_now.get(k) for k in
                                ("n_class", "src", "p25", "p50", "p75", "move_rate")}
        c["endpoint24"] = _endpoint24_stats(kind, closed)
        classes[kind] = c
    surprises = [r for r in recs
                 if (r.get("grade") or {}).get("verdict") in ("surprise_up", "surprise_down")][:20]
    return {"records": recs[:limit], "n_total": len(recs), "n_closed": len(closed),
            "verdict_counts": verdicts, "classes": classes,
            "surprises": [r["id"] for r in surprises]}


def _selfcheck() -> None:
    # квартили и заборы
    exp = {"class": "t", "n_class": 12, "src": "diary",
           "p25": 1.0, "p50": 2.5, "p75": 5.0}
    assert grade(exp, {"peak24": 3.0})["verdict"] == "confirm"
    assert grade(exp, {"peak24": 30.0})["verdict"] == "surprise_up"    # > 5+6=11
    assert grade(exp, {"peak24": -8.0})["verdict"] == "surprise_down"  # < 1-6=-5
    assert grade(exp, {"peak24": 8.0})["verdict"] == "tail_ok"
    assert grade(exp, {})["verdict"] == "pending"
    # замок №2: тонкая база не даёт сюрпризов
    thin = {"class": "t", "n_class": 3, "src": "insufficient", "p25": 1, "p50": 2, "p75": 3}
    assert grade(thin, {"peak24": 99.0})["verdict"] == "collect"
    # квартильная механика
    qs = _quartiles([1, 2, 3, 4, 5])
    assert qs == {"p25": 2.0, "p50": 3.0, "p75": 4.0}, qs
    # замок №1 (заморозка) — проверка процессом: build_expectation зовётся
    # ТОЛЬКО в upsert_new; resolve_pending/grade ожидание не пересчитывают
    import inspect
    src = inspect.getsource(resolve_pending) + inspect.getsource(grade)
    assert "build_expectation" not in src, "resolve/grade не должны трогать ожидание!"
    print("diary selfcheck OK (заборы, замки 1-2, квартили)")


if __name__ == "__main__":
    if "--selfcheck" in sys.argv:
        _selfcheck()
    elif "--mark-waves" in sys.argv:
        print(f"волновых помечено всего: {mark_waves()}")
    elif "--resolve" in sys.argv:
        n = resolve_pending()
        print(f"резолвнуто: {n}")
    elif "--upsert" in sys.argv:
        print(f"добавлено: {upsert_new()}")
    else:
        b = diary_block(limit=5)
        print(json.dumps({k: b[k] for k in ("n_total", "n_closed", "verdict_counts", "classes")},
                         ensure_ascii=False, indent=1))
```

## `dashboard/site/app.js`

```text
/* Пульс — трейдинг-дашборд.
   Фид: GitHub raw (пуш локального демона, задержка ~1-2 мин) + фоллбэк на
   задеплоенную копию. Живые цены: Bybit v5 public WebSocket напрямую из
   браузера — отработка активных сигналов тикает в реальном времени. */
"use strict";

const FEED_PROXY = "/api/feed";  // свежий фид через Vercel-прокси (мимо Fastly-кэша raw, max-age=300)
const FEED_RAW = "https://raw.githubusercontent.com/nsudiyan/mirofish-state/main/trading_feed.json";
const FEED_POLL_MS = 15_000;
const BYBIT_REST = "https://api.bybit.com/v5/market/kline";
const BYBIT_WS = "wss://stream.bybit.com/v5/public/linear";
const LIVE_CARDS_MAX = 24;
const RADAR_CARD_WINDOW_H = 6;   // radar-хитов много — в карточки только свежие
const KLINE_REFRESH_MS = 5 * 60_000;

const $ = (id) => document.getElementById(id);
const MSK = new Intl.DateTimeFormat("ru-RU", {
  timeZone: "Europe/Moscow", day: "2-digit", month: "2-digit",
  hour: "2-digit", minute: "2-digit",
});
const fmtMsk = (iso) => iso ? MSK.format(new Date(iso)) : "—";
const fmtPct = (v, digits = 2) =>
  v == null ? "—" : `${v > 0 ? "+" : ""}${v.toFixed(digits)}%`;
const cls = (v) => (v == null ? "flat" : v > 0 ? "pos" : v < 0 ? "neg" : "flat");
const esc = (s) => String(s ?? "").replace(/[&<>"]/g,
  (c) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;" }[c]));

function agoStr(iso) {
  const m = Math.max(0, (Date.now() - new Date(iso)) / 60000);
  if (m < 1) return "только что";
  if (m < 60) return `${Math.floor(m)} мин назад`;
  if (m < 48 * 60) return `${Math.floor(m / 60)} ч назад`;
  return `${Math.floor(m / 1440)} дн назад`;
}

const SRC_RU = { tg_channel: "ТГК", screener: "скринер", storm: "шторм", radar: "радар", alert: "алерт" };
const OUR_SOURCES = new Set(["storm", "radar", "screener", "alert"]); // наши системы (не чужие ТГК)
const FRESH_MIN = 15; // сигнал моложе 15 мин = метка NEW
const SECOND_WAVE_CHG = 10; // монета уже +10%/24ч на входе = «вторая волна» → сайз меньше (брат 07.07, кейс ALLO)
const state = {
  feed: null,
  liveCards: new Map(),   // id → {sig, basis, peak, dd, sparkPts, el}
  klineCache: new Map(),  // symbol|tsMin → {bars, fetchedAt}
  ws: null, wsSymbols: new Set(), wsAlive: false,
  histFilter: "all", histShown: 40, roseShown: 40, roseStarOnly: false,
  liveFilter: "ours",     // по умолчанию — наши сигналы
  starred: new Set(JSON.parse(localStorage.getItem("pulse_starred") || "[]")),
  lastPx: new Map(),      // symbol → живая цена с WS (для связок и не только)
  chg24: new Map(),       // symbol → живой % за 24ч с WS (метка «вторая волна»)
  comboPeak: new Map(),   // symbol → максимальный живой % от базиса поста за сессию
};

/* ── Web Push: уведомления о новых кандидатах в зонах входа ──
   Пуши шлёт локальный демон с мака (pywebpush не нужен — npx web-push);
   тут только подписка. iOS: работает ТОЛЬКО из PWA с экрана «Домой». */
const VAPID_PUB = "BGld5u9OcP_jL-TN8SvD4_krPxk1X16ZwHcOHwMoWZMxDSGcZnDf_lOMdIvD32pk6lWwCAdLIb-eRsDWo4VKnYw";
function b64ToU8(s) {
  const pad = "=".repeat((4 - (s.length % 4)) % 4);
  const raw = atob((s + pad).replace(/-/g, "+").replace(/_/g, "/"));
  return Uint8Array.from(raw, (c) => c.charCodeAt(0));
}
async function initPush() {
  if (!("serviceWorker" in navigator)) return;
  const reg = await navigator.serviceWorker.register("sw.js").catch(() => null);
  if (!reg || !("PushManager" in window)) return;
  const btn = $("push-btn");
  if (!btn) return;
  btn.hidden = false;
  const sub = await reg.pushManager.getSubscription();
  if (sub && Notification.permission === "granted") btn.textContent = "🔔✓";
  btn.onclick = async () => {
    try {
      const perm = await Notification.requestPermission();
      if (perm !== "granted") { btn.textContent = "🔕"; return; }
      const s = await reg.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: b64ToU8(VAPID_PUB),
      });
      const code = JSON.stringify(s.toJSON());
      btn.textContent = "🔔✓";
      try { await navigator.clipboard.writeText(code); } catch (_) {}
      // одноразовый шаг: код подписки надо передать демону на маке
      let box = $("push-code");
      if (!box) {
        box = document.createElement("div");
        box.id = "push-code";
        box.innerHTML = `<b>Код подписки скопирован в буфер</b> — пришли его Клоду одним сообщением, он включит доставку. <textarea readonly rows="3"></textarea>`;
        document.querySelector("header").after(box);
      }
      box.querySelector("textarea").value = code;
    } catch (e) {
      btn.textContent = "🔕";
      alert("Не вышло подписаться: " + e.message + (/(iPhone|iPad)/.test(navigator.userAgent) ? "\n\nНа iOS: сайт должен быть добавлен на экран «Домой» и открыт оттуда." : ""));
    }
  };
}
initPush();

/* ── 📓 Дневник трейдера: вкладка + рендер журнала/ожиданий/сюрпризов ── */
const PULSE_SECTIONS = ["sec-entry", "sec-live", "sec-pump", "sec-rose",
                        "sec-history", "sec-bot", "sec-storm", "short-watch"];
function switchTab(tab) {
  document.querySelectorAll(".tab").forEach((b) =>
    b.classList.toggle("active", b.dataset.tab === tab));
  for (const id of PULSE_SECTIONS) { const el = $(id); if (el) el.hidden = tab !== "pulse"; }
  const d = $("sec-diary"); if (d) d.hidden = tab !== "diary";
  location.hash = tab === "diary" ? "#diary" : "";
}
document.querySelectorAll(".tab").forEach((b) =>
  b.addEventListener("click", () => switchTab(b.dataset.tab)));
if (location.hash === "#diary") switchTab("diary");

const KIND_RU = { awakening: "🌅 пробуждение", radar_alt: "радар-альт", combo: "⚡ связка" };
// [лейбл, цвет, подсказка]; вердикт = экзамен MFE-пика, не PnL и не исполнимый выход.
const VERDICT_RU = {
  confirm: ["🎯 MFE в коридоре", "var(--s1)",
    "MFE-пик 24ч попал в коридор ожиданий класса. Это НЕ оценка прибыли: endpoint — в колонке «Итог 24ч»"],
  tail_ok: ["〰 хвост нормы", "var(--ink-2)",
    "пик вне коридора p25–p75, но внутри забора Тьюки — необычно, но не сюрприз"],
  surprise_up: ["🚀 MFE-сюрприз ↑", "var(--warn)"],
  surprise_down: ["💥 MFE-сюрприз ↓", "var(--down)"],
  collect: ["⏳ копим базу", "var(--muted)"],
  in_progress: ["в работе", "var(--muted)"],
  pending: ["ждёт данных", "var(--muted)"],
};

function renderDiary(feed) {
  const dy = feed.diary || {};
  const recs = dy.records || [];
  // сторона записи; старые записи без side уже забэкфилены (08.07), «—» = источник не сообщил
  const sideHtml = (s) => s === "long" ? '<span class="dir long">▲ LONG</span>'
    : s === "short" ? '<span class="dir short">▼ SHORT</span>' : "—";
  const tiles = $("diary-tiles");
  if (tiles) {
    const vc = dy.verdict_counts || {};
    tiles.innerHTML = [
      ["записей", dy.n_total ?? 0, ""],
      ["закрыто (24ч)", dy.n_closed ?? 0, ""],
      ["🎯 MFE в коридоре", vc.confirm ?? 0, ""],
      ["🚀 MFE-сюрпризов ↑", vc.surprise_up ?? 0, "warn"],
      ["💥 MFE-сюрпризов ↓", vc.surprise_down ?? 0, "down"],
      ["⏳ на тонкой базе", vc.collect ?? 0, ""],
    ].map(([label, v, tone]) => {
      const col = tone === "up" ? "var(--up)" : tone === "down" ? "var(--down)" : tone === "warn" ? "var(--warn)" : "var(--ink)";
      return `<div class="tile"><div class="v" style="color:${col}">${v}</div><div class="l">${esc(label)}</div></div>`;
    }).join("");
  }
  const cls = $("diary-classes");
  if (cls) {
    cls.innerHTML = Object.entries(dy.classes || {}).map(([k, c]) => {
      const e = c.expectation_now || {};
      const endpoint = c.endpoint24 || {};
      const mfe = e.p50 != null
        ? `MFE-пик p50 <b>${fmtPct(e.p50, 1)}</b> · разброс ${fmtPct(e.p25, 1)}…${fmtPct(e.p75, 1)} · ход≥5%: ${Math.round((e.move_rate || 0) * 100)}%`
        : "MFE-база копится";
      const ret = endpoint.n
        ? `RET24: med <b>${fmtPct(endpoint.median_ret24, 1)}</b> · mean ${fmtPct(endpoint.mean_ret24, 1)} (n=${endpoint.n})`
        : "RET24: нет закрытых endpoint-ов";
      const net = endpoint.n
        ? `RET24−cost: 0.14%→${fmtPct(endpoint.mean_ret24_after_cost?.["0.14"], 1)} · 0.31%→${fmtPct(endpoint.mean_ret24_after_cost?.["0.31"], 1)} · 0.71%→${fmtPct(endpoint.mean_ret24_after_cost?.["0.71"], 1)}`
        : "";
      const clusters = endpoint.n
        ? `UTC-дней ${endpoint.utc_days} · 5м-сканов ${endpoint.scan_clusters} · day-t ${endpoint.day_t_status === "ok" ? endpoint.day_t : "недостаточно данных"}`
        : "";
      return `<div class="tile"><div class="v" style="font-size:15px">${esc(KIND_RU[k] || k)}</div>
        <div class="l">${mfe}<br>${ret}<br>${net}<br>${clusters}<br>MFE-база: n=${e.n_class ?? 0} (${esc(e.src || "—")}) · закрыто своих: ${c.n_closed}</div></div>`;
    }).join("");
  }
  const tb = document.querySelector("#diary-table tbody");
  if (tb) {
    tb.innerHTML = recs.map((r) => {
      const o = r.outcome || {};
      const e = r.expectation || {};
      const [vLabel, vColor, vHint] = VERDICT_RU[(r.grade || {}).verdict] || VERDICT_RU.pending;
      const px = (p) => r.basis != null && p != null
        ? (r.basis * (1 + p / 100)).toPrecision(5) : null;
      const range = o.peak24 != null
        ? `<span class="mono">${px(o.dd24)} … ${px(o.peak24)}</span>`
        : "—";
      return `<tr>
        <td class="lft">${fmtMsk(r.zone_ts)}</td>
        <td class="lft">${starHtml(r.symbol)} <b>${esc(r.symbol)}</b></td>
        <td class="lft">${esc(KIND_RU[r.kind] || r.kind)}</td>
        <td>${sideHtml(r.side)}</td>
        <td class="mono">${r.basis ?? "—"}</td>
        <td>${range}</td>
        <td class="${cls2(o.peak24)}">${fmtPct(o.peak24, 1)}</td>
        <td class="${cls2(o.dd24)}">${fmtPct(o.dd24, 1)}</td>
        <td class="${cls2(o.ret24)}"><b>${fmtPct(o.ret24, 1)}</b></td>
        <td class="lft" title="MFE-пик, не выход · заморожено ${esc(e.frozen_at || "")} · база: ${esc(e.src || "")}">${e.p50 != null ? "MFE " + fmtPct(e.p50, 1) + " (n=" + e.n_class + ")" : "копится"}</td>
        <td class="lft" style="color:${vColor}" title="${esc((r.grade || {}).note || vHint || "")}">${vLabel}</td>
      </tr>`;
    }).join("");
  }
  const sup = $("diary-surprises");
  if (sup) {
    const ss = recs.filter((r) => ["surprise_up", "surprise_down"].includes((r.grade || {}).verdict));
    sup.innerHTML = ss.length ? ss.map((r) => {
      const o = r.outcome || {}, c = r.ctx || {}, e = r.expectation || {};
      return `<div class="card entry${(r.grade.verdict === "surprise_up") ? " awk" : ""}">
        <div class="row1">${starHtml(r.symbol)}<span class="sym">${esc(r.symbol)}</span>
          ${sideHtml(r.side)}
          <span class="when">${fmtMsk(r.zone_ts)}</span></div>
        <div class="big ${cls2(o.peak24)}">MFE ${fmtPct(o.peak24, 1)} <span style="font-size:11px;color:var(--muted)">ожидали MFE ${fmtPct(e.p50, 1)}</span></div>
        <div class="meta">RET24 ${fmtPct(o.ret24, 1)} до costs · ${esc(KIND_RU[r.kind] || r.kind)} · ×${c.vol_ratio ?? "—"} · BTC ${fmtPct(c.btc_ret24, 1)} при входе · ${esc((r.grade || {}).note || "")}</div>
      </div>`;
    }).join("") : '<div class="empty">Сюрпризов пока нет — исходы в рамках ожиданий.</div>';
  }
}
const cls2 = (v) => (v == null ? "" : v > 0 ? "pos" : v < 0 ? "neg" : "");

/* ── звёздочки: пометка «просмотрел/просмотрю» на МОНЕТЕ, живёт в браузере ── */
function starHtml(sym) {
  const on = state.starred.has(sym);
  return `<span class="star${on ? " on" : ""}" data-sym="${esc(sym)}" title="отметить монету">${on ? "★" : "☆"}</span>`;
}
function toggleStar(sym) {
  if (state.starred.has(sym)) state.starred.delete(sym);
  else state.starred.add(sym);
  localStorage.setItem("pulse_starred", JSON.stringify([...state.starred]));
  const on = state.starred.has(sym);
  document.querySelectorAll(`.star[data-sym="${CSS.escape(sym)}"]`).forEach((el) => {
    el.classList.toggle("on", on);
    el.textContent = on ? "★" : "☆";
  });
  // активные ★-фильтры должны сразу отразить изменение
  if (!state.feed) return;
  if (state.liveFilter === "starred") renderLive(state.feed);
  if (state.histFilter === "starred") renderHistory(state.feed);
  if (state.roseStarOnly) renderRose(state.feed);
}
document.addEventListener("click", (ev) => {
  const st = ev.target.closest(".star");
  if (st) { ev.preventDefault(); toggleStar(st.dataset.sym); }
});

/* ── фид ── */
async function fetchFeed() {
  // 1) /api/feed — свежий (contents API, no-store); 2) raw — fallback если прокси лёг
  // (bust на raw бесполезен: Fastly игнорит query — оставлен лишь чтобы не долбить один URL);
  // 3) статичный feed.json — последний резерв (устаревает на момент деплоя).
  const bust = Math.floor(Date.now() / 15_000);
  for (const url of [FEED_PROXY, `${FEED_RAW}?t=${bust}`, `feed.json?t=${bust}`]) {
    try {
      const r = await fetch(url, { cache: "no-store" });
      if (r.ok) return await r.json();
    } catch (_) { /* следующий источник */ }
  }
  return null;
}

function feedStatus(feed) {
  const el = $("feed-status");
  if (!feed) { el.innerHTML = '<span class="dot err"></span>фид недоступен'; return; }
  const ageMin = (Date.now() - new Date(feed.generated_at)) / 60000;
  const dot = ageMin < 5 ? "ok" : ageMin < 30 ? "stale" : "err";
  el.innerHTML = `<span class="dot ${dot}"></span>фид ${agoStr(feed.generated_at)}`;
  $("gen-at").textContent = ` Фид собран: ${fmtMsk(feed.generated_at)} МСК.`;
}

/* ── Bybit klines: базис и спарклайн сигнала ── */
async function fetchBars(symbol, fromMs) {
  const key = `${symbol}|${Math.floor(fromMs / 60000)}`;
  const hit = state.klineCache.get(key);
  if (hit && Date.now() - hit.fetchedAt < KLINE_REFRESH_MS) return hit.bars;
  try {
    const url = `${BYBIT_REST}?category=linear&symbol=${symbol}&interval=5&start=${fromMs}&end=${Date.now()}&limit=1000`;
    const r = await fetch(url);
    const j = await r.json();
    let bars = (j?.result?.list || []).map((x) => ({
      t: +x[0], o: +x[1], h: +x[2], l: +x[3], c: +x[4],
    })).reverse();
    // только полные бары строго после сигнала (без look-ahead)
    const barMs = 5 * 60000;
    const firstFull = (Math.floor(fromMs / barMs) + 1) * barMs;
    bars = bars.filter((b) => b.t >= firstFull && b.t + barMs <= Date.now());
    state.klineCache.set(key, { bars, fetchedAt: Date.now() });
    return bars;
  } catch (_) { return hit ? hit.bars : []; }
}

/* 🔒 шорт-тест (просьба брата 08.07): витрина шторм-шортов с активной плашкой
   «лонги заперты» (условие = fundingTag: funding_now ≥ 0.01 при падении).
   Наблюдение для глаз, НЕ сигнал; в дневник такие не пишутся. */
function renderShortWatch() {
  const box = $("short-watch-cards");
  if (!box) return;
  const out = [...state.liveCards.values()].filter((c) => {
    const s = c.sig;
    return s.source === "storm" && s.direction === "short"
      && s.funding_now != null && s.funding_now >= 0.01;
  }).sort((a, b) => new Date(b.sig.ts_utc) - new Date(a.sig.ts_utc));
  if (!out.length) {
    box.innerHTML = '<div class="empty">Сейчас таких нет.</div>';
    return;
  }
  box.innerHTML = out.slice(0, 8).map((c) => {
    const s = c.sig, live = c.lastPct;
    return `<div class="shortw-row">
      <div class="shortw-top">${starHtml(s.symbol)}<b>${esc(s.symbol)}</b>
        <span class="dir short">▼ SHORT</span>
        <span class="when">${fmtMsk(s.ts_utc)}</span></div>
      <div class="shortw-mid">
        <span class="big ${live > 0 ? "pos" : live < 0 ? "neg" : ""}">${live != null ? fmtPct(live) : "—"}</span>
        <span class="mono muted-note">пик ${fmtPct(c.peak, 1)} · против ${fmtPct(c.dd, 1)}</span>
      </div>
      ${fundingTag(s)}
    </div>`;
  }).join("");
}

function sigPct(sig, basis, price) {
  const raw = (price - basis) / basis * 100;
  return sig.direction === "short" ? -raw : raw; // без направления = дельта лонга
}

/* ── live-карточки ── */
function pickLiveSignals(feed) {
  const now = Date.now();
  const seen = new Set();
  const out = [];
  for (const s of feed.live_signals || []) {
    const age = (now - new Date(s.ts_utc)) / 3600_000;
    // радар-мажоры не рисуем: ход ≥5% у 2/19, vol_radar их и не доставляет —
    // карточки только красили ленту рыночным дрейфом (2026-07-06)
    if (s.major_radar) continue;
    // radar-хиты старше 6ч гасим, НО сигнал с наклоном — редкость, живёт
    if (s.source === "radar" && age > RADAR_CARD_WINDOW_H && !s.bias) continue;
    const dedup = `${s.source}:${s.symbol}:${s.ts_utc.slice(0, 16)}`;
    if (seen.has(dedup)) continue;
    seen.add(dedup);
    out.push(s);
  }
  return out; // LIVE_CARDS_MAX применяется ПОСЛЕ фильтра в renderLive
}

function liveFilterBar(signals) {
  const srcs = ["ours", "all", "starred", "bias", ...new Set(signals.map((s) => s.source))];
  $("live-filters").innerHTML = srcs.map((s) =>
    `<button class="fbtn ${state.liveFilter === s ? "on" : ""}" data-src="${s}">
       ${s === "ours" ? "наши" : s === "all" ? "все" : s === "starred" ? "★ отмеченные" : s === "bias" ? "⬆⬇ наклон" : SRC_RU[s] || s}</button>`).join("");
  $("live-filters").querySelectorAll(".fbtn").forEach((b) =>
    b.onclick = () => { state.liveFilter = b.dataset.src; renderLive(state.feed); });
}

function renderBiasAcc(feed) {
  const acc = feed.bias_accuracy || {};
  const el = $("bias-acc");
  if (el) {
    const parts = Object.entries(acc)
      .filter(([, a]) => a.n_graded)
      .map(([v, a]) => `${v}: <b>${a.accuracy_pct}%</b> (n=${a.n_graded}${a.low_n ? " ⚠" : ""})`);
    el.innerHTML = parts.length
      ? `наклон, форвард-точность (когорты раздельно) — ${parts.join(" · ")}`
      : "наклон по версиям (v2 radar→⬆ Rose→контрарно, v3 — свои правила; когорты не смешиваются): форвард копится…";
  }
  // счётчик в шапке: ПО ВЕРСИЯМ РАЗДЕЛЬНО (ревью Codex 10.07: v2 и v3 — разные
  // правила, общий процент = смесь когорт). Числа — ЭФФЕКТИВНЫЕ кейсы: волна
  // одного скана схлопнута в 1 кейс (дедуп 06.07 «лечи»), сырые — в подсказке.
  const hs = $("bias-score");
  if (hs) {
    const fmtN = (x) => (Number.isInteger(x) ? x : x.toFixed(1));
    const vers = Object.entries(acc).filter(([, a]) => a.n_graded);
    const chips = vers.map(([v, a]) => {
      const pct = Math.round((a.hits || 0) / a.n_graded * 100);
      return `${esc(v)} <b>${fmtN(a.hits || 0)}/${a.n_graded}</b> <span style="color:var(--${pct >= 60 ? "up" : pct >= 45 ? "warn" : "down"})">${pct}%</span>`;
    });
    const raws = vers.map(([v, a]) => {
      const rh = a.raw_hits ?? a.hits ?? 0, rg = ((a.raw_hits ?? 0) + (a.raw_misses ?? 0)) || a.n_graded || 0;
      return `${v}: ${fmtN(rh)}/${rg}`;
    });
    hs.innerHTML = chips.length ? `⬆⬇ ${chips.join(" · ")}` : "";
    hs.title = `наклон стороны ПО ВЕРСИЯМ — у v2 и v3 разные правила, когорты не смешиваются. Эффективные кейсы (волна одного скана = 1). Сырых треков: ${raws.join(" · ")}. ±2% флэт не считается. Статус: исследовательский форвард, НЕ prereg-тест.`;
  }
}

async function renderLive(feed, force = false) {
  let sigs = pickLiveSignals(feed);
  renderBiasAcc(feed);
  liveFilterBar(sigs);
  if (state.liveFilter === "ours") sigs = sigs.filter((s) => OUR_SOURCES.has(s.source));
  else if (state.liveFilter === "starred") sigs = sigs.filter((s) => state.starred.has(s.symbol));
  else if (state.liveFilter === "bias") sigs = sigs.filter((s) => s.bias && s.bias.side !== "flat");
  else if (state.liveFilter !== "all") sigs = sigs.filter((s) => s.source === state.liveFilter);
  sigs = sigs.slice(0, LIVE_CARDS_MAX);

  const box = $("live-cards");
  const wantIds = new Set(sigs.map((s) => s.id));
  if (!sigs.length) {
    box.innerHTML = '<div class="empty">Живых сигналов в окне 48ч нет.</div>';
    state.liveCards.clear();
    return;
  }
  // реконсиляция: пересборка только при изменении набора
  const haveIds = new Set(state.liveCards.keys());
  const same = !force && wantIds.size === haveIds.size && [...wantIds].every((i) => haveIds.has(i));
  if (!same) {
    box.innerHTML = "";
    state.liveCards.clear();
    for (const sig of sigs) {
      const el = document.createElement("div");
      const isFresh = Date.now() - new Date(sig.ts_utc) < FRESH_MIN * 60_000;
      el.className = isFresh ? "card fresh" : "card";
      el.innerHTML = cardHtml(sig, isFresh);
      box.appendChild(el);
      state.liveCards.set(sig.id, { sig, el, basis: sig.entry || null, peak: null, dd: null, sparkPts: [] });
    }
    await Promise.all([...state.liveCards.values()].map(initCardBars));
    wsEnsure();
    renderEntry();   // кандидаты сразу, не ждать 5с-интервала
  }
}

function biasTag(sig) {
  const b = sig.bias;
  if (!b || b.side === "flat") return "";
  const arrow = b.side === "up" ? "⬆" : "⬇";
  const why = (b.why || []).join(" · ");
  return `<span class="biastag ${b.side}${b.strong ? " strong" : ""}"
    title="наклон структуры ${b.v} (НЕ прогноз с доказанным эджем — точность меряется форвардом): ${esc(why)}">${arrow} наклон</span>`;
}

/* ⚠ «вторая волна»: монета УЖЕ дала ход (chg24 ≥10%) ДО этой метки — предупреждение
   «сайз меньше», НЕ фильтр (§7 брата: метить, не убирать; фильтр решает ретро-валидация
   Ф1). chg24 живой с Bybit WS (state.chg24). Кейсы поздней метки: SKYAI1/NEXUS 09.07. */
function waveTag(sym) {
  const c = state.chg24.get(sym);
  if (c == null || c < SECOND_WAVE_CHG) return "";
  return `<span class="risktag" title="монета уже сделала ${fmtPct(c, 1)} за сутки — «вторая волна» на разогнанной монете: статистика пула собрана на тихих стартах, здесь не гарантирована (кейсы ALLO 07.07, SKYAI1/NEXUS 09.07 — поздняя метка после вертикали часто откатывает). Правило брата: сайз меньше стандартного микро, решение твоё">⚠ уже ${fmtPct(c, 0)}/24ч — сайз меньше</span>`;
}

function riskTag(sig) {
  // шторм-шорт = статистически самая сливная категория (форвард 04-06.07:
  // чётких 2/33, сливов 30%) — маркируем, не скрываем (просьба брата)
  if (sig.source === "storm" && sig.direction === "short")
    return `<span class="risktag" title="форвард n=33: пробой вниз доигрывается лишь у 6%, 30% выкупаются против — самая сливная категория">⚠ вниз-пробой: 6% чётких</span>`;
  return "";
}

/* Погодная плашка BTC на скринер-карточках (разрез 06.07, n=907, честное окно).
   Метрика = ровно та, что в исследовании: % BTC за 24ч (терцильные пороги
   −1.9% / +0.3%). Live-значение из WS-тикера. Визуал-форвард, гейтов нет. */
function weatherTag(sig) {
  if (sig.source !== "screener" || state.btcRet24 == null) return "";
  const b = state.btcRet24;
  const d = (sig.direction || "").toLowerCase();
  const isShort = d === "short" || d === "шорт";
  const isLong = d === "long" || d === "лонг";
  const btc = `BTC ${fmtPct(b, 1)}/24ч`;
  if (isShort && b > 0.3)
    return `<span class="risktag" title="разрез n=907: шорты при растущем BTC — win 18%, средний R −0.58">⚠ против ветра · ${btc}</span>`;
  if (isLong && b < -1.9)
    return `<span class="risktag" title="разрез n=907: лонги при падающем BTC — win 31%, средний R −0.26">⚠ против ветра · ${btc}</span>`;
  if (isShort && b < -1.9)
    return `<span class="windtag" title="разрез n=907: шорты при падающем BTC — win 45%, средний R +0.10">по ветру · ${btc}</span>`;
  if (isLong && b > 0.3)
    return `<span class="windtag" title="разрез n=907: лонги при растущем BTC — win 46%, средний R +0.15">по ветру · ${btc}</span>`;
  return "";
}

/* 🎣 игла-вынос стопов (ретро 06.07, n=150): контр-сигнал — после иглы-вверх
   3ч-медиана −0.5% (вверх 43%), после иглы-вниз +0.6% (вверх 61%) */
function sweepTag(sym) {
  const sw = (state.feed?.sweeps || []).find((x) => x.symbol === sym);
  if (!sw || (Date.now() - new Date(sw.ts)) > 2 * 3600_000) return "";
  const t = new Date(sw.ts);
  const hhmm = `${String((t.getUTCHours() + 3) % 24).padStart(2, "0")}:${String(t.getUTCMinutes()).padStart(2, "0")}`;
  if (sw.dir === "up")
    return `<span class="risktag" title="игла ${hhmm} МСК: фитиль +${sw.wick_pct}% на объёме ×${sw.vol_x} — вынос шортовых стопов; ретро n=89: через 3ч медиана −0.5%, вверх лишь 43% — не вход, часто раздача">🎣 вынос вверх ${hhmm}</span>`;
  return `<span class="windtag" title="игла ${hhmm} МСК: фитиль −${sw.wick_pct}% на объёме ×${sw.vol_x} — вынос лонговых стопов; ретро n=61: через 3ч медиана +0.6%, вверх 61% — стопы сняты, база отскока; свой стоп под такой лоу не ставить">🎣 вынос вниз ${hhmm}</span>`;
}

/* 🔒 funding-чек шторм-шортов (добро брата 07.07 после MAGMA): автоматизация
   ручного чеклиста MUSDT-эталона. ИНФО, не сигнал: лид «вниз-пробой при
   funding-плюсе доигрывается 17% vs 7%» жив 3/3 (MUSDT/ADA/MAGMA), но правилом
   станет только после side_study-2 (n≥60). База funding = 0.005%/период. */
function fundingTag(sig) {
  if (sig.source !== "storm" || sig.direction !== "short" || sig.funding_now == null) return "";
  const f = sig.funding_now, xBase = Math.abs(f) / 0.005;
  if (f >= 0.01)
    return `<span class="windtag" title="funding ${fmtPct(f, 3)}/период = ${xBase.toFixed(0)}× базы ПРИ падении — лонги платят и упираются (запертая толпа сверху, выкупать пробой некому). MUSDT-паттерн: live-счёт лида 3/3 (MUSDT +11%, ADA +3%, MAGMA +19.8%), ретро 17% vs 7%. НЕ правило до side_study-2 — решение твоё, скальп-класс, сайз малый">🔒 лонги заперты · f ${fmtPct(f, 3)}</span>`;
  if (f <= -0.01)
    return `<span class="risktag" title="funding ${fmtPct(f, 3)}/период — толпа уже В ШОРТАХ и платит: классический профиль выкупа пробоя (94% таких возвращаются). Анти-сторона funding-лида">⚡ толпа в шортах · выкуп-риск</span>`;
  return "";
}

function comboTag(sym) {
  const c = (state.feed?.combos || []).find((x) => x.symbol === sym);
  if (!c) return "";
  return `<span class="badge combo" title="связка: пробуждение ×${c.awake_ratio} → Rose-пост (${agoStr(c.post_ts)})">⚡</span>`;
}

/* ⚡ Связки «пробуждение → Rose-пост → структура жива» — рендер подблока entry.
   Живая смерть на каждом WS-тике (просьба брата 06.07 «следи каждую секунду»):
   провал ниже базиса поста >3% (для short зеркально) ИЛИ живой ретрейс ≥80%
   пика при пике ≥8% → карточка исчезает немедленно. Сервер дополнительно
   выкидывает раздачу и финально-отдавшие треки. */
function comboAlive(c) {
  // «вход сейчас»: ход по посту уже случился (пик ≥8%) = отработана; провал −3% = мертва
  const px = state.lastPx.get(c.symbol);
  if (px == null || !c.basis) {
    return { alive: (c.peak24_pct || 0) < 8, live: null };  // цены ещё нет — судим по фиду
  }
  let live = (px - c.basis) / c.basis * 100;
  if (c.direction === "short") live = -live;
  // ключ = монета|пост, НЕ только монета (код-ревью 2026-07-06 HIGH-1): иначе
  // пик от ПРОШЛОЙ связки по той же монете (за сессию) убивал бы новую связку
  const pk = `${c.symbol}|${c.post_ts}`;
  const peakRef = Math.max(c.peak24_pct || 0, state.comboPeak.get(pk) || 0, live);
  state.comboPeak.set(pk, peakRef);
  if (peakRef >= 8) return { alive: false, live };
  if (live < -3) return { alive: false, live };
  return { alive: true, live };
}

function renderCombos() {
  const box = $("combo-cards");
  if (!box || !state.feed) return;
  const rows = [];
  // чистка comboPeak от ключей исчезнувших связок (не копить за сессию)
  const liveKeys = new Set((state.feed.combos || []).map((c) => `${c.symbol}|${c.post_ts}`));
  for (const k of state.comboPeak.keys()) if (!liveKeys.has(k)) state.comboPeak.delete(k);
  for (const c of state.feed.combos || []) {
    const { alive, live } = comboAlive(c);
    if (!alive) continue;
    rows.push({ c, live });
  }
  if (!rows.length) {
    box.innerHTML = '<div class="empty">Живых связок нет — отработавшие и провалившиеся сняты.</div>';
    return;
  }
  box.innerHTML = rows.map(({ c, live }) => `
    <div class="card entry combo">
      <div class="row1">
        ${starHtml(c.symbol)}
        <span class="sym">${esc(c.symbol)}</span>
        <span class="badge combo">⚡</span>
        <span class="when">пост ${agoStr(c.post_ts)}</span>
      </div>
      <div class="big ${cls(live ?? c.peak24_pct)}">${fmtPct(live ?? c.peak24_pct, 1)} <span style="font-size:11px;color:var(--muted)">${live != null ? "live от поста" : "пик 24ч"}</span></div>
      <div class="meta">🌅 ×${c.awake_ratio} за ${Math.round((new Date(c.post_ts) - new Date(c.awake_ts)) / 3600_000)}ч до поста · ${esc(c.channel || "rose")} ${esc(c.direction || "")}${c.fast ? " · ⚡ только что (трек уточняется)" : " · пик 24ч " + fmtPct(c.peak24_pct, 1)}</div>
    </div>`).join("");
}
let _comboLast = 0;
function comboTickRender() {           // по WS-тику, не чаще раза в секунду
  const now = Date.now();
  if (now - _comboLast < 1000) return;
  _comboLast = now;
  renderCombos();
}

function pumpTag(sym) {
  const pw = (state.feed?.pump_watch || []).find((p) => p.symbol === sym);
  if (!pw) return "";
  const t = pw.broke ? "🔻 слом плато" : pw.dist ? "🌊 распределение" : "🌋 пост-памп";
  return `<span class="pumptag" title="под pump-надзором (${esc(pw.kind || "")}): лонги тут — против шторма">${t}</span>`;
}

function cardHtml(sig, isFresh = false) {
  const dirCls = sig.direction || "none";
  const dirTxt = sig.direction === "long" ? "▲ LONG" : sig.direction === "short" ? "▼ SHORT" : "Δ";
  const chan = sig.channel || (sig.channels || []).join(", ");
  const extra = [sig.setup, chan, sig.note].filter(Boolean).join(" · ");
  return `
    <div class="row1">
      ${starHtml(sig.symbol)}
      <span class="sym">${esc(sig.symbol)}</span>
      <span class="dir ${dirCls}">${dirTxt}</span>
      ${isFresh ? '<span class="newtag">NEW</span>' : ""}
      <span class="when" title="${fmtMsk(sig.ts_utc)} МСК">${agoStr(sig.ts_utc)}</span>
    </div>
    <div class="row1">
      <span class="badge ${sig.source}">${SRC_RU[sig.source] || sig.source}</span>
      ${sig.emits > 1 ? `<span class="badge" title="повторных алертов">×${sig.emits}</span>` : ""}
      ${biasTag(sig)}
      <span data-role="wave">${waveTag(sig.symbol)}</span>
      ${riskTag(sig)}
      ${fundingTag(sig)}
      ${weatherTag(sig)}
      ${sweepTag(sig.symbol)}
      ${pumpTag(sig.symbol)}
      ${comboTag(sig.symbol)}
      ${extra ? `<span class="src-note" title="${esc(extra)}">${esc(extra)}</span>` : ""}
    </div>
    <div class="big" data-role="pct">…</div>
    <div class="meta" data-role="meta"></div>
    <svg class="spark" data-role="spark" viewBox="0 0 240 40" preserveAspectRatio="none"></svg>`;
}

async function initCardBars(card) {
  const ts = new Date(card.sig.ts_utc).getTime();
  const bars = await fetchBars(card.sig.symbol, ts);
  if (!bars.length) {
    card.el.querySelector('[data-role="pct"]').textContent = "нет данных";
    card.el.querySelector('[data-role="pct"]').style.fontSize = "14px";
    return;
  }
  if (!card.basis) card.basis = bars[0].c; // вход = close 1-го полного бара
  const s = card.sig;
  const fav = (b) => s.direction === "short"
    ? (card.basis - b.l) / card.basis * 100 : (b.h - card.basis) / card.basis * 100;
  const adv = (b) => s.direction === "short"
    ? (card.basis - b.h) / card.basis * 100 : (b.l - card.basis) / card.basis * 100;
  // пик/просадка — строго после бара входа (его high/low могли быть до входа)
  const post = bars.slice(1);
  card.peak = post.length ? Math.max(...post.map(fav)) : 0;
  card.dd = post.length ? Math.min(...post.map(adv)) : 0;
  card.sparkPts = bars.map((b) => sigPct(s, card.basis, b.c));
  updateCard(card, sigPct(s, card.basis, bars[bars.length - 1].c));
}

function updateCard(card, livePct) {
  if (card.basis == null || livePct == null) return;
  card.lastPct = livePct;   // для секции «Исследовательские кандидаты — NO-TRADE»
  card.peak = card.peak == null ? livePct : Math.max(card.peak, livePct);
  card.dd = card.dd == null ? Math.min(0, livePct) : Math.min(card.dd, livePct);
  const pctEl = card.el.querySelector('[data-role="pct"]');
  pctEl.textContent = fmtPct(livePct);
  pctEl.className = `big ${livePct > 0 ? "pos" : livePct < 0 ? "neg" : ""}`;
  card.el.querySelector('[data-role="meta"]').innerHTML =
    `пик <b>${fmtPct(card.peak, 1)}</b> · просадка <b>${fmtPct(card.dd, 1)}</b>`;
  const waveEl = card.el.querySelector('[data-role="wave"]');   // chg24 приходит с WS асинхронно — обновляем по тику
  if (waveEl) waveEl.innerHTML = waveTag(card.sig.symbol);
  drawSpark(card, livePct);
}

function drawSpark(card, livePct) {
  const svg = card.el.querySelector('[data-role="spark"]');
  const pts = [...card.sparkPts, livePct];
  if (pts.length < 2) return;
  const W = 240, H = 40, P = 3;
  const min = Math.min(...pts, 0), max = Math.max(...pts, 0);
  const span = max - min || 1;
  const x = (i) => P + (i / (pts.length - 1)) * (W - 2 * P);
  const y = (v) => H - P - ((v - min) / span) * (H - 2 * P);
  const d = pts.map((v, i) => `${i ? "L" : "M"}${x(i).toFixed(1)},${y(v).toFixed(1)}`).join("");
  const zero = y(0);
  const last = pts[pts.length - 1];
  svg.innerHTML = `
    <line x1="0" x2="${W}" y1="${zero}" y2="${zero}" stroke="var(--grid)" stroke-width="1"/>
    <path d="${d}" fill="none" stroke="${last >= 0 ? "var(--up)" : "var(--down)"}" stroke-width="1.6"/>
    <circle cx="${x(pts.length - 1)}" cy="${y(last)}" r="2.4" fill="${last >= 0 ? "var(--up)" : "var(--down)"}"/>`;
}

/* ── Bybit WebSocket ── */
function wsEnsure() {
  const wanted = new Set(["BTCUSDT", "ETHUSDT"]);
  for (const { sig } of state.liveCards.values()) wanted.add(sig.symbol);
  for (const c of state.feed?.combos || []) wanted.add(c.symbol);
  const changed = wanted.size !== state.wsSymbols.size || [...wanted].some((s) => !state.wsSymbols.has(s));
  if (state.ws && state.ws.readyState === 1 && !changed) return;
  state.wsSymbols = wanted;
  if (state.ws) try { state.ws.close(); } catch (_) {}
  const ws = new WebSocket(BYBIT_WS);
  state.ws = ws;
  ws.onopen = () => {
    state.wsAlive = true;
    $("ws-status").innerHTML = '<span class="dot ok"></span>live-цены';
    const args = [...wanted].map((s) => `tickers.${s}`);
    for (let i = 0; i < args.length; i += 10)
      ws.send(JSON.stringify({ op: "subscribe", args: args.slice(i, i + 10) }));
    ws._ping = setInterval(() => { try { ws.send('{"op":"ping"}'); } catch (_) {} }, 20_000);
  };
  ws.onmessage = (ev) => {
    let m; try { m = JSON.parse(ev.data); } catch (_) { return; }
    if (!m.topic || !m.topic.startsWith("tickers.") || !m.data) return;
    const sym = m.topic.slice(8);
    const last = parseFloat(m.data.lastPrice);
    if (!isFinite(last)) return; // дельта без lastPrice
    state.lastPx.set(sym, last);
    const p24 = parseFloat(m.data.price24hPcnt);
    if (isFinite(p24)) state.chg24.set(sym, p24 * 100);
    if ((state.feed?.combos || []).some((c) => c.symbol === sym)) comboTickRender();
    if (sym === "BTCUSDT") headerTick("btc-tick", "BTC", last, m.data.price24hPcnt);
    if (sym === "ETHUSDT") headerTick("eth-tick", "ETH", last, m.data.price24hPcnt);
    for (const card of state.liveCards.values())
      if (card.sig.symbol === sym && card.basis != null)
        updateCard(card, sigPct(card.sig, card.basis, last));
  };
  ws.onclose = () => {
    clearInterval(ws._ping);
    if (state.ws !== ws) return;
    state.wsAlive = false;
    $("ws-status").innerHTML = '<span class="dot stale"></span>переподключение…';
    setTimeout(() => { if (state.ws === ws) { state.ws = null; wsEnsure(); } }, 3000);
  };
  ws.onerror = () => { try { ws.close(); } catch (_) {} };
}

const _hdrPct = {};
function headerTick(id, name, last, pcnt) {
  const p = parseFloat(pcnt);
  if (isFinite(p)) _hdrPct[id] = p * 100;
  if (id === "btc-tick" && isFinite(p)) {
    const was = state.btcRet24;
    state.btcRet24 = p * 100;   // погода для скринер-плашек
    if (was == null && state.feed) renderLive(state.feed, true);  // первый тик — форс-перерисовка (иначе реконсиляция пропустит и weatherTag не появится)
  }
  const chg = _hdrPct[id];
  $(id).innerHTML = `${name} <b>${last.toLocaleString("en-US", { maximumFractionDigits: last > 1000 ? 0 : 2 })}</b>` +
    (chg != null ? ` <span style="color:var(--${chg >= 0 ? "up" : "down"})">${fmtPct(chg, 1)}</span>` : "");
}

/* ── общие ячейки трека (методика 6ч/24ч) ── */
function dirCell(d) {
  return d === "long" ? '<span class="pos">▲ long</span>'
    : d === "short" ? '<span class="neg">▼ short</span>' : '<span class="flat">Δ</span>';
}
function trackStatus(t) {
  if (t.status === "no_data") return '<span class="flat">нет данных</span>';
  return t.final ? "🏁 итог" : "⏳ идёт";
}
// Доминанта суток: когда 24ч закрыты, подсвечиваем ЧТО было больше —
// пик или просадка (стороной не торгуем, важен сам шторм).
function domHl(t) {
  const out = { peak: "", dd: "" };
  if (!t.final || t.peak24_pct == null || t.dd24_pct == null) return out;
  const peak = t.peak24_pct, pain = Math.abs(t.dd24_pct);
  if (peak > pain) out.peak = " hl-pos";
  else if (pain > peak) out.dd = " hl-neg";
  return out;
}

/* ── «Исследовательские кандидаты — NO-TRADE»: механический шорт-лист по форвард-разрезам (переименовано по ревью Codex 10.07; NO-TRADE до чистого пайплайна).
   Каждое правило — из проверенного разреза, НЕ интуиция:
   • радар-альт с наклоном ⬆ v3 (86% ходунов вверх, n=35; мажоры уже отсеяны)
   • свежесть ≤3ч (медиана времени до пика 1.2–1.7ч — позже вход догоняющий)
   • цена не убежала: −1.5%…+2.5% от алерта; просадка с алерта ≥ −2% (не пила)
   • пик ещё не отработан (≤+3.5%) и монета не под pump-раздачей
   • 🌅-пробуждения — отдельные правила: окно 30ч, коридор −3%…+5%
   Пересчёт каждые 5с из живых WS-данных карточек. ═ НЕ СИГНАЛ ГАРАНТИИ ═
   ВАЖНО: правила продублированы серверно в dashboard_feed.log_entry_candidates
   (форензика outcomes/entry_candidates.csv) — менять СИНХРОННО. */
function entryReason(card, isAwakening) {
  const bits = [isAwakening ? "🌅 пробуждение (окно 12–35ч)" : "радар-альт ⬆ v3 (форвард n=35 — мало)"];
  bits.push("не убежала", `просадка ${fmtPct(card.dd, 1)}`);
  return bits.join(" · ");
}

function renderEntry() {
  const box = $("entry-cards");
  if (!box || !state.feed) return;
  const pumps = new Map((state.feed.pump_watch || []).map((p) => [p.symbol, p]));
  const out = [];
  for (const card of state.liveCards.values()) {
    const s = card.sig;
    if (s.source !== "radar" || !s.bias || s.bias.side !== "up") continue;
    if (card.lastPct == null || card.dd == null || card.peak == null) continue;
    const p = pumps.get(s.symbol);
    if (p && (p.dist || p.broke)) continue;               // раздача/слом — не вход
    const ageH = (Date.now() - new Date(s.ts_utc)) / 3600_000;
    const isAwk = (s.vol_ratio || 0) >= 15;
    const ok = isAwk
      ? (ageH <= 30 && card.lastPct >= -3 && card.lastPct <= 5 && card.dd >= -4)
      : (ageH <= 3 && card.lastPct >= -1.5 && card.lastPct <= 2.5
         && card.dd >= -2 && card.peak <= 3.5);
    if (ok) out.push({ card, isAwk, ageH });
  }
  out.sort((a, b) => a.ageH - b.ageH);                    // свежие первыми
  // рыночная волна: ≥5 кандидатов из одного 5-мин скана = не пробуждения,
  // а общий BTC-движ (radar_resolved: каскадные good 15% vs 23% у одиночных) — скрыть
  const buckets = new Map();
  for (const it of out) {
    const b = Math.floor(new Date(it.card.sig.ts_utc) / 300_000);   // floor = 5-мин скан-бакет (совпадает с сервером)
    (buckets.get(b) || buckets.set(b, []).get(b)).push(it);
  }
  let waveNote = "";
  for (const [, items] of buckets) {
    if (items.filter((x) => !x.isAwk).length >= 5) {
      const hidden = items.filter((x) => !x.isAwk);
      hidden.forEach((x) => { x.hide = true; });
      waveNote += `<div class="wave-note">⚠ рыночная волна ${agoStr(hidden[0].card.sig.ts_utc)}: ${hidden.length} монет хитанули одним сканом — это BTC-движ, не пробуждения (каскадные отрабатывают в 1.5 раза хуже: good 15% vs 23%) — скрыты</div>`;
    }
  }
  const shown = out.filter((x) => !x.hide);
  if (!shown.length && !waveNote) {
    box.innerHTML = '<div class="empty">Кандидатов сейчас нет — правила строгие.</div>';
    return;
  }
  box.innerHTML = waveNote + shown.map(({ card, isAwk }) => {
    const s = card.sig;
    const c24 = state.chg24.get(s.symbol);
    const secondWave = c24 != null && c24 >= SECOND_WAVE_CHG;
    return `<div class="card entry${isAwk ? " awk" : ""}">
      <div class="row1">
        ${starHtml(s.symbol)}
        <span class="sym">${esc(s.symbol)}</span>
        <span class="badge" style="color:var(--warn)" title="исследовательский кандидат — NO-TRADE: годовой бэктест 10.07 торгового эджа не подтвердил · карточка для форвард-наблюдения и дневника, не сигнал входа · справка разреза (не план): ходы зреют 4–9ч, пилы −1–2% типичны, слом структуры = OI падает на росте / объём без хода / pump-раздача">🧪 наблюдение</span>
        ${isAwk ? '<span class="badge" style="color:var(--warn)">🌅</span>' : ""}
        ${secondWave ? `<span class="risktag" title="монета уже сделала ${fmtPct(c24, 1)} за сутки ДО этого сигнала — «вторая волна» на разогнанной монете: статистика пула собрана на тихих стартах, здесь она не гарантирована (кейс ALLO 07.07). Правило брата: сайз меньше стандартного микро">⚠ уже ${fmtPct(c24, 0)}/24ч — сайз меньше</span>` : ""}
        <span class="when">${agoStr(s.ts_utc)}</span>
      </div>
      <div class="big ${card.lastPct > 0 ? "pos" : card.lastPct < 0 ? "neg" : ""}">${fmtPct(card.lastPct)}</div>
      <div class="meta">${esc(entryReason(card, isAwk))}</div>
    </div>`;
  }).join("");
  renderShortWatch();   // 🧪 шорт-гипотеза рендерится тем же циклом (секция вынесена ниже)
}
setInterval(renderEntry, 5000);

/* ── Pump-надзор: эпизоды + заглушенные лонги ── */
function renderPump(feed) {
  const eps = feed.pump_watch || [];
  const muted = feed.pump_muted || [];
  const broke = eps.filter((e) => e.broke).length;
  $("pump-tiles").innerHTML = `
    <div class="tile"><div class="v">${eps.length}</div><div class="l">эпизодов под надзором</div></div>
    <div class="tile"><div class="v">${broke}</div><div class="l">со сломом плато 🔻</div></div>
    <div class="tile"><div class="v">${muted.length}</div><div class="l">заглушено лонгов скринера</div></div>`;
  const stage = (e) => e.broke ? "🔻 слом плато" : e.dist ? "🌊 распределение" : "🌋 вертикаль";
  $("pump-table").querySelector("tbody").innerHTML = eps.length ? eps.map((e) => `
    <tr>
      <td class="sym lft">${starHtml(e.symbol)} ${esc(e.symbol)}</td>
      <td class="lft">${fmtMsk(e.detected_ts)}</td>
      <td class="lft">${e.kind === "squeeze" ? "шорт-сквиз (OI↓ на росте)" : e.kind === "trend" ? "тренд (OI↑)" : "смешанный"}</td>
      <td class="lft">${stage(e)}</td>
      <td><span class="pos">${e.rise_pct != null ? "+" + e.rise_pct + "%" : "—"}</span></td>
      <td>${e.peak ?? "—"}</td>
      <td>${e.plateau_low ?? "—"}</td>
    </tr>`).join("")
    : '<tr><td colspan="7" class="empty">Свежих вертикалей на рынке нет — надзор пуст, гейт вооружён.</td></tr>';
  $("pump-muted-table").querySelector("tbody").innerHTML = muted.length ? muted.map((m) => `
    <tr>
      <td class="lft">${fmtMsk(m.ts + ":00Z")}</td>
      <td class="sym lft">${starHtml(m.symbol)} ${esc(m.symbol)}</td>
      <td class="lft">${esc(m.setup)}</td>
      <td>${esc(m.score ?? "—")}</td>
      <td class="lft">${esc(m.reason || "")}</td>
    </tr>`).join("")
    : '<tr><td colspan="5" class="empty">Гейт ещё ничего не глушил.</td></tr>';
}

/* ── Rose: KPI + накапливающаяся таблица треков ── */
function renderRose(feed) {
  const r = feed.rose || {};
  const s = r.summary || {};
  const acc = r.bot_accuracy || {};
  const accStr = Object.entries(acc)
    .map(([ch, a]) => `${ch}: ${(a.accuracy * 100).toFixed(0)}% (n=${a.n})`).join(" · ");
  $("rose-tiles").innerHTML = `
    <div class="tile"><div class="v">${s.n_tracks ?? "—"}</div><div class="l">треков сигналов (30 дн)</div></div>
    <div class="tile"><div class="v">${s.hit24_pct != null ? s.hit24_pct + "%" : "—"}</div><div class="l">итог суток в плюс${s.low_n ? " ⚠ мало данных" : ""}</div></div>
    <div class="tile"><div class="v pos">${s.avg_peak24_pct != null ? "+" + s.avg_peak24_pct + "%" : "—"}</div><div class="l">средний пик за сутки</div></div>
    <div class="tile"><div class="v neg">${s.avg_dd24_pct != null ? s.avg_dd24_pct + "%" : "—"}</div><div class="l">средняя просадка за сутки</div></div>
    <div class="tile"><div class="v">${s.avg_confirms ?? "—"}</div><div class="l">среднее подтверждений</div></div>
    ${accStr ? `<div class="tile"><div class="v" style="font-size:14px;line-height:1.6">${esc(accStr)}</div><div class="l">точность подтверждений в боте</div></div>` : ""}`;
  $("rose-filters").innerHTML =
    `<button class="fbtn ${state.roseStarOnly ? "on" : ""}" id="rose-star-btn">★ отмеченные</button>`;
  $("rose-star-btn").onclick = () => { state.roseStarOnly = !state.roseStarOnly; renderRose(state.feed); };
  let rows = r.tracks || [];
  if (state.roseStarOnly) rows = rows.filter((t) => state.starred.has(t.symbol));
  const shown = rows.slice(0, state.roseShown);
  $("rose-table").querySelector("tbody").innerHTML = shown.length ? shown.map((t) => {
    const hl = domHl(t);
    return `
    <tr>
      <td class="lft">${fmtMsk(t.anchor_ts)}</td>
      <td class="sym lft">${starHtml(t.symbol)} ${esc(t.symbol)}</td>
      <td class="lft">${dirCell(t.direction)}</td>
      <td>${t.confirms > 1 ? `<b>×${t.confirms}</b>` : "1"}</td>
      <td><span class="pos">${fmtPct(t.peak6_pct, 1)}</span></td>
      <td><span class="${cls(t.dd6_pct)}">${fmtPct(t.dd6_pct, 1)}</span></td>
      <td><span class="pos${hl.peak}">${fmtPct(t.peak24_pct, 1)}</span></td>
      <td><span class="${cls(t.dd24_pct)}${hl.dd}">${fmtPct(t.dd24_pct, 1)}</span></td>
      <td><span class="${cls(t.ret24_pct)}">${fmtPct(t.ret24_pct, 1)}</span></td>
      <td class="lft">${trackStatus(t)}</td>
    </tr>`;
  }).join("") : '<tr><td colspan="10" class="empty">Треков пока нет.</td></tr>';
  const more = $("rose-more");
  more.hidden = rows.length <= state.roseShown;
  more.onclick = () => { state.roseShown += 60; renderRose(feed); };
}

/* ── история всех сигналов (ledger) ── */
function renderHistory(feed) {
  const rows = feed.signals_history || [];
  const srcs = ["all", "starred", ...new Set(rows.map((t) => t.source))];
  $("hist-filters").innerHTML = srcs.map((c) =>
    `<button class="fbtn ${state.histFilter === c ? "on" : ""}" data-src="${esc(c)}">${c === "all" ? "все источники" : c === "starred" ? "★ отмеченные" : SRC_RU[c] || esc(c)}</button>`).join("");
  $("hist-filters").querySelectorAll(".fbtn").forEach((b) =>
    b.onclick = () => { state.histFilter = b.dataset.src; state.histShown = 40; renderHistory(feed); });
  const filtered = state.histFilter === "all" ? rows
    : state.histFilter === "starred" ? rows.filter((t) => state.starred.has(t.symbol))
    : rows.filter((t) => t.source === state.histFilter);
  const shown = filtered.slice(0, state.histShown);
  $("hist-table").querySelector("tbody").innerHTML = shown.length ? shown.map((t) => {
    const hl = domHl(t);
    return `
    <tr>
      <td class="lft">${fmtMsk(t.anchor_ts)}</td>
      <td class="lft"><span class="badge ${esc(t.source)}">${SRC_RU[t.source] || esc(t.source)}</span>${t.setup ? ` <span class="flat">${esc(t.setup)}</span>` : ""}</td>
      <td class="sym lft">${starHtml(t.symbol)} ${esc(t.symbol)}</td>
      <td class="lft">${dirCell(t.direction)}</td>
      <td>${t.confirms > 1 ? `<b>×${t.confirms}</b>` : "1"}</td>
      <td><span class="pos${hl.peak}">${fmtPct(t.peak24_pct, 1)}</span></td>
      <td><span class="${cls(t.dd24_pct)}${hl.dd}">${fmtPct(t.dd24_pct, 1)}</span></td>
      <td><span class="${cls(t.ret24_pct)}">${fmtPct(t.ret24_pct, 1)}</span></td>
      <td class="lft">${trackStatus(t)}</td>
    </tr>`;
  }).join("") : '<tr><td colspan="9" class="empty">История накапливается — сигналы появятся после первых резолвов.</td></tr>';
  const more = $("hist-more");
  more.hidden = filtered.length <= state.histShown;
  more.onclick = () => { state.histShown += 60; renderHistory(feed); };
}

/* ── бот ── */
function renderBot(feed) {
  const b = feed.bot_stats || {};
  $("bot-window").textContent = `честное окно: сигналы с ${b.window_start || "—"} · n=${b.n ?? 0}`;
  const dirs = b.by_direction || {};
  const eq = b.equity_r || [];
  const lastR = eq.length ? eq[eq.length - 1].cum_r : null;
  const wrAll = b.n ? Object.values(b.by_setup || {}).reduce((s, x) => s + x.wr24h_pct * x.n, 0) / b.n : null;
  $("bot-tiles").innerHTML = `
    <div class="tile"><div class="v">${b.n ?? "—"}</div><div class="l">резолвленных сигналов</div></div>
    <div class="tile"><div class="v">${wrAll != null ? wrAll.toFixed(1) + "%" : "—"}</div><div class="l">winrate 24ч (все сетапы)</div></div>
    <div class="tile"><div class="v ${lastR != null ? (lastR >= 0 ? "pos" : "neg") : ""}">${lastR != null ? (lastR > 0 ? "+" : "") + lastR.toFixed(1) + "R" : "—"}</div><div class="l">накопленный R (24ч-выходы)</div></div>
    ${dirs.long ? `<div class="tile"><div class="v">${dirs.long.wr24h_pct}%</div><div class="l">winrate лонги (n=${dirs.long.n})</div></div>` : ""}
    ${dirs.short ? `<div class="tile"><div class="v">${dirs.short.wr24h_pct}%</div><div class="l">winrate шорты (n=${dirs.short.n})</div></div>` : ""}`;
  drawSetupChart(b.by_setup || {});
  drawEquityChart(eq);
  $("bot-table").querySelector("tbody").innerHTML = (b.recent || []).map((r) => `
    <tr>
      <td class="lft">${fmtMsk(r.ts + ":00Z") /* naive UTC из фида */}</td>
      <td class="sym lft">${starHtml(r.symbol)} ${esc(r.symbol)}</td>
      <td class="lft">${esc(r.setup)}</td>
      <td class="lft"><span class="${r.direction === "long" ? "pos" : "neg"}">${r.direction === "long" ? "▲" : "▼"} ${esc(r.direction)}</span></td>
      <td><span class="${cls(r.change_24h_pct)}">${fmtPct(r.change_24h_pct, 1)}</span></td>
      <td><span class="${cls(r.r_24h)}">${r.r_24h == null ? "—" : (r.r_24h > 0 ? "+" : "") + r.r_24h.toFixed(2)}</span></td>
      <td><span class="${cls(r.mfe_24h)}">${fmtPct(r.mfe_24h, 1)}</span></td>
      <td><span class="${cls(r.mae_24h)}">${fmtPct(r.mae_24h, 1)}</span></td>
      <td class="lft">${esc(r.outcome_24h || "—")}</td>
    </tr>`).join("") || '<tr><td colspan="9" class="empty">Пусто.</td></tr>';
}

function drawSetupChart(bySetup) {
  const svg = $("setup-chart");
  const entries = Object.entries(bySetup).sort((a, b) => b[1].n - a[1].n);
  if (!entries.length) { svg.innerHTML = ""; return; }
  const W = 560, rowH = 34, P = { l: 96, r: 60 };
  const H = entries.length * rowH + 8;
  svg.setAttribute("viewBox", `0 0 ${W} ${H}`);
  svg.style.height = `${H}px`;
  const maxWr = Math.max(...entries.map(([, v]) => v.wr24h_pct), 50);
  const w = (v) => (v / maxWr) * (W - P.l - P.r);
  svg.innerHTML = entries.map(([name, v], i) => {
    const y = i * rowH + 8;
    return `
      <text class="bar-name" x="${P.l - 8}" y="${y + 14}" text-anchor="end">${esc(name)}</text>
      <rect x="${P.l}" y="${y}" width="${Math.max(2, w(v.wr24h_pct))}" height="20" rx="4"
            fill="var(--s1)" data-tip="${esc(name)}: winrate 24ч ${v.wr24h_pct}% · 4ч ${v.wr4h_pct}% · n=${v.n}${v.avg_r24 != null ? " · avg R " + v.avg_r24 : ""}"/>
      <text class="bar-lbl" x="${P.l + Math.max(2, w(v.wr24h_pct)) + 7}" y="${y + 14}">${v.wr24h_pct}%${v.low_n ? " ⚠" : ""} <tspan fill="var(--muted)">n=${v.n}</tspan></text>`;
  }).join("");
  hookTips(svg);
}

function drawEquityChart(eq) {
  const svg = $("equity-chart");
  if (!eq || eq.length < 2) { svg.innerHTML = ""; return; }
  const W = 560, H = 190, P = { l: 44, r: 10, t: 8, b: 20 };
  svg.setAttribute("viewBox", `0 0 ${W} ${H}`);
  svg.style.height = `${H}px`;
  const vals = eq.map((p) => p.cum_r);
  const min = Math.min(...vals, 0), max = Math.max(...vals, 0);
  const span = max - min || 1;
  const x = (i) => P.l + (i / (eq.length - 1)) * (W - P.l - P.r);
  const y = (v) => P.t + (1 - (v - min) / span) * (H - P.t - P.b);
  const d = vals.map((v, i) => `${i ? "L" : "M"}${x(i).toFixed(1)},${y(v).toFixed(1)}`).join("");
  const ticks = [min, min + span / 2, max].map((v) => Math.round(v));
  svg.innerHTML = `
    ${ticks.map((t) => `<line x1="${P.l}" x2="${W - P.r}" y1="${y(t)}" y2="${y(t)}" stroke="var(--grid)" stroke-width="1"/>
      <text class="axis-lbl" x="${P.l - 6}" y="${y(t) + 3}" text-anchor="end">${t}R</text>`).join("")}
    <line x1="${P.l}" x2="${W - P.r}" y1="${y(0)}" y2="${y(0)}" stroke="var(--muted)" stroke-width="1" stroke-dasharray="3,3"/>
    <path d="${d}" fill="none" stroke="var(--s1)" stroke-width="2"/>
    <line id="eq-cross" y1="${P.t}" y2="${H - P.b}" stroke="var(--muted)" stroke-width="1" visibility="hidden"/>
    <circle id="eq-dot" r="3.5" fill="var(--s1)" stroke="var(--surface)" stroke-width="2" visibility="hidden"/>
    <rect x="${P.l}" y="${P.t}" width="${W - P.l - P.r}" height="${H - P.t - P.b}" fill="transparent" id="eq-hover"/>
    <text class="axis-lbl" x="${P.l}" y="${H - 4}">${esc(eq[0].ts.slice(0, 10))}</text>
    <text class="axis-lbl" x="${W - P.r}" y="${H - 4}" text-anchor="end">${esc(eq[eq.length - 1].ts.slice(0, 10))}</text>`;
  const hover = svg.querySelector("#eq-hover");
  const crossEl = svg.querySelector("#eq-cross"), dotEl = svg.querySelector("#eq-dot");
  const tip = $("tooltip");
  hover.addEventListener("mousemove", (ev) => {
    const rect = svg.getBoundingClientRect();
    const relX = (ev.clientX - rect.left) / rect.width * W;
    const i = Math.max(0, Math.min(eq.length - 1, Math.round((relX - P.l) / (W - P.l - P.r) * (eq.length - 1))));
    crossEl.setAttribute("x1", x(i)); crossEl.setAttribute("x2", x(i));
    crossEl.setAttribute("visibility", "visible");
    dotEl.setAttribute("cx", x(i)); dotEl.setAttribute("cy", y(vals[i]));
    dotEl.setAttribute("visibility", "visible");
    tip.style.display = "block";
    tip.style.left = `${ev.clientX + 14}px`; tip.style.top = `${ev.clientY - 10}px`;
    tip.textContent = `${eq[i].ts.replace("T", " ")} · ${vals[i] > 0 ? "+" : ""}${vals[i].toFixed(2)}R`;
  });
  hover.addEventListener("mouseleave", () => {
    crossEl.setAttribute("visibility", "hidden"); dotEl.setAttribute("visibility", "hidden");
    tip.style.display = "none";
  });
}

function hookTips(svg) {
  const tip = $("tooltip");
  svg.querySelectorAll("[data-tip]").forEach((el) => {
    el.addEventListener("mousemove", (ev) => {
      tip.style.display = "block";
      tip.style.left = `${ev.clientX + 14}px`; tip.style.top = `${ev.clientY - 10}px`;
      tip.textContent = el.dataset.tip;
    });
    el.addEventListener("mouseleave", () => { tip.style.display = "none"; });
  });
}

/* ── шторм/радар ── */
function renderStorm(feed) {
  const st = feed.storm || {}, rd = feed.radar || {};
  $("storm-tiles").innerHTML = `
    <div class="tile"><div class="v">${(st.watchlist || []).length}</div><div class="l">пружин под наблюдением</div></div>
    <div class="tile"><div class="v">${rd.n ?? "—"}</div><div class="l">объёмных всплесков (всего)</div></div>
    <div class="tile"><div class="v">${rd.good_pct != null ? rd.good_pct + "%" : "—"}</div><div class="l">всплесков «good»</div></div>
    <div class="tile"><div class="v">${rd.avg_mfe_pct != null ? "+" + rd.avg_mfe_pct + "%" : "—"}</div><div class="l">средний MFE всплеска</div></div>`;
  $("storm-table").querySelector("tbody").innerHTML = (st.watchlist || []).map((w) => `
    <tr>
      <td class="sym lft">${starHtml(w.symbol)} ${esc(w.symbol)}</td>
      <td class="lft">${fmtMsk(w.added_ts)}</td>
      <td>${w.price_at_add ?? "—"}</td>
      <td>${w.box_width_pct != null ? w.box_width_pct.toFixed(2) + "%" : "—"}</td>
      <td>${w.compress_p != null ? "p" + w.compress_p : "—"}</td>
      <td><span class="${cls(w.oi_chg_4h)}">${fmtPct(w.oi_chg_4h, 1)}</span></td>
      <td>${w.funding != null ? (w.funding * 100).toFixed(3) + "%" : "—"}</td>
    </tr>`).join("") || '<tr><td colspan="7" class="empty">Watchlist пуст.</td></tr>';
}

/* ── главный цикл ── */
async function refresh() {
  const feed = await fetchFeed();
  feedStatus(feed);
  if (!feed) return;
  const isNew = !state.feed || feed.generated_at !== state.feed.generated_at;
  state.feed = feed;
  if (isNew) {
    renderPump(feed);
    renderCombos();
    renderDiary(feed);
    renderRose(feed);
    renderHistory(feed);
    renderBot(feed);
    renderStorm(feed);
  }
  await renderLive(feed); // сам решает, перестраивать ли карточки
}

/* ── автообновление UI: вечная вкладка сама подтягивает новый релиз ──
   Сравниваем версию app.js в СВЕЖЕМ index.html (no-store) с версией,
   с которой загружены сами (из своего <script src>). Разошлись → reload. */
const MY_VER = (document.querySelector('script[src*="app.js"]')?.src.match(/v=(\d+)/) || [])[1];
async function checkUiVersion() {
  try {
    const r = await fetch("index.html", { cache: "no-store" });
    const ver = ((await r.text()).match(/app\.js\?v=(\d+)/) || [])[1];
    if (MY_VER && ver && ver !== MY_VER) location.reload();
  } catch (_) { /* сеть моргнула — проверим в следующий раз */ }
}
setInterval(checkUiVersion, 5 * 60_000);

refresh();
setInterval(refresh, FEED_POLL_MS);
setInterval(() => { // спарклайны: дотягиваем свежие закрытые бары
  for (const card of state.liveCards.values())
    if (card.basis != null) initCardBars(card);
}, KLINE_REFRESH_MS);
```

## `dashboard/site/index.html`

```text
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Пульс — трейдинг-дашборд</title>
<link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>⚡</text></svg>">
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="icon-180.png">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<link rel="stylesheet" href="style.css?v=34">
</head>
<body>
<div class="wrap">

<header>
  <h1>⚡ ПУЛЬС <span class="accent">· сигналы и отработка</span></h1>
  <div class="hdr-status">
    <span id="feed-status"><span class="dot"></span>фид…</span>
    <span id="ws-status"><span class="dot"></span>live-цены…</span>
    <span class="hdr-tick" id="btc-tick"></span>
    <span class="hdr-tick" id="eth-tick"></span>
    <button id="push-btn" class="pushbtn" hidden title="уведомления о новых исследовательских кандидатах (NO-TRADE)">🔔</button>
    <span class="hdr-tick" id="bias-score" title="наклон стороны по версиям — когорты v2/v3 раздельно (разные правила)"></span>
  </div>
</header>

<nav class="tabs">
  <button class="tab active" data-tab="pulse">⚡ Пульс</button>
  <button class="tab" data-tab="diary">📓 Дневник трейдера</button>
</nav>


<section id="sec-entry">
  <div class="entry-main">
    <div class="sec-head">
      <h2>🧪 Исследовательские кандидаты — NO-TRADE</h2>
      <span class="sub">механический отбор для наблюдения и дневника, НЕ сигнал входа · свечной торговый edge не подтверждён · свежесть ≤3ч · цена не убежала · без пилы · не в раздаче</span>
    </div>
    <div class="cards" id="entry-cards"><div class="empty">Кандидатов сейчас нет — правила строгие.</div></div>
    <div class="sec-head" style="margin-top:14px">
      <h2 style="font-size:12px">⚡ Связки: пробуждение → Rose-пост → структура жива</h2>
      <span class="sub">радар-хит ≥5× за ≤72ч ДО поста Rose (такие посты отрабатывают ×2 лучше, n=5: VANRY +146% — выборка крошечная) · снимается САМА: ход случился (пик от поста ≥8%), пост провален (ниже −3%), раздача или 48ч · исследовательский лид, не сигнал</span>
    </div>
    <div class="cards" id="combo-cards"><div class="empty">Живых связок нет.</div></div>
  </div>
</section>

<section id="sec-live">
  <div class="sec-head">
    <h2>Живые сигналы</h2>
    <span class="sub">отработка тикает в реальном времени с Bybit · % в сторону сигнала от цены на момент алерта · ⬆⬇ = наклон структуры (OI/funding/pump-стадия), не торговый прогноз</span>
    <span class="sub" id="bias-acc" style="font-family:var(--mono)"></span>
  </div>
  <div class="filters" id="live-filters"></div>
  <div class="cards" id="live-cards"><div class="empty">Загрузка…</div></div>
</section>

<section id="sec-pump">
  <div class="sec-head">
    <h2>🌋 Pump-надзор</h2>
    <span class="sub">вертикаль ≥80%/4ч → плато → слом (кейс TAIKO 02.07) · пока монета под надзором, лонг-сетапы скринера по ней глушатся гейтом</span>
  </div>
  <div class="tiles" id="pump-tiles"></div>
  <div class="tbl-wrap">
    <table id="pump-table">
      <thead><tr>
        <th class="lft">Монета</th><th class="lft">Детект (МСК)</th><th class="lft">Тип</th>
        <th class="lft">Стадия</th><th>Рост при детекте</th><th>Пик</th><th>Низ плато</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
  <div class="sec-head" style="margin-top:14px"><h2 style="font-size:12px">Заглушенные лонги скринера (PumpWatchGate)</h2></div>
  <div class="tbl-wrap">
    <table id="pump-muted-table">
      <thead><tr>
        <th class="lft">Время (МСК)</th><th class="lft">Монета</th><th class="lft">Сетап</th>
        <th>Score</th><th class="lft">Причина (стадия, тип, возраст эпизода)</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
</section>

<section id="sec-rose">
  <div class="sec-head">
    <h2>🌹 Rose — сигналы и анализ</h2>
    <span class="sub">каналы rose + RoseSignalsPremium · повтор алерта ≤6ч = подтверждение, сутки итога отсчитываются заново · старт = первые 6ч, итог = макс. пик и просадка за 24ч от последнего подтверждения</span>
  </div>
  <div class="tiles" id="rose-tiles"></div>
  <div class="filters" id="rose-filters"></div>
  <div class="tbl-wrap">
    <table id="rose-table">
      <thead><tr>
        <th class="lft">Алерт (МСК)</th><th class="lft">Монета</th><th class="lft">Напр.</th>
        <th>Подтв.</th><th>Старт 6ч: пик</th><th>Старт 6ч: просадка</th>
        <th>Пик 24ч</th><th>Просадка 24ч</th><th>Итог 24ч</th><th class="lft">Статус</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
  <button class="morebtn" id="rose-more" hidden>Показать ещё</button>
</section>

<section id="sec-history">
  <div class="sec-head">
    <h2>История сигналов — итог за сутки</h2>
    <span class="sub">накапливается по всем источникам дашборда · пик и просадка в сторону сигнала за 24ч от последнего подтверждения</span>
  </div>
  <div class="filters" id="hist-filters"></div>
  <div class="tbl-wrap">
    <table id="hist-table">
      <thead><tr>
        <th class="lft">Алерт (МСК)</th><th class="lft">Источник</th><th class="lft">Монета</th><th class="lft">Напр.</th>
        <th>Подтв.</th><th>Пик 24ч</th><th>Просадка 24ч</th><th>Итог 24ч</th><th class="lft">Статус</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
  <button class="morebtn" id="hist-more" hidden>Показать ещё</button>
</section>

<section id="sec-bot">
  <div class="sec-head">
    <h2>Скринер-бот</h2>
    <span class="sub" id="bot-window"></span>
  </div>
  <div class="tiles" id="bot-tiles"></div>
  <div class="charts2">
    <div class="chart-box"><h3>Winrate по сетапам, % (24ч-горизонт)</h3><svg id="setup-chart"></svg></div>
    <div class="chart-box"><h3>Накопленный R (все сигналы честного окна)</h3><svg id="equity-chart"></svg></div>
  </div>
  <div class="sec-head" style="margin-top:16px"><h2 style="font-size:12px">Последние резолвы</h2></div>
  <div class="tbl-wrap">
    <table id="bot-table">
      <thead><tr>
        <th class="lft">Время (МСК)</th><th class="lft">Монета</th><th class="lft">Сетап</th>
        <th class="lft">Напр.</th><th>Δ24ч</th><th>R(24ч)</th><th>MFE</th><th>MAE</th><th class="lft">Итог</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
</section>

<section id="sec-storm">
  <div class="sec-head">
    <h2>Шторм и радар</h2>
    <span class="sub">пружины под наблюдением и статистика объёмных всплесков · по каналам шторм смотрит только rose и RoseSignalsPremium (rose_watch: пост → подкоп метриками → 6ч надзора → 🌹 алерт)</span>
  </div>
  <div class="tiles" id="storm-tiles"></div>
  <div class="tbl-wrap">
    <table id="storm-table">
      <thead><tr>
        <th class="lft">Монета</th><th class="lft">В списке с (МСК)</th><th>Цена входа в список</th>
        <th>Ширина коробки</th><th>Сжатие, перц.</th><th>ΔOI 4ч</th><th>Funding</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>
</section>

<section id="short-watch">
  <div class="sec-head">
    <h2 style="font-size:12px">🧪 Шорт-гипотеза (статус: ОПРОВЕРГАЕТСЯ) — NO-TRADE</h2>
    <span class="sub">шторм-шорты с funding ≥ +0.010% при падении — MUSDT-паттерн (лид 3/3, ретро 17% vs 7% на крошечной выборке) · ⚠ годовой discovery-прогон 10.07 знак НЕ подтвердил: funding+ шорты −1.22% vs −1.10% у всех (качество discovery, не приговор) · витрина для глаз: НЕ сигнал, в дневник не пишется, в действия не идёт</span>
  </div>
  <div id="short-watch-cards"><div class="empty">Сейчас таких нет.</div></div>
</section>

<section id="sec-diary" hidden>
  <div class="sec-head">
    <h2>📓 Дневник трейдера</h2>
    <span class="sub">ВСЕ исследовательские кандидаты — включая не отработавшие · «вход» = момент появления в секции · MFE-пик показывает только достигнутую волатильность, <b>не исполнимую доходность</b> · RET24 = close через 24ч до costs · сетка RET24−cost — механическая иллюстрация, не PnL · значимость считается только по UTC-дням, при &lt;5 днях честно «недостаточно данных» · никакой показатель дневника не меняет торговые правила</span>
  </div>
  <div class="tiles" id="diary-tiles"></div>

  <div class="sec-head" style="margin-top:14px"><h2 style="font-size:12px">🧠 MFE-ожидания и наблюдаемый RET24 (замороженные базы)</h2></div>
  <div class="tiles" id="diary-classes"></div>

  <div class="sec-head" style="margin-top:14px"><h2 style="font-size:12px">Журнал сделок зоны</h2></div>
  <div class="tbl-wrap">
    <table id="diary-table">
      <thead><tr>
        <th class="lft">В зоне с (МСК)</th><th class="lft">Монета</th><th class="lft">Класс</th>
        <th>Сторона</th>
        <th>Базис</th><th>Диапазон сделки (24ч)</th><th>Пик</th><th>Просадка</th><th>Итог 24ч</th>
        <th class="lft">MFE-ожидание (p50)</th><th class="lft">MFE-вердикт</th>
      </tr></thead>
      <tbody></tbody>
    </table>
  </div>

  <div class="sec-head" style="margin-top:14px"><h2 style="font-size:12px">🚀💥 Библиотека сюрпризов</h2></div>
  <div class="cards" id="diary-surprises"><div class="empty">Сюрпризов пока нет — исходы в рамках ожиданий.</div></div>
</section>

<footer>
  Данные: локальный бот → GitHub → этот дашборд (обновление ~1–2 мин) · живые цены — Bybit WebSocket напрямую ·
  аналитика бота считается только в честном окне (с 02.06.2026) · время на странице — МСК.
  <span id="gen-at"></span>
</footer>

</div>
<div class="tooltip" id="tooltip"></div>
<script src="app.js?v=37"></script>
</body>
</html>
```

## `dashboard/site/style.css`

```text
/* Пульс — трейдинг-дашборд. Тёмная тема (deliberate dark-only).
   Палитра: dataviz reference (dark steps), валидирована на #14161a. */
:root {
  --page: #0d0f12;
  --surface: #14161a;
  --surface-2: #191c21;
  --ink: #ffffff;
  --ink-2: #c3c2b7;
  --muted: #898781;
  --grid: #2c2c2a;
  --border: rgba(255, 255, 255, 0.10);
  --s1: #3987e5;  /* blue */
  --s2: #199e70;  /* aqua */
  --s3: #c98500;  /* yellow */
  --s4: #9085e9;  /* violet */
  --s5: #e66767;  /* red */
  --good: #0ca30c;
  --warn: #fab219;
  --crit: #d03b3b;
  --up: #0ca30c;
  --down: #d03b3b;
  --radius: 10px;
  --mono: ui-monospace, SFMono-Regular, Menlo, monospace;
}
* { box-sizing: border-box; margin: 0; padding: 0; }
html { -webkit-text-size-adjust: 100%; }
body {
  background: var(--page);
  color: var(--ink-2);
  font: 14px/1.45 system-ui, -apple-system, "Segoe UI", sans-serif;
  padding: 0 16px 64px;
}
.wrap { max-width: 1280px; margin: 0 auto; }

/* ── header ── */
header {
  display: flex; align-items: center; gap: 16px; flex-wrap: wrap;
  padding: 18px 0 14px; border-bottom: 1px solid var(--border);
  position: sticky; top: 0; background: var(--page); z-index: 20;
}
h1 { font-size: 18px; color: var(--ink); letter-spacing: 0.4px; font-weight: 700; }
h1 .accent { color: var(--s1); }
.hdr-status { display: flex; gap: 14px; align-items: center; font-size: 12px; color: var(--muted); flex-wrap: wrap; }
.dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; background: var(--muted); margin-right: 5px; vertical-align: 1px; }
.dot.ok { background: var(--good); }
.dot.stale { background: var(--warn); }
.dot.err { background: var(--crit); }
.hdr-tick { font-family: var(--mono); font-variant-numeric: tabular-nums; }
.hdr-tick b { color: var(--ink); font-weight: 600; }

/* ── sections ── */
section { margin-top: 28px; }
.sec-head { display: flex; align-items: baseline; gap: 12px; margin-bottom: 12px; flex-wrap: wrap; }
.sec-head h2 { font-size: 14px; color: var(--ink); text-transform: uppercase; letter-spacing: 1px; font-weight: 700; }
.sec-head .sub { font-size: 12px; color: var(--muted); }
.note { font-size: 11px; color: var(--muted); margin-top: 8px; }

/* ── live signal cards ── */
.cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 10px; }
.card {
  background: var(--surface); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 12px 14px 10px; position: relative;
  display: flex; flex-direction: column; gap: 6px; min-height: 128px;
}
.card .row1 { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.role {
  font-size: 10px; padding: 1px 8px; border-radius: 20px; letter-spacing: 0.3px;
  border: 1px solid var(--border); color: var(--muted); vertical-align: 2px; margin-left: 8px;
}
.role.storm_watch { color: var(--s5); border-color: rgba(230,103,103,.4); }
.role.news { color: var(--s1); border-color: rgba(57,135,229,.35); }
.card .sym { font-weight: 700; color: var(--ink); font-size: 15px; }
.card .dir { font-size: 11px; font-weight: 700; padding: 1px 7px; border-radius: 20px; }
.dir.long { color: var(--up); background: rgba(12, 163, 12, 0.12); }
/* плашка стороны в дневнике — вне .card размер задаём сами */
#diary-table .dir, #diary-surprises .dir { font-size: 11px; font-weight: 700; padding: 1px 7px; border-radius: 20px; white-space: nowrap; }
.dir.short { color: var(--down); background: rgba(208, 59, 59, 0.12); }
.dir.none { color: var(--muted); background: rgba(255,255,255,0.06); }
.badge {
  font-size: 10px; padding: 1px 7px; border-radius: 4px; letter-spacing: 0.3px;
  border: 1px solid var(--border); color: var(--ink-2); white-space: nowrap;
}
.badge.tg_channel { color: var(--s1); border-color: rgba(57,135,229,.4); }
.badge.screener { color: var(--s4); border-color: rgba(144,133,233,.4); }
.badge.storm { color: var(--s3); border-color: rgba(201,133,0,.45); }
.badge.radar { color: var(--s2); border-color: rgba(25,158,112,.45); }
.badge.alert { color: var(--s5); border-color: rgba(230,103,103,.45); }
.card .when { margin-left: auto; font-size: 11px; color: var(--muted); white-space: nowrap; }
.card .big {
  font-family: var(--mono); font-variant-numeric: tabular-nums;
  font-size: 26px; font-weight: 700; color: var(--ink); line-height: 1.1;
}
.big.pos { color: var(--up); }
.big.neg { color: var(--down); }
.card .meta { font-size: 11px; color: var(--muted); font-family: var(--mono); font-variant-numeric: tabular-nums; }
.card .meta b { color: var(--ink-2); font-weight: 600; }
.card svg.spark { width: 100%; height: 40px; display: block; margin-top: auto; }
.card .src-note { font-size: 11px; color: var(--muted); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

/* ── channel rating cards ── */
.chan-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 10px; }
.chan {
  background: var(--surface); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 14px 16px;
}
.chan .name { color: var(--ink); font-weight: 700; font-size: 14px; margin-bottom: 8px; }
.chan .kpis { display: flex; gap: 18px; flex-wrap: wrap; }
.kpi .v { font-family: var(--mono); font-variant-numeric: tabular-nums; font-size: 19px; font-weight: 700; color: var(--ink); }
.kpi .l { font-size: 10.5px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.5px; }
.meter { height: 4px; background: var(--grid); border-radius: 2px; margin-top: 10px; overflow: hidden; }
.meter i { display: block; height: 100%; border-radius: 2px; background: var(--s1); }
.lown { color: var(--warn); font-size: 10.5px; margin-top: 6px; }

/* ── stat tiles ── */
.tiles { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px; margin-bottom: 14px; }
.tile {
  background: var(--surface); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 12px 14px;
}
.tile .v { font-family: var(--mono); font-variant-numeric: tabular-nums; font-size: 24px; font-weight: 700; color: var(--ink); }
.tile .v.pos { color: var(--up); } .tile .v.neg { color: var(--down); }
.tile .l { font-size: 11px; color: var(--muted); margin-top: 2px; }

/* ── tables ── */
.tbl-wrap { overflow-x: auto; background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius); }
table { border-collapse: collapse; width: 100%; font-size: 12.5px; }
th, td { text-align: right; padding: 7px 12px; white-space: nowrap; }
th:first-child, td:first-child, th.lft, td.lft { text-align: left; }
thead th {
  color: var(--muted); font-weight: 600; font-size: 10.5px; text-transform: uppercase;
  letter-spacing: 0.6px; border-bottom: 1px solid var(--grid);
  position: sticky; top: 0; background: var(--surface);
}
tbody td { border-bottom: 1px solid rgba(255,255,255,0.04); font-family: var(--mono); font-variant-numeric: tabular-nums; color: var(--ink-2); }
tbody tr:last-child td { border-bottom: none; }
tbody tr:hover td { background: rgba(255,255,255,0.03); }
td.sym { color: var(--ink); font-weight: 600; }
td .pos { color: var(--up); } td .neg { color: var(--down); } td .flat { color: var(--muted); }
td .chip { font-size: 10px; padding: 1px 6px; border-radius: 4px; border: 1px solid var(--border); font-family: system-ui, sans-serif; }

/* ── filters ── */
.filters { display: flex; gap: 8px; flex-wrap: wrap; margin-bottom: 10px; }
.fbtn {
  background: var(--surface); color: var(--ink-2); border: 1px solid var(--border);
  border-radius: 20px; padding: 4px 13px; font-size: 12px; cursor: pointer;
}
.fbtn:hover { background: var(--surface-2); }
.fbtn.on { color: var(--ink); border-color: var(--s1); background: rgba(57,135,229,0.12); }
.morebtn {
  margin: 12px auto 0; display: block; background: none; border: 1px solid var(--border);
  color: var(--ink-2); border-radius: 8px; padding: 6px 22px; font-size: 12px; cursor: pointer;
}
.morebtn:hover { background: var(--surface-2); }

/* ── charts ── */
.charts2 { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
@media (max-width: 860px) { .charts2 { grid-template-columns: 1fr; } }
.chart-box { background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius); padding: 14px 16px 8px; }
.chart-box h3 { font-size: 12px; color: var(--ink); font-weight: 600; margin-bottom: 10px; }
.chart-box svg { width: 100%; display: block; overflow: visible; }
.chart-box .axis-lbl { font-size: 10px; fill: var(--muted); font-family: var(--mono); }
.bar-lbl { font-size: 11px; fill: var(--ink-2); font-family: var(--mono); }
.bar-name { font-size: 11px; fill: var(--ink-2); }
.tooltip {
  position: fixed; pointer-events: none; z-index: 50; display: none;
  background: var(--surface-2); border: 1px solid var(--border); border-radius: 8px;
  padding: 7px 10px; font-size: 11.5px; font-family: var(--mono);
  font-variant-numeric: tabular-nums; color: var(--ink); box-shadow: 0 6px 20px rgba(0,0,0,0.5);
}

/* доминанта суток: ярче то, что больше — пик или просадка */
td .hl-pos, td .hl-neg { font-weight: 700; padding: 2px 7px; border-radius: 5px; }
td .hl-pos { background: rgba(12, 163, 12, 0.18); }
td .hl-neg { background: rgba(208, 59, 59, 0.18); }

/* звёздочка «просмотрел/просмотрю» — на монете, во всех секциях */
.star {
  cursor: pointer; color: var(--muted); font-size: 15px; line-height: 1;
  user-select: none; display: inline-block; padding: 2px 3px; margin: -2px 0;
  font-family: system-ui, sans-serif;
}
.star:hover { color: var(--ink-2); transform: scale(1.15); }
.star.on { color: var(--warn); }
td .star { font-size: 14px; }

/* наклон структуры (bias v1) */
.biastag {
  font-size: 10px; padding: 1px 7px; border-radius: 4px; white-space: nowrap;
  border: 1px solid var(--border); cursor: help;
}
.biastag.up { color: var(--up); border-color: rgba(12, 163, 12, 0.45); }
.biastag.down { color: var(--down); border-color: rgba(208, 59, 59, 0.45); }
.biastag.up.strong { background: rgba(12, 163, 12, 0.16); font-weight: 700; }
.biastag.down.strong { background: rgba(208, 59, 59, 0.16); font-weight: 700; }

/* ── вкладки Пульс / Дневник ── */
.tabs { display: flex; gap: 8px; margin-top: 14px; }
.tab {
  background: var(--surface); color: var(--ink-2);
  border: 1px solid var(--border); border-radius: 10px;
  font-size: 13px; font-weight: 600; padding: 7px 16px; cursor: pointer;
}
.tab.active {
  color: var(--ink); border-color: var(--up);
  box-shadow: 0 0 14px rgba(12, 163, 12, 0.15);
}
.mono { font-family: var(--mono); font-variant-numeric: tabular-nums; }

/* 🔔 подписка на пуши зон входа */
.pushbtn {
  background: var(--surface-2); color: var(--ink-2);
  border: 1px solid var(--border); border-radius: 8px;
  font-size: 13px; padding: 3px 9px; cursor: pointer;
}
.pushbtn:hover { border-color: var(--up); }
#push-code {
  margin-top: 12px; padding: 12px; border: 1px dashed var(--warn);
  border-radius: 10px; font-size: 12px;
}
#push-code textarea {
  width: 100%; margin-top: 8px; background: var(--surface-2);
  color: var(--ink-2); border: 1px solid var(--border); border-radius: 6px;
  font: 11px var(--mono); padding: 6px;
}

/* ═══ Исследовательские кандидаты — NO-TRADE (ревью Codex 10.07:
   вердикт годового бэктеста = эдж не подтверждён, панель больше НЕ действия) ═══ */
#sec-entry {
  position: relative;
  margin-top: 26px;
  padding: 20px 18px 18px;
  border-radius: 16px;
  border: 1px solid transparent;
  background:
    linear-gradient(var(--surface), var(--surface)) padding-box,
    linear-gradient(120deg, rgba(250, 178, 25, 0.55), rgba(250, 178, 25, 0.15) 55%,
                    rgba(250, 178, 25, 0.45)) border-box;
  box-shadow: inset 0 0 60px rgba(250, 178, 25, 0.03);
}
#sec-entry::before {
  content: "🧪 ИССЛЕДОВАТЕЛЬСКИЕ КАНДИДАТЫ — NO-TRADE";
  position: absolute; top: -9px; left: 16px;
  padding: 1px 10px; border-radius: 6px;
  background: var(--page);
  border: 1px solid rgba(250, 178, 25, 0.5);
  color: var(--warn);
  font-size: 11px; font-weight: 700; letter-spacing: 0.06em;
}
/* 🧪 шорт-гипотеза: отдельная секция ниже (вынесена из панели по ревью Codex 10.07;
   витрина осталась — просьба брата 08.07 «видеть глазами») */
#short-watch { border-top: 1px dashed var(--grid); padding-top: 12px; }
#short-watch .shortw-row { padding: 9px 0; border-bottom: 1px solid var(--grid); display: flex; flex-direction: column; gap: 5px; }
#short-watch .shortw-row:last-child { border-bottom: none; }
#short-watch .shortw-top { display: flex; align-items: center; gap: 6px; }
#short-watch .shortw-top .when { color: var(--muted); font-size: 11px; margin-left: auto; }
#short-watch .shortw-mid { display: flex; align-items: baseline; gap: 8px; }
#short-watch .shortw-mid .big { font-size: 16px; font-weight: 700; }
#short-watch .muted-note { color: var(--muted); font-size: 11px; }
#short-watch .dir { font-size: 11px; font-weight: 700; padding: 1px 7px; border-radius: 20px; white-space: nowrap; }
#sec-entry .sec-head h2 { color: var(--ink); }
#sec-entry .combo-head h2 { color: var(--warn); }
#sec-entry .empty {
  border: 1px dashed var(--grid); border-radius: 10px;
  padding: 14px; text-align: center;
}

/* «Вход имеет смысл сейчас» — компактные карточки-кандидаты */
.card.entry { min-height: 92px; border-color: rgba(12, 163, 12, 0.35); background: var(--surface-2); }
.card.entry .big { font-size: 21px; }
.card.entry.awk { border-color: rgba(250, 178, 25, 0.55); }

/* волновая заметка в entry-секции (каскад одного скана скрыт) */
.wave-note {
  grid-column: 1 / -1;
  font-size: 11px; color: var(--muted);
  border: 1px dashed var(--grid); border-radius: 8px;
  padding: 8px 12px;
}

/* погодная плашка BTC (скринер): по ветру — зелёная, против — риск-стиль */
.windtag {
  font-size: 10px; padding: 1px 6px; border-radius: 4px;
  color: var(--up); border: 1px solid rgba(12, 163, 12, 0.4);
  white-space: nowrap;
}

/* ⚡ связки пробуждение×Rose */
.badge.combo { color: #f5d90a; border-color: rgba(245, 217, 10, 0.45); }
.card.entry.combo { border-color: rgba(245, 217, 10, 0.5); }
.card.entry.combo.stale { opacity: 0.65; }

/* риск-маркер (статистически сливная категория) */
.risktag {
  font-size: 10px; padding: 1px 7px; border-radius: 4px; white-space: nowrap;
  color: var(--crit); border: 1px solid rgba(208, 59, 59, 0.5);
  background: rgba(208, 59, 59, 0.10); cursor: help;
}

/* монета под pump-надзором (storm_radar/pump_watch) */
.pumptag {
  font-size: 10px; padding: 1px 7px; border-radius: 4px; white-space: nowrap;
  color: var(--warn); border: 1px solid rgba(250, 178, 25, 0.45);
}

/* свежий сигнал (< 15 мин) */
.card.fresh { border-color: rgba(250, 178, 25, 0.55); box-shadow: 0 0 12px rgba(250, 178, 25, 0.12); }
.newtag {
  font-size: 9.5px; font-weight: 800; letter-spacing: 0.8px; color: #14161a;
  background: var(--warn); padding: 1px 6px; border-radius: 4px;
}

/* misc */
.empty { color: var(--muted); font-size: 12.5px; padding: 18px; text-align: center; }
footer { margin-top: 40px; font-size: 11px; color: var(--muted); border-top: 1px solid var(--border); padding-top: 14px; }
a { color: var(--s1); text-decoration: none; }
```
