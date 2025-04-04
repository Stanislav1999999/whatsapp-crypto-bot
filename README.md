# WhatsApp Crypto Signal Bot (Flask + Twilio + CoinPayments)

from flask import Flask, request
from twilio.twiml.messaging_response import MessagingResponse
from twilio.rest import Client
import requests
import json
import datetime
import os

app = Flask(__name__)

# === CONFIG ===
COINPAYMENTS_API_KEY = 'your_coinpayments_api_key'
COINPAYMENTS_API_SECRET = 'your_coinpayments_api_secret'
YOUR_USDT_WALLET = 'your_usdt_wallet_address'
SUBSCRIPTION_PRICE_USD = 100

# Twilio config (from environment)
TWILIO_SID = os.environ.get("TWILIO_SID")
TWILIO_AUTH_TOKEN = os.environ.get("TWILIO_AUTH_TOKEN")
TWILIO_WHATSAPP_NUMBER = "whatsapp:+14155238886"
client = Client(TWILIO_SID, TWILIO_AUTH_TOKEN)

# Подписчики { phone_number: expiration_date }
subscribers = {
    "+380991112233": datetime.datetime(2025, 5, 1),
    "+380981234567": datetime.datetime(2025, 4, 20),
}

# === Проверка оплаты через CoinPayments ===
def check_payment(phone):
    # Здесь ты можешь подключить реальный запрос к CoinPayments API
    # В демо мы просто всегда возвращаем True для примера
    return True

# === Рассылка сигнала всем активным ===
def send_broadcast(message):
    now = datetime.datetime.now()
    for number, expiry in subscribers.items():
        if expiry > now:
            client.messages.create(
                from_=TWILIO_WHATSAPP_NUMBER,
                to=f"whatsapp:{number}",
                body=message
            )

# === Обработка входящих сообщений WhatsApp ===
@app.route("/bot", methods=['POST'])
def bot():
    incoming_msg = request.values.get('Body', '').strip().lower()
    from_number = request.values.get('From', '').replace('whatsapp:', '')

    resp = MessagingResponse()
    msg = resp.message()

    now = datetime.datetime.now()
    is_subscribed = from_number in subscribers and subscribers[from_number] > now

    if incoming_msg == 'старт':
        if is_subscribed:
            msg.body("✅ У тебя уже активна подписка! Ожидай сигналы в ближайшее время.")
        else:
            msg.body(
                f"👋 Привет! Чтобы получать сигналы 2 раза в день, оплати {SUBSCRIPTION_PRICE_USD}$ в USDT.\n"
                f"💸 Кошелек для оплаты: {YOUR_USDT_WALLET}\n\n"
                "После оплаты напиши *Готово* для активации подписки."
            )

    elif incoming_msg == 'готово':
        if check_payment(from_number):
            expiration = now + datetime.timedelta(days=30)
            subscribers[from_number] = expiration
            msg.body("🎉 Подписка активирована! Ожидай сигналы утром и вечером каждый день.")
        else:
            msg.body("⏳ Платеж не найден. Пожалуйста, проверь транзакцию и попробуй позже.")

    elif incoming_msg == 'сигнал':
        if is_subscribed:
            msg.body("🚀 Сигнал на сегодня:\n📌 BTC/USDT Лонг от $84 390\n🎯 Цель: $86 500\n🛑 Стоп: $83 200")
        else:
            msg.body("🔒 Подписка не активна. Напиши *Старт*, чтобы получить инструкцию по оплате.")

    elif incoming_msg == 'рассылка' and from_number == 'твой_номер_админа':
        send_broadcast("🚀 Сигнал: BTC/USDT лонг от $84 390 | Цель $86 500 | Стоп $83 200")
        msg.body("✅ Рассылка отправлена всем активным подписчикам.")

    else:
        msg.body("👋 Привет! Напиши *Старт* для начала или *Сигнал*, чтобы получить сигнал (если есть подписка).")

    return str(resp)

if __name__ == '__main__':
    app.run(debug=True)
