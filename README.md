import tkinter as tk
from tkinter import messagebox, filedialog, ttk
import yfinance as yf
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime, timedelta
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Global variable to hold stock data
current_data = None
current_symbol = None


def validate_date(date_str):
    """Validate date format YYYY-MM-DD"""
    try:
        datetime.strptime(date_str, '%Y-%m-%d')
        return True
    except ValueError:
        return False


def validate_symbol(symbol):
    """Basic symbol validation"""
    if not symbol or len(symbol) < 1 or len(symbol) > 10:
        return False
    return symbol.replace('.', '').replace('-', '').isalnum()


def fetch_stock():
    """Fetch stock data with comprehensive error handling"""
    global current_data, current_symbol

    symbol = entry_symbol.get().strip().upper()
    start = entry_start.get().strip()
    end = entry_end.get().strip()

    # Input validation
    if not symbol:
        messagebox.showwarning("Missing Input", "Please enter a stock symbol.")
        return

    if not validate_symbol(symbol):
        messagebox.showwarning("Invalid Symbol", "Please enter a valid stock symbol (1-10 characters).")
        return

    if not start or not end:
        messagebox.showwarning("Missing Dates", "Please enter both start and end dates.")
        return

    if not validate_date(start) or not validate_date(end):
        messagebox.showwarning("Invalid Date", "Please enter dates in YYYY-MM-DD format.")
        return

    start_date = datetime.strptime(start, '%Y-%m-%d')
    end_date = datetime.strptime(end, '%Y-%m-%d')

    if start_date >= end_date:
        messagebox.showwarning("Invalid Date Range", "Start date must be before end date.")
        return

    if end_date > datetime.now():
        messagebox.showwarning("Future Date", "End date cannot be in the future.")
        return

    # Show loading message
    progress_window = tk.Toplevel(root)
    progress_window.title("Loading...")
    progress_window.geometry("250x100")
    progress_window.transient(root)
    progress_window.grab_set()

    tk.Label(progress_window, text=f"Fetching data for {symbol}...").pack(pady=20)
    progress_bar = ttk.Progressbar(progress_window, mode='indeterminate')
    progress_bar.pack(pady=10)
    progress_bar.start()

    root.update()

    try:
        # Fetch data
        ticker = yf.Ticker(symbol)
        df = ticker.history(start=start, end=end, auto_adjust=True)

        if df.empty:
            progress_window.destroy()
            messagebox.showerror("No Data", f"No data found for '{symbol}'. Please check the symbol and dates.")
            return

        current_data = df
        current_symbol = symbol

        # Update status
        status_label.config(text=f"✅ Data loaded: {symbol} ({len(df)} days)")

        progress_window.destroy()
        messagebox.showinfo("Success", f"Data for {symbol} loaded successfully.\nRecords: {len(df)}")

        # Enable buttons
        for button in [export_btn, plot_btn, analyze_btn]:
            button.config(state='normal')

        logger.info(f"Successfully fetched data for {symbol}")

    except Exception as e:
        progress_window.destroy()
        error_msg = f"Error fetching data for {symbol}: {str(e)}"
        messagebox.showerror("Error", error_msg)
        logger.error(error_msg)


def export_data():
    """Export data to CSV with error handling"""
    if current_data is None:
        messagebox.showwarning("No Data", "Please fetch stock data first.")
        return

    try:
        file_path = filedialog.asksaveasfilename(
            defaultextension=".csv",
            filetypes=[("CSV files", "*.csv"), ("All files", "*.*")],
            title="Save Stock Data"
        )

        if file_path:
            # Add additional calculated columns
            export_df = current_data.copy()
            export_df['Daily_Change_%'] = export_df['Close'].pct_change() * 100
            export_df['MA7'] = export_df['Close'].rolling(window=7).mean()
            export_df['MA30'] = export_df['Close'].rolling(window=30).mean()

            export_df.to_csv(file_path)
            messagebox.showinfo("Export Successful", f"Data exported to:\n{file_path}")
            logger.info(f"Data exported to {file_path}")

    except Exception as e:
        messagebox.showerror("Export Error", f"Failed to export data: {str(e)}")
        logger.error(f"Export failed: {str(e)}")


def plot_chart():
    """Plot comprehensive stock chart"""
    if current_data is None:
        messagebox.showwarning("No Data", "Please fetch stock data first.")
        return

    try:
        fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(12, 10))

        # Price chart with moving averages
        ax1.plot(current_data.index, current_data['Close'], label='Close Price', color='blue', linewidth=2)

        # Calculate and plot moving averages
        ma7 = current_data['Close'].rolling(window=7).mean()
        ma30 = current_data['Close'].rolling(window=30).mean()

        ax1.plot(current_data.index, ma7, label='7-Day MA', linestyle='--', color='orange', alpha=0.8)
        ax1.plot(current_data.index, ma30, label='30-Day MA', linestyle='--', color='green', alpha=0.8)

        ax1.set_title(f'{current_symbol} Stock Price Analysis', fontsize=16, fontweight='bold')
        ax1.set_ylabel('Price (USD)', fontsize=12)
        ax1.legend()
        ax1.grid(True, alpha=0.3)

        # Volume chart
        ax2.bar(current_data.index, current_data['Volume'], alpha=0.7, color='gray')
        ax2.set_title('Trading Volume', fontsize=14)
        ax2.set_xlabel('Date', fontsize=12)
        ax2.set_ylabel('Volume', fontsize=12)
        ax2.grid(True, alpha=0.3)

        plt.tight_layout()
        plt.show()

        logger.info(f"Chart displayed for {current_symbol}")

    except Exception as e:
        messagebox.showerror("Plot Error", f"Failed to create chart: {str(e)}")
        logger.error(f"Plot failed: {str(e)}")


def analyze_trends():
    """Enhanced trend analysis with multiple indicators"""
    if current_data is None:
        messagebox.showwarning("No Data", "Please fetch stock data first.")
        return

    try:
        data = current_data.copy()
        data['Daily_Change_%'] = data['Close'].pct_change() * 100

        # Basic statistics
        best_day = data['Daily_Change_%'].idxmax()
        worst_day = data['Daily_Change_%'].idxmin()
        best_change = data.loc[best_day, 'Daily_Change_%']
        worst_change = data.loc[worst_day, 'Daily_Change_%']

        # Moving averages
        ma7 = data['Close'].rolling(window=7).mean()
        ma30 = data['Close'].rolling(window=30).mean()
        current_price = data['Close'].iloc[-1]

        # Volatility (standard deviation of daily changes)
        volatility = data['Daily_Change_%'].std()

        # Price change over period
        start_price = data['Close'].iloc[0]
        total_return = ((current_price - start_price) / start_price) * 100

        # Trend determination
        recent_ma7 = ma7.iloc[-1]
        recent_ma30 = ma30.iloc[-1]

        if recent_ma7 > recent_ma30 and current_price > recent_ma7:
            trend = "📈 Strong Uptrend - Consider Buying"
            trend_color = "🟢"
        elif recent_ma7 < recent_ma30 and current_price < recent_ma7:
            trend = "📉 Strong Downtrend - Consider Selling"
            trend_color = "🔴"
        elif current_price > recent_ma7:
            trend = "📊 Mild Uptrend - Monitor Closely"
            trend_color = "🟡"
        else:
            trend = "📊 Sideways/Unclear - Hold Position"
            trend_color = "🟡"

        # Risk assessment
        if volatility > 3:
            risk = "⚠️ High Risk (High Volatility)"
        elif volatility > 1.5:
            risk = "⚡ Medium Risk"
        else:
            risk = "✅ Low Risk (Low Volatility)"

        message = f"""
📊 STOCK ANALYSIS REPORT - {current_symbol}
{'=' * 50}

📈 PERFORMANCE METRICS:
• Total Return: {total_return:.2f}%
• Current Price: ${current_price:.2f}
• Volatility: {volatility:.2f}%

📅 EXTREME DAYS:
• Best Day: {best_day.strftime('%Y-%m-%d')} ({best_change:.2f}%)
• Worst Day: {worst_day.strftime('%Y-%m-%d')} ({worst_change:.2f}%)

🔍 TREND ANALYSIS:
• 7-Day MA: ${recent_ma7:.2f}
• 30-Day MA: ${recent_ma30:.2f}
• Trend: {trend}

⚠️ RISK ASSESSMENT:
• {risk}

{trend_color} RECOMMENDATION: {trend.split(' - ')[1] if ' - ' in trend else 'Monitor closely'}
        """

        # Create custom dialog for better formatting
        analysis_window = tk.Toplevel(root)
        analysis_window.title(f"Trend Analysis - {current_symbol}")
        analysis_window.geometry("600x500")
        analysis_window.transient(root)

        text_widget = tk.Text(analysis_window, wrap=tk.WORD, font=('Consolas', 10))
        scrollbar = tk.Scrollbar(analysis_window, orient=tk.VERTICAL, command=text_widget.yview)
        text_widget.configure(yscrollcommand=scrollbar.set)

        text_widget.insert(tk.END, message)
        text_widget.config(state=tk.DISABLED)

        text_widget.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=10, pady=10)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

        logger.info(f"Trend analysis completed for {current_symbol}")

    except Exception as e:
        messagebox.showerror("Analysis Error", f"Failed to analyze trends: {str(e)}")
        logger.error(f"Analysis failed: {str(e)}")


def reset_form():
    """Reset all form fields and data"""
    global current_data, current_symbol
    current_data = None
    current_symbol = None

    entry_symbol.delete(0, tk.END)
    entry_symbol.insert(0, "AAPL")

    # Set default dates (last year)
    end_date = datetime.now()
    start_date = end_date - timedelta(days=365)

    entry_start.delete(0, tk.END)
    entry_start.insert(0, start_date.strftime('%Y-%m-%d'))

    entry_end.delete(0, tk.END)
    entry_end.insert(0, end_date.strftime('%Y-%m-%d'))

    status_label.config(text="Ready - Enter stock symbol and dates")

    # Disable buttons
    for button in [export_btn, plot_btn, analyze_btn]:
        button.config(state='disabled')


# GUI setup
root = tk.Tk()
root.title("📊 Advanced Stock Analyzer")
root.geometry("500x600")
root.resizable(True, True)

# Create main frame
main_frame = ttk.Frame(root, padding="10")
main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))

# Configure grid weights
root.columnconfigure(0, weight=1)
root.rowconfigure(0, weight=1)
main_frame.columnconfigure(1, weight=1)

# Title
title_label = tk.Label(main_frame, text="📊 Advanced Stock Analyzer",
                       font=('Arial', 16, 'bold'), fg='#2E8B57')
title_label.grid(row=0, column=0, columnspan=2, pady=(0, 20))

# Input fields with better layout
ttk.Label(main_frame, text="Stock Symbol (e.g., AAPL, MSFT):").grid(row=1, column=0, sticky=tk.W, pady=5)
entry_symbol = ttk.Entry(main_frame, width=20)
entry_symbol.grid(row=1, column=1, sticky=(tk.W, tk.E), pady=5)
entry_symbol.insert(0, "AAPL")

ttk.Label(main_frame, text="Start Date (YYYY-MM-DD):").grid(row=2, column=0, sticky=tk.W, pady=5)
entry_start = ttk.Entry(main_frame, width=20)
entry_start.grid(row=2, column=1, sticky=(tk.W, tk.E), pady=5)
# Default to 1 year ago
default_start = (datetime.now() - timedelta(days=365)).strftime('%Y-%m-%d')
entry_start.insert(0, default_start)

ttk.Label(main_frame, text="End Date (YYYY-MM-DD):").grid(row=3, column=0, sticky=tk.W, pady=5)
entry_end = ttk.Entry(main_frame, width=20)
entry_end.grid(row=3, column=1, sticky=(tk.W, tk.E), pady=5)
entry_end.insert(0, datetime.now().strftime('%Y-%m-%d'))

# Buttons frame
button_frame = ttk.Frame(main_frame)
button_frame.grid(row=4, column=0, columnspan=2, pady=20)

# Primary action button
fetch_btn = ttk.Button(button_frame, text="📥 Fetch Stock Data", command=fetch_stock)
fetch_btn.pack(pady=5, fill=tk.X)

# Secondary action buttons (initially disabled)
export_btn = ttk.Button(button_frame, text="📤 Export to CSV", command=export_data, state='disabled')
export_btn.pack(pady=5, fill=tk.X)

plot_btn = ttk.Button(button_frame, text="📈 Plot Chart", command=plot_chart, state='disabled')
plot_btn.pack(pady=5, fill=tk.X)

analyze_btn = ttk.Button(button_frame, text="🔍 Analyze Trends", command=analyze_trends, state='disabled')
analyze_btn.pack(pady=5, fill=tk.X)

# Reset button
reset_btn = ttk.Button(button_frame, text="🔄 Reset", command=reset_form)
reset_btn.pack(pady=5, fill=tk.X)

# Status label
status_label = ttk.Label(main_frame, text="Ready - Enter stock symbol and dates",
                         font=('Arial', 10), foreground='#666')
status_label.grid(row=5, column=0, columnspan=2, pady=20)

# Help text
help_text = """
📋 Instructions:
1. Enter a valid stock symbol (e.g., AAPL, GOOGL, MSFT)
2. Set your desired date range
3. Click 'Fetch Stock Data' to download data
4. Use other buttons to analyze and export data

💡 Tips:
• Use recent dates for better data availability
• Popular symbols: AAPL, MSFT, GOOGL, AMZN, TSLA
• Data source: Yahoo Finance
"""

help_label = tk.Label(main_frame, text=help_text, font=('Arial', 9),
                      justify=tk.LEFT, fg='#666', wraplength=400)
help_label.grid(row=6, column=0, columnspan=2, pady=10)

# Start GUI loop
if __name__ == "__main__":
    root.mainloop()
