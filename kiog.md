import os

import time

import sqlite3

from fastapi import FastAPI, Request

from telebot import TeleBot, types



\# ⚙️ মূল কনফিগারেশন ও টোকেন

BOT\_TOKEN = "AAG6XLGne9xnFxR7o3kaInjAVcEpF996zmE

ADMIN\_ID = 123456789  # এখানে আপনার নিজের ব্যক্তিগত টেলিগ্রাম আইডি নম্বর বসাবেন

bot = TeleBot(BOT\_TOKEN)

app = FastAPI()



\# 🗄️ ফ্রি ডাটাবেজ সেটআপ (SQLite)

conn = sqlite3.connect("database.db", check\_same\_thread=False)

cursor = conn.cursor()

cursor.execute("""

CREATE TABLE IF NOT EXISTS workers (

&#x20;   id TEXT PRIMARY KEY, 

&#x20;   balance REAL, 

&#x20;   last\_active TEXT, 

&#x20;   inactive\_days INTEGER, 

&#x20;   is\_blocked INTEGER DEFAULT 0

)

""")

cursor.execute("CREATE TABLE IF NOT EXISTS tasks (id TEXT PRIMARY KEY, title TEXT, link TEXT, reward REAL)")

cursor.execute("CREATE TABLE IF NOT EXISTS withdrawals (id INTEGER PRIMARY KEY AUTOINCREMENT, user\_id TEXT, number TEXT, method TEXT, amount REAL, status TEXT DEFAULT 'PENDING')")

conn.commit()



\# 🛡️ সিকিউরিটি চেক মিডলওয়্যার

def check\_user(message):

&#x20;   user\_id = str(message.from\_user.id)

&#x20;   cursor.execute("SELECT is\_blocked FROM workers WHERE id=?", (user\_id,))

&#x20;   res = cursor.fetchone()

&#x20;   if res and res\[0] == 1:

&#x20;       bot.send\_message(message.chat.id, "🚨 আপনি পরপর ২ দিন কোনো কাজ করেননি। মাত্র ৮০ মিনিটের সহজ কাজ না করার কারণে আপনার অ্যাকাউন্টটি ব্লক করা হয়েছে! আনব্লক করতে অ্যাডমিনের সাথে যোগাযোগ করুন।")

&#x20;       return False

&#x20;   return True



\# 👋 স্টার্ট মেনু ও নোটিশ

@bot.message\_handler(commands=\['start'])

def start\_cmd(message):

&#x20;   if not check\_user(message): return

&#x20;   user\_id = str(message.from\_user.id)

&#x20;   cursor.execute("INSERT OR IGNORE INTO workers (id, balance, last\_active, inactive\_days, is\_blocked) VALUES (?, 0.0, ?, 0, 0)", (user\_id, time.strftime("%Y-%m-%d")))

&#x20;   conn.commit()

&#x20;   

&#x20;   markup = types.ReplyKeyboardMarkup(resize\_keyboard=True)

&#x20;   markup.add("🎯 Available Tasks", "💸 Withdraw / টাকা তুলুন")

&#x20;   markup.add("💰 Balance Check")

&#x20;   

&#x20;   bot.send\_message(message.chat.id, "👋 স্বাগতম! আমাদের বটে কাজ করে দৈনিক ইনকাম করুন।\\n\\n📌 \*\*নিয়ম:\*\* পরপর ২ দিন কাজ না করলে আইডি ব্লক করা হবে।", reply\_markup=markup)



\# 🎯 কাজের তালিকা শো করা

@bot.message\_handler(func=lambda msg: msg.text == "🎯 Available Tasks")

def show\_tasks(msg):

&#x20;   if not check\_user(msg): return

&#x20;   cursor.execute("SELECT id, title, reward FROM tasks")

&#x20;   tasks = cursor.fetchall()

&#x20;   if not tasks:

&#x20;       bot.send\_message(msg.chat.id, "এখন কোনো কাজ উপলব্ধ নেই। দয়া করে পরে চেষ্টা করুন।")

&#x20;       return

&#x20;       

&#x20;   text = "🎯 \*\*চলতি কাজের তালিকা:\*\*\\nনিচের যেকোনো বাটনে ক্লিক করে কাজ সম্পন্ন করুন:\\n"

&#x20;   markup = types.InlineKeyboardMarkup()

&#x20;   for t in tasks:

&#x20;       text += f"\\n🔹 {t\[1]} - রেট: {t\[2]} টাকা"

&#x20;       cpa\_link = f"{t\[2]}?subid={msg.from\_user.id}"

&#x20;       markup.add(types.InlineKeyboardButton(text=f"👉 {t\[1]}", url=cpa\_link))

&#x20;       

&#x20;   bot.send\_message(msg.chat.id, text, reply\_markup=markup)



\# 💰 ব্যালেন্স চেক

@bot.message\_handler(func=lambda msg: msg.text == "💰 Balance Check")

def check\_bal(msg):

&#x20;   if not check\_user(msg): return

&#x20;   user\_id = str(msg.from\_user.id)

&#x20;   cursor.execute("SELECT balance FROM workers WHERE id=?", (user\_id,))

&#x20;   bal = cursor.fetchone()\[0]

&#x20;   bot.send\_message(msg.chat.id, f"💰 আপনার বর্তমান ব্যালেন্স: {bal} টাকা।")



\# 💸 উইথড্র উইন্ডো নোটিশ (মাসে ২ বার পেমেন্ট)

@bot.message\_handler(func=lambda msg: msg.text == "💸 Withdraw / টাকা তুলুন")

def withdraw\_cmd(msg):

&#x20;   if not check\_user(msg): return

&#x20;   text = (

&#x20;       "💸 \*\*টাকা তোলার নিয়ম ও নোটিশ\*\*\\n\\n"

&#x20;       "📌 \*\*জরুরি নোটিশ:\*\* আমাদের বটে প্রতিদিন পেমেন্ট দেওয়া হয় না। মাসে মোট ২ বার সবার পেমেন্ট ক্লিয়ার করা হয়।\\n\\n"

&#x20;       "📅 \*\*১ম পেমেন্ট:\*\* মাসের ২ তারিখের মধ্যে সবার বিকাশ/নগদে টাকা পৌঁছে যাবে।\\n"

&#x20;       "📅 \*\*২য় পেমেন্ট:\*\* মাসের ১৭ তারিখের মধ্যে সবার বিকাশ/নগদে টাকা পৌঁছে যাবে।\\n\\n"

&#x20;       "🔹 \*\*সর্বনিম্ন উইথড্র:\*\* ৫০ টাকা\\n"

&#x20;       "🔹 \*\*পেমেন্টের মাধ্যম:\*\* বিকাশ / নগদ\\n\\n"

&#x20;       "টাকা তুলতে চাইলে নিচের ফরম্যাটে মেসেজ লিখুন:\\n`/req \[বিকাশ/নগদ] | \[আপনার নম্বর] | \[টাকার পরিমাণ]`"

&#x20;   )

&#x20;   bot.send\_message(msg.chat.id, text, parse\_mode="Markdown")



\# 📥 কর্মী দ্বারা উইথড্র রিকোয়েস্ট সাবমিট

@bot.message\_handler(commands=\['req'])

def process\_withdraw(msg):

&#x20;   if not check\_user(msg): return

&#x20;   try:

&#x20;       parts = msg.text.split('/req ')\[1].split('|')

&#x20;       method = parts\[0].strip()

&#x20;       number = parts\[1].strip()

&#x20;       amount = float(parts\[2].strip())

&#x20;       

&#x20;       user\_id = str(msg.from\_user.id)

&#x20;       cursor.execute("SELECT balance FROM workers WHERE id=?", (user\_id,))

&#x20;       bal = cursor.fetchone()\[0]

&#x20;       

&#x20;       if amount < 50:

&#x20;           bot.send\_message(msg.chat.id, "❌ সর্বনিম্ন উইথড্র ৫০ টাকা।")

&#x20;           return

&#x20;       if bal < amount:

&#x20;           bot.send\_message(msg.chat.id, "❌ আপনার অ্যাকাউন্টে পর্যাপ্ত ব্যালেন্স নেই।")

&#x20;           return

&#x20;           

&#x20;       cursor.execute("UPDATE workers SET balance = balance - ? WHERE id=?", (amount, user\_id))

&#x20;       cursor.execute("INSERT INTO withdrawals (user\_id, number, method, amount) VALUES (?, ?, ?, ?)", (user\_id, number, method, amount))

&#x20;       conn.commit()

&#x20;       bot.send\_message(msg.chat.id, "✅ আপনার উইথড্র রিকোয়েস্ট সফলভাবে জমা হয়েছে। আগামী ২ বা ১৭ তারিখের মধ্যে পেমেন্ট পেয়ে যাবেন।")

&#x20;   except:

&#x20;       bot.send\_message(msg.chat.id, "❌ ভুল ফরম্যাট! দয়া করে সঠিক নিয়ম মেনে আবার লিখুন।")



\# 👑 ৩. সুরক্ষিত ইন-চ্যাট অ্যাডমিন প্যানেল কমান্ড

@bot.message\_handler(commands=\['addtask'])

def admin\_add\_task(msg):

&#x20;   if msg.from\_user.id != ADMIN\_ID: return

&#x20;   try:

&#x20;       parts = msg.text.split('/addtask ')\[1].split('|')

&#x20;       t\_id = parts\[0].strip()

&#x20;       title = parts\[1].strip()

&#x20;       link = parts\[2].strip()

&#x20;       reward = float(parts\[3].strip())

&#x20;       

&#x20;       cursor.execute("INSERT OR REPLACE INTO tasks (id, title, link, reward) VALUES (?, ?, ?, ?)", (t\_id, title, link, reward))

&#x20;       conn.commit()

&#x20;       bot.send\_message(ADMIN\_ID, f"✅ কাজ সফলভাবে যুক্ত হয়েছে! ID: {t\_id}")

&#x20;   except:

&#x20;       bot.send\_message(ADMIN\_ID, "❌ ফরম্যাট ভুল! উদাহরণ: `/addtask 101 | নগদ ইনস্টল | লিংক | 30`")



@bot.message\_handler(commands=\['deletetask'])

def admin\_del\_task(msg):

&#x20;   if msg.from\_user.id != ADMIN\_ID: return

&#x20;   t\_id = msg.text.split('/deletetask ')\[1].strip()

&#x20;   cursor.execute("DELETE FROM tasks WHERE id=?", (t\_id,))

&#x20;   conn.commit()

&#x20;   bot.send\_message(ADMIN\_ID, f"❌ কাজ ডিলিট করা হয়েছে! ID: {t\_id}")



@bot.message\_handler(commands=\['withdrawals'])

def view\_withdrawals(msg):

&#x20;   if msg.from\_user.id != ADMIN\_ID: return

&#x20;   cursor.execute("SELECT id, user\_id, number, method, amount FROM withdrawals WHERE status='PENDING'")

&#x20;   rows = cursor.fetchall()

&#x20;   if not rows:

&#x20;       bot.send\_message(ADMIN\_ID, "পেন্ডিং কোনো উইথড্র রিকোয়েস্ট নেই।")

&#x20;       return

&#x20;   for r in rows:

&#x20;       text = f"📥 \*\*পেন্ডিং উইথড্র:\*\*\\nID: {r\[0]}\\nইউজার ID: {r\[1]}\\nমাধ্যম: {r\[3]}\\nনম্বর: `{r\[2]}`\\nপরিমাণ: {r\[4]} টাকা"

&#x20;       markup = types.InlineKeyboardMarkup()

&#x20;       markup.add(types.InlineKeyboardButton("✅ Confirm Payout", callback\_data=f"pay\_{r\[0]}\_{r\[1]}\_{r\[4]}"))

&#x20;       bot.send\_message(ADMIN\_ID, text, reply\_markup=markup, parse\_mode="Markdown")



@bot.callback\_query\_handler(func=lambda call: call.data.startswith('pay\_'))

def approve\_payment(call):

&#x20;   if call.from\_user.id != ADMIN\_ID: return

&#x20;   \_, req\_id, user\_id, amount = call.data.split('\_')

&#x20;   cursor.execute("UPDATE withdrawals SET status='SUCCESS' WHERE id=?", (req\_id,))

&#x20;   conn.commit()

&#x20;   

&#x20;   bot.edit\_message\_text(chat\_id=ADMIN\_ID, message\_id=call.message.message\_id, text=f"✅ পেমেন্ট সফলভাবে রিলিজ করা হয়েছে! পরিমাণ: {amount} টাকা।")

&#x20;   try:

&#x20;       bot.send\_message(user\_id, f"🎉 অভিনন্দন! আপনার উইথড্র সফল হয়েছে এবং {amount} টাকা আপনার বিকাশ/নগদে পাঠিয়ে দেওয়া হয়েছে।")

&#x20;   except:

&#x20;       pass



\# 📡 2. CPALead Postback রিসিভ করার অটোমেটিক মেকানিজম (Webhook)

@app.get("/postback")

async def cpa\_postback(request: Request):

&#x20;   params = request.query\_params

&#x20;   user\_id = params.get("subid")

&#x20;   

&#x20;   if user\_id:

&#x20;       # ৮০ সেন্টের কাজে আপনি পাচ্ছেন প্রায় ৯৬ টাকা, কর্মীকে অটোমেটিক ৩০ টাকা ডিরেক্ট ওয়ালেটে অ্যাড করা হবে

&#x20;       cursor.execute("UPDATE workers SET balance = balance + 30.0, last\_active = ? WHERE id = ?", (time.strftime("%Y-%m-%d"), user\_id))

&#x20;       conn.commit()

&#x20;       try:

&#x20;           bot.send\_message(user\_id, "🎉 অভিনন্দন! আপনার একটি কাজ সফল হয়েছে এবং ৩০ টাকা আপনার ব্যালেন্সে যোগ করা হয়েছে।")

&#x20;       except:

&#x20;           pass

&#x20;       return {"status": "success"}

&#x20;   return {"status": "failed"}



