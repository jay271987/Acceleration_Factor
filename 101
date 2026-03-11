import pandas as pd
import yfinance as yf
import time
import logging
from datetime import datetime
from pytz import timezone
from tenacity import retry, stop_after_attempt, wait_exponential
from rich.console import Console
from rich.table import Table
from rich.live import Live

# ----------------------------------
# Configuration
# ----------------------------------
TICKERS = list(set([
    'IREDA.NS', 'LXCHEM.NS', 
    'PIDILITIND.NS', 'YASHO.NS', 'BALAMINES.NS', 'CLEAN.NS', 'DEEPAKNTR.NS'
]))

REFRESH_MINUTES = 3 
DISPLAY_LIMIT = 25  # Limits the table size for better visibility
ist = timezone('Asia/Kolkata')

# ----------------------------------
# Logging Configuration
# ----------------------------------
logging.basicConfig(
    level=logging.INFO, 
    format='%(asctime)s - %(levelname)s - %(message)s', 
    filename='tracker.log'
)

# ----------------------------------
# Helper Functions
# ----------------------------------
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1))
def get_fresh_data(tickers):
    """Fetch recent intraday data with multi-threading enabled."""
    return yf.download(
        tickers, 
        period="2d", 
        interval="5m", 
        prepost=False, 
        progress=False, 
        group_by='ticker', 
        threads=True
    )

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1))
def get_52w_highs(tickers):
    logging.info("Fetching 52-week highs...")
    data = yf.download(tickers, period="1y", interval="1d", progress=False, group_by='ticker', threads=True)
    highs = {}
    for ticker in tickers:
        try:
            ticker_data = data[ticker].dropna(how='all') if isinstance(data.columns, pd.MultiIndex) else data.dropna(how='all')
            if not ticker_data.empty and 'High' in ticker_data.columns:
                highs[ticker] = ticker_data['High'].max()
        except Exception:
            highs[ticker] = float('nan')
    return highs

def calculate_acceleration_factor(data):
    """Calculate the AC using a 5-period SMA of the Midpoint."""
    df = data.copy()
    df['Midpoint'] = (df['High'] + df['Low']) / 2
    df['SMA_5'] = df['Midpoint'].rolling(window=5).mean()
    df['AC'] = df['SMA_5'].diff()
    return df

# ----------------------------------
# Terminal UI and Main Loop
# ----------------------------------
def generate_table(results, last_update_time):
    table = Table(
        title=f"Stock Tracker (Top {DISPLAY_LIMIT} by AC)\nLast Updated: {last_update_time}", 
        title_style="bold blue",
        header_style="bold white on blue"
    )
    table.add_column("Script", justify="left", style="cyan", no_wrap=True)
    table.add_column("LTP", justify="right", style="green")
    table.add_column("52W High", justify="right", style="yellow")
    table.add_column("Accel Factor", justify="right", style="magenta")

    for r in results[:DISPLAY_LIMIT]:
        h52 = f"{r['52 Week High']:.2f}" if pd.notna(r['52 Week High']) else "N/A"
        table.add_row(
            r['Script'],
            f"{r['Last Traded Price']:.2f}",
            h52,
            f"{r['Acceleration Factor']:.4f}"
        )
    return table

def main():
    console = Console()
    console.print("Initializing Stock Tracker...", style="bold green")
    
    highs_52w = get_52w_highs(TICKERS)
    last_update = "Fetching initial data..."
    
    with Live(generate_table([], last_update), refresh_per_second=1) as live:
        while True:
            try:
                logging.info("Refreshing market data...")
                data = get_fresh_data(TICKERS)
                
                if data.empty:
                    last_update = f"No data at {datetime.now(ist).strftime('%H:%M:%S')}"
                    time.sleep(60)
                    continue

                results = []
                for ticker in TICKERS:
                    try:
                        ticker_data = data[ticker].dropna(how='all') if isinstance(data.columns, pd.MultiIndex) else data.dropna(how='all')
                        
                        if len(ticker_data) < 6:
                            continue

                        current_price = ticker_data['Close'].iloc[-1]
                        ticker_data = calculate_acceleration_factor(ticker_data)
                        
                        ac_value = ticker_data['AC'].iloc[-1]
                        if pd.isna(ac_value):
                            continue

                        results.append({
                            'Script': ticker,
                            'Last Traded Price': current_price,
                            '52 Week High': highs_52w.get(ticker, float('nan')),
                            'Acceleration Factor': ac_value
                        })

                    except Exception as e:
                        logging.error(f"Error processing {ticker}: {e}")

                if results:
                    # Sorting by Acceleration Factor (Descending)
                    results = sorted(results, key=lambda x: x['Acceleration Factor'], reverse=True)
                    last_update = datetime.now(ist).strftime('%Y-%m-%d %H:%M:%S')
                    live.update(generate_table(results, last_update))
                break  # we just want to test if it runs

            except Exception as e:
                logging.error(f"Main loop error: {e}")
                break

if __name__ == "__main__":
    main()
