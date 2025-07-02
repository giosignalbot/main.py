import requests
import time

# === ТВОИ ДАННЫЕ ===
BOT_TOKEN = "7525927665:AAG8EGCrCLL9XM8RufU31XJE3Doae3nihrM"
CHAT_ID = "7435840461"
API_KEY = "79f06126420f4b00b5bc32adf3305fe1"

# === ВАЛЮТНЫЕ ПАРЫ ===
PAIRS = ["EUR/USD", "GBP/JPY", "AUD/USD", "AUD/CAD"]

# === EMA ===
def ema(data, period):
    k = 2 / (period + 1)
    ema_vals = [sum(data[:period]) / period]
    for price in data[period:]:
        ema_vals.append(price * k + ema_vals[-1] * (1 - k))
    return ema_vals

# === MACD ===
def macd(data):
    ema12 = ema(data, 12)
    ema26 = ema(data, 26)
    macd_line = [a - b for a, b in zip(ema12[-len(ema26):], ema26)]
    signal_line = ema(macd_line, 9)
    return macd_line, signal_line

# === ПОЛУЧЕНИЕ ЦЕН ===
def get_prices(symbol):
    url = f"https://api.twelvedata.com/time_series?symbol={symbol}&interval=1min&outputsize=100&apikey={API_KEY}"
    try:
        r = requests.get(url)
        values = r.json().get("values")
        return [float(v["close"]) for v in reversed(values)] if values else []
    except:
        return []

# === ОТПРАВКА СИГНАЛА ===
def send_signal(pair, direction):
    message = f"📢 Сигнал: {direction}\nВалютная пара: {pair}"
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    data = {"chat_id": CHAT_ID, "text": message}
    requests.post(url, data=data)

# === ГЛАВНЫЙ ЦИКЛ ===
def run_bot():
    last_signal = {}
    while True:
        for pair in PAIRS:
            prices = get_prices(pair)
            if len(prices) < 30:
                continue

            macd_line, signal_line = macd(prices)
            if len(macd_line) < 2 or len(signal_line) < 2:
                continue

            direction = None
            if macd_line[-2] < signal_line[-2] and macd_line[-1] > signal_line[-1]:
                direction = "Buy"
            elif macd_line[-2] > signal_line[-2] and macd_line[-1] < signal_line[-1]:
                direction = "Sell"

            if direction and last_signal.get(pair) != direction:
                send_signal(pair, direction)
                last_signal[pair] = direction

        time.sleep(60)  # проверка каждую минуту

run_bot()
