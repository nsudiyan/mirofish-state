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
