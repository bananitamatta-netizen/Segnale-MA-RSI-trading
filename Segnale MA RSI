# -*- coding: utf-8 -*-
"""
Script eseguito automaticamente da GitHub Actions (gratis, nel cloud,
nessun PC/VPS necessario). Controlla i simboli indicati, calcola il
segnale MA+RSI usando dati gratuiti di Yahoo Finance, e manda una
notifica Telegram SOLO quando cambia il segnale (evita spam ripetuto).

Non richiede MT5: i prezzi vengono da Yahoo Finance (fonte pubblica
gratuita), molto vicini al prezzo del broker ma non identici al 100%.
"""

import os
import json
import requests
import pandas as pd
import yfinance as yf

# ============== CONFIGURAZIONE ==============
# Mappa "nome che vuoi vedere nei messaggi" -> "ticker Yahoo Finance"
SIMBOLI = {
    "EURUSD": "EURUSD=X",
    "GBPUSD": "GBPUSD=X",
    "ORO (XAUUSD)": "GC=F",
}

FAST_MA_PERIOD = 10
SLOW_MA_PERIOD = 50
RSI_PERIOD = 14
RSI_OVERBOUGHT = 70.0
RSI_OVERSOLD = 30.0

ATR_PERIOD = 14
SL_ATR_MULTIPLIER = 1.5
TP_ATR_MULTIPLIER = 3.0

STATE_FILE = "state.json"
# ==============================================

TELEGRAM_TOKEN = os.environ["TELEGRAM_TOKEN"]
CHAT_ID = os.environ["CHAT_ID"]


def calcola_rsi(chiusure, periodo):
    delta = chiusure.diff()
    guadagni = delta.clip(lower=0)
    perdite = -delta.clip(upper=0)
    media_guadagni = guadagni.rolling(periodo).mean()
    media_perdite = perdite.rolling(periodo).mean()
    rs = media_guadagni / media_perdite
    return 100 - (100 / (1 + rs))


def calcola_atr(alti, bassi, chiusure, periodo):
    chiusura_prec = chiusure.shift(1)
    tr1 = alti - bassi
    tr2 = (alti - chiusura_prec).abs()
    tr3 = (bassi - chiusura_prec).abs()
    true_range = pd.concat([tr1, tr2, tr3], axis=1).max(axis=1)
    return true_range.rolling(periodo).mean()


def carica_stato():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            return json.load(f)
    return {}


def salva_stato(stato):
    with open(STATE_FILE, "w") as f:
        json.dump(stato, f, indent=2)


def manda_messaggio(testo):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    resp = requests.post(url, data={"chat_id": CHAT_ID, "text": testo})
    if resp.status_code != 200:
        print(f"Errore invio Telegram: {resp.text}")


def controlla_simbolo(nome, ticker):
    df = yf.download(ticker, period="5d", interval="1h", progress=False)
    if df.empty or len(df) < SLOW_MA_PERIOD + RSI_PERIOD:
        print(f"{nome}: dati insufficienti, salto.")
        return None

    chiusure = df["Close"]
    alti = df["High"]
    bassi = df["Low"]
    if hasattr(chiusure, "squeeze"):
        chiusure = chiusure.squeeze()
        alti = alti.squeeze()
        bassi = bassi.squeeze()

    fast_ma = chiusure.ewm(span=FAST_MA_PERIOD, adjust=False).mean()
    slow_ma = chiusure.ewm(span=SLOW_MA_PERIOD, adjust=False).mean()
    rsi = calcola_rsi(chiusure, RSI_PERIOD)
    atr = calcola_atr(alti, bassi, chiusure, ATR_PERIOD)

    f_ultima, f_prec = float(fast_ma.iloc[-1]), float(fast_ma.iloc[-2])
    s_ultima, s_prec = float(slow_ma.iloc[-1]), float(slow_ma.iloc[-2])
    rsi_ultima = float(rsi.iloc[-1])
    atr_ultimo = float(atr.iloc[-1])
    prezzo = float(chiusure.iloc[-1])

    cross_up = f_prec <= s_prec and f_ultima > s_ultima
    cross_down = f_prec >= s_prec and f_ultima < s_ultima

    rsi_ok_buy = rsi_ultima < RSI_OVERBOUGHT
    rsi_ok_sell = rsi_ultima > RSI_OVERSOLD

    sl, tp = None, None

    if cross_up and rsi_ok_buy:
        tipo = "BUY"
        sl = prezzo - atr_ultimo * SL_ATR_MULTIPLIER
        tp = prezzo + atr_ultimo * TP_ATR_MULTIPLIER
    elif cross_down and rsi_ok_sell:
        tipo = "SELL"
        sl = prezzo + atr_ultimo * SL_ATR_MULTIPLIER
        tp = prezzo - atr_ultimo * TP_ATR_MULTIPLIER
    else:
        tipo = "ASPETTA"

    return tipo, prezzo, rsi_ultima, sl, tp


def main():
    stato = carica_stato()

    for nome, ticker in SIMBOLI.items():
        risultato = controlla_simbolo(nome, ticker)
        if risultato is None:
            continue

        tipo, prezzo, rsi_val, sl, tp = risultato
        precedente = stato.get(nome)

        if tipo in ("BUY", "SELL") and tipo != precedente:
            emoji = "🟢" if tipo == "BUY" else "🔴"
            azione = "COMPRA" if tipo == "BUY" else "VENDI"
            testo = (
                f"{emoji} Nuovo segnale: {nome}\n"
                f"Azione: {azione}\n"
                f"Prezzo: {prezzo:.5f}\n"
                f"RSI: {rsi_val:.2f}\n"
                f"Stop Loss suggerito: {sl:.5f}\n"
                f"Take Profit suggerito: {tp:.5f}\n\n"
                f"⚠️ Analisi tecnica automatica, non è consiglio finanziario."
            )
            manda_messaggio(testo)
            print(f"Notifica inviata per {nome}: {tipo}")
        else:
            print(f"{nome}: segnale = {tipo} (nessuna notifica, invariato o in attesa)")

        stato[nome] = tipo

    salva_stato(stato)


if __name__ == "__main__":
    main()
