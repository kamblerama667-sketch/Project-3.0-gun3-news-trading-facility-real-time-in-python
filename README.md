# Project-3.0-gun3-news-trading-facility-real-time-in-python








# ╔═══════════════════════════════════════════════════════════════════════════╗
# ║   PROJECT 3.0 — MASTER CORE HD MONITOR  v6.0  [THE ORACLE EDITION]       ║
# ║   Multi-Source Sentiment · Cross-Verification · Leverage · Multi-Asset    ║
# ║   Built on v5.0 — all existing layers preserved exactly                   ║
# ╚═══════════════════════════════════════════════════════════════════════════╝

import hashlib, hmac, os, time, math, json, random, struct, threading, re
from collections import deque
from dataclasses import dataclass, field
from typing import List, Optional, Dict
from IPython.display import HTML, Javascript, display

try:
    import requests as _req
    _HAS_REQUESTS = True
except ImportError:
    _HAS_REQUESTS = False

try:
    import feedparser as _fp
    _HAS_FEEDPARSER = True
except ImportError:
    _HAS_FEEDPARSER = False
    print("[P3 v6.0] feedparser not found. Install: pip install feedparser")

try:
    from google.colab import output as _colab_output
    _IN_COLAB = True
except ImportError:
    _IN_COLAB = False

# ════════════════════════════════════════════════════════════════════════════
# LAYER 1 — QUANTUM-PROOF HASHING  [PLACEHOLDER — unchanged from v5.0]
# ════════════════════════════════════════════════════════════════════════════
class LatticeHashPlaceholder:
    _MOD = (1 << 256) - 2**32 + 2**9 + 2**8 - 2**7 + 2**6 - 2**4 + 1

    def __init__(self, seed: bytes = None):
        self._seed = seed or os.urandom(32)

    def digest(self, data: bytes) -> bytes:
        tag  = b"P3\x00LATTICEv4\x00" + struct.pack(">Q", len(data))
        blob = tag + data
        k    = hmac.new(self._seed, blob[:64], hashlib.sha3_256).digest()
        r1   = hmac.new(k, blob, hashlib.sha3_512).digest()
        r2   = hmac.new(r1[:32], r1[32:] + blob[-32:], hashlib.sha3_256).digest()
        return (int.from_bytes(r2, "big") % self._MOD).to_bytes(32, "big")

    def verify(self, data: bytes, expected: bytes) -> bool:
        return hmac.compare_digest(self.digest(data), expected)


# ════════════════════════════════════════════════════════════════════════════
# LAYER 2 — RL CONSENSUS AGENT  [unchanged from v5.0]
# ════════════════════════════════════════════════════════════════════════════
@dataclass
class NetworkTelemetry:
    latency_ms:         float
    throughput_bps:     float
    validation_time_ms: float
    mempool_depth:      int
    peer_count:         int


class RLConsensusAgent:
    _DA = [-1, 0, +1]
    _TA = [-50, 0, +50]

    def __init__(self, alpha=0.15, gamma=0.92, epsilon=0.25):
        self.alpha = alpha; self.gamma = gamma; self.epsilon = epsilon
        self.epsilon_decay = 0.995; self.epsilon_min = 0.02
        self.Q: dict = {}
        self.difficulty = 3; self.timeout_ms = 500
        self._hist = deque(maxlen=128)
        self._ls = self._la = None
        self.episodes = 0; self.last_reward = 0.0

    def _state(self, t):
        return (min(int(t.latency_ms / 100), 9),
                min(int(t.throughput_bps / 10), 9),
                min(int(t.mempool_depth / 500), 4))

    def _Q(self, s):
        if s not in self.Q: self.Q[s] = [0.0] * 9
        return self.Q[s]

    def _reward(self, t):
        if len(self._hist) < 2: return 0.0
        p = self._hist[-2]
        return ((t.throughput_bps - p.throughput_bps) / (p.throughput_bps + 1e-6)
                + max(0, (p.mempool_depth - t.mempool_depth) / 1000)
                - max(0, (t.latency_ms - 200) / 1000)
                - max(0, (self.difficulty - 8) * 0.5))

    def step(self, t: NetworkTelemetry) -> dict:
        self._hist.append(t)
        s = self._state(t); q = self._Q(s)
        if self._ls is not None:
            r = self._reward(t); self.last_reward = round(r, 4)
            pq = self._Q(self._ls)
            pq[self._la] += self.alpha * (r + self.gamma * max(q) - pq[self._la])
        ai = random.randint(0, 8) if random.random() < self.epsilon else q.index(max(q))
        self.epsilon = max(self.epsilon_min, self.epsilon * self.epsilon_decay)
        self.difficulty = max(1, min(12, self.difficulty + self._DA[ai // 3]))
        self.timeout_ms = max(100, min(2000, self.timeout_ms + self._TA[ai % 3]))
        self._ls, self._la = s, ai; self.episodes += 1
        return {"difficulty": self.difficulty, "timeout_ms": self.timeout_ms,
                "epsilon": round(self.epsilon, 4), "reward": self.last_reward}

    def pretrain(self, n=300):
        for i in range(n):
            p = i / n
            self.step(NetworkTelemetry(
                50 + 300 * math.sin(p * math.pi) + random.gauss(0, 20),
                40 + 30 * math.cos(p * math.pi * 2) + random.gauss(0, 5),
                80 + 120 * random.random(),
                int(2000 * p * random.random()), random.randint(50, 500)))


# ════════════════════════════════════════════════════════════════════════════
# LAYER 3 — HEDGE ENGINE + VOLATILITY ORACLE  [unchanged from v5.0]
# ════════════════════════════════════════════════════════════════════════════
@dataclass
class Stablecoin:
    symbol: str; address: str; risk: int; apy_bp: int; liquidity: float
    active: bool = False
    @property
    def apy(self): return self.apy_bp / 100


class VolatilityOracle:
    _CG_URL  = ("https://api.coingecko.com/api/v3/simple/price"
                "?ids=bitcoin,ethereum,solana&vs_currencies=usd&include_24hr_change=true")
    _BN_BTCURL = "https://api.binance.com/api/v3/ticker/24hr?symbol=BTCUSDT"
    _BN_ETHURL = "https://api.binance.com/api/v3/ticker/24hr?symbol=ETHUSDT"

    def __init__(self, window=20):
        self._buf   = deque(maxlen=window)
        self._lock  = threading.Lock()
        self._prices = {"BTC": 0.0, "ETH": 0.0, "SOL": 0.0}
        self._24h    = {"BTC": 0.0, "ETH": 0.0, "SOL": 0.0}
        self._source = "SIMULATED"

    def _fetch_live(self) -> Optional[float]:
        if not _HAS_REQUESTS: return None
        try:
            r = _req.get(self._CG_URL, timeout=5)
            if r.status_code == 200:
                d = r.json()
                self._prices["BTC"] = d.get("bitcoin", {}).get("usd", 0.0)
                self._prices["ETH"] = d.get("ethereum", {}).get("usd", 0.0)
                self._prices["SOL"] = d.get("solana", {}).get("usd", 0.0)
                self._24h["BTC"]    = d.get("bitcoin", {}).get("usd_24h_change", 0.0)
                self._24h["ETH"]    = d.get("ethereum", {}).get("usd_24h_change", 0.0)
                self._24h["SOL"]    = d.get("solana", {}).get("usd_24h_change", 0.0)
                self._source = "COINGECKO"
                return abs(self._24h["BTC"]) * 3.0
        except Exception: pass
        try:
            r = _req.get(self._BN_BTCURL, timeout=4)
            if r.status_code == 200:
                d = r.json()
                self._prices["BTC"] = float(d["lastPrice"])
                self._24h["BTC"]    = float(d["priceChangePercent"])
                self._source = "BINANCE"
                return abs(self._24h["BTC"]) * 5.0
        except Exception: pass
        return None

    def fetch(self) -> float:
        live = self._fetch_live()
        if live is not None: return max(0.0, live)
        self._source = "SIMULATED"
        # Simulated prices when offline
        if self._prices["BTC"] == 0.0:
            self._prices["BTC"] = 65000.0 + random.gauss(0, 400)
            self._prices["ETH"] = 3200.0  + random.gauss(0, 60)
            self._prices["SOL"] = 145.0   + random.gauss(0, 3)
        else:
            for k in self._prices:
                self._prices[k] *= (1 + random.gauss(0, 0.001))
        return max(0.0, 45.0 + random.gauss(0, 10)
                   + random.choice([0, 0, 0, 0, 42, 0, -25]))

    def tick(self) -> float:
        v = self.fetch()
        with self._lock: self._buf.append(v)
        return v

    def now(self):
        with self._lock: return self._buf[-1] if self._buf else 0.0
    def mean(self):
        with self._lock:
            return sum(self._buf) / len(self._buf) if self._buf else 0.0
    def delta(self):
        m = self.mean(); return (self.now() - m) / m * 100 if m else 0.0
    def price(self, sym="BTC"): return self._prices.get(sym, 0.0)
    def change_24h(self, sym="BTC"): return self._24h.get(sym, 0.0)
    def source(self): return self._source


class HedgeEngine:
    DROP = -15.0; RISE = +8.0; COOL = 60

    def __init__(self, coins: List[Stablecoin]):
        self.coins  = sorted(coins, key=lambda c: c.risk)
        self.oracle = VolatilityOracle()
        self.state  = "NEUTRAL"; self.active: Stablecoin = None
        self._last  = 0.0; self.log: List[dict] = []
        self._lock  = threading.Lock()
        self._switch(self.coins[0], "INIT")

    def _switch(self, coin, reason):
        if self.active: self.active.active = False
        coin.active = True; self.active = coin; self._last = time.time()
        self.log.append({"ts": time.strftime("%H:%M:%S"), "reason": reason,
                         "contract": coin.symbol, "risk": coin.risk,
                         "apy": coin.apy, "vol": round(self.oracle.now(), 1)})

    def _cool(self): return (time.time() - self._last) < self.COOL

    def tick(self) -> dict:
        idx = self.oracle.tick(); delta = self.oracle.delta()
        with self._lock:
            if not self._cool():
                if delta <= self.DROP and self.state != "DEFENSIVE":
                    self._switch(self.coins[0], "VOL_DROP(" + str(round(delta,1)) + "%)")
                    self.state = "DEFENSIVE"
                elif delta >= self.RISE and self.state == "DEFENSIVE":
                    best = max((c for c in self.coins if c.risk <= 2), key=lambda c: c.apy_bp)
                    self._switch(best, "RECOVERY(" + str(round(delta,1)) + "%)")
                    self.state = "YIELD"
                elif abs(delta) < 5 and self.state != "NEUTRAL":
                    self._switch(self.coins[0], "STABLE"); self.state = "NEUTRAL"
        return {"vol": round(idx,1), "mean": round(self.oracle.mean(),1),
                "delta": round(delta,1), "state": self.state,
                "asset": self.active.symbol, "apy": self.active.apy,
                "prices":  {k: round(self.oracle.price(k), 2) for k in ["BTC","ETH","SOL"]},
                "changes": {k: round(self.oracle.change_24h(k), 4) for k in ["BTC","ETH","SOL"]},
                "data_source": self.oracle.source()}


# ════════════════════════════════════════════════════════════════════════════
# LAYER 4 — SHARD CONTROLLER  [unchanged from v5.0]
# ════════════════════════════════════════════════════════════════════════════
@dataclass
class BlockMeta:
    index: int; shard: int; pqc_hash: bytes; prev: bytes
    ts: float; txs: int; sender: str; amount: float


class ShardController:
    CAP = 1_000_000; HOT = 500
    _SENDERS = ["TIGER_MASTER","QUANTUM_NODE","LATTICE_01",
                "ALPHA_RELAY","NODE_ZERO","DELTA_CORE","SIGMA_7"]

    def __init__(self):
        self._cache = {}; self._info = {}
        self._lock  = threading.Lock(); self.total = 0

    def shard(self, idx): return idx // self.CAP

    def write(self, m: BlockMeta) -> int:
        with self._lock:
            sid = self.shard(m.index)
            if sid not in self._cache:
                self._cache[sid] = deque(maxlen=self.HOT)
                self._info[sid]  = {"start": sid*self.CAP,
                                    "end": (sid+1)*self.CAP-1, "count": 0}
            self._cache[sid].append(m); self._info[sid]["count"] += 1
            self.total = m.index + 1
        return sid

    def recent(self, n=2):
        with self._lock:
            if not self._cache: return []
            s = self._cache[max(self._cache)]
            return list(s)[-n:]

    def manifest(self):
        with self._lock:
            return [{"id": k, **v} for k, v in self._info.items()]

    def active_count(self): return len(self._info)


# ════════════════════════════════════════════════════════════════════════════
# LAYER 9A — SENTIMENT ENGINE
# Keyword-based NLP scorer (no external ML deps needed).
# Weights: Official News 60%, Social Media 40%
# ════════════════════════════════════════════════════════════════════════════

# Keyword dictionaries — scored -2 to +2
_BULL_STRONG = ["surge","rally","breakout","ath","all-time high","bullish",
                "soar","moon","adoption","etf approved","institutional buy",
                "record high","parabolic","explode","pump","accumulate"]
_BULL_MILD   = ["rise","gain","recover","positive","growth","buy","green",
                "rebound","support","upgrade","inflow","uptrend","beat"]
_BEAR_STRONG = ["crash","collapse","war","ban","hack","rug","scam","plunge",
                "capitulate","liquidate","bubble","fraud","seized","arrest",
                "market crash","black swan","recession","default","crisis"]
_BEAR_MILD   = ["fall","drop","concern","bearish","sell","caution","decline",
                "outflow","downtrend","miss","weak","uncertainty","fear"]

def _score_text(text: str) -> float:
    """Return sentiment score: -1.0 (very bearish) to +1.0 (very bullish)."""
    t   = text.lower()
    raw = 0.0
    for w in _BULL_STRONG: raw += t.count(w) * 2.0
    for w in _BULL_MILD:   raw += t.count(w) * 1.0
    for w in _BEAR_STRONG: raw -= t.count(w) * 2.0
    for w in _BEAR_MILD:   raw -= t.count(w) * 1.0
    # Normalise to [-1, 1]
    return max(-1.0, min(1.0, raw / 10.0))

def _detect_global_event(text: str) -> Optional[str]:
    """Return event label if a critical macro event is detected."""
    t = text.lower()
    events = {
        "WAR/CONFLICT":    ["war","military","invasion","missile","airstrike","troops"],
        "MARKET CRASH":    ["market crash","black monday","circuit breaker",
                            "flash crash","panic sell","mass liquidation"],
        "REGULATORY BAN":  ["ban crypto","crypto ban","sec charges","cftc","crackdown",
                            "illegal","regulate","shutdown exchange"],
        "HACK/EXPLOIT":    ["hacked","exploit","stolen","bridge attack","rug pull",
                            "vulnerability","smart contract bug"],
        "MACRO SHOCK":     ["fed rate","interest rate hike","inflation surge",
                            "gdp contraction","bank failure","default"],
    }
    for label, keywords in events.items():
        if any(k in t for k in keywords):
            return label
    return None


@dataclass
class SentimentSignal:
    source:    str          # "NEWS" | "REDDIT" | "SIMULATION"
    score:     float        # -1.0 to +1.0
    headline:  str          # sample headline driving score
    event:     Optional[str]  # global event label if detected
    ts:        str


class MultiSourceSentimentAggregator:
    """
    Layer 9A — fetches sentiment from two independent streams:
      Stream A (weight 60%): Google News RSS via feedparser
      Stream B (weight 40%): Reddit r/CryptoCurrency, r/investing, r/Bitcoin
    Cross-verifies global events across streams before flagging them.
    """
    WEIGHT_NEWS   = 0.60
    WEIGHT_SOCIAL = 0.40

    # RSS feeds
    _GNEWS_FEEDS = [
        "https://news.google.com/rss/search?q=bitcoin+crypto&hl=en-US&gl=US&ceid=US:en",
        "https://news.google.com/rss/search?q=stock+market+economy&hl=en-US&gl=US&ceid=US:en",
    ]
    # Reddit JSON endpoints (no auth needed for public listing)
    _REDDIT_URLS = [
        "https://www.reddit.com/r/CryptoCurrency/hot.json?limit=15",
        "https://www.reddit.com/r/investing/hot.json?limit=10",
        "https://www.reddit.com/r/Bitcoin/hot.json?limit=10",
    ]
    _REDDIT_HDR = {"User-Agent": "Mozilla/5.0 P3Oracle/6.0"}

    def __init__(self, cache_seconds: int = 120):
        self._cache_sec  = cache_seconds
        self._last_fetch = 0.0
        self._lock       = threading.Lock()
        self._cached: Dict = {}

    # ── Stream A: Google News RSS ──────────────────
    def _fetch_news(self) -> SentimentSignal:
        if not _HAS_FEEDPARSER or not _HAS_REQUESTS:
            return self._simulate("NEWS")
        texts, top_headline = [], "No headline"
        try:
            for url in self._GNEWS_FEEDS:
                feed = _fp.parse(url)
                for entry in feed.entries[:8]:
                    title = getattr(entry, "title", "")
                    summ  = getattr(entry, "summary", "")
                    texts.append(title + " " + summ)
                    if top_headline == "No headline" and title:
                        top_headline = title[:80]
        except Exception as e:
            return self._simulate("NEWS")
        blob  = " ".join(texts)
        score = _score_text(blob)
        event = _detect_global_event(blob)
        return SentimentSignal("NEWS", round(score, 4),
                               top_headline, event, time.strftime("%H:%M:%S"))

    # ── Stream B: Reddit ──────────────────────────
    def _fetch_reddit(self) -> SentimentSignal:
        if not _HAS_REQUESTS:
            return self._simulate("REDDIT")
        texts, top_headline = [], "No post"
        try:
            for url in self._REDDIT_URLS:
                r = _req.get(url, headers=self._REDDIT_HDR, timeout=5)
                if r.status_code != 200: continue
                posts = r.json().get("data", {}).get("children", [])
                for p in posts[:6]:
                    d = p.get("data", {})
                    title = d.get("title", "")
                    body  = d.get("selftext", "")[:200]
                    texts.append(title + " " + body)
                    if top_headline == "No post" and title:
                        top_headline = title[:80]
        except Exception:
            return self._simulate("REDDIT")
        blob  = " ".join(texts)
        score = _score_text(blob)
        event = _detect_global_event(blob)
        return SentimentSignal("REDDIT", round(score, 4),
                               top_headline, event, time.strftime("%H:%M:%S"))

    # ── Simulation fallback ───────────────────────
    def _simulate(self, source: str) -> SentimentSignal:
        score = round(random.gauss(0.05, 0.35), 4)
        score = max(-1.0, min(1.0, score))
        headlines = {
            "NEWS":   ["BTC consolidates near key support level",
                       "Fed signals cautious approach to rate cuts",
                       "Crypto ETF inflows hit weekly record",
                       "Regulators review digital asset framework"],
            "REDDIT": ["This dip is a gift - accumulate now",
                       "Bear trap confirmed, breakout incoming",
                       "Everyone panic selling, I'm buying",
                       "Macro looks rough, staying in stables"],
        }
        hl = random.choice(headlines.get(source, ["Market activity observed"]))
        return SentimentSignal(source, score, hl, None, time.strftime("%H:%M:%S"))

    # ── Cross-Verification Logic ───────────────────
    def verify_intelligence(self, news: SentimentSignal,
                            social: SentimentSignal) -> dict:
        """
        A Global Event is confirmed ONLY if detected in both streams.
        Returns verification dict consumed by the trading agent.
        """
        confirmed_event = None
        if news.event and social.event:
            # Both streams agree on same event category
            if news.event == social.event:
                confirmed_event = news.event
            else:
                # Different events — flag as MULTI_EVENT but don't act
                confirmed_event = "MULTI_EVENT"
        elif news.event or social.event:
            # Only one stream — flag for awareness but lower confidence
            confirmed_event = (news.event or social.event) + "_UNVERIFIED"

        # Weighted composite score (60/40)
        composite = (news.score * self.WEIGHT_NEWS +
                     social.score * self.WEIGHT_SOCIAL)

        # Agreement metric: how closely do streams agree? [-1,1] → [0,1]
        agreement  = 1.0 - abs(news.score - social.score) / 2.0

        # Confidence Score [0-100]
        # High agreement + matching event confirmation = high confidence
        base_conf  = agreement * 100
        if confirmed_event and "_UNVERIFIED" not in confirmed_event:
            base_conf = min(base_conf * 1.2, 100)  # boost for verified event
        elif confirmed_event and "_UNVERIFIED" in confirmed_event:
            base_conf *= 0.7                         # penalise unverified
        confidence = round(base_conf, 1)

        # Confidence tier
        if confidence >= 72:
            tier, tier_color = "HIGH",   "green"
        elif confidence >= 45:
            tier, tier_color = "MEDIUM", "yellow"
        else:
            tier, tier_color = "LOW",    "red"

        # Market direction consensus
        if composite > 0.15:   direction = "BULLISH"
        elif composite < -0.15: direction = "BEARISH"
        else:                   direction = "NEUTRAL"

        return {
            "composite":        round(composite, 4),
            "confidence":       confidence,
            "tier":             tier,
            "tier_color":       tier_color,
            "direction":        direction,
            "agreement":        round(agreement, 4),
            "confirmed_event":  confirmed_event,
            "news_score":       news.score,
            "social_score":     social.score,
            "news_headline":    news.headline,
            "social_headline":  social.headline,
            "news_event":       news.event,
            "social_event":     social.event,
        }

    # ── Public tick method ────────────────────────
    def tick(self) -> dict:
        now = time.time()
        with self._lock:
            if now - self._last_fetch < self._cache_sec and self._cached:
                return self._cached
        # Fetch in parallel threads for speed
        results = {}
        def fetch_news():   results["news"]   = self._fetch_news()
        def fetch_reddit(): results["reddit"] = self._fetch_reddit()
        t1 = threading.Thread(target=fetch_news,   daemon=True)
        t2 = threading.Thread(target=fetch_reddit, daemon=True)
        t1.start(); t2.start()
        t1.join(timeout=8); t2.join(timeout=8)
        news   = results.get("news",   self._simulate("NEWS"))
        social = results.get("reddit", self._simulate("REDDIT"))
        intel  = self.verify_intelligence(news, social)
        out = {
            "news":    {"score": news.score,   "headline": news.headline,
                        "event": news.event,   "ts": news.ts},
            "social":  {"score": social.score, "headline": social.headline,
                        "event": social.event, "ts": social.ts},
            "intel":   intel,
            "fetched_at": time.strftime("%H:%M:%S"),
        }
        with self._lock:
            self._cached     = out
            self._last_fetch = time.time()
        return out


# ════════════════════════════════════════════════════════════════════════════
# LAYER 9B — MULTI-ASSET RL TRADING AGENT  (v6.0 upgrade)
# Inherits v5.0 RL logic, adds:
#   • Leverage 1x-20x per trade
#   • Multi-asset support (BTC, ETH, SOL)
#   • Sentiment-gated execution (won't trade if confidence < 40)
#   • Confidence score attached to every trade
# ════════════════════════════════════════════════════════════════════════════
SUPPORTED_ASSETS = ["BTC", "ETH", "SOL"]


@dataclass
class Trade:
    id:             int
    asset:          str
    side:           str
    entry_price:    float
    size_usdt:      float
    leverage:       int
    confidence:     float
    conf_tier:      str
    open_ts:        str
    close_price:    float = 0.0
    closed:         bool  = False
    pnl:            float = 0.0
    pnl_pct:        float = 0.0
    direction_match: bool = False


class MultiAssetRLTrader:
    """
    RL Trader with leverage (1x-20x), multi-asset, and sentiment gating.
    Actions per asset: 0=HOLD, 1=BUY, 2=SELL
    Sentiment gate: skip trade if confidence < MIN_CONFIDENCE
    Leverage: amplifies both PnL and SL/TP thresholds accordingly.
    """
    TRADE_SIZE      = 400.0    # base USDT per trade (before leverage)
    MIN_CONFIDENCE  = 40.0     # don't open new trades below this
    SL_PCT          = 1.5      # stop-loss % (on raw price, not leveraged)
    TP_PCT          = 2.5      # take-profit %
    MAX_OPEN_TRADES = 3        # across all assets simultaneously

    def __init__(self, start_balance: float = 10_000.0,
                 alpha=0.18, gamma=0.94, epsilon=0.30,
                 default_leverage: int = 3):
        self.balance    = start_balance
        self.start_bal  = start_balance
        self.alpha      = alpha
        self.gamma      = gamma
        self.epsilon    = epsilon
        self.eps_decay  = 0.997
        self.eps_min    = 0.05
        self.leverage   = max(1, min(20, default_leverage))
        self.Q: dict    = {}
        self._price_hist: Dict[str, deque] = {a: deque(maxlen=30) for a in SUPPORTED_ASSETS}
        self._ls: Dict[str, tuple]  = {a: None for a in SUPPORTED_ASSETS}
        self._la: Dict[str, int]    = {a: None for a in SUPPORTED_ASSETS}
        self.episodes    = 0
        self.last_reward = 0.0
        self.open_trades: Dict[str, Optional[Trade]] = {a: None for a in SUPPORTED_ASSETS}
        self.trade_log:  List[Trade] = []
        self._trade_id   = 0
        self._lock       = threading.Lock()

    def set_leverage(self, lev: int):
        self.leverage = max(1, min(20, lev))

    # ── State encoding ────────────────────────────
    def _state(self, asset: str, price: float, intel: dict) -> tuple:
        hist = self._price_hist[asset]
        hist.append(price)
        prices = list(hist)
        lo, hi = min(prices), max(prices)
        rng = hi - lo if hi != lo else 1.0
        pb  = min(int((price - lo) / rng * 10), 9)
        tail = prices[-5:] if len(prices) >= 5 else prices
        mom  = (tail[-1] - tail[0]) / (tail[0] + 1e-6) * 100
        mb   = min(max(int(mom + 5), 0), 9)
        posb = 1 if self.open_trades[asset] else 0
        # Sentiment bucket: 0-4
        comp = intel.get("composite", 0.0)
        sb   = min(int((comp + 1.0) / 2.0 * 5), 4)
        return (pb, mb, posb, sb)

    def _Q(self, s):
        if s not in self.Q: self.Q[s] = [0.0, 0.0, 0.0]
        return self.Q[s]

    def _reward(self, asset: str, price: float, action_idx: int,
                intel: dict) -> float:
        r = 0.0
        t = self.open_trades.get(asset)
        if t and t.closed: r += t.pnl_pct * self.leverage * 2.0
        if action_idx == 0:
            hist = list(self._price_hist[asset])
            if len(hist) >= 2:
                mom = (hist[-1] - hist[-2]) / (hist[-2] + 1e-6) * 100
                r  -= abs(mom) * 0.5
        if not t: r -= 0.01
        # Sentiment alignment bonus
        direction = intel.get("direction", "NEUTRAL")
        if action_idx == 1 and direction == "BULLISH": r += 0.15
        if action_idx == 2 and direction == "BEARISH": r += 0.15
        if action_idx == 1 and direction == "BEARISH": r -= 0.20
        if action_idx == 2 and direction == "BULLISH": r -= 0.20
        return round(r, 4)

    # ── Trade lifecycle ───────────────────────────
    def _open(self, asset: str, price: float, intel: dict):
        open_count = sum(1 for t in self.open_trades.values() if t)
        if (self.open_trades[asset] or
                self.balance < self.TRADE_SIZE or
                open_count >= self.MAX_OPEN_TRADES):
            return
        conf  = intel.get("confidence", 0.0)
        if conf < self.MIN_CONFIDENCE:
            return   # sentiment gate: confidence too low
        with self._lock:
            self._trade_id += 1
            # direction_match: price momentum agrees with sentiment
            direction = intel.get("direction", "NEUTRAL")
            hist = list(self._price_hist[asset])
            price_bullish = len(hist) >= 2 and hist[-1] > hist[-2]
            dmatch = ((direction == "BULLISH" and price_bullish) or
                      (direction == "BEARISH" and not price_bullish))
            self.open_trades[asset] = Trade(
                id=self._trade_id, asset=asset, side="BUY",
                entry_price=price, size_usdt=self.TRADE_SIZE,
                leverage=self.leverage,
                confidence=conf,
                conf_tier=intel.get("tier", "LOW"),
                open_ts=time.strftime("%H:%M:%S"),
                direction_match=dmatch)
            self.balance -= self.TRADE_SIZE

    def _check_sl_tp(self, asset: str, price: float):
        t = self.open_trades.get(asset)
        if not t or t.closed: return
        pnl_raw = (price - t.entry_price) / t.entry_price * 100
        # Leverage amplifies realised PnL
        pnl_lev = pnl_raw * t.leverage
        if pnl_lev >= self.TP_PCT * t.leverage or pnl_lev <= -self.SL_PCT * t.leverage:
            self._close(asset, price, pnl_raw)

    def _close(self, asset: str, price: float, pnl_raw_pct: float):
        t = self.open_trades.get(asset)
        if not t: return
        with self._lock:
            pnl_lev  = pnl_raw_pct * t.leverage
            pnl_usdt = t.size_usdt * pnl_lev / 100
            t.close_price = price
            t.pnl         = round(pnl_usdt, 2)
            t.pnl_pct     = round(pnl_lev, 4)
            t.closed      = True
            self.balance  += t.size_usdt + pnl_usdt
            self.trade_log.append(t)
            if len(self.trade_log) > 60: self.trade_log.pop(0)
            self.open_trades[asset] = None

    # ── Per-asset RL step ─────────────────────────
    def _step_asset(self, asset: str, price: float, intel: dict):
        self._check_sl_tp(asset, price)
        s = self._state(asset, price, intel)
        q = self._Q(s)
        ls, la = self._ls[asset], self._la[asset]
        if la is not None:
            r = self._reward(asset, price, la, intel)
            self.last_reward = r
            pq = self._Q(ls)
            pq[la] += self.alpha * (r + self.gamma * max(q) - pq[la])
        if random.random() < self.epsilon:
            ai = random.randint(0, 2)
        else:
            ai = q.index(max(q))
        # Mask illegal
        if ai == 1 and (self.open_trades[asset] or
                        self.balance < self.TRADE_SIZE or
                        sum(1 for t in self.open_trades.values() if t) >= self.MAX_OPEN_TRADES):
            ai = 0
        if ai == 2 and not self.open_trades[asset]: ai = 0
        # Sentiment gate: block BUY in strong BEARISH, SELL in strong BULLISH
        direction = intel.get("direction", "NEUTRAL")
        conf      = intel.get("confidence", 0.0)
        if conf >= 70:
            if ai == 1 and direction == "BEARISH": ai = 0
            if ai == 2 and direction == "BULLISH": ai = 0
        action = ["HOLD","BUY","SELL"][ai]
        if action == "BUY":
            self._open(asset, price, intel)
        elif action == "SELL" and self.open_trades[asset]:
            pnl_raw = (price - self.open_trades[asset].entry_price) / self.open_trades[asset].entry_price * 100
            self._close(asset, price, pnl_raw)
        self._ls[asset] = s; self._la[asset] = ai
        return action

    # ── Master step (all assets) ──────────────────
    def step(self, prices: dict, intel: dict) -> dict:
        self.epsilon = max(self.eps_min, self.epsilon * self.eps_decay)
        self.episodes += 1
        actions = {}
        for asset in SUPPORTED_ASSETS:
            p = prices.get(asset, 0.0)
            if p > 0: actions[asset] = self._step_asset(asset, p, intel)
            else:     actions[asset] = "HOLD"

        total_pnl     = self.balance - self.start_bal
        total_pnl_pct = total_pnl / self.start_bal * 100
        wins  = sum(1 for t in self.trade_log if t.pnl > 0)
        total_closed = len(self.trade_log)
        win_rate = round(wins / total_closed * 100, 1) if total_closed else 0.0

        # Open positions summary
        open_positions = {}
        for asset, t in self.open_trades.items():
            if t:
                cur_price = prices.get(asset, t.entry_price)
                raw_pct = (cur_price - t.entry_price) / t.entry_price * 100
                lev_pct = raw_pct * t.leverage
                open_positions[asset] = {
                    "side":       t.side,
                    "entry":      round(t.entry_price, 2),
                    "current":    round(cur_price, 2),
                    "size":       t.size_usdt,
                    "leverage":   t.leverage,
                    "open_pnl":   round(t.size_usdt * lev_pct / 100, 2),
                    "open_pnl_pct": round(lev_pct, 4),
                    "confidence": t.confidence,
                    "conf_tier":  t.conf_tier,
                    "ts":         t.open_ts,
                }

        recent = [
            {"id": t.id, "asset": t.asset, "side": t.side,
             "entry": round(t.entry_price, 2), "close": round(t.close_price, 2),
             "pnl": t.pnl, "pnl_pct": t.pnl_pct,
             "leverage": t.leverage, "confidence": t.confidence,
             "conf_tier": t.conf_tier, "ts": t.open_ts}
            for t in reversed(self.trade_log[-6:])
        ]

        return {
            "actions":         actions,
            "balance":         round(self.balance, 2),
            "total_pnl":       round(total_pnl, 2),
            "total_pnl_pct":   round(total_pnl_pct, 4),
            "open_positions":  open_positions,
            "win_rate":        win_rate,
            "trades_closed":   total_closed,
            "recent_trades":   recent,
            "epsilon":         round(self.epsilon, 4),
            "episodes":        self.episodes,
            "last_reward":     self.last_reward,
            "leverage":        self.leverage,
        }


# ════════════════════════════════════════════════════════════════════════════
# LAYER 5 — MASTER ENGINE  (v6.0 — integrates sentiment + multi-asset trader)
# ════════════════════════════════════════════════════════════════════════════
class P3MasterEngine:
    def __init__(self, leverage: int = 3):
        self.hasher    = LatticeHashPlaceholder()
        self.rl        = RLConsensusAgent()
        self.shards    = ShardController()
        self.hedge     = HedgeEngine([
            Stablecoin("USDC-v3","0xA0b8..eB48",1, 420,50_000_000),
            Stablecoin("USDT",   "0xdAC1..ec7", 1, 310,80_000_000),
            Stablecoin("DAI",    "0x6B17..d0F", 2, 580,30_000_000),
            Stablecoin("FRAX",   "0x853d..99e", 2, 820,15_000_000),
            Stablecoin("LUSD",   "0x5f98..4e4e",3,1100, 5_000_000),
        ])
        self.sentiment = MultiSourceSentimentAggregator(cache_seconds=120)
        self.trader    = MultiAssetRLTrader(start_balance=10_000.0,
                                            default_leverage=leverage)
        self._idx       = 0; self._lock = threading.Lock()
        self._boot      = time.time(); self._last_hash = "—"
        self.rl.pretrain(300)

    def set_leverage(self, lev: int):
        self.trader.set_leverage(lev)

    def tick(self) -> dict:
        hedge     = self.hedge.tick()
        sentiment = self.sentiment.tick()
        intel     = sentiment["intel"]
        telem     = NetworkTelemetry(
            random.uniform(30, 380), random.uniform(12, 68),
            random.uniform(40, 260), random.randint(0, 3000),
            random.randint(60, 600))
        rl = self.rl.step(telem)

        payload  = os.urandom(512)
        pqc_hash = self.hasher.digest(payload)
        hi       = int.from_bytes(pqc_hash, "big")
        accepted = (hi >> (256 - rl["difficulty"])) == 0
        if accepted:
            with self._lock:
                m = BlockMeta(self._idx, self.shards.shard(self._idx),
                              pqc_hash, self.hasher.digest(pqc_hash + b"\x00"),
                              time.time(), len(payload) // 64,
                              random.choice(ShardController._SENDERS),
                              round(random.uniform(100, 50000), 2))
                self.shards.write(m); self._idx += 1
                self._last_hash = pqc_hash.hex()

        recent = self.shards.recent(2)
        feed   = [{"index": b.index, "sender": b.sender, "amount": b.amount,
                   "hash": b.pqc_hash.hex()[:12], "shard": b.shard}
                  for b in reversed(recent)]

        prices     = hedge.get("prices",  {"BTC": 65000.0, "ETH": 3200.0, "SOL": 145.0})
        trade_state = self.trader.step(prices, intel)

        return {
            # ── Core monitor (unchanged keys) ──────
            "hedge_state":   hedge["state"],
            "vol_index":     hedge["vol"],
            "vol_mean":      hedge["mean"],
            "vol_delta":     hedge["delta"],
            "active_asset":  hedge["asset"],
            "asset_apy":     hedge["apy"],
            "difficulty":    rl["difficulty"],
            "timeout_ms":    rl["timeout_ms"],
            "rl_epsilon":    rl["epsilon"],
            "rl_reward":     rl["reward"],
            "rl_episodes":   self.rl.episodes,
            "total_blocks":  self.shards.total,
            "active_shards": self.shards.active_count(),
            "accepted":      accepted,
            "last_hash":     self._last_hash,
            "block_feed":    feed,
            "shards":        self.shards.manifest(),
            "latency":       round(telem.latency_ms, 1),
            "throughput":    round(telem.throughput_bps, 1),
            "mempool":       telem.mempool_depth,
            "peers":         telem.peer_count,
            "uptime":        int(time.time() - self._boot),
            "ts":            time.strftime("%H:%M:%S"),
            "audit":         self.hedge.log[-6:],
            # ── Price feeds ────────────────────────
            "prices":        prices,
            "changes":       hedge.get("changes", {}),
            "data_source":   hedge.get("data_source", "SIMULATED"),
            # ── Sentiment (Layer 9) ─────────────────
            "sentiment":     sentiment,
            # ── Trading (Layer 9B) ──────────────────
            "trade":         trade_state,
        }


# ════════════════════════════════════════════════════════════════════════════
# LAYER 6 — TRIPLE-VIEW HD SHELL  (v6.0)
#   Tab 0: Core Monitor  (v5.0 shell, zero changes)
#   Tab 1: Oracle Intelligence  (NEW — sentiment dashboard)
#   Tab 2: Trading Terminal  (v5.0 terminal + leverage + multi-asset + confidence)
# ════════════════════════════════════════════════════════════════════════════
_SHELL = """
<link href="https://fonts.googleapis.com/css2?family=Share+Tech+Mono&display=swap" rel="stylesheet">

<style>
#p3-hd-monitor {
  width:100%; height:580px;
  background:#020408; border:2px solid #00f2ff; border-radius:10px;
  position:relative; overflow:hidden;
  font-family:'Share Tech Mono','Courier New',monospace;
  box-shadow:0 0 40px rgba(0,242,255,0.2);
  display:grid;
  grid-template-columns:1fr 1.5fr 1fr;
  grid-template-rows:50px 1fr 40px;
  gap:10px; padding:15px; box-sizing:border-box;
  transition:border-color .7s, box-shadow .7s;
}
#p3-hd-monitor.DEFENSIVE{border-color:#ff4444;box-shadow:0 0 50px rgba(255,68,68,0.4);}
#p3-hd-monitor.YIELD     {border-color:#00ff88;box-shadow:0 0 50px rgba(0,255,136,0.3);}

/* Header */
#p3-header{grid-column:1/-1;display:flex;justify-content:space-between;align-items:center;
  border-bottom:1px solid rgba(0,242,255,0.3);padding-bottom:10px;}
#p3-title{color:#00f2ff;font-weight:bold;font-size:16px;letter-spacing:2px;}
#p3-tabs {display:flex;gap:5px;}
.p3-tab  {font-size:8px;padding:4px 8px;cursor:pointer;letter-spacing:1px;
  border:1px solid rgba(0,242,255,0.35);color:rgba(0,242,255,0.55);
  background:rgba(0,242,255,0.05);transition:all .2s;}
.p3-tab.active{border-color:#00f2ff;color:#00f2ff;
  background:rgba(0,242,255,0.16);box-shadow:0 0 7px rgba(0,242,255,0.25);}
#p3-status{color:#fff;font-size:10px;}

/* Left / Centre / Right — CORE VIEW (unchanged) */
#p3-left{grid-row:2;border:1px solid rgba(0,242,255,0.15);border-radius:5px;
  padding:10px;display:flex;flex-direction:column;gap:6px;
  background:rgba(0,242,255,0.02);overflow:hidden;}
.p3-block-card{border-left:3px solid #00f2ff;background:rgba(0,242,255,0.07);
  padding:7px;font-size:10px;cursor:pointer;animation:slideIn .35s ease;transition:background .2s;}
.p3-block-card:hover{background:rgba(0,242,255,0.15);}
@keyframes slideIn{from{opacity:0;transform:translateX(-8px);}to{opacity:1;transform:none;}}
.bc-h{color:#fff;font-weight:bold;} .bc-d{opacity:.55;}
#p3-center{grid-row:2;border:1px solid rgba(0,242,255,0.15);border-radius:5px;
  display:flex;flex-direction:column;align-items:center;justify-content:space-between;
  overflow:hidden;position:relative;}
#v4-canvas{width:100%;height:68%;display:block;}
#p3-pills {display:flex;gap:7px;width:95%;margin-bottom:14px;}
.p3-pill  {flex:1;font-size:9px;text-align:center;padding:5px;
  border:1px solid rgba(0,242,255,0.5);background:rgba(0,242,255,0.08);
  cursor:pointer;color:#fff;transition:background .2s;line-height:1.6;}
.p3-pill:hover{background:rgba(0,242,255,0.22);}
#p3-right{grid-row:2;display:flex;flex-direction:column;gap:7px;}
.p3-panel{flex:1;border:1px solid rgba(0,242,255,0.15);background:rgba(0,0,0,0.45);
  padding:8px;font-size:10px;color:#00f2ff;display:flex;flex-direction:column;
  gap:3px;overflow:hidden;}
.p3-panel-title{font-size:8px;letter-spacing:2px;opacity:.45;
  border-bottom:1px solid rgba(0,242,255,0.15);padding-bottom:3px;margin-bottom:2px;}
.p3-stat{display:flex;justify-content:space-between;font-size:10px;}
.p3-stat .k{opacity:.5;} .p3-stat .v{color:#fff;}
.p3-stat .v.g{color:#00ff88;} .p3-stat .v.r{color:#ff4444;} .p3-stat .v.y{color:#ffcc00;}
#hedge-badge{text-align:center;font-size:12px;font-weight:bold;letter-spacing:3px;
  padding:5px;border:1px solid #00f2ff;margin-bottom:4px;transition:color .5s,border-color .5s;}

/* Footer */
#p3-footer{grid-column:1/-1;display:flex;justify-content:space-between;align-items:center;
  font-size:10px;color:rgba(0,242,255,0.65);
  border-top:1px solid rgba(0,242,255,0.25);padding-top:5px;}
.pdot{display:inline-block;width:6px;height:6px;border-radius:50%;background:#00ff88;
  animation:pulse 1.2s infinite;margin-right:5px;vertical-align:middle;}
.pdot.r{background:#ff4444;}
@keyframes pulse{0%,100%{opacity:1;box-shadow:0 0 5px #00ff88;}50%{opacity:.15;box-shadow:none;}}
::-webkit-scrollbar{width:3px;}
::-webkit-scrollbar-thumb{background:rgba(0,242,255,.2);border-radius:2px;}

/* ══ OVERLAY VIEWS — span all 3 columns in row 2 ══ */
.p3-overlay{display:none;grid-row:2;grid-column:1/-1;flex-direction:row;gap:10px;}
.p3-overlay.visible{display:flex;}

/* Confidence badge */
.conf-badge{display:inline-block;padding:2px 7px;font-size:9px;font-weight:bold;
  letter-spacing:1px;border-radius:2px;}
.conf-badge.green {color:#00ff88;border:1px solid #00ff88;background:rgba(0,255,136,0.08);}
.conf-badge.yellow{color:#ffcc00;border:1px solid #ffcc00;background:rgba(255,204,0,0.08);}
.conf-badge.red   {color:#ff4444;border:1px solid #ff4444;background:rgba(255,68,68,0.08);}

/* Sentiment meter bar */
.sent-bar-wrap{height:7px;background:rgba(255,255,255,0.06);
  border:1px solid rgba(0,242,255,0.15);border-radius:2px;overflow:hidden;margin:3px 0;}
.sent-bar{height:100%;transition:width .6s ease,background .6s ease;}

/* Asset price row */
.asset-row{display:flex;justify-content:space-between;font-size:10px;
  padding:4px 0;border-bottom:1px solid rgba(0,242,255,0.07);}

/* Open position card */
.pos-card{border-left:3px solid #00f2ff;background:rgba(0,242,255,0.05);
  padding:6px;font-size:9px;margin-bottom:4px;}

/* Trade log row */
.tr-row{display:flex;justify-content:space-between;font-size:9px;
  padding:3px 0;border-bottom:1px solid rgba(0,242,255,.06);}
.tr-row .win {color:#00ff88;} .tr-row .loss{color:#ff4444;}

/* Target bar */
#tr-target-bar-wrap{background:rgba(0,242,255,.08);border:1px solid rgba(0,242,255,.2);
  border-radius:2px;height:7px;overflow:hidden;margin-top:3px;}
#tr-target-bar{height:100%;background:linear-gradient(90deg,#00f2ff,#00ff88);transition:width .6s;}

/* Leverage selector */
#lev-selector{display:flex;gap:3px;flex-wrap:wrap;margin-top:4px;}
.lev-btn{padding:2px 5px;font-size:8px;cursor:pointer;
  border:1px solid rgba(0,242,255,0.3);color:rgba(0,242,255,0.6);
  background:transparent;font-family:inherit;}
.lev-btn.active{border-color:#00f2ff;color:#00f2ff;background:rgba(0,242,255,0.14);}
</style>

<!-- ═══════════════════ HTML ═══════════════════ -->
<div id="p3-hd-monitor">

  <!-- HEADER -->
  <div id="p3-header">
    <span id="p3-title">&gt;&gt; PROJECT 3.0 ORACLE</span>
    <div id="p3-tabs">
      <div class="p3-tab active" id="tab-core"      onclick="p3Tab('core')">CORE MONITOR</div>
      <div class="p3-tab"        id="tab-oracle"    onclick="p3Tab('oracle')">ORACLE INTEL</div>
      <div class="p3-tab"        id="tab-trade"     onclick="p3Tab('trade')">TRADING TERMINAL</div>
    </div>
    <span id="p3-status">LATTICE-v4 : <b style="color:#00ff00">ACTIVE</b></span>
  </div>

  <!-- ══ CORE VIEW (v5.0 layout, zero changes) ══ -->
  <div id="view-core" style="display:contents;">

    <div id="p3-left">
      <div style="text-align:center;font-size:12px;color:#fff;
                  border-bottom:1px solid rgba(0,242,255,.15);padding-bottom:5px;">
        LIVE BLOCKCHAIN
      </div>
      <div id="p3-feed" style="flex:1;display:flex;flex-direction:column;gap:5px;overflow:hidden;"></div>
      <div style="margin-top:auto;padding-top:5px;border-top:1px solid rgba(0,242,255,.12);
                  font-size:9px;display:flex;flex-direction:column;gap:2px;">
        <div class="p3-stat"><span class="k">TOTAL BLOCKS</span><span class="v" id="px-total">0</span></div>
        <div class="p3-stat"><span class="k">ACTIVE SHARDS</span><span class="v g" id="px-shards">0</span></div>
        <div class="p3-stat"><span class="k">MEMPOOL</span><span class="v" id="px-mem">—</span></div>
      </div>
    </div>

    <div id="p3-center">
      <canvas id="v4-canvas"></canvas>
      <div id="p3-pills">
        <div class="p3-pill" onclick="(function(){var s=window.__p3state||{};alert('Total Blocks: '+(s.total_blocks||0)+'\\nShards: '+(s.active_shards||0));})()">
          &#8383;<br><span id="pill-blocks-val">—</span><br>BLOCKS
        </div>
        <div class="p3-pill" onclick="(function(){var s=window.__p3state||{};alert('Epsilon: '+(s.rl_epsilon||0)+'\\nReward: '+(s.rl_reward||0)+'\\nEpisodes: '+(s.rl_episodes||0));})()">
          &#11042;<br><span id="pill-ai-val">—</span><br>RL-AGENT
        </div>
        <div class="p3-pill" onclick="(function(){var s=window.__p3state||{};alert('Difficulty: '+(s.difficulty||0)+'\\nTimeout: '+(s.timeout_ms||0)+'ms');})()">
          &#9672;<br><span id="pill-diff-val">—</span><br>DIFFICULTY
        </div>
        <div class="p3-pill" onclick="(function(){var s=window.__p3state||{};var i=(s.sentiment||{}).intel||{};alert('Confidence: '+i.confidence+'%\\nDirection: '+i.direction+'\\nEvent: '+(i.confirmed_event||'None'));})()">
          &#9678;<br><span id="pill-conf-val">—%</span><br>CONFIDENCE
        </div>
      </div>
    </div>

    <div id="p3-right">
      <div class="p3-panel">
        <div class="p3-panel-title">HEDGE ENGINE</div>
        <div id="hedge-badge">NEUTRAL</div>
        <div class="p3-stat"><span class="k">ACTIVE ASSET</span><span class="v" id="r-asset">—</span></div>
        <div class="p3-stat"><span class="k">APY</span><span class="v g" id="r-apy">—</span></div>
        <div class="p3-stat"><span class="k">VOL INDEX</span><span class="v" id="r-vol">—</span></div>
        <div class="p3-stat"><span class="k">VOL DELTA</span><span class="v" id="r-delta">—</span></div>
        <div class="p3-stat"><span class="k">VOL MEAN</span><span class="v" id="r-mean">—</span></div>
      </div>
      <div class="p3-panel">
        <div class="p3-panel-title">RL CONSENSUS</div>
        <div class="p3-stat"><span class="k">DIFFICULTY</span><span class="v y" id="r-diff">—</span></div>
        <div class="p3-stat"><span class="k">TIMEOUT</span><span class="v" id="r-tmout">—</span></div>
        <div class="p3-stat"><span class="k">EPSILON</span><span class="v" id="r-eps">—</span></div>
        <div class="p3-stat"><span class="k">REWARD</span><span class="v" id="r-rew">—</span></div>
        <div class="p3-stat"><span class="k">EPISODES</span><span class="v g" id="r-ep">—</span></div>
      </div>
      <div class="p3-panel">
        <div class="p3-panel-title">NETWORK TELEMETRY</div>
        <div class="p3-stat"><span class="k">LATENCY</span><span class="v" id="r-lat">—</span></div>
        <div class="p3-stat"><span class="k">THROUGHPUT</span><span class="v g" id="r-tput">—</span></div>
        <div class="p3-stat"><span class="k">MEMPOOL</span><span class="v" id="r-mpool">—</span></div>
        <div class="p3-stat"><span class="k">PEERS</span><span class="v" id="r-peers">—</span></div>
      </div>
      <div class="p3-panel" style="flex:1.5;">
        <div class="p3-panel-title">HEDGE AUDIT LOG</div>
        <div id="r-audit" style="display:flex;flex-direction:column;gap:3px;font-size:9px;overflow:hidden;"></div>
      </div>
    </div>

  </div>

  <!-- ══ ORACLE INTELLIGENCE VIEW (NEW) ══ -->
  <div id="view-oracle" class="p3-overlay">

    <!-- Column 1: News stream -->
    <div class="p3-panel" style="flex:1.1;min-width:0;">
      <div class="p3-panel-title">STREAM A · OFFICIAL NEWS (60% WEIGHT)</div>
      <div class="p3-stat">
        <span class="k">SCORE</span>
        <span class="v" id="or-news-score">—</span>
      </div>
      <div class="sent-bar-wrap">
        <div class="sent-bar" id="or-news-bar" style="width:50%;"></div>
      </div>
      <div style="font-size:9px;color:rgba(255,255,255,0.5);margin-top:4px;" id="or-news-hl">—</div>
      <div class="p3-stat" style="margin-top:6px;">
        <span class="k">EVENT DETECTED</span>
        <span class="v y" id="or-news-event">NONE</span>
      </div>
      <div class="p3-stat"><span class="k">FETCHED AT</span><span class="v" id="or-news-ts">—</span></div>

      <div class="p3-panel-title" style="margin-top:8px;">STREAM B · SOCIAL/REDDIT (40% WEIGHT)</div>
      <div class="p3-stat">
        <span class="k">SCORE</span>
        <span class="v" id="or-soc-score">—</span>
      </div>
      <div class="sent-bar-wrap">
        <div class="sent-bar" id="or-soc-bar" style="width:50%;"></div>
      </div>
      <div style="font-size:9px;color:rgba(255,255,255,0.5);margin-top:4px;" id="or-soc-hl">—</div>
      <div class="p3-stat" style="margin-top:6px;">
        <span class="k">EVENT DETECTED</span>
        <span class="v y" id="or-soc-event">NONE</span>
      </div>
      <div class="p3-stat"><span class="k">FETCHED AT</span><span class="v" id="or-soc-ts">—</span></div>
    </div>

    <!-- Column 2: Cross-verification output -->
    <div class="p3-panel" style="flex:1.2;min-width:0;">
      <div class="p3-panel-title">CROSS-VERIFICATION · verify_intelligence()</div>

      <div style="text-align:center;margin:8px 0 4px;">
        <div id="or-conf-num"
             style="font-size:36px;font-weight:bold;letter-spacing:2px;">—%</div>
        <div id="or-conf-label" style="font-size:10px;letter-spacing:2px;">CONFIDENCE</div>
      </div>

      <div style="text-align:center;margin-bottom:8px;">
        <span class="conf-badge" id="or-conf-badge">—</span>
      </div>

      <div class="p3-stat">
        <span class="k">COMPOSITE SCORE</span>
        <span class="v" id="or-composite">—</span>
      </div>
      <div class="p3-stat">
        <span class="k">STREAM AGREEMENT</span>
        <span class="v g" id="or-agree">—</span>
      </div>
      <div class="p3-stat">
        <span class="k">MARKET DIRECTION</span>
        <span class="v" id="or-direction">—</span>
      </div>
      <div class="p3-stat" style="margin-top:6px;">
        <span class="k">CONFIRMED EVENT</span>
        <span class="v r" id="or-event">NONE</span>
      </div>
      <div style="font-size:8px;opacity:.4;margin-top:3px;line-height:1.4;">
        Events confirmed only when detected across BOTH streams
      </div>

      <div class="p3-panel-title" style="margin-top:10px;">WEIGHTING MATRIX</div>
      <div class="p3-stat"><span class="k">OFFICIAL NEWS</span>
        <span class="v g">60%</span></div>
      <div class="p3-stat"><span class="k">SOCIAL MEDIA</span>
        <span class="v y">40%</span></div>
      <div class="p3-stat"><span class="k">DATA SOURCE</span>
        <span class="v" id="or-source">—</span></div>
      <div class="p3-stat"><span class="k">LAST UPDATED</span>
        <span class="v" id="or-updated">—</span></div>
    </div>

    <!-- Column 3: Multi-asset prices + direction -->
    <div class="p3-panel" style="flex:0.9;min-width:0;">
      <div class="p3-panel-title">MULTI-ASSET PRICES</div>
      <div id="or-assets">
        <div class="asset-row">
          <span style="color:#fff;">BTC/USDT</span>
          <span id="or-btc-price" class="v">—</span>
          <span id="or-btc-chg" class="v">—</span>
        </div>
        <div class="asset-row">
          <span style="color:#fff;">ETH/USDT</span>
          <span id="or-eth-price" class="v">—</span>
          <span id="or-eth-chg" class="v">—</span>
        </div>
        <div class="asset-row">
          <span style="color:#fff;">SOL/USDT</span>
          <span id="or-sol-price" class="v">—</span>
          <span id="or-sol-chg" class="v">—</span>
        </div>
      </div>

      <div class="p3-panel-title" style="margin-top:8px;">SIGNAL STRENGTH</div>
      <div style="font-size:9px;margin-bottom:3px;">NEWS vs SOCIAL AGREEMENT</div>
      <div class="sent-bar-wrap">
        <div class="sent-bar" id="or-agree-bar" style="width:50%;background:#00f2ff;"></div>
      </div>

      <div class="p3-panel-title" style="margin-top:8px;">TRADE GATE STATUS</div>
      <div class="p3-stat">
        <span class="k">MIN CONFIDENCE</span>
        <span class="v">40%</span>
      </div>
      <div class="p3-stat">
        <span class="k">CURRENT</span>
        <span class="v" id="or-gate-conf">—</span>
      </div>
      <div class="p3-stat">
        <span class="k">GATE</span>
        <span class="v" id="or-gate-status">—</span>
      </div>
      <div class="p3-stat" style="margin-top:4px;">
        <span class="k">SENTIMENT GATE</span>
        <span style="font-size:8px;opacity:.4;">Blocks trades if confidence &lt; 40%</span>
      </div>
    </div>

  </div><!-- end oracle view -->

  <!-- ══ TRADING TERMINAL VIEW (v5.0 + confidence + leverage + multi-asset) ══ -->
  <div id="view-trade" class="p3-overlay">

    <!-- Column 1: P&L HUD -->
    <div class="p3-panel" style="flex:1.0;min-width:0;">
      <div class="p3-panel-title">VIRTUAL P&amp;L · START $10,000</div>
      <div id="tr-pnl-big" style="font-size:28px;font-weight:bold;letter-spacing:2px;
                                   text-align:center;padding:6px 0;">$0.00</div>
      <div id="tr-pnl-pct" style="font-size:13px;text-align:center;margin-bottom:4px;">0.0000%</div>
      <div class="p3-stat"><span class="k">BALANCE</span><span class="v" id="tr-balance">$10,000</span></div>
      <div class="p3-stat"><span class="k">WIN RATE</span><span class="v g" id="tr-winrate">—</span></div>
      <div class="p3-stat"><span class="k">TRADES CLOSED</span><span class="v" id="tr-closed">0</span></div>
      <div class="p3-stat"><span class="k">CONFIDENCE</span><span class="v" id="tr-cur-conf">—</span></div>
      <div class="p3-panel-title" style="margin-top:6px;">DAILY TARGET: 10%</div>
      <div id="tr-target-bar-wrap"><div id="tr-target-bar"></div></div>
      <div style="font-size:8px;opacity:.4;margin-top:2px;" id="tr-target-label">0.00% of 10%</div>
      <div class="p3-panel-title" style="margin-top:6px;">LEVERAGE</div>
      <div id="lev-selector">
        <div class="lev-btn" onclick="setLeverage(1)">1x</div>
        <div class="lev-btn" onclick="setLeverage(2)">2x</div>
        <div class="lev-btn active" onclick="setLeverage(3)">3x</div>
        <div class="lev-btn" onclick="setLeverage(5)">5x</div>
        <div class="lev-btn" onclick="setLeverage(10)">10x</div>
        <div class="lev-btn" onclick="setLeverage(20)">20x</div>
      </div>
      <div style="font-size:8px;opacity:.4;margin-top:2px;">
        SL: 1.5% · TP: 2.5% (leveraged)
      </div>
    </div>

    <!-- Column 2: Open positions (multi-asset) -->
    <div class="p3-panel" style="flex:1.2;min-width:0;">
      <div class="p3-panel-title">OPEN POSITIONS (MAX 3)</div>
      <div id="tr-positions" style="display:flex;flex-direction:column;gap:4px;"></div>

      <div class="p3-panel-title" style="margin-top:6px;">RL TRADER · MULTI-ASSET</div>
      <div class="p3-stat"><span class="k">BTC ACTION</span><span class="v" id="tr-act-btc">HOLD</span></div>
      <div class="p3-stat"><span class="k">ETH ACTION</span><span class="v" id="tr-act-eth">HOLD</span></div>
      <div class="p3-stat"><span class="k">SOL ACTION</span><span class="v" id="tr-act-sol">HOLD</span></div>
      <div class="p3-stat" style="margin-top:4px;">
        <span class="k">EPISODES</span><span class="v g" id="tr-ep">—</span>
      </div>
      <div class="p3-stat">
        <span class="k">EPSILON</span><span class="v" id="tr-eps">—</span>
      </div>
      <div class="p3-stat">
        <span class="k">LAST REWARD</span><span class="v" id="tr-rew">—</span>
      </div>
      <div class="p3-stat" style="margin-top:4px;">
        <span class="k">ACTIVE LEVERAGE</span>
        <span class="v y" id="tr-lev-display">3x</span>
      </div>
    </div>

    <!-- Column 3: Closed trade log with confidence -->
    <div class="p3-panel" style="flex:1.0;min-width:0;">
      <div class="p3-panel-title">CLOSED TRADES · CONFIDENCE SCORES</div>
      <div id="tr-log" style="display:flex;flex-direction:column;gap:3px;overflow:hidden;flex:1;"></div>
    </div>

  </div><!-- end trade view -->

  <!-- FOOTER -->
  <div id="p3-footer">
    <span><span class="pdot" id="p3-dot"></span>UP: <span id="f-uptime">0s</span></span>
    <span id="f-prices" style="color:#00f2ff;font-size:9px;">BTC:$— ETH:$— SOL:$—</span>
    <span id="f-intel" style="font-size:9px;">INTEL: —</span>
    <span>TICK: <span id="f-ts">—</span></span>
  </div>

</div>

<script>
(function(){
'use strict';

/* ═══════ STATE & TAB CONTROL ═══════ */
window.__p3state = {};
var activeTab    = 'core';
var pendingLev   = 3;

window.p3Tab = function(tab) {
  activeTab = tab;
  ['core','oracle','trade'].forEach(function(t) {
    var vc = document.getElementById('view-'+t);
    var bt = document.getElementById('tab-'+t);
    if (t === 'core') {
      var kids = vc.children;
      for (var i=0;i<kids.length;i++) kids[i].style.display = (tab==='core'?'':'none');
    } else {
      vc.classList.toggle('visible', t===tab);
    }
    bt.classList.toggle('active', t===tab);
  });
  if (tab==='core') startCanvas(); else if(tab!=='core') stopCanvas();
};

window.setLeverage = function(lev) {
  pendingLev = lev;
  document.querySelectorAll('.lev-btn').forEach(function(b){
    b.classList.toggle('active', b.textContent===lev+'x');
  });
  document.getElementById('tr-lev-display').textContent = lev+'x';
  if (typeof google!=='undefined' && google.colab && google.colab.kernel)
    google.colab.kernel.invokeFunction('p3.setlev',[lev],{});
};

/* ═══════ CANVAS (full cyberpunk bloom — v5.0 renderer) ═══════ */
var canvas=document.getElementById('v4-canvas');
var ctx=canvas.getContext('2d');
var W,H,coreHue=185,pulseT=0,animRunning=true;
var ORBITS=3,PCT=[4,6,8];
var trails=[];
for(var oi=0;oi<ORBITS;oi++){trails.push([]);for(var pi=0;pi<PCT[oi];pi++)trails[oi].push([]);}
var lastT=performance.now();

function resize(){W=canvas.width=canvas.offsetWidth;H=canvas.height=canvas.offsetHeight;}
resize(); window.addEventListener('resize',resize);
function stopCanvas(){animRunning=false;}
function startCanvas(){if(animRunning)return;animRunning=true;lastT=performance.now();requestAnimationFrame(df);}

function glowLine(x0,y0,x1,y1,w,r,g,b,a){
  ctx.save();ctx.lineCap='round';
  ctx.strokeStyle='rgba('+r+','+g+','+b+','+(a*0.06)+')';ctx.lineWidth=w*5;
  ctx.beginPath();ctx.moveTo(x0,y0);ctx.lineTo(x1,y1);ctx.stroke();
  ctx.strokeStyle='rgba('+r+','+g+','+b+','+(a*0.18)+')';ctx.lineWidth=w*2.2;
  ctx.beginPath();ctx.moveTo(x0,y0);ctx.lineTo(x1,y1);ctx.stroke();
  ctx.strokeStyle='rgba('+r+','+g+','+b+','+a+')';ctx.lineWidth=w;
  ctx.beginPath();ctx.moveTo(x0,y0);ctx.lineTo(x1,y1);ctx.stroke();ctx.restore();
}
function radialBloom(cx,cy,r,R,G,B,A){
  for(var i=7;i>=1;i--){
    var fr=i/7,rr=r*(1+fr*3.6),al=A*(1-fr)*(1-fr)*0.5;
    if(al<0.01)continue;
    var g2=ctx.createRadialGradient(cx,cy,0,cx,cy,rr);
    g2.addColorStop(0,'rgba('+R+','+G+','+B+','+al+')');
    g2.addColorStop(1,'rgba('+R+','+G+','+B+',0)');
    ctx.fillStyle=g2;ctx.beginPath();ctx.arc(cx,cy,rr,0,Math.PI*2);ctx.fill();
  }
}
function hsl2rgb(h,s,l){
  h/=360;s/=100;l/=100;var r,g,b;
  if(!s){r=g=b=l;}else{
    var q=l<.5?l*(1+s):l+s-l*s,p=2*l-q;
    var h2r=function(p,q,t){if(t<0)t+=1;if(t>1)t-=1;
      if(t<1/6)return p+(q-p)*6*t;if(t<1/2)return q;
      if(t<2/3)return p+(q-p)*(2/3-t)*6;return p;};
    r=h2r(p,q,h+1/3);g=h2r(p,q,h);b=h2r(p,q,h-1/3);
  }
  return[Math.round(r*255),Math.round(g*255),Math.round(b*255)];
}

var targetHue=185;
function df(now){
  if(!animRunning)return;
  requestAnimationFrame(df);
  var dt=Math.min((now-lastT)/1000,0.05);lastT=now;pulseT+=dt;
  var st=window.__p3state||{};
  targetHue=st.hedge_state==='DEFENSIVE'?0:st.hedge_state==='YIELD'?140:185;
  var hd=targetHue-coreHue;if(Math.abs(hd)>180)hd-=Math.sign(hd)*360;
  coreHue+=hd*Math.min(dt*1.5,1);
  var rgb=hsl2rgb(coreHue,100,60),R=rgb[0],G=rgb[1],B=rgb[2];
  ctx.clearRect(0,0,W,H);
  var cx=W/2,cy=H/2,base=Math.min(W,H)/2;
  var gs=base*0.20;
  ctx.save();ctx.strokeStyle='rgba('+R+','+G+','+B+',0.04)';ctx.lineWidth=0.5;
  for(var gx=cx%gs;gx<W;gx+=gs){ctx.beginPath();ctx.moveTo(gx,0);ctx.lineTo(gx,H);ctx.stroke();}
  for(var gy=cy%gs;gy<H;gy+=gs){ctx.beginPath();ctx.moveTo(0,gy);ctx.lineTo(W,gy);ctx.stroke();}
  ctx.restore();
  for(var sy=0;sy<H;sy+=4){ctx.fillStyle='rgba(0,0,0,0.06)';ctx.fillRect(0,sy,W,1.5);}
  var RF=[0.13,0.24,0.37,0.54],SG=[6,12,24,48],SP=[0.38,-0.28,0.17,-0.12];
  for(var li=0;li<4;li++){
    var r2=base*RF[li],seg=SG[li],rot=pulseT*SP[li]+li*Math.PI/seg;
    var br=0.50+0.30*Math.sin(pulseT*1.2+li*1.1);
    for(var i=0;i<seg;i++){
      var a0=rot+i/seg*Math.PI*2,a1=rot+(i+1)/seg*Math.PI*2;
      glowLine(cx+Math.cos(a0)*r2,cy+Math.sin(a0)*r2,
               cx+Math.cos(a1)*r2,cy+Math.sin(a1)*r2,0.6,R,G,B,br*0.7);
    }
  }
  var diff=st.difficulty||3;
  for(var i=0;i<diff;i++){
    var a=i/diff*Math.PI*2+pulseT*0.13;
    var re=base*(0.54+0.07*Math.sin(pulseT*2.5+i));
    glowLine(cx,cy,cx+Math.cos(a)*re,cy+Math.sin(a)*re,0.5,R,G,B,0.09+0.11*Math.abs(Math.sin(pulseT*1.8+i)));
  }
  var OR=[0.19,0.34,0.52],OSP=[0.55,0.85,1.20];
  for(var ri=0;ri<ORBITS;ri++){
    var orR=base*OR[ri],cnt=PCT[ri],spd=OSP[ri],dir=(ri%2===0)?1:-1;
    var oc=hsl2rgb((coreHue+ri*20)%360,100,65);
    var oR=oc[0],oG=oc[1],oB=oc[2];
    for(var pi2=0;pi2<cnt;pi2++){
      var angle=pi2/cnt*Math.PI*2+pulseT*spd*dir;
      var px=cx+Math.cos(angle)*orR,py=cy+Math.sin(angle)*orR;
      var trail=trails[ri][pi2],af=(1.8+ri*0.4)*dt;
      for(var ti=0;ti<trail.length;ti++)trail[ti].age+=af;
      while(trail.length>0&&trail[0].age>1.0)trail.shift();
      trail.push({x:px,y:py,age:0});
      for(var ti=1;ti<trail.length;ti++){
        var a0t=Math.max(0,1-trail[ti].age),a1t=Math.max(0,1-trail[ti-1].age);
        var avg=(a0t+a1t)*0.5;if(avg<0.01)continue;
        ctx.save();ctx.strokeStyle='rgba('+oR+','+oG+','+oB+','+(avg*avg*0.75)+')';
        ctx.lineWidth=0.4+avg*1.8;ctx.lineCap='round';
        ctx.beginPath();ctx.moveTo(trail[ti-1].x,trail[ti-1].y);
        ctx.lineTo(trail[ti].x,trail[ti].y);ctx.stroke();ctx.restore();
      }
      var ha=0.55+0.45*Math.abs(Math.sin(pulseT*2+pi2));
      var hr=2.2*(0.8+0.2*Math.sin(pulseT*4+pi2));
      ctx.save();
      for(var gl=3;gl>=1;gl--){ctx.fillStyle='rgba('+oR+','+oG+','+oB+','+(ha*0.04/gl)+')';
        ctx.beginPath();ctx.arc(px,py,hr*(1+gl*0.8),0,Math.PI*2);ctx.fill();}
      ctx.fillStyle='rgba('+oR+','+oG+','+oB+','+ha+')';
      ctx.beginPath();ctx.arc(px,py,hr,0,Math.PI*2);ctx.fill();ctx.restore();
    }
  }
  var lat=st.latency||100,ln=Math.min(lat/380,1);
  var lRc=Math.round(ln*255),lGc=Math.round((1-ln)*180);
  var lr2=base*(0.09+0.03*Math.sin(pulseT*4.2));
  ctx.save();ctx.strokeStyle='rgba('+lRc+','+lGc+',60,0.35)';ctx.lineWidth=1.2;
  ctx.beginPath();ctx.arc(cx,cy,lr2,0,Math.PI*2);ctx.stroke();
  ctx.strokeStyle='rgba('+lRc+','+lGc+',60,0.12)';ctx.lineWidth=3.5;
  ctx.beginPath();ctx.arc(cx,cy,lr2,0,Math.PI*2);ctx.stroke();ctx.restore();
  var pulse=0.68+0.32*Math.sin(pulseT*2.4),cr=base*0.115*pulse;
  radialBloom(cx,cy,cr,R,G,B,0.85);
  for(var gl=5;gl>=1;gl--){
    ctx.save();ctx.fillStyle='rgba('+R+','+G+','+B+','+(0.22/(gl*gl)+0.01)+')';
    ctx.beginPath();ctx.arc(cx,cy,cr*(0.5+gl*0.28),0,Math.PI*2);ctx.fill();ctx.restore();
  }
  ctx.save();ctx.fillStyle='rgba(255,255,255,0.90)';ctx.beginPath();ctx.arc(cx,cy,cr*0.38,0,Math.PI*2);ctx.fill();
  ctx.fillStyle='rgba(255,255,255,1)';ctx.beginPath();ctx.arc(cx,cy,cr*0.17,0,Math.PI*2);ctx.fill();ctx.restore();
  for(var i=0;i<4;i++){
    var fa=i*Math.PI/2+pulseT*0.07,fl=base*(0.22+0.06*Math.sin(pulseT*1.5+i));
    ctx.save();ctx.strokeStyle='rgba(255,255,255,0.07)';ctx.lineWidth=0.5;
    ctx.beginPath();ctx.moveTo(cx,cy);ctx.lineTo(cx+Math.cos(fa)*fl,cy+Math.sin(fa)*fl);ctx.stroke();ctx.restore();
  }
}
requestAnimationFrame(df);

/* ═══════ HELPER: colour from sentiment score [-1,1] ═══════ */
function sentColor(s){ return s>0.1?'#00ff88':s<-0.1?'#ff4444':'#ffcc00'; }
function sentBarStyle(s){
  var pct=Math.round((s+1)/2*100);
  var col=s>0.1?'#00ff88':s<-0.1?'#ff4444':'#ffcc00';
  return 'width:'+pct+'%;background:'+col+';';
}
function fmtPrice(p){return p>0?'$'+p.toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2}):'—';}
function fmtChg(c){var s=(c>=0?'+':'')+c.toFixed(2)+'%';return {text:s,cls:c>=0?'g':'r'};}

/* ═══════ CORE MONITOR SYNC ═══════ */
function syncCore(state){
  var root=document.getElementById('p3-hd-monitor');
  root.className=state.hedge_state||'';
  document.getElementById('pill-blocks-val').textContent=(state.total_blocks||0).toLocaleString();
  document.getElementById('pill-ai-val').textContent='e '+(state.rl_epsilon||0);
  document.getElementById('pill-diff-val').textContent=state.difficulty||0;
  var intel=(state.sentiment||{}).intel||{};
  var confEl=document.getElementById('pill-conf-val');
  confEl.textContent=(intel.confidence||0)+'%';
  confEl.style.color=intel.tier_color==='green'?'#00ff88':intel.tier_color==='red'?'#ff4444':'#ffcc00';

  var feed=document.getElementById('p3-feed');
  if(state.block_feed&&state.block_feed.length){
    feed.innerHTML='';
    state.block_feed.forEach(function(b){
      var el=document.createElement('div');el.className='p3-block-card';
      el.innerHTML='<div class="bc-h">BLK #'+b.index+' SHARD '+b.shard+'</div>'+
        '<div class="bc-d">'+b.sender+'</div>'+
        '<div class="bc-d">$'+(b.amount||0).toLocaleString()+' '+(b.hash||'').substring(0,10)+'...</div>';
      feed.appendChild(el);
    });
  }
  document.getElementById('px-total').textContent=(state.total_blocks||0).toLocaleString();
  document.getElementById('px-shards').textContent=state.active_shards||0;
  document.getElementById('px-mem').textContent=(state.mempool||0).toLocaleString();

  var badge=document.getElementById('hedge-badge');
  badge.textContent=state.hedge_state||'NEUTRAL';
  var bc=state.hedge_state==='DEFENSIVE'?'#ff4444':state.hedge_state==='YIELD'?'#00ff88':'#00f2ff';
  badge.style.color=bc;badge.style.borderColor=bc;
  document.getElementById('r-asset').textContent=state.active_asset||'—';
  document.getElementById('r-apy').textContent=(state.asset_apy||0)+'%';
  document.getElementById('r-vol').textContent=state.vol_index||0;
  var dEl=document.getElementById('r-delta');
  dEl.textContent=(state.vol_delta||0)+'%';
  dEl.className='v'+((state.vol_delta||0)<-10?' r':(state.vol_delta||0)>8?' g':'');
  document.getElementById('r-mean').textContent=state.vol_mean||0;
  document.getElementById('r-diff').textContent=state.difficulty||0;
  document.getElementById('r-tmout').textContent=(state.timeout_ms||0)+'ms';
  document.getElementById('r-eps').textContent=state.rl_epsilon||0;
  var rEl=document.getElementById('r-rew');rEl.textContent=state.rl_reward||0;
  rEl.className='v'+((state.rl_reward||0)>0?' g':' r');
  document.getElementById('r-ep').textContent=state.rl_episodes||0;
  var latEl=document.getElementById('r-lat');
  latEl.textContent=(state.latency||0)+'ms';latEl.className='v'+((state.latency||0)>200?' r':' g');
  document.getElementById('r-tput').textContent=(state.throughput||0)+' bps';
  document.getElementById('r-mpool').textContent=(state.mempool||0).toLocaleString();
  document.getElementById('r-peers').textContent=state.peers||0;
  var aEl=document.getElementById('r-audit');
  if(state.audit&&state.audit.length){
    aEl.innerHTML='';
    state.audit.slice().reverse().forEach(function(e){
      var d=document.createElement('div');
      d.style.cssText='display:flex;justify-content:space-between;border-bottom:1px solid rgba(0,242,255,.06);padding-bottom:2px;';
      d.innerHTML='<span style="color:#ffcc00">'+(e.ts||'')+'</span>'+
        '<span style="color:#fff">'+(e.contract||'')+'</span>'+
        '<span style="color:#00ff88">'+(e.reason||'')+'</span>';
      aEl.appendChild(d);
    });
  }
  var dot=document.getElementById('p3-dot');
  dot.className='pdot'+(state.accepted?'':' r');
  document.getElementById('f-uptime').textContent=(state.uptime||0)+'s';
  document.getElementById('f-ts').textContent=state.ts||'—';
}

/* ═══════ ORACLE INTEL SYNC ═══════ */
function syncOracle(state){
  var sent=state.sentiment||{};
  var news=sent.news||{},soc=sent.social||{},intel=sent.intel||{};
  /* Stream A */
  var nsEl=document.getElementById('or-news-score');
  nsEl.textContent=news.score||0;nsEl.style.color=sentColor(news.score||0);
  document.getElementById('or-news-bar').style.cssText=sentBarStyle(news.score||0);
  document.getElementById('or-news-hl').textContent=news.headline||'—';
  document.getElementById('or-news-event').textContent=news.event||'NONE';
  document.getElementById('or-news-ts').textContent=news.ts||'—';
  /* Stream B */
  var ssEl=document.getElementById('or-soc-score');
  ssEl.textContent=soc.score||0;ssEl.style.color=sentColor(soc.score||0);
  document.getElementById('or-soc-bar').style.cssText=sentBarStyle(soc.score||0);
  document.getElementById('or-soc-hl').textContent=soc.headline||'—';
  document.getElementById('or-soc-event').textContent=soc.event||'NONE';
  document.getElementById('or-soc-ts').textContent=soc.ts||'—';
  /* Cross-verification */
  var conf=intel.confidence||0;
  var confEl=document.getElementById('or-conf-num');
  confEl.textContent=conf+'%';
  confEl.style.color=intel.tier_color==='green'?'#00ff88':intel.tier_color==='red'?'#ff4444':'#ffcc00';
  document.getElementById('or-conf-label').textContent='CONFIDENCE';
  var badgeEl=document.getElementById('or-conf-badge');
  badgeEl.textContent=intel.tier||'—';badgeEl.className='conf-badge '+(intel.tier_color||'yellow');
  document.getElementById('or-composite').textContent=intel.composite||0;
  var agEl=document.getElementById('or-agree');
  agEl.textContent=((intel.agreement||0)*100).toFixed(1)+'%';
  agEl.className='v'+((intel.agreement||0)>0.65?' g':(intel.agreement||0)<0.35?' r':' y');
  var dirEl=document.getElementById('or-direction');
  dirEl.textContent=intel.direction||'NEUTRAL';
  dirEl.className='v'+(intel.direction==='BULLISH'?' g':intel.direction==='BEARISH'?' r':' y');
  document.getElementById('or-event').textContent=intel.confirmed_event||'NONE';
  document.getElementById('or-source').textContent=state.data_source||'SIMULATED';
  document.getElementById('or-updated').textContent=sent.fetched_at||'—';
  /* Agreement bar */
  document.getElementById('or-agree-bar').style.width=((intel.agreement||0)*100)+'%';
  /* Asset prices */
  var prices=state.prices||{},changes=state.changes||{};
  ['btc','eth','sol'].forEach(function(a){
    var sym=a.toUpperCase();
    document.getElementById('or-'+a+'-price').textContent=fmtPrice(prices[sym]||0);
    var c=fmtChg(changes[sym]||0);
    var cel=document.getElementById('or-'+a+'-chg');
    cel.textContent=c.text;cel.className='v '+c.cls;
  });
  /* Gate */
  var gConf=document.getElementById('or-gate-conf');
  gConf.textContent=conf+'%';gConf.className='v'+(conf>=40?' g':' r');
  var gStat=document.getElementById('or-gate-status');
  gStat.textContent=conf>=40?'OPEN':'BLOCKED';gStat.className='v'+(conf>=40?' g':' r');
  /* Footer intel */
  document.getElementById('f-intel').textContent=
    'INTEL: '+(intel.direction||'—')+' '+conf+'% '+(intel.tier||'—');
  /* Footer prices */
  document.getElementById('f-prices').textContent=
    'BTC:'+fmtPrice(prices.BTC||0)+' ETH:'+fmtPrice(prices.ETH||0)+' SOL:'+fmtPrice(prices.SOL||0);
}

/* ═══════ TRADING TERMINAL SYNC ═══════ */
function syncTrade(state){
  var tr=state.trade||{},intel=(state.sentiment||{}).intel||{};
  var pnl=tr.total_pnl||0,pnlPct=tr.total_pnl_pct||0;
  var pnlEl=document.getElementById('tr-pnl-big');
  pnlEl.textContent=(pnl>=0?'+$':'−$')+Math.abs(pnl).toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2});
  pnlEl.style.color=pnl>=0?'#00ff88':'#ff4444';
  var pnlPctEl=document.getElementById('tr-pnl-pct');
  pnlPctEl.textContent=(pnlPct>=0?'+':'')+pnlPct.toFixed(4)+'%';
  pnlPctEl.style.color=pnlPct>=0?'#00ff88':'#ff4444';
  document.getElementById('tr-balance').textContent='$'+(tr.balance||0).toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2});
  document.getElementById('tr-winrate').textContent=(tr.win_rate||0)+'%';
  document.getElementById('tr-closed').textContent=tr.trades_closed||0;
  var curConf=document.getElementById('tr-cur-conf');
  var conf=intel.confidence||0;
  curConf.textContent=conf+'%';
  curConf.className='v conf-badge '+(intel.tier_color||'yellow');

  var pct=Math.max(0,Math.min(pnlPct,10));
  document.getElementById('tr-target-bar').style.width=(pct/10*100)+'%';
  document.getElementById('tr-target-label').textContent=pnlPct.toFixed(2)+'% of 10% daily target';

  /* RL actions */
  var actions=tr.actions||{};
  ['BTC','ETH','SOL'].forEach(function(a){
    var el=document.getElementById('tr-act-'+a.toLowerCase());
    var act=actions[a]||'HOLD';
    el.textContent=act;
    el.className='v'+(act==='BUY'?' g':act==='SELL'?' r':' ');
  });
  document.getElementById('tr-ep').textContent=tr.episodes||0;
  document.getElementById('tr-eps').textContent=tr.epsilon||0;
  var rewEl=document.getElementById('tr-rew');
  rewEl.textContent=tr.last_reward||0;
  rewEl.className='v'+((tr.last_reward||0)>0?' g':' r');
  document.getElementById('tr-lev-display').textContent=(tr.leverage||3)+'x';

  /* Open positions */
  var posDiv=document.getElementById('tr-positions');
  var positions=tr.open_positions||{};
  posDiv.innerHTML='';
  var hasOpen=false;
  ['BTC','ETH','SOL'].forEach(function(a){
    var p=positions[a];
    if(!p)return;hasOpen=true;
    var opnl=p.open_pnl||0;
    var confBadge='<span class="conf-badge '+p.conf_tier.toLowerCase()+'">'+p.confidence+'%</span>';
    var card=document.createElement('div');card.className='pos-card';
    card.style.borderLeftColor=opnl>=0?'#00ff88':'#ff4444';
    card.innerHTML=
      '<div style="display:flex;justify-content:space-between;margin-bottom:2px;">'+
        '<span style="color:#fff;font-weight:bold;">'+a+'/USDT</span>'+
        confBadge+
        '<span style="color:#ffcc00;">'+p.leverage+'x</span>'+
      '</div>'+
      '<div style="display:flex;justify-content:space-between;">'+
        '<span class="k">Entry: $'+p.entry+'</span>'+
        '<span class="k">Now: $'+p.current+'</span>'+
      '</div>'+
      '<div style="display:flex;justify-content:space-between;">'+
        '<span style="color:'+(opnl>=0?'#00ff88':'#ff4444')+';">'+(opnl>=0?'+':'')+opnl+'</span>'+
        '<span style="color:'+(opnl>=0?'#00ff88':'#ff4444')+';">'+(p.open_pnl_pct>=0?'+':'')+p.open_pnl_pct+'%</span>'+
      '</div>';
    posDiv.appendChild(card);
  });
  if(!hasOpen){
    posDiv.innerHTML='<div style="opacity:.35;font-size:9px;margin:8px 0;">No open positions</div>';
  }

  /* Trade log with confidence scores */
  var log=document.getElementById('tr-log');
  if(tr.recent_trades&&tr.recent_trades.length){
    log.innerHTML='';
    tr.recent_trades.forEach(function(t){
      var row=document.createElement('div');row.className='tr-row';
      var pnlC=t.pnl>=0?'win':'loss';
      var cb='<span class="conf-badge '+(t.conf_tier||'low').toLowerCase()+'">'+t.confidence+'%</span>';
      row.innerHTML=
        '<span style="color:#ffcc00;font-size:8px;">'+t.ts+'</span>'+
        '<span style="color:#fff;">'+t.asset+'</span>'+
        '<span style="color:#aaa;">'+t.leverage+'x</span>'+
        cb+
        '<span class="'+pnlC+'">'+(t.pnl>=0?'+':'')+t.pnl+'</span>'+
        '<span class="'+pnlC+'">'+(t.pnl_pct>=0?'+':'')+t.pnl_pct+'%</span>';
      log.appendChild(row);
    });
  } else {
    log.innerHTML='<div style="opacity:.35;font-size:9px;margin-top:8px;">No closed trades yet...</div>';
  }
}

/* ═══════ MASTER SYNC ═══════ */
window.__p3sync=function(state){
  window.__p3state=state;
  syncCore(state);
  syncOracle(state);
  syncTrade(state);
};

/* ═══════ POLLING ═══════ */
function scheduleTick(){
  if(typeof google!=='undefined'&&google.colab&&google.colab.kernel)
    google.colab.kernel.invokeFunction('p3.tick',[],{});
}
setInterval(scheduleTick,1500);

})();
</script>
"""


# ════════════════════════════════════════════════════════════════════════════
# LAUNCH
# ════════════════════════════════════════════════════════════════════════════
def _inject(state: dict) -> str:
    return "window.__p3sync(" + json.dumps(state) + ");"


def launch(leverage: int = 3):
    # Install feedparser silently if missing
    if not _HAS_FEEDPARSER:
        import subprocess, sys
        subprocess.run([sys.executable, "-m", "pip", "install",
                        "feedparser", "-q"], check=False)

    engine = P3MasterEngine(leverage=leverage)

    if _IN_COLAB:
        def _tick_handler(*args, **kwargs):
            display(Javascript(_inject(engine.tick())))
        def _setlev_handler(lev, **kwargs):
            engine.set_leverage(int(lev))
        _colab_output.register_callback('p3.tick',   _tick_handler)
        _colab_output.register_callback('p3.setlev', _setlev_handler)
    else:
        _stop = threading.Event()
        def _ticker():
            while not _stop.is_set():
                display(Javascript(_inject(engine.tick())))
                time.sleep(1.5)
        threading.Thread(target=_ticker, daemon=True).start()

    display(HTML(_SHELL))
    display(Javascript(_inject(engine.tick())))

    print("[P3 v6.0 ORACLE] Engine online.")
    print("[P3 v6.0] Tabs: CORE MONITOR | ORACLE INTEL | TRADING TERMINAL")
    print("[P3 v6.0] Leverage buttons wired to kernel. Default: " + str(leverage) + "x")
    print("[P3 v6.0] Sentiment cache: 120s. Parallel fetch: Google News + Reddit.")
    return engine


engine = launch(leverage=3)
