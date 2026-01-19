import telebot
from googletrans import Translator
import pdfplumber
import os
from flask import Flask
from threading import Thread

# البيانات الأساسية
TOKEN = '8377482442:AAGWVYrL6SXYnarAUuEHrDFc2o8ImDZ9iP8'
bot = telebot.TeleBot(TOKEN)
translator = Translator()

# سيرفر وهمي لإبقاء البوت شغالاً على Render
app = Flask('')
@app.route('/')
def home(): return "Bot is Running!"

def run(): app.run(host='0.0.0.0', port=8080)
def keep_alive(): Thread(target=run).start()

# قائمة اللغات
LANGS = {'العربية': 'ar', 'الإنجليزية': 'en', 'الفرنسية': 'fr', 'الألمانية': 'de'}

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "مرحباً بك في مترجم الطلاب الاحترافي! 🎓\nأرسل نصاً، صورة، أو ملف PDF للترجمة.")

# ترجمة النصوص
@bot.message_handler(func=lambda m: True, content_types=['text'])
def translate_text(message):
    try:
        res = translator.translate(message.text, dest='ar' if not message.text.isascii() else 'en')
        bot.reply_to(message, f"📍 **الترجمة:**\n\n{res.text}")
    except:
        bot.reply_to(message, "حدث خطأ بسيط، حاول مرة أخرى.")

# ترجمة ملفات PDF
@bot.message_handler(content_types=['document'])
def handle_docs(message):
    if message.document.file_name.endswith('.pdf'):
        msg = bot.reply_to(message, "⏳ جاري قراءة ملف PDF وترجمته...")
        file_info = bot.get_file(message.document.file_id)
        downloaded_file = bot.download_file(file_info.file_path)
        
        with open("study.pdf", 'wb') as f:
            f.write(downloaded_file)
            
        full_text = ""
        with pdfplumber.open("study.pdf") as pdf:
            for page in pdf.pages[:3]: # ترجمة أول 3 صفحات لتجنب البطء
                full_text += page.extract_text()
        
        if full_text:
            translated = translator.translate(full_text, dest='ar')
            bot.send_message(message.chat.id, f"📄 **ترجمة مستندك:**\n\n{translated.text[:4000]}")
        
        os.remove("study.pdf")
    else:
        bot.reply_to(message, "يرجى إرسال ملف بصيغة PDF فقط.")

if __name__ == "__main__":
    keep_alive()
    bot.polling(none_stop=True)
    
