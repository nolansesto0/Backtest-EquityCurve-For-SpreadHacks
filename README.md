# Backtest-EquityCurve-For-SpreadHacks
Uses a CSV file created from 5 minute data of QQQ from June 2023-May 2024 to backtest my spread hacking strategy with the parameters I have created. Then plots the % made on an equity curve.
----------------------------------------
import pandas as pd
import matplotlib.pyplot as plt

# Load the CSV file with the correct column name for timestamps
data = pd.read_csv('QQQ_5min_data_April1_April6.csv', parse_dates=['timestamp'])

# Function to calculate MACD
def calculate_macd(df):
    df['EMA_12'] = df['close'].ewm(span=12, adjust=False).mean()
    df['EMA_26'] = df['close'].ewm(span=26, adjust=False).mean()
    df['MACD'] = df['EMA_12'] - df['EMA_26']
    df['Signal_Line'] = df['MACD'].ewm(span=9, adjust=False).mean()
    df['MACD_Histogram'] = df['MACD'] - df['Signal_Line']
    return df

# Function to calculate RSI
def calculate_rsi(df, period=14):
    delta = df['close'].diff(1)
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

# Apply the MACD calculation
data = calculate_macd(data)

# Resample to 30-minute intervals to calculate RSI
data_30min = data.resample('30min', on='timestamp').agg({
    'open': 'first',
    'high': 'max',
    'low': 'min',
    'close': 'last',
    'volume': 'sum'
}).dropna()

# Apply the RSI calculation on the 30-minute resampled data
data_30min['RSI'] = calculate_rsi(data_30min)

# Merge the 30-minute RSI back to the original 5-minute data
data['timestamp_floor'] = data['timestamp'].dt.floor('30min')
data = data.merge(data_30min[['RSI']], left_on='timestamp_floor', right_index=True, how='left')
data.drop(columns=['timestamp_floor'], inplace=True)

# Function to check for three consecutive higher highs and higher lows
def check_consecutive_days(daily_data):
    if len(daily_data) < 3:
        return False
    return (daily_data.iloc[-3]['high'] < daily_data.iloc[-2]['high'] < daily_data.iloc[-1]['high'] and
            daily_data.iloc[-3]['low'] < daily_data.iloc[-2]['low'] < daily_data.iloc[-1]['low'])

# Function to process each day's data
def process_day(day_data, account_balance, wait_for_lower_open, reference_open):
    trade_entries = []
    result = None
    entry_range = 0.05  # Define the range within which MACD and Signal Line are considered close

    for i in range(1, len(day_data)):
        current_row = day_data.iloc[i]

        # Skip trading if RSI is too high or if waiting for a lower open
        if current_row['RSI'] > 50 or wait_for_lower_open:
            continue

        # Check for MACD and Signal Line being within a certain range below the zero line
        if (current_row['MACD'] < 0 and current_row['Signal_Line'] < 0 and
            abs(current_row['MACD'] - current_row['Signal_Line']) <= entry_range):

            entry_price = current_row['close']
            entry_time = current_row['timestamp']

            # Track prices after entry
            after_entry_data = day_data[day_data['timestamp'] > entry_time]

            # Determine win or loss
            min_price = after_entry_data['low'].min()
            if min_price <= entry_price - 3.5:
                account_balance *= 0.87  # Loss day
                result = 'Loss'
            else:
                account_balance *= 1.0412  # Win day
                result = 'Win'

            trade_entries.append(entry_time)
            break

    return account_balance, trade_entries, result

# Initial account balance
initial_balance = 5000
account_balance = initial_balance
equity_curve = []
dates = []
trade_entries_log = []
results_log = []

# Track daily highs and lows
daily_data = data.resample('D', on='timestamp').agg({
    'open': 'first',
    'high': 'max',
    'low': 'min',
    'close': 'last',
    'volume': 'sum'
}).dropna()

# Process each day
wait_for_lower_open = False
reference_open = None
consecutive_highs_lows_counter = 0

for date in pd.date_range(start='2023-06-01', end='2024-05-24'):
    day_data = data[data['timestamp'].dt.date == date.date()]
    if not day_data.empty:
        if wait_for_lower_open:
            if day_data.iloc[0]['open'] < reference_open - 5:
                wait_for_lower_open = False  # Condition met, resume trading
        else:
            if check_consecutive_days(daily_data):
                consecutive_highs_lows_counter += 1
                if consecutive_highs_lows_counter >= 3:
                    wait_for_lower_open = True
                    reference_open = day_data.iloc[0]['open']
                    consecutive_highs_lows_counter = 0
            else:
                consecutive_highs_lows_counter = 0

            account_balance, trade_entries, result = process_day(day_data, account_balance, wait_for_lower_open, reference_open)
            equity_curve.append((account_balance / initial_balance - 1) * 100)  # Calculate percent change
            dates.append(date)
            trade_entries_log.extend(trade_entries)
            results_log.append((date, result if result else 'None'))

# Create a DataFrame for the equity curve
equity_df = pd.DataFrame({'Date': dates, 'Percent Increase': equity_curve})

# Calculate maximum drawdown
equity_df['Drawdown'] = equity_df['Percent Increase'].cummax() - equity_df['Percent Increase']
max_drawdown = equity_df['Drawdown'].max()

# Function to calculate maximum advantage/gain
def calculate_max_gain(df):
    df['Max_Gain'] = df['Percent Increase'] - df['Percent Increase'].cummin()
    return df

# Calculate maximum advantage/gain
equity_df = calculate_max_gain(equity_df)
max_gain = equity_df['Max_Gain'].max()

# Plot the equity curve
plt.figure(figsize=(10, 6))
plt.plot(equity_df['Date'], equity_df['Percent Increase'], marker='o')
plt.title('Equity Curve (Percent Increase)')
plt.xlabel('Date')
plt.ylabel('Percent Increase')
plt.grid(True)
plt.show()

# Save the equity curve to a CSV file
equity_df.to_csv('Equity_Curve_April1_April6.csv', index=False)

# Print maximum drawdown and maximum gain
print(f'Maximum Drawdown: {max_drawdown:.2f}%')
print(f'Maximum Gain: {max_gain:.2f}%')

# Calculate and print the difference and ratio
difference = max_gain - max_drawdown
ratio = max_gain / max_drawdown if max_drawdown != 0 else float('inf')
print(f'Difference between Max Gain and Max Drawdown: {difference:.2f}%')
print(f'Ratio of Max Gain to Max Drawdown: {ratio:.2f}')

# Print trade entries and results
for date, result in results_log:
    print(f'{date.date()}: {result}')

for entry in trade_entries_log:
    print(f'Trade entry at: {entry}')

print(equity_df)
