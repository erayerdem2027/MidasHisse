import io
import asyncio
import yfinance as yf
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import os
from typing import Final
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes

# TOKEN bilgisini Render'daki "Environment Variables" kısmından alacağız
TOKEN: Final = os.getenv('BOT_TOKEN')

tickers = ["SN", "AAPL", "LEU", "HOOD", "TM", "ELF", "NOC", "AMZN"]
benchmarks = ["QQQ", "SPY"]
all_symbols = tickers + benchmarks
weights = {
    "AAPL": 0.15, "AMZN": 0.15, "NOC": 0.15,
    "TM": 0.125, "ELF": 0.125, "SN": 0.10,
    "LEU": 0.10, "HOOD": 0.10
}

def perform_analysis():
    try:
        # Veri Çekme (Daha stabil olması için son 1 yılı çekiyoruz)
        data = yf.download(all_symbols, period="1y")['Close']
        data['HYBRID_50_50'] = (data['QQQ'] + data['SPY']) / 2

        # Analiz başlangıç tarihini dinamik yapabilir veya sabit tutabilirsiniz
        df_filtered = data[data.index >= '2025-01-01'].copy()
        
        if df_filtered.empty:
            return None, "Yeterli veri bulunamadı."

        base_date = df_filtered.index[0]
        df_norm = (df_filtered / df_filtered.loc[base_date]) * 100
        df_norm['MY_PORTFOLIO'] = sum(df_norm[ticker] * weight for ticker, weight in weights.items())
        df_norm['RELATIVE_PERF'] = (df_norm['MY_PORTFOLIO'] / df_norm['HYBRID_50_50']) - 1

        # Sharpe Ratio
        returns = data.tail(252).pct_change().dropna()
        returns['MY_PORTFOLIO'] = sum(returns[ticker] * weight for ticker, weight in weights.items())
        returns['HYBRID_50_50'] = (returns['QQQ'] + returns['SPY']) / 2

        daily_rf = (1 + 0.04)**(1/252) - 1
        sharpe_dict = {col: ((returns[col] - daily_rf).mean() / returns[col].std()) * np.sqrt(252)
                       for col in list(all_symbols) + ['HYBRID_50_50', 'MY_PORTFOLIO']}
        df_sharpe = pd.DataFrame.from_dict(sharpe_dict, orient='index', columns=['SR']).sort_values(by='SR', ascending=False)

        plots = []
        # Grafik 1: Rölatif Performans
        plt.figure(figsize=(10, 5))
        colors = ['green' if x >= 0 else 'red' for x in df_norm['RELATIVE_PERF']]
        plt.bar(df_norm.index, df_norm['RELATIVE_PERF'] * 100, color=colors, alpha=0.7)
        plt.title(f"Portföy Rölatif Performansı (Baz: {base_date.date()})")
        buf1 = io.BytesIO(); plt.savefig(buf1, format='png'); buf1.seek(0); plots.append(buf1); plt.close()

        # Grafik 2: Sharpe Oranları
        plt.figure(figsize=(10, 5))
        plt.bar(df_sharpe.index, df_sharpe['SR'], color=['#2c3e50' if x == 'MY_PORTFOLIO' else '#3498db' for x in df_sharpe.index])
        plt.title("Sharpe Oranları (Risk Ayarlı Getiri)")
        plt.xticks(rotation=45)
        buf2 = io.BytesIO(); plt.savefig(buf2, format='png'); buf2.seek(0); plots.append(buf2); plt.close()

        return plots, "Analiz raporu başarıyla hazırlandı."
    except Exception as e:
        return None, f"Hata oluştu: {str(e)}"

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Finans Analiz Botu Aktif! /analiz komutu ile rapor alabilirsiniz.")

async def send_analysis(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Veriler analiz ediliyor, lütfen bekleyin...")
    plots, msg = await asyncio.to_thread(perform_analysis)

    if plots:
        for plot in plots:
            await context.bot.send_photo(chat_id=update.effective_chat.id, photo=plot)
        await update.message.reply_text(msg)
    else:
        await update.message.reply_text(msg)

async def main():
    if not TOKEN:
        print("HATA: BOT_TOKEN bulunamadı!")
        return

    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("analiz", send_analysis))

    print("Bot başlatılıyor...")
    await application.initialize()
    await application.start()
    await application.updater.start_polling()
    
    # Render'da botun kapanmaması için sonsuz döngü
    while True:
        await asyncio.sleep(3600)

if __name__ == '__main__':
    asyncio.run(main())