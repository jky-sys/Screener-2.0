import streamlit as st
import yfinance as yf
import pandas as pd
import numpy as np
import requests
import io
import plotly.graph_objects as go # 引入绘图库

# ==============================================================================
# 配置与页面设置
# ==============================================================================
st.set_page_config(page_title="Trinity Pro V2.0", page_icon="📈", layout="wide")

# [内置列表]
CUSTOM_TICKERS = [
    # === 半导体 & 芯片 ===
    "NVDA", "AMD", "TSM", "AVGO", "INTC", "QCOM", "MU", "TXN", 
    "AMAT", "LRCX", "ASML", "ARM", "SMCI", "MRVL", "ON", "ADI", 
    "KLAC", "SNPS", "CDNS", "TER", "WDC", "PSTG",
    # === 航天 & 太空 ===
    "RKLB", "SPCE", "LUNR", "ASTS", "BA", "LMT", "NOC", "RTX", 
    "GD", "AXON", "PLTR", "SPIR", "BKSY", "RDW",
    # === 加密货币 ===
    "MSTR", "COIN", "MARA", "RIOT", "CLSK", "IREN", "HUT", 
    "BITF", "HOOD", "SQ", "PYPL", "CIFR", "WULF", "CORZ", "SDIG",
    # === 热门科技 ===
    "TSLA", "AAPL", "MSFT", "GOOGL", "META", "AMZN", 
    "NET", "SNOW", "U", "DKNG", "RBLX", "AI", "PATH", "JOBY",
    # === 核能 & 新能源 ===
    "SMR", "OKLO", "CCJ", "UEC", "NNE", "BWXT", "LEU", "FLR", 
    "CEG", "VST", "TLN", "GCT",
    # === 网络安全 & 未来科技 ===
    "CRWD", "NBIS", "PANW", "ZS", "FTNT", "S", "SENT", "OKTA",
    "IONQ", "RGTI", "QUBT", "DNA"
]

NAS100_FALLBACK_TICKERS = [
    "AAPL", "MSFT", "NVDA", "AMZN", "META", "AVGO", "TSLA", "GOOGL", "GOOG", "COST",
    "NFLX", "AMD", "ADBE", "PEP", "LIN", "TMUS", "CSCO", "QCOM", "INTU", "TXN",
    "AMGN", "AMAT", "BKNG", "HON", "ISRG", "SBUX", "LRCX", "VRTX", "ADP", "MDLZ",
    "GILD", "ADI", "PANW", "REGN", "MU", "SNPS", "KLAC", "PDD", "CDNS", "MELI",
    "MAR", "PYPL", "CSX", "ORLY", "MNST", "ASML", "CTAS", "WDAY", "ROP", "LULU",
    "NXPI", "PCAR", "FTNT", "DXCM", "MRVL", "ADSK", "CRWD", "KDP", "ABNB", "PAYX",
    "ODFL", "CHTR", "IDEXX", "ROST", "FAST", "MCHP", "CPRT", "SIRI", "CTSH", "EA",
    "EXC", "VRSK", "BIIB", "XEL", "CEG", "DDOG", "GEHC", "BKR", "GFS", "ON",
    "TTD", "DLTR", "CDW", "ANSS", "WBD", "FANG", "TEAM", "AZN", "CCEP", "TTWO",
    "ZM", "ILMN", "ALGN", "WBA", "EBAY", "ENPH", "ZS", "JD", "LCID", "ARM", "SMCI"
]

SP500_FALLBACK_TICKERS = [
    "MSFT", "AAPL", "NVDA", "AMZN", "META", "GOOGL", "GOOG", "BRK-B", "LLY", "AVGO",
    "JPM", "TSLA", "XOM", "UNH", "V", "PG", "MA", "COST", "JNJ", "HD",
    "MRK", "ABBV", "CVX", "BAC", "CRM", "AMD", "NFLX", "PEP", "KO", "WMT",
    "T-Mobile", "ADBE", "LIN", "ACN", "MCD", "DIS", "CSCO", "ABT", "INTU", "QCOM",
    "WFC", "VZ", "AMGN", "TXN", "IBM", "PFE", "PM", "CAT", "ISRG", "UBER"
]

# ==============================================================================
# 核心逻辑函数
# ==============================================================================
@st.cache_data(ttl=3600)
def get_stock_list(mode):
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"}
    try:
        if mode == "NAS100":
            url = "https://en.wikipedia.org/wiki/Nasdaq-100"
            response = requests.get(url, headers=headers, timeout=5)
            response.raise_for_status()
            tables = pd.read_html(io.StringIO(response.text))
            df = tables[0]
            col = 'Symbol' if 'Symbol' in df.columns else 'Ticker'
            return list(set([t.replace('.', '-') for t in df[col].tolist()]))
        elif mode == "SP500":
            url = "https://en.wikipedia.org/wiki/List_of_S%26P_500_companies"
            response = requests.get(url, headers=headers, timeout=5)
            response.raise_for_status()
            tables = pd.read_html(io.StringIO(response.text))
            return list(set([t.replace('.', '-') for t in tables[0]['Symbol'].tolist()]))
        else:
            return CUSTOM_TICKERS
    except:
        if mode == "NAS100": return NAS100_FALLBACK_TICKERS
        if mode == "SP500": return SP500_FALLBACK_TICKERS
        return CUSTOM_TICKERS

def calculate_ema(series, span):
    return series.ewm(span=span, adjust=False).mean()

def calculate_trinity_indicators(df):
    # NX Channels
    df['nx_up1'] = calculate_ema(df['High'], 26)
    df['nx_dw1'] = calculate_ema(df['Low'], 26)
    df['nx_rising'] = (df['nx_up1'] > df['nx_up1'].shift(1)) & (df['nx_dw1'] > df['nx_dw1'].shift(1))
    
    # MACD
    fast_ema = calculate_ema(df['Close'], 12)
    slow_ema = calculate_ema(df['Close'], 26)
    df['dif'] = fast_ema - slow_ema
    df['dea'] = calculate_ema(df['dif'], 9)
    df['macd_gold_cross'] = (df['dif'] > df['dea']) & (df['dif'].shift(1) < df['dea'].shift(1))

    # CD Divergence
    min_price_60 = df['Low'].rolling(60).min()
    min_dif_60 = df['dif'].rolling(60).min()
    price_is_low = df['Low'] <= min_price_60 * 1.05
    dif_is_stronger = df['dif'] > min_dif_60 + 0.1
    df['cd_potential'] = price_is_low & dif_is_stronger & df['macd_gold_cross']

    # INST
    if len(df) < 250:
        df['inst_buy'] = 0
        return df
    
    def rma(series, length): 
        return series.ewm(alpha=1/length, adjust=False).mean()
    
    high_long = df['High'].rolling(250).max()
    low_long  = df['Low'].rolling(250).min()
    
    low_diff = df['Low'] - df['Low'].shift(1)
    instc = rma(low_diff.abs(), 3) / rma(low_diff.clip(lower=0), 3).replace(0, np.nan) * 100
    instc = instc.fillna(0)
    
    is_oversold = df['Low'] <= df['Low'].rolling(30).min()
    inst_signal = np.where(is_oversold, instc, 0)
    
    df['inst_buy'] = calculate_ema(pd.Series(inst_signal, index=df.index), 3)
    
    return df

# ==============================================================================
# V2.0 新增: 交互式绘图函数
# ==============================================================================
def create_chart(df, ticker):
    # 只取最近 150 天的数据，避免图表太长
    plot_df = df.iloc[-150:]
    
    fig = go.Figure()

    # 1. 绘制 K 线 (Candlestick)
    fig.add_trace(go.Candlestick(
        x=plot_df.index,
        open=plot_df['Open'], high=plot_df['High'],
        low=plot_df['Low'], close=plot_df['Close'],
        name='K线',
        increasing_line_color='#26a69a', decreasing_line_color='#ef5350'
    ))

    # 2. 绘制 NX 短期通道 (蓝色线)
    fig.add_trace(go.Scatter(x=plot_df.index, y=plot_df['nx_up1'], mode='lines', line=dict(color='rgba(41, 98, 255, 0.5)', width=1), name='NX短通道上沿'))
    fig.add_trace(go.Scatter(x=plot_df.index, y=plot_df['nx_dw1'], mode='lines', line=dict(color='rgba(41, 98, 255, 0.5)', width=1), name='NX短通道下沿'))

    # 3. 标记 INST 信号 (绿色三角形)
    inst_signals = plot_df[plot_df['inst_buy'] > 0.5]
    if not inst_signals.empty:
        fig.add_trace(go.Scatter(
            x=inst_signals.index, 
            y=inst_signals['Low'] * 0.98, # 画在K线下方一点点
            mode='markers',
            marker=dict(symbol='triangle-up', size=10, color='#00e676'),
            name='INST吸筹'
        ))

    # 4. 标记 CD 信号 (红色圆点)
    cd_signals = plot_df[plot_df['cd_potential']]
    if not cd_signals.empty:
        fig.add_trace(go.Scatter(
            x=cd_signals.index, 
            y=cd_signals['Low'] * 0.96, # 画在更下方
            mode='markers',
            marker=dict(symbol='circle', size=8, color='red'),
            name='CD背离'
        ))

    # 图表美化配置
    fig.update_layout(
        title=f"{ticker} - 交互式图表 (三位一体)",
        xaxis_rangeslider_visible=False, # 隐藏底部滑块
        height=500,
        template="plotly_dark", # 使用深色主题
        margin=dict(l=20, r=20, t=40, b=20)
    )
    
    return fig

# ==============================================================================
# 主界面逻辑 (UI)
# ==============================================================================
st.title("🛰️ Trinity Pro: 三位一体量化雷达 V2.0")
st.markdown("---")

# 侧边栏配置
st.sidebar.header("配置选项")
scan_mode = st.sidebar.selectbox("选择扫描板块", ["CUSTOM (定制科技/核能)", "NAS100 (纳指100)", "SP500 (标普500)"])
period = st.sidebar.selectbox("数据回溯时间", ["2y", "5y"], index=0)

mode_map = {"CUSTOM (定制科技/核能)": "CUSTOM", "NAS100 (纳指100)": "NAS100", "SP500 (标普500)": "SP500"}
current_mode = mode_map[scan_mode]

if st.button("🚀 启动全市场扫描", type="primary"):
    tickers = get_stock_list(current_mode)
    st.info(f"正在深度扫描 {len(tickers)} 只标的，请耐心等待...")
    
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    results = []
    
    for i, ticker in enumerate(tickers):
        progress = (i + 1) / len(tickers)
        progress_bar.progress(progress)
        status_text.text(f"正在分析: {ticker} ...")
        
        try:
            stock = yf.Ticker(ticker)
            df = stock.history(period=period, interval="1d", auto_adjust=True)
            if df.empty or len(df) < 200: continue
            
            df = calculate_trinity_indicators(df)
            curr = df.iloc[-1]
            
            # === 筛选逻辑 ===
            recent_accumulation = df['inst_buy'].iloc[-90:].max() > 0.5
            recent_trend_days = df['nx_rising'].iloc[-12:]
            is_trend_up_now = curr['nx_rising']
            trend_just_started = is_trend_up_now and (not recent_trend_days.all())
            has_momentum_signal = df['cd_potential'].iloc[-10:].any() or df['macd_gold_cross'].iloc[-5:].any()
            
            if recent_accumulation and trend_just_started and has_momentum_signal:
                score = 0
                if df['cd_potential'].iloc[-5:].any(): score += 2
                if curr['inst_buy'] > 0.5: score += 1
                
                # 存入数据，用于后续展示
                results.append({
                    "Ticker": ticker,
                    "Price": curr['Close'],
                    "Score": score,
                    "Msg": "双底雏形" + (" + CD背离 (强)" if score >=2 else ""),
                    "Data": df # 把数据也存下来，方便画图
                })
                
        except Exception:
            continue

    progress_bar.empty()
    status_text.empty()
    
    if results:
        st.success(f"扫描完成！共发现 {len(results)} 个高潜机会")
        st.markdown("### 🎯 筛选结果 (点击下方条目查看图表)")
        
        # 遍历结果，生成可展开的图表
        for res in results:
            # 使用 expander 创建折叠面板
            with st.expander(f"📊 {res['Ticker']} - ${res['Price']:.2f} | {res['Msg']}"):
                # 1. 显示基础信息
                st.write(f"**信号评分**: {'⭐' * (res['Score'] + 1)}")
                # 2. 调用画图函数
                chart = create_chart(res['Data'], res['Ticker'])
                st.plotly_chart(chart, use_container_width=True)
    else:
        st.warning("本次扫描未发现符合条件的标的，建议切换板块或稍后再试。")