# conversion.py
# Currency Converter (Конвертер валют)

**Автор:** Никифорова Диана

**Вариант:** Currency Converter GUI

**Дата сдачи:** 25.06.2026

## Описание программы

**Currency Converter** — это desktop-приложение с графическим интерфейсом для конвертации валют с использованием актуальных курсов от [exchangerate-api.com](https://www.exchangerate-api.com). Программа позволяет:

- Выбирать исходную и целевую валюту из списка.
- Вводить сумму для конвертации (только положительные числа).
- Получать результат конвертации в реальном времени.
- Просматривать историю всех выполненных конвертаций в таблице.
- Сохранять историю в JSON-файл автоматически при каждой операции.
- Загружать историю из JSON-файла при запуске (файл `history.json`).
- Очищать историю по желанию.

## Требования для запуска

- Python 3.8 или выше.
- Установленная библиотека `requests` (для HTTP-запросов к API).
- Интернет-соединение для получения курсов валют.

## Установка зависимостей

```bash
pip install requests
# Модель одной операции конвертации

from datetime import datetime

class Conversion:
    """
    Класс Conversion - модель конвертации валюты.
    Поля:
    - from_currency: исходная валюта (строка)
    - to_currency: целевая валюта (строка)
    - amount: сумма в исходной валюте (float)
    - result: сумма в целевой валюте (float)
    - rate: использованный курс (float)
    - timestamp: дата и время операции (строка)
    """

    def __init__(self, from_currency, to_currency, amount, result, rate):
        """
        Конструктор операции.
        """
        self._from_currency = from_currency
        self._to_currency = to_currency
        self._amount = amount
        self._result = result
        self._rate = rate
        self._timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # Геттеры (сеттеры не нужны, т.к. данные неизменяемы)
    @property
    def from_currency(self):
        return self._from_currency

    @property
    def to_currency(self):
        return self._to_currency

    @property
    def amount(self):
        return self._amount

    @property
    def result(self):
        return self._result

    @property
    def rate(self):
        return self._rate

    @property
    def timestamp(self):
        return self._timestamp

    def to_dict(self):
        """
        Преобразование в словарь для сохранения в JSON.
        """
        return {
            'from_currency': self._from_currency,
            'to_currency': self._to_currency,
            'amount': self._amount,
            'result': self._result,
            'rate': self._rate,
            'timestamp': self._timestamp
        }

    @classmethod
    def from_dict(cls, data):
        """
        Создание объекта из словаря (для загрузки из JSON).
        """
        conv = cls(
            data['from_currency'],
            data['to_currency'],
            data['amount'],
            data['result'],
            data['rate']
        )
        conv._timestamp = data['timestamp']  # восстанавливаем время
        return conv

    def __str__(self):
        return (f"{self._from_currency} {self._amount:.2f} → "
        # history_manager.py
# Сохранение и загрузка истории в/из JSON

import json
import os
from collections import deque
from conversion import Conversion

class HistoryManager:
    """
    Класс для управления историей конвертаций.
    Хранит записи в deque, поддерживает добавление, получение, фильтрацию,
    сохранение и загрузку из JSON.
    """

    def __init__(self):
        self._entries = deque()  # очередь для хранения записей

    def add_entry(self, conversion):
        """Добавить запись в историю."""
        self._entries.append(conversion)

    def get_all(self):
        """Получить все записи (список)."""
        return list(self._entries)

    def get_last(self, count=1):
        """Получить последние N записей."""
        if count <= 0:
            return []
        entries_list = list(self._entries)
        return entries_list[-count:] if count < len(entries_list) else entries_list

    def clear(self):
        """Очистить историю."""
        self._entries.clear()

    def save_to_file(self, filename='history.json'):
        """Сохранить историю в JSON-файл."""
        data = [entry.to_dict() for entry in self._entries]
        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def load_from_file(self, filename='history.json'):
        """
        Загрузить историю из JSON-файла.
        Возвращает True при успехе, False если файл не найден.
        """
        if not os.path.exists(filename):
            return False
        with open(filename, 'r', encoding='utf-8') as f:
            data = json.load(f)
        self._entries.clear()
        for item in data:
            conv = Conversion.from_dict(item)
            self._entries.append(conv)
        return True

    def __len__(self):
        return len(self._entries)

    def __str__(self):
        if not self._entries:
            return "История пуста"
        lines = ["=== ИСТОРИЯ КОНВЕРТАЦИЙ ==="]
        for i, entry in enumerate(self._entries, 1):
            lines.append(f"{i}. {entry}")
        lines.append("=============================")
        return "\n".join(lines)
        # currency_api.py
# Получение курсов валют через exchangerate-api.com

import requests

class CurrencyAPI:
    """
    Класс-обёртка для запросов к exchangerate-api.com.
    Требует API_KEY.
    """
    BASE_URL = "https://v6.exchangerate-api.com/v6"

    def __init__(self, api_key):
        self.api_key = api_key

    def get_exchange_rate(self, from_currency, to_currency):
        """
        Возвращает курс from_currency → to_currency.
        В случае ошибки возвращает None.
        """
        url = f"{self.BASE_URL}/{self.api_key}/pair/{from_currency}/{to_currency}"
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            data = response.json()
            if data.get("result") == "success":
                return data["conversion_rate"]
            else:
                return None
        except Exception as e:
            # В реальном приложении можно логировать ошибку
            return None

    def get_supported_currencies(self):
        """
        Возвращает список поддерживаемых валют (опционально).
        Можно использовать для динамического обновления списка.
        """
        url = f"{self.BASE_URL}/{self.api_key}/codes"
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            data = response.json()
            if data.get("result") == "success":
                return [code for code, _ in data["supported_codes"]]
            else:
                return None
        except Exception:
            return None
            # converter.py
# Логика конвертации, объединяет API и историю

from conversion import Conversion
from history_manager import HistoryManager
from currency_api import CurrencyAPI

class Converter:
    """
    Основной класс приложения.
    Содержит API-клиент и менеджер истории.
    Предоставляет метод для выполнения конвертации.
    """

    def __init__(self, api_key):
        self.api = CurrencyAPI(api_key)
        self.history = HistoryManager()
        # Попытка загрузить историю при инициализации
        self.history.load_from_file('history.json')

    def convert(self, from_currency, to_currency, amount):
        """
        Выполняет конвертацию, сохраняет запись в историю.
        Возвращает объект Conversion или None при ошибке.
        """
        if amount <= 0:
            raise ValueError("Сумма должна быть положительным числом")

        if from_currency == to_currency:
            rate = 1.0
            result = amount
        else:
            rate = self.api.get_exchange_rate(from_currency, to_currency)
            if rate is None:
                return None  # ошибка API
            result = amount * rate

        conversion = Conversion(from_currency, to_currency, amount, result, rate)
        self.history.add_entry(conversion)
        self.history.save_to_file('history.json')  # автосохранение
        return conversion

    def get_history(self):
        """Получить всю историю."""
        return self.history.get_all()

    def clear_history(self):
        """Очистить историю."""
        self.history.clear()
        self.history.save_to_file('history.json')

    def load_history(self, filename='history.json'):
        """Загрузить историю из указанного файла."""
        return self.history.load_from_file(filename)
        # main.py
# GUI-приложение для конвертера валют

import tkinter as tk
from tkinter import ttk, messagebox
from converter import Converter

# Замените на ваш реальный API-ключ
API_KEY = "ваш_ключ_здесь"

class CurrencyConverterApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Конвертер валют")
        self.root.geometry("600x500")
        self.root.resizable(False, False)

        # Инициализация логики
        self.converter = Converter(API_KEY)

        # Список валют (можно дополнить или получить через API)
        self.currencies = ["USD", "EUR", "RUB", "GBP", "JPY", "CNY", "CAD", "AUD", "CHF"]

        # Переменные для привязки к виджетам
        self.from_currency = tk.StringVar(value="USD")
        self.to_currency = tk.StringVar(value="EUR")
        self.amount = tk.StringVar()
        self.result_var = tk.StringVar(value="")

        # Построение интерфейса
        self._create_widgets()
        self._refresh_history_table()

    def _create_widgets(self):
        # ---- Фрейм конвертации ----
        input_frame = ttk.LabelFrame(self.root, text="Конвертация", padding=10)
        input_frame.pack(fill="x", padx=10, pady=10)

        # Из
        ttk.Label(input_frame, text="Из:").grid(row=0, column=0, sticky="w", padx=5)
        from_combo = ttk.Combobox(input_frame, textvariable=self.from_currency,
                                  values=self.currencies, state="readonly", width=10)
        from_combo.grid(row=0, column=1, padx=5, pady=5)

        # В
        ttk.Label(input_frame, text="В:").grid(row=0, column=2, sticky="w", padx=5)
        to_combo = ttk.Combobox(input_frame, textvariable=self.to_currency,
                                values=self.currencies, state="readonly", width=10)
        to_combo.grid(row=0, column=3, padx=5, pady=5)

        # Сумма
        ttk.Label(input_frame, text="Сумма:").grid(row=1, column=0, sticky="w", padx=5)
        amount_entry = ttk.Entry(input_frame, textvariable=self.amount, width=15)
        amount_entry.grid(row=1, column=1, padx=5, pady=5)

        # Кнопка конвертации
        convert_btn = ttk.Button(input_frame, text="Конвертировать", command=self._on_convert)
        convert_btn.grid(row=1, column=2, padx=10, pady=5)

        # Метка результата
        self.result_label = ttk.Label(input_frame, textvariable=self.result_var,
                                      font=("Arial", 12, "bold"), foreground="blue")
        self.result_label.grid(row=1, column=3, padx=10)

        # Привязка Enter к полю суммы
        amount_entry.bind("<Return>", lambda e: self._on_convert())

        # ---- Таблица истории ----
        history_frame = ttk.LabelFrame(self.root, text="История конвертаций", padding=10)
        history_frame.pack(fill="both", expand=True, padx=10, pady=10)

        columns = ("Дата", "Из", "Сумма", "В", "Результат", "Курс")
        self.tree = ttk.Treeview(history_frame, columns=columns, show="headings", height=12)
        for col in columns:
            self.tree.heading(col, text=col)
            self.tree.column(col, width=90, anchor="center")
        # Корректируем ширину под данные
        self.tree.column("Дата", width=140)
        self.tree.column("Курс", width=80)

        scrollbar = ttk.Scrollbar(history_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # ---- Кнопки управления ----
        btn_frame = ttk.Frame(self.root)
        btn_frame.pack(pady=5)

        clear_btn = ttk.Button(btn_frame, text="Очистить историю", command=self._on_clear)
        clear_btn.pack(side="left", padx=5)

        load_btn = ttk.Button(btn_frame, text="Загрузить историю", command=self._on_load)
        load_btn.pack(side="left", padx=5)

    def _on_convert(self):
        """Обработчик нажатия кнопки конвертации."""
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

        try:
            conversion = self.converter.convert(from_cur, to_cur, amount)
            if conversion is None:
                messagebox.showerror("Ошибка", "Не удалось получить курс. Проверьте API-ключ и интернет.")
                return
            # Отображаем результат
            self.result_var.set(f"{conversion.result:.2f} {to_cur}")
            # Обновляем таблицу истории
            self._refresh_history_table()
        except Exception as e:
            messagebox.showerror("Ошибка", str(e))

    def _refresh_history_table(self):
        """Обновить данные в таблице истории."""
        for row in self.tree.get_children():
            self.tree.delete(row)
        for entry in self.converter.get_history():
            self.tree.insert("", "end", values=(
                entry.timestamp,
                entry.from_currency,
                f"{entry.amount:.2f}",
                entry.to_currency,
                f"{entry.result:.2f}",
                f"{entry.rate:.4f}"
            ))

    def _on_clear(self):
        """Очистка истории."""
        if messagebox.askyesno("Подтверждение", "Удалить всю историю?"):
            self.converter.clear_history()
            self._refresh_history_table()
            self.result_var.set("")

    def _on_load(self):
        """Загрузка истории из указанного файла."""
        filename = "history.json"  # можно запросить через диалог
        success = self.converter.load_history(filename)
        if success:
            self._refresh_history_table()
            messagebox.showinfo("Успех", f"История загружена из {filename}")
        else:
            messagebox.showerror("Ошибка", f"Файл {filename} не найден.")

if __name__ == "__main__":
    root = tk.Tk()
    app = CurrencyConverterApp(root)
    root.mainloop()
    # tests.py
# Модуль тестирования приложения

import unittest
import os
import json
from conversion import Conversion
from history_manager import HistoryManager
from currency_api import CurrencyAPI
from converter import Converter

TEST_API_KEY = "didi010829"

class TestConversion(unittest.TestCase):
    def test_creation(self):
        conv = Conversion("USD", "EUR", 100, 85.0, 0.85)
        self.assertEqual(conv.from_currency, "USD")
        self.assertEqual(conv.to_currency, "EUR")
        self.assertEqual(conv.amount, 100)
        self.assertEqual(conv.result, 85.0)
        self.assertEqual(conv.rate, 0.85)
        self.assertIsNotNone(conv.timestamp)

    def test_to_dict_and_from_dict(self):
        conv = Conversion("USD", "EUR", 100, 85.0, 0.85)
        data = conv.to_dict()
        new_conv = Conversion.from_dict(data)
        self.assertEqual(conv.from_currency, new_conv.from_currency)
        self.assertEqual(conv.to_currency, new_conv.to_currency)
        self.assertEqual(conv.amount, new_conv.amount)
        self.assertEqual(conv.result, new_conv.result)
        self.assertEqual(conv.rate, new_conv.rate)
        self.assertEqual(conv.timestamp, new_conv.timestamp)


class TestHistoryManager(unittest.TestCase):
    def setUp(self):
        self.hm = HistoryManager()
        self.conv1 = Conversion("USD", "EUR", 100, 85.0, 0.85)
        self.conv2 = Conversion("RUB", "USD", 5000, 60.0, 0.012)

    def test_add_and_get_all(self):
        self.hm.add_entry(self.conv1)
        self.hm.add_entry(self.conv2)
        all_entries = self.hm.get_all()
        self.assertEqual(len(all_entries), 2)
        self.assertEqual(all_entries[0], self.conv1)

    def test_get_last(self):
        self.hm.add_entry(self.conv1)
        self.hm.add_entry(self.conv2)
        last = self.hm.get_last(1)
        self.assertEqual(len(last), 1)
        self.assertEqual(last[0], self.conv2)

    def test_clear(self):
        self.hm.add_entry(self.conv1)
        self.hm.clear()
        self.assertEqual(len(self.hm), 0)

    def test_save_and_load(self):
        self.hm.add_entry(self.conv1)
        self.hm.add_entry(self.conv2)
        self.hm.save_to_file("test_history.json")
        new_hm = HistoryManager()
        success = new_hm.load_from_file("test_history.json")
        self.assertTrue(success)
        self.assertEqual(len(new_hm), 2)
        # Проверка содержимого
        loaded = new_hm.get_all()
        self.assertEqual(loaded[0].from_currency, "USD")
        self.assertEqual(loaded[1].to_currency, "USD")
        os.remove("test_history.json")

    def test_load_nonexistent(self):
        hm = HistoryManager()
        success = hm.load_from_file("non_existent.json")
        self.assertFalse(success)


class TestConverter(unittest.TestCase):
    def setUp(self):
        # Используем реальный API-ключ только если он задан, иначе пропускаем тесты с API
        self.api_key = "dummy"  # замените при необходимости
        self.converter = Converter(self.api_key)

    def test_convert_same_currency(self):
        conv = self.converter.convert("USD", "USD", 100)
        self.assertIsNotNone(conv)
        self.assertEqual(conv.amount, 100)
        self.assertEqual(conv.result, 100)
        self.assertEqual(conv.rate, 1.0)

    def test_convert_negative_amount(self):
        with self.assertRaises(ValueError):
            self.converter.convert("USD", "EUR", -10)

    def test_history_added(self):
        self.converter.convert("USD", "EUR", 50)
        self.assertEqual(len(self.converter.get_history()), 1)

    def test_clear_history(self):
        self.converter.convert("USD", "EUR", 10)
        self.converter.clear_history()
        self.assertEqual(len(self.converter.get_history()), 0)

    # Пропускаем тесты, требующие реального API, если ключ не задан
    @unittest.skipIf(not os.getenv("EXCHANGE_API_KEY"), "API key not set")
    def test_real_api(self):
        api = CurrencyAPI(os.getenv("EXCHANGE_API_KEY"))
        rate = api.get_exchange_rate("USD", "EUR")
        self.assertIsNotNone(rate)
        self.assertGreater(rate, 0)

if __name__ == "__main__":
    unittest.main(verbosity=2)
                f"{self._to_currency} {self._result:.2f} "
                f"(курс: {self._rate:.4f}, {self._timestamp})")
