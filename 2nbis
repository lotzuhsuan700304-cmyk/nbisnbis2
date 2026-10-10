import datetime
import numpy as np
import pandas as pd
import plotly.graph_objects as go
import streamlit as st
import yfinance as yf

# 設定手機適配寬度與標題
st.set_page_config(page_title="NBIS 監控儀表板", layout="centered")


# 載入數據
@st.cache_data(ttl=900)  # 每 15 分鐘快取更新一次
def load_data(symbol="NBIS"):
    ticker = yf.Ticker(symbol)
    df = ticker.history(period="3mo")

    # 計算 ATR (14日)
    df["H-L"] = df["High"] - df["Low"]
    df["H-PC"] = (df["High"] - df["Close"].shift(1)).abs()
    df["L-PC"] = (df["Low"] - df["Close"].shift(1)).abs()
    df["TR"] = df[["H-L", "H-PC", "L-PC"]].max(axis=1)
    df["ATR"] = df["TR"].rolling(14).mean()

    # 計算動態停損軌道與費波納契
    latest_close = df["Close"].iloc[-1]
    latest_atr = df["ATR"].iloc[-1]
    trailing_stop = latest_close - (1.5 * latest_atr)

    high_20 = df["High"].tail(20).max()
    low_20 = df["Low"].tail(20).min()
    fib_618 = high_20 - (high_20 - low_20) * 0.382

    news = ticker.news[:3]
    return df, latest_close, latest_atr, trailing_stop, fib_618, high_20, news


# 資料讀取
df, close, atr, trailing_stop, fib_target, high_20, news = load_data("NBIS")

# 帳戶參數
HOLDINGS = 580
COST_BASIS = 226.5
unrealized_pnl = (close - COST_BASIS) * HOLDINGS
pnl_pct = ((close - COST_BASIS) / COST_BASIS) * 100

# ----------------- UI 介面設計 -----------------
st.title("📊 NBIS 盤前量化監控")

# 1. 頂部核心指標卡片
col1, col2 = st.columns(2)
col1.metric("最新股價", f"${close:.2f}", f"{close - COST_BASIS:+.2f}")
col2.metric("未實現損益", f"${unrealized_pnl:,.0f}", f"{pnl_pct:+.2f}%")

st.markdown("---")

# 2. 關鍵價位參考卡片
col3, col4 = st.columns(2)
col3.metric("🔴 動態防守線 (ATR)", f"${trailing_stop:.2f}")
col4.metric("🟢 上行目標 (Fib)", f"${fib_target:.2f}")

# 3. 視覺化 K 線與軌道圖
fig = go.Figure()

# K 線
fig.add_trace(
    go.Candlestick(
        x=df.index,
        open=df["Open"],
        high=df["High"],
        low=df["Low"],
        close=df["Close"],
        name="K線",
    )
)

# 成本線
fig.add_hline(
    y=COST_BASIS,
    line_dash="dot",
    line_color="gray",
    annotation_text=f"成本線 ${COST_BASIS}",
)

# 動態停損水平線
fig.add_hline(
    y=trailing_stop,
    line_dash="dash",
    line_color="crimson",
    annotation_text=f"ATR 防守 ${trailing_stop:.2f}",
)

# 近期目標水平線
fig.add_hline(
    y=fib_target,
    line_dash="dash",
    line_color="mediumseagreen",
    annotation_text=f"目標 ${fib_target:.2f}",
)

fig.update_layout(
    height=420,
    margin=dict(l=10, r=10, t=25, b=10),
    xaxis_rangeslider_visible=False,
    template="plotly_dark",
)
st.plotly_chart(fig, use_container_width=True)

# 4. 近期輿情新聞
st.subheader("📰 最新外電輿情")
for item in news:
    ts = datetime.datetime.fromtimestamp(
        item["providerPublishTime"]
    ).strftime("%m/%d %H:%M")
    st.markdown(f"- **[{ts}]** [{item['title']}]({item['link']})")
