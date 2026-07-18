# koolcoin<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>LEDGER//LIVE — Crypto Tracker</title>

<script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>

<style>
  html, body { margin: 0; padding: 0; height: 100%; background: #0d1117; }
  #root { height: 100%; }
  @keyframes flashUp { 0% { background-color: rgba(57,255,140,0.35); } 100% { background-color: transparent; } }
  @keyframes flashDown { 0% { background-color: rgba(255,77,77,0.35); } 100% { background-color: transparent; } }
  @keyframes tickerScroll { 0% { transform: translateX(0); } 100% { transform: translateX(-50%); } }
  .flash-up { animation: flashUp 0.7s ease-out; }
  .flash-down { animation: flashDown 0.7s ease-out; }
  ::-webkit-scrollbar { width: 6px; height: 6px; }
  ::-webkit-scrollbar-thumb { background: #2a3038; border-radius: 3px; }
  * { box-sizing: border-box; }
  input:focus, select:focus { outline: none; border-color: #39ff8c !important; }
  button:hover { filter: brightness(1.1); }
  button { cursor: pointer; }
</style>
</head>
<body>
<div id="root"></div>

<script type="text/babel" data-presets="react">
const { useState, useEffect, useRef } = React;

// ---------- Config ----------
const ASSETS = [
  { symbol: "BTC", name: "Bitcoin", base: 64200, vol: 0.0009 },
  { symbol: "ETH", name: "Ethereum", base: 3450, vol: 0.0012 },
  { symbol: "SOL", name: "Solana", base: 168, vol: 0.002 },
  { symbol: "XRP", name: "XRP", base: 0.62, vol: 0.0022 },
  { symbol: "DOGE", name: "Dogecoin", base: 0.14, vol: 0.003 },
  { symbol: "ADA", name: "Cardano", base: 0.44, vol: 0.0018 },
  { symbol: "USD", name: "US Dollar", base: 1, vol: 0 },
];

const TRADEABLE = ASSETS.filter((a) => a.symbol !== "USD");
const STARTING_BALANCE = 10000; // starting USD account balance

const fmt = (n, d = 2) =>
  n.toLocaleString("en-US", { minimumFractionDigits: d, maximumFractionDigits: d });

const fmtRate = (n) => {
  if (n >= 1000) return fmt(n, 2);
  if (n >= 1) return fmt(n, 4);
  if (n >= 0.0001) return fmt(n, 6);
  return n.toExponential(2);
};

const fmtSigned = (n, suffix = "") => {
  const sign = n >= 0 ? "+" : "-";
  return sign + fmt(Math.abs(n), 2) + suffix;
};

// simple inline icon glyphs (no external icon library needed)
const IconUp = ({ size = 12 }) => <span style={{ fontSize: size, lineHeight: 1 }}>▲</span>;
const IconDown = ({ size = 12 }) => <span style={{ fontSize: size, lineHeight: 1 }}>▼</span>;
const IconPlus = ({ size = 14 }) => <span style={{ fontSize: size, lineHeight: 1 }}>+</span>;
const IconSwap = ({ size = 14 }) => <span style={{ fontSize: size, lineHeight: 1, color: "#6b7280" }}>⇄</span>;
const IconActivity = ({ size = 16 }) => <span style={{ fontSize: size, lineHeight: 1, color: "#39ff8c" }}>◆</span>;

function CryptoTracker() {
  const [prices, setPrices] = useState(() =>
    Object.fromEntries(ASSETS.map((a) => [a.symbol, a.base]))
  );
  const [prevPrices, setPrevPrices] = useState(() =>
    Object.fromEntries(ASSETS.map((a) => [a.symbol, a.base]))
  );
  const [flash, setFlash] = useState({});
  const [history, setHistory] = useState(() =>
    Object.fromEntries(ASSETS.map((a) => [a.symbol, Array(40).fill(a.base)]))
  );
  const [tick, setTick] = useState(0);
  const priceRef = useRef(prices);
  priceRef.current = prices;

  // orders: open + closed
  const [openOrders, setOpenOrders] = useState([]);
  const [closedOrders, setClosedOrders] = useState([]);
  const [showAdd, setShowAdd] = useState(false);
  const [activeTab, setActiveTab] = useState("open");
  const [form, setForm] = useState({
    side: "buy",
    base: "BTC",
    quote: "USD",
    qty: "",
    entry: "",
  });

  // account holdings: cash + coin balances, starts all-USD
  const [holdings, setHoldings] = useState(() => {
    const h = Object.fromEntries(ASSETS.map((a) => [a.symbol, 0]));
    h.USD = STARTING_BALANCE;
    return h;
  });

  // simulate live prices (USD-denominated), pairs are derived
  useEffect(() => {
    const id = setInterval(() => {
      setPrevPrices(priceRef.current);
      setPrices((old) => {
        const next = {};
        for (const a of ASSETS) {
          if (a.symbol === "USD") {
            next.USD = 1;
            continue;
          }
          const cur = old[a.symbol];
          const drift = (Math.random() - 0.5) * 2 * a.vol;
          let np = cur * (1 + drift);
          if (np < a.base * 0.4) np = a.base * 0.4;
          next[a.symbol] = np;
        }
        return next;
      });
      setTick((t) => t + 1);
    }, 1800);
    return () => clearInterval(id);
  }, []);

  useEffect(() => {
    const f = {};
    for (const a of ASSETS) {
      const cur = prices[a.symbol];
      const prev = prevPrices[a.symbol];
      if (cur > prev) f[a.symbol] = "up";
      else if (cur < prev) f[a.symbol] = "down";
    }
    setFlash(f);
    setHistory((h) => {
      const next = { ...h };
      for (const a of ASSETS) {
        next[a.symbol] = [...h[a.symbol].slice(-39), prices[a.symbol]];
      }
      return next;
    });
    const t = setTimeout(() => setFlash({}), 700);
    return () => clearTimeout(t);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [tick]);

  // current rate for a base/quote pair, in quote units per 1 base
  const rateOf = (base, quote) => prices[base] / prices[quote];

  const placeOrder = () => {
    const qty = parseFloat(form.qty);
    const entry = parseFloat(form.entry);
    if (!qty || qty <= 0 || form.base === form.quote) return;
    const effectiveEntry = entry > 0 ? entry : rateOf(form.base, form.quote);

    // Buying `base` with `quote`: spend quote, receive base.
    // Selling `base` for `quote`: spend base, receive quote.
    const spendCurrency = form.side === "buy" ? form.quote : form.base;
    const spendAmount = form.side === "buy" ? qty * effectiveEntry : qty;

    if ((holdings[spendCurrency] || 0) < spendAmount) return; // insufficient funds, silently block

    setHoldings((h) => ({
      ...h,
      [spendCurrency]: h[spendCurrency] - spendAmount,
    }));

    setOpenOrders((o) => [
      ...o,
      {
        id: Date.now(),
        side: form.side,
        base: form.base,
        quote: form.quote,
        qty,
        entry: effectiveEntry,
        openedAt: Date.now(),
      },
    ]);
    setForm({ ...form, qty: "", entry: "" });
    setShowAdd(false);
  };

  const closeOrder = (id) => {
    setOpenOrders((orders) => {
      const order = orders.find((o) => o.id === id);
      if (!order) return orders;
      const mark = rateOf(order.base, order.quote);
      const diff = order.side === "buy" ? mark - order.entry : order.entry - mark;
      const pnl = diff * order.qty;

      // Return the held asset back to its origin currency at current mark,
      // crediting back principal + P&L in the quote currency.
      const returnCurrency = order.side === "buy" ? order.quote : order.base;
      let returnAmount;
      if (order.side === "buy") {
        // was holding base qty worth entry*qty in quote; now sell base at mark -> receive mark*qty in quote
        returnAmount = order.qty * mark;
      } else {
        // was holding quote worth entry*qty (short); buy back base at mark, net back in quote terms
        returnAmount = order.qty * order.entry + pnl;
      }

      setHoldings((h) => ({
        ...h,
        [returnCurrency]: (h[returnCurrency] || 0) + returnAmount,
      }));

      setClosedOrders((c) => [
        { ...order, closeRate: mark, pnl, closedAt: Date.now() },
        ...c,
      ]);
      return orders.filter((o) => o.id !== id);
    });
  };

  const totalPnl = openOrders.reduce((sum, o) => {
    const mark = rateOf(o.base, o.quote);
    const diff = o.side === "buy" ? mark - o.entry : o.entry - mark;
    return sum + diff * o.qty;
  }, 0);

  const totalRealized = closedOrders.reduce((sum, o) => sum + o.pnl, 0);

  // Account: cash + value of any non-USD holdings priced in USD, plus notional tied up in open orders
  const cashUSD = holdings.USD || 0;
  const holdingsValueUSD = TRADEABLE.reduce(
    (sum, a) => sum + (holdings[a.symbol] || 0) * prices[a.symbol],
    0
  );
  const openOrdersValueUSD = openOrders.reduce((sum, o) => {
    const mark = rateOf(o.base, o.quote);
    const notionalQuote = o.side === "buy" ? o.qty * mark : o.qty * o.entry + (o.entry - mark) * o.qty;
    return sum + notionalQuote * (prices[o.quote] || 1);
  }, 0);
  const totalEquity = cashUSD + holdingsValueUSD + openOrdersValueUSD;

  const spendCurrency = form.side === "buy" ? form.quote : form.base;
  const qtyNum = parseFloat(form.qty) || 0;
  const entryNum = parseFloat(form.entry) || 0;
  const effectiveEntryPreview = entryNum > 0 ? entryNum : rateOf(form.base, form.quote);
  const spendAmountPreview = form.side === "buy" ? qtyNum * effectiveEntryPreview : qtyNum;
  const insufficientFunds = (holdings[spendCurrency] || 0) < spendAmountPreview;
  const orderInvalid = form.base === form.quote || !form.qty || qtyNum <= 0 || insufficientFunds;

  return (
    <div style={styles.page}>
      {/* Ticker tape */}
      <div style={styles.tickerWrap}>
        <div style={{ ...styles.tickerTrack, animation: "tickerScroll 22s linear infinite" }}>
          {[...TRADEABLE, ...TRADEABLE].map((a, i) => {
            const cur = prices[a.symbol];
            const prev = prevPrices[a.symbol];
            const up = cur >= prev;
            return (
              <span key={i} style={styles.tickerItem}>
                <span style={{ color: "#6b7280", fontWeight: 600 }}>{a.symbol}/USD</span>
                <span style={{ color: up ? "#39ff8c" : "#ff4d4d" }}>${fmtRate(cur)}</span>
                <span style={{ color: up ? "#39ff8c" : "#ff4d4d", fontSize: 11 }}>
                  {up ? "▲" : "▼"}
                </span>
              </span>
            );
          })}
        </div>
      </div>

      <div style={styles.container}>
        <header style={styles.header}>
          <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
            <IconActivity size={16} />
            <h1 style={styles.h1}>LEDGER//LIVE</h1>
          </div>
          <div style={styles.summaryInline}>
            <span style={{ color: "#8b949e" }}>EQUITY</span>
            <span style={{ fontWeight: 700 }}>${fmt(totalEquity)}</span>
            <span style={{ color: "#8b949e" }}>OPEN</span>
            <span style={{ color: totalPnl >= 0 ? "#39ff8c" : "#ff4d4d", fontWeight: 700 }}>
              {fmtSigned(totalPnl)}
            </span>
            <span style={{ color: "#8b949e" }}>REALIZED</span>
            <span style={{ color: totalRealized >= 0 ? "#39ff8c" : "#ff4d4d", fontWeight: 700 }}>
              {fmtSigned(totalRealized)}
            </span>
          </div>
          <div style={styles.liveTag}>
            <span style={styles.liveDot} />
            LIVE
          </div>
        </header>

        <div style={styles.mainGrid}>
          {/* Left: Account + Market */}
          <div style={styles.leftCol}>
            <div style={styles.sectionHead}><span>ACCOUNT</span></div>
            <div style={styles.accountPanel}>
              <div style={styles.accountRow}>
                <span style={styles.accountLabel}>Cash (USD)</span>
                <span style={styles.accountValue}>${fmt(cashUSD)}</span>
              </div>
              <div style={styles.accountRow}>
                <span style={styles.accountLabel}>Holdings value</span>
                <span style={styles.accountValue}>${fmt(holdingsValueUSD)}</span>
              </div>
              <div style={styles.accountRow}>
                <span style={styles.accountLabel}>In open orders</span>
                <span style={styles.accountValue}>${fmt(openOrdersValueUSD)}</span>
              </div>
              <div style={{ ...styles.accountRow, borderTop: "1px solid #1f2730", paddingTop: 6, marginTop: 2 }}>
                <span style={{ ...styles.accountLabel, color: "#e6edf3", fontWeight: 700 }}>Total equity</span>
                <span style={{ ...styles.accountValue, color: "#39ff8c", fontWeight: 700 }}>${fmt(totalEquity)}</span>
              </div>
              {TRADEABLE.some((a) => holdings[a.symbol] > 0.00000001) && (
                <div style={styles.holdingsList}>
                  {TRADEABLE.filter((a) => holdings[a.symbol] > 0.00000001).map((a) => (
                    <div key={a.symbol} style={styles.holdingChip}>
                      <span style={{ fontWeight: 700 }}>{a.symbol}</span>
                      <span style={{ color: "#8b949e" }}>{fmtRate(holdings[a.symbol])}</span>
                    </div>
                  ))}
                </div>
              )}
            </div>

            <div style={styles.sectionHead}><span>MARKET (vs USD)</span></div>
            <div style={styles.marketGrid}>
              {TRADEABLE.map((a) => {
                const cur = prices[a.symbol];
                const first = history[a.symbol][0];
                const pct = ((cur - first) / first) * 100;
                const up = pct >= 0;
                const points = history[a.symbol];
                const min = Math.min(...points);
                const max = Math.max(...points);
                const range = max - min || 1;
                const path = points
                  .map((p, i) => {
                    const x = (i / (points.length - 1)) * 100;
                    const y = 26 - ((p - min) / range) * 22 - 1;
                    return `${i === 0 ? "M" : "L"}${x},${y}`;
                  })
                  .join(" ");
                return (
                  <div
                    key={a.symbol}
                    className={flash[a.symbol] ? `flash-${flash[a.symbol]}` : ""}
                    style={styles.marketCard}
                  >
                    <div style={{ display: "flex", justifyContent: "space-between" }}>
                      <div>
                        <div style={styles.symbol}>{a.symbol}</div>
                        <div style={styles.assetName}>{a.name}</div>
                      </div>
                      <svg width="64" height="26" style={{ opacity: 0.9 }}>
                        <path d={path} fill="none" stroke={up ? "#39ff8c" : "#ff4d4d"} strokeWidth="1.3" />
                      </svg>
                    </div>
                    <div style={{ display: "flex", alignItems: "baseline", gap: 6, marginTop: 6 }}>
                      <span style={styles.price}>${fmtRate(cur)}</span>
                      <span style={{ color: up ? "#39ff8c" : "#ff4d4d", fontSize: 11, display: "flex", alignItems: "center", gap: 2 }}>
                        {up ? <IconUp size={11} /> : <IconDown size={11} />}
                        {fmtSigned(pct, "%")}
                      </span>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* Right: Orders */}
          <div style={styles.rightCol}>
            {/* Place order */}
            <div style={styles.sectionHead}>
              <span>ORDERS</span>
              <button style={styles.addBtn} onClick={() => setShowAdd((s) => !s)}>
                <IconPlus size={13} /> Place order
              </button>
            </div>

            {showAdd && (
              <div style={styles.orderForm}>
                <div style={styles.sideToggle}>
                  <button
                    style={{ ...styles.sideBtn, ...(form.side === "buy" ? styles.sideBtnActiveBuy : {}) }}
                    onClick={() => setForm({ ...form, side: "buy" })}
                  >
                    BUY
                  </button>
                  <button
                    style={{ ...styles.sideBtn, ...(form.side === "sell" ? styles.sideBtnActiveSell : {}) }}
                    onClick={() => setForm({ ...form, side: "sell" })}
                  >
                    SELL
                  </button>
                </div>

                <div style={styles.pairRow}>
                  <select
                    value={form.base}
                    onChange={(e) => setForm({ ...form, base: e.target.value })}
                    style={styles.select}
                  >
                    {TRADEABLE.map((a) => (
                      <option key={a.symbol} value={a.symbol}>{a.symbol}</option>
                    ))}
                  </select>
                  <IconSwap size={14} />
                  <select
                    value={form.quote}
                    onChange={(e) => setForm({ ...form, quote: e.target.value })}
                    style={styles.select}
                  >
                    {ASSETS.map((a) => (
                      <option key={a.symbol} value={a.symbol}>{a.symbol}</option>
                    ))}
                  </select>
                </div>

                <div style={styles.pairRow}>
                  <input
                    placeholder={`Quantity of ${form.base}`}
                    value={form.qty}
                    onChange={(e) => setForm({ ...form, qty: e.target.value })}
                    style={styles.input}
                    type="number"
                  />
                  <input
                    placeholder={`Rate (blank = live ${fmtRate(rateOf(form.base, form.quote))})`}
                    value={form.entry}
                    onChange={(e) => setForm({ ...form, entry: e.target.value })}
                    style={styles.input}
                    type="number"
                  />
                </div>

                <button
                  style={{
                    ...styles.confirmBtn,
                    ...(orderInvalid ? styles.confirmBtnDisabled : {}),
                  }}
                  disabled={orderInvalid}
                  onClick={placeOrder}
                >
                  {form.side === "buy" ? "Buy" : "Sell"} {form.base}/{form.quote}
                </button>
              </div>
            )}

            {/* Tabs */}
            <div style={styles.tabRow}>
              <button
                style={{ ...styles.tabBtn, ...(activeTab === "open" ? styles.tabBtnActive : {}) }}
                onClick={() => setActiveTab("open")}
              >
                Open ({openOrders.length})
              </button>
              <button
                style={{ ...styles.tabBtn, ...(activeTab === "closed" ? styles.tabBtnActive : {}) }}
                onClick={() => setActiveTab("closed")}
              >
                Closed ({closedOrders.length})
              </button>
            </div>

            <div style={styles.scrollPanel}>
              {activeTab === "open" ? (
                openOrders.length === 0 ? (
                  <div style={styles.empty}>No open orders — place one to start tracking live P&L.</div>
                ) : (
                  <div style={styles.posTable}>
                    <div style={styles.posHeaderRow}>
                      <span>PAIR</span>
                      <span>SIDE</span>
                      <span>QTY</span>
                      <span>ENTRY</span>
                      <span>MARK</span>
                      <span>P&L</span>
                      <span></span>
                    </div>
                    {openOrders.map((o) => {
                      const mark = rateOf(o.base, o.quote);
                      const diff = o.side === "buy" ? mark - o.entry : o.entry - mark;
                      const pnl = diff * o.qty;
                      const up = pnl >= 0;
                      const flashKey = flash[o.base] || flash[o.quote];
                      return (
                        <div key={o.id} className={flashKey ? `flash-${flashKey}` : ""} style={styles.posRow}>
                          <span style={{ fontWeight: 700 }}>{o.base}/{o.quote}</span>
                          <span style={{ color: o.side === "buy" ? "#39ff8c" : "#ff4d4d", fontWeight: 600 }}>
                            {o.side.toUpperCase()}
                          </span>
                          <span>{o.qty}</span>
                          <span>{fmtRate(o.entry)}</span>
                          <span>{fmtRate(mark)}</span>
                          <span style={{ color: up ? "#39ff8c" : "#ff4d4d" }}>
                            {fmtSigned(pnl)} {o.quote}
                          </span>
                          <button style={styles.closeBtn} onClick={() => closeOrder(o.id)}>
                            Close
                          </button>
                        </div>
                      );
                    })}
                  </div>
                )
              ) : closedOrders.length === 0 ? (
                <div style={styles.empty}>Closed orders will appear here with realized P&L.</div>
              ) : (
                <div style={styles.posTable}>
                  <div style={{ ...styles.posHeaderRow, gridTemplateColumns: "1fr 0.8fr 1fr 1fr 1fr 1.4fr 1fr" }}>
                    <span>PAIR</span>
                    <span>SIDE</span>
                    <span>QTY</span>
                    <span>ENTRY</span>
                    <span>CLOSE</span>
                    <span>REALIZED P&L</span>
                    <span>CLOSED</span>
                  </div>
                  {closedOrders.map((o) => {
                    const up = o.pnl >= 0;
                    return (
                      <div key={o.id} style={{ ...styles.posRow, gridTemplateColumns: "1fr 0.8fr 1fr 1fr 1fr 1.4fr 1fr", opacity: 0.85 }}>
                        <span style={{ fontWeight: 700 }}>{o.base}/{o.quote}</span>
                        <span style={{ color: o.side === "buy" ? "#39ff8c" : "#ff4d4d" }}>{o.side.toUpperCase()}</span>
                        <span>{o.qty}</span>
                        <span>{fmtRate(o.entry)}</span>
                        <span>{fmtRate(o.closeRate)}</span>
                        <span style={{ color: up ? "#39ff8c" : "#ff4d4d" }}>{fmtSigned(o.pnl)} {o.quote}</span>
                        <span style={{ fontSize: 11, color: "#6b7280" }}>
                          {new Date(o.closedAt).toLocaleTimeString()}
                        </span>
                      </div>
                    );
                  })}
                </div>
              )}
            </div>
          </div>
        </div>

        <div style={styles.footNote}>Simulated prices — not live market data.</div>
      </div>
    </div>
  );
}

const styles = {
  page: {
    height: "100vh",
    width: "100%",
    background: "#0d1117",
    color: "#e6edf3",
    fontFamily: "'JetBrains Mono', 'SF Mono', Consolas, monospace",
    display: "flex",
    flexDirection: "column",
    overflow: "hidden",
  },
  tickerWrap: {
    flexShrink: 0,
    background: "#111820",
    borderBottom: "1px solid #1f2730",
    overflow: "hidden",
    whiteSpace: "nowrap",
    padding: "6px 0",
  },
  tickerTrack: { display: "inline-flex", gap: 24, width: "max-content" },
  tickerItem: {
    display: "inline-flex",
    alignItems: "center",
    gap: 6,
    fontSize: 11,
    paddingRight: 24,
    borderRight: "1px solid #1f2730",
  },
  container: {
    flex: 1,
    minHeight: 0,
    display: "flex",
    flexDirection: "column",
    padding: "10px 14px",
    maxWidth: 1200,
    width: "100%",
    margin: "0 auto",
  },
  header: {
    flexShrink: 0,
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 10,
    flexWrap: "wrap",
    gap: 8,
  },
  h1: { fontSize: 15, letterSpacing: 2, margin: 0, fontWeight: 700 },
  summaryInline: {
    display: "flex",
    alignItems: "center",
    gap: 8,
    fontSize: 12,
  },
  liveTag: {
    display: "flex",
    alignItems: "center",
    gap: 6,
    fontSize: 10,
    letterSpacing: 1,
    color: "#39ff8c",
    border: "1px solid #1f3a2a",
    background: "#0f1f16",
    padding: "4px 9px",
    borderRadius: 20,
  },
  liveDot: { width: 6, height: 6, borderRadius: "50%", background: "#39ff8c", boxShadow: "0 0 6px #39ff8c" },
  mainGrid: {
    flex: 1,
    minHeight: 0,
    display: "grid",
    gridTemplateColumns: "1fr 1fr",
    gap: 14,
  },
  leftCol: {
    minHeight: 0,
    display: "flex",
    flexDirection: "column",
    overflow: "hidden",
  },
  rightCol: {
    minHeight: 0,
    display: "flex",
    flexDirection: "column",
    overflow: "hidden",
  },
  sectionHead: {
    flexShrink: 0,
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    fontSize: 10,
    letterSpacing: 1.5,
    color: "#8b949e",
    margin: "0 0 8px",
  },
  addBtn: {
    display: "flex",
    alignItems: "center",
    gap: 4,
    background: "#16211a",
    color: "#39ff8c",
    border: "1px solid #21402c",
    borderRadius: 6,
    padding: "4px 9px",
    fontSize: 11,
    cursor: "pointer",
    fontFamily: "inherit",
  },
  marketGrid: {
    display: "grid",
    gridTemplateColumns: "repeat(auto-fit, minmax(150px, 1fr))",
    gap: 8,
    overflowY: "auto",
    paddingRight: 2,
  },
  marketCard: {
    background: "#111820",
    border: "1px solid #1f2730",
    borderRadius: 8,
    padding: "10px 12px",
    transition: "background-color 0.3s",
  },
  symbol: { fontWeight: 700, fontSize: 13 },
  assetName: { fontSize: 10, color: "#6b7280" },
  price: { fontSize: 15, fontWeight: 700 },
  orderForm: {
    flexShrink: 0,
    display: "flex",
    flexDirection: "column",
    gap: 8,
    background: "#111820",
    border: "1px solid #1f2730",
    borderRadius: 8,
    padding: 10,
    marginBottom: 10,
  },
  sideToggle: { display: "flex", gap: 8 },
  sideBtn: {
    flex: 1,
    padding: "6px",
    borderRadius: 6,
    border: "1px solid #2a3038",
    background: "#0d1117",
    color: "#6b7280",
    fontWeight: 700,
    fontSize: 11,
    cursor: "pointer",
    fontFamily: "inherit",
    letterSpacing: 1,
  },
  sideBtnActiveBuy: { background: "#16211a", color: "#39ff8c", border: "1px solid #21402c" },
  sideBtnActiveSell: { background: "#211616", color: "#ff4d4d", border: "1px solid #402121" },
  pairRow: { display: "flex", alignItems: "center", gap: 8 },
  select: {
    flex: 1,
    background: "#0d1117",
    color: "#e6edf3",
    border: "1px solid #2a3038",
    borderRadius: 6,
    padding: "6px",
    fontFamily: "inherit",
    fontSize: 12,
  },
  input: {
    flex: 1,
    background: "#0d1117",
    color: "#e6edf3",
    border: "1px solid #2a3038",
    borderRadius: 6,
    padding: "6px",
    fontFamily: "inherit",
    fontSize: 12,
  },
  warnText: { color: "#ff4d4d", fontSize: 10 },
  confirmBtn: {
    background: "#39ff8c",
    color: "#0d1117",
    border: "none",
    borderRadius: 6,
    padding: "8px 12px",
    fontWeight: 700,
    fontSize: 11,
    cursor: "pointer",
    fontFamily: "inherit",
    letterSpacing: 1,
  },
  confirmBtnDisabled: {
    background: "#1f2730",
    color: "#6b7280",
    cursor: "not-allowed",
  },
  accountPanel: {
    flexShrink: 0,
    background: "#111820",
    border: "1px solid #1f2730",
    borderRadius: 8,
    padding: "10px 12px",
    marginBottom: 10,
  },
  accountRow: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    fontSize: 12,
    padding: "3px 0",
  },
  accountLabel: { color: "#8b949e" },
  accountValue: { fontWeight: 600 },
  holdingsList: {
    display: "flex",
    flexWrap: "wrap",
    gap: 6,
    marginTop: 8,
    paddingTop: 8,
    borderTop: "1px solid #1a2028",
  },
  holdingChip: {
    display: "flex",
    alignItems: "center",
    gap: 6,
    fontSize: 11,
    background: "#0d1117",
    border: "1px solid #1f2730",
    borderRadius: 6,
    padding: "3px 8px",
  },
  tabRow: { flexShrink: 0, display: "flex", gap: 6, marginBottom: 8 },
  tabBtn: {
    padding: "5px 12px",
    borderRadius: 6,
    border: "1px solid #1f2730",
    background: "#111820",
    color: "#8b949e",
    fontSize: 11,
    cursor: "pointer",
    fontFamily: "inherit",
    fontWeight: 600,
  },
  tabBtnActive: { color: "#39ff8c", border: "1px solid #21402c", background: "#16211a" },
  scrollPanel: { flex: 1, minHeight: 0, overflowY: "auto" },
  empty: {
    color: "#6b7280",
    fontSize: 12,
    border: "1px dashed #2a3038",
    borderRadius: 10,
    padding: "20px 14px",
    textAlign: "center",
  },
  posTable: { border: "1px solid #1f2730", borderRadius: 8, overflow: "hidden" },
  posHeaderRow: {
    display: "grid",
    gridTemplateColumns: "1fr 0.8fr 0.8fr 1fr 1fr 1.4fr 55px",
    padding: "8px 10px",
    fontSize: 9,
    letterSpacing: 1,
    color: "#6b7280",
    background: "#111820",
    position: "sticky",
    top: 0,
  },
  posRow: {
    display: "grid",
    gridTemplateColumns: "1fr 0.8fr 0.8fr 1fr 1fr 1.4fr 55px",
    alignItems: "center",
    padding: "9px 10px",
    fontSize: 12,
    borderTop: "1px solid #1a2028",
    background: "#0d1117",
    transition: "background-color 0.3s",
  },
  closeBtn: {
    background: "#1a1f26",
    border: "1px solid #2a3038",
    color: "#e6edf3",
    borderRadius: 6,
    padding: "4px 7px",
    fontSize: 10,
    cursor: "pointer",
    fontFamily: "inherit",
  },
  footNote: { flexShrink: 0, marginTop: 8, fontSize: 10, color: "#4b5563", textAlign: "center" },
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<CryptoTracker />);
</script>
</body>
</html>
