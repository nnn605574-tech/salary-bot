import os
import logging
import sqlite3
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes

# التوكن من BotFather
BOT_TOKEN = os.getenv('BOT_TOKEN')

# إعداد التسجيل
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

class SalaryBot:
    def __init__(self):
        self.setup_database()
    
    def setup_database(self):
        self.conn = sqlite3.connect('salary_bot.db', check_same_thread=False)
        self.conn.execute('''
            CREATE TABLE IF NOT EXISTS employees (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                phone TEXT UNIQUE NOT NULL,
                wallet TEXT,
                salary_balance REAL DEFAULT 0,
                position TEXT DEFAULT 'موظف'
            )
        ''')
        self.conn.commit()
        logging.info("✅ قاعدة البيانات جاهزة")
    
    async def start(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        user = update.effective_user
        await update.message.reply_text(
            f"🎯 مرحباً {user.first_name}!\n\n"
            "🤖 بوت إدارة الرواتب\n\n"
            "📋 الأوامر المتاحة:\n"
            "/add اسم هاتف محفظة راتب - إضافة موظف\n"
            "/list - عرض الموظفين\n"
            "/balance - عرض الرواتب\n"
            "/update هاتف مبلغ - تحديث راتب\n"
            "/help - المساعدة"
        )
    
    async def add_employee(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if len(context.args) < 4:
            await update.message.reply_text(
                "❌ طريقة الاستخدام:\n"
                "/add الاسم الهاتف المحفظة الراتب\n\n"
                "مثال:\n"
                "/add محمد 123456 wallet123 1500"
            )
            return
        
        try:
            name = context.args[0]
            phone = context.args[1]
            wallet = context.args[2]
            salary = float(context.args[3])
            position = context.args[4] if len(context.args) > 4 else "موظف"
            
            self.conn.execute(
                "INSERT INTO employees (name, phone, wallet, salary_balance, position) VALUES (?, ?, ?, ?, ?)",
                (name, phone, wallet, salary, position)
            )
            self.conn.commit()
            
            await update.message.reply_text(
                f"✅ تم إضافة الموظف بنجاح!\n\n"
                f"👤 الاسم: {name}\n"
                f"📞 الهاتف: {phone}\n"
                f"💳 المحفظة: {wallet}\n"
                f"💰 الراتب: {salary}\n"
                f"🎯 المنصب: {position}"
            )
            
        except sqlite3.IntegrityError:
            await update.message.reply_text("❌ رقم الهاتف مسجل مسبقاً!")
        except ValueError:
            await update.message.reply_text("❌ الراتب يجب أن يكون رقماً!")
    
    async def list_employees(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        employees = self.conn.execute(
            "SELECT name, phone, wallet, salary_balance, position FROM employees"
        ).fetchall()
        
        if not employees:
            await update.message.reply_text("📭 لا يوجد موظفين")
            return
        
        text = "📋 قائمة الموظفين:\n\n"
        for emp in employees:
            text += f"👤 {emp[0]}\n"
            text += f"📞 {emp[1]} | 💰 {emp[3]}\n"
            text += f"💳 {emp[2]} | 🎯 {emp[4]}\n"
            text += "─" * 20 + "\n"
        
        await update.message.reply_text(text)
    
    async def balance(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        employees = self.conn.execute(
            "SELECT name, salary_balance FROM employees"
        ).fetchall()
        
        if not employees:
            await update.message.reply_text("📭 لا يوجد موظفين")
            return
        
        text = "💰 رواتب الموظفين:\n\n"
        total = 0
        for emp in employees:
            text += f"👤 {emp[0]}\n💵 {emp[1]} دولار\n\n"
            total += emp[1]
        
        text += f"📊 الإجمالي: {total} دولار"
        await update.message.reply_text(text)
    
    async def update_salary(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        if len(context.args) < 2:
            await update.message.reply_text(
                "❌ طريقة الاستخدام:\n"
                "/update الهاتف المبلغ\n\n"
                "مثال:\n"
                "/update 123456 500\n"
                "/update 123456 -200"
            )
            return
        
        try:
            phone = context.args[0]
            amount = float(context.args[1])
            
            self.conn.execute(
                "UPDATE employees SET salary_balance = salary_balance + ? WHERE phone = ?",
                (amount, phone)
            )
            self.conn.commit()
            
            employee = self.conn.execute(
                "SELECT name FROM employees WHERE phone = ?", (phone,)
            ).fetchone()
            
            if employee:
                await update.message.reply_text(
                    f"✅ تم تحديث الراتب!\n\n"
                    f"👤 الموظف: {employee[0]}\n"
                    f"📞 الهاتف: {phone}\n"
                    f"💵 المبلغ: {amount} دولار"
                )
            else:
                await update.message.reply_text("❌ رقم الهاتف غير موجود")
                
        except ValueError:
            await update.message.reply_text("❌ المبلغ يجب أن يكون رقماً!")

def main():
    """تشغيل البوت"""
    if not BOT_TOKEN:
        logging.error("❌ لم يتم تعيين BOT_TOKEN")
        return
    
    application = Application.builder().token(BOT_TOKEN).build()
    bot = SalaryBot()
    
    # إضافة الأوامر
    application.add_handler(CommandHandler("start", bot.start))
    application.add_handler(CommandHandler("add", bot.add_employee))
    application.add_handler(CommandHandler("list", bot.list_employees))
    application.add_handler(CommandHandler("balance", bot.balance))
    application.add_handler(CommandHandler("update", bot.update_salary))
    
    logging.info("🚀 البوت يعمل...")
    application.run_polling()

if __name__ == "__main__":
    main()
