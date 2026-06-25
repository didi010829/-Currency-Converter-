# -Currency-Converter
import tkinter as tk
from tkinter import ttk, messagebox
import requests
import json
import os
from datetime import datetime

#  Конфигурация 
API_KEY = "didi010829"
BASE_URL = "https://v6.exchangerate-api.com/v6"
HISTORY_FILE = "history.json"

# Загрузка/сохранение истории 
def load_history():
    if os.path.exists(HISTORY_FILE):
        with open(HISTORY_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return []

def save_history(history):
    with open(HISTORY_FILE, "w", encoding="utf-8") as f:
        json.dump(history, f, ensure_ascii=False, indent=2)

#  Получение курса 
def get_exchange_rate(from_currency, to_currency):
    """Возвращает курс из from_currency в to_currency или None при ошибке."""
    url = f"{BASE_URL}/{API_KEY}/pair/{from_currency}/{to_currency}"
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        if data["result"] == "success":
            return data["conversion_rate"]
        else:
            return None
    except Exception as e:
        messagebox.showerror("Ошибка", f"Не удалось получить курс: {e}")
        return None
# Основной класс приложения 
class CurrencyConverterApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Конвертер валют")
        self.root.geometry("500x500")
        self.root.resizable(False, False)

        # Данные о валютах (популярные)
        self.currencies = ["USD", "EUR", "RUB", "GBP", "JPY", "CNY", "CAD", "AUD", "CHF"]
        self.history = load_history()

        # Переменные
        self.from_currency = tk.StringVar(value="USD")
        self.to_currency = tk.StringVar(value="EUR")
        self.amount = tk.StringVar()
        self.result_var = tk.StringVar(value="")

        #  Интерфейс 
        # Фрейм для ввода
        input_frame = ttk.LabelFrame(root, text="Конвертация", padding=10)
        input_frame.pack(fill="x", padx=10, pady=10)

        # Выбор валюты "из"
        ttk.Label(input_frame, text="Из:").grid(row=0, column=0, sticky="w", padx=5)
        from_combo = ttk.Combobox(input_frame, textvariable=self.from_currency,
                                  values=self.currencies, state="readonly", width=10)
        from_combo.grid(row=0, column=1, padx=5, pady=5)

        # Выбор валюты "в"
        ttk.Label(input_frame, text="В:").grid(row=0, column=2, sticky="w", padx=5)
        to_combo = ttk.Combobox(input_frame, textvariable=self.to_currency,
                                values=self.currencies, state="readonly", width=10)
        to_combo.grid(row=0, column=3, padx=5, pady=5)

        # Поле ввода суммы
        ttk.Label(input_frame, text="Сумма:").grid(row=1, column=0, sticky="w", padx=5)
        amount_entry = ttk.Entry(input_frame, textvariable=self.amount, width=15)
        amount_entry.grid(row=1, column=1, padx=5, pady=5)

        # Кнопка конвертации
        convert_btn = ttk.Button(input_frame, text="Конвертировать", command=self.convert)
        convert_btn.grid(row=1, column=2, padx=10, pady=5)

        # Метка результата
        self.result_label = ttk.Label(input_frame, textvariable=self.result_var,
                                      font=("Arial", 12, "bold"), foreground="blue")
        self.result_label.grid(row=1, column=3, padx=10)

        # Таблица истории 
        history_frame = ttk.LabelFrame(root, text="История конвертаций", padding=10)
        history_frame.pack(fill="both", expand=True, padx=10, pady=10)

        columns = ("Дата", "Сумма", "Из", "В", "Результат")
        self.tree = ttk.Treeview(history_frame, columns=columns, show="headings", height=12)
        for col in columns:
            self.tree.heading(col, text=col)
            self.tree.column(col, width=90, anchor="center")

        scrollbar = ttk.Scrollbar(history_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Кнопка очистки истории
        clear_btn = ttk.Button(root, text="Очистить историю", command=self.clear_history)
        clear_btn.pack(pady=5)

        # Загрузка истории при запуске
        self.refresh_history()

        # Привязка Enter к полю ввода
        amount_entry.bind("<Return>", lambda e: self.convert())

    def convert(self):
        """Основная логика конвертации."""
        # Проверка ввода
        amount_str = self.amount.get().strip()
        if not amount_str:
            messagebox.showwarning("Предупреждение", "Введите сумму.")
            return
        try:
            amount = float(amount_str)
            if amount <= 0:
                raise ValueError("Сумма должна быть положительной")
        except ValueError:
            messagebox.showerror("Ошибка", "Сумма должна быть положительным числом.")
            return

        from_cur = self.from_currency.get()
        to_cur = self.to_currency.get()

        if from_cur == to_cur:
            result = amount
            rate = 1.0
        else:
            rate = get_exchange_rate(from_cur, to_cur)
            if rate is None:
                return
            result = amount * rate

        # Отображение результата
        self.result_var.set(f"{result:.2f} {to_cur}")

        # Сохранение в истории
        entry = {
            "date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "amount": amount,
            "from": from_cur,
            "to": to_cur,
            "result": round(result, 2),
            "rate": rate if from_cur != to_cur else 1.0
        }
        self.history.insert(0, entry)  # новые записи сверху
        if len(self.history) > 100:    # ограничим историю 100 записями
            self.history.pop()
        save_history(self.history)
        self.refresh_history()

    def refresh_history(self):
        """Обновляет таблицу истории."""
        for row in self.tree.get_children():
            self.tree.delete(row)
        for entry in self.history:
            self.tree.insert("", "end", values=(
                entry["date"],
                entry["amount"],
                entry["from"],
                entry["to"],
                f"{entry['result']} {entry['to']}"
            ))

    def clear_history(self):
        """Очищает историю."""
        if messagebox.askyesno("Подтверждение", "Удалить всю историю?"):
            self.history.clear()
            save_history(self.history)
            self.refresh_history()

#  Запуск 
if __name__ == "__main__":
    root = tk.Tk()
    app = CurrencyConverterApp(root)
    root.mainloop()
