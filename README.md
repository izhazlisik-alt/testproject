# testproject
"""
Weather Diary - Дневник погоды
Консольное приложение для ведения дневника погоды.
Позволяет добавлять записи о температуре, осадках, фильтровать их по дате и строить графики температуры.
"""

import json
import os
from datetime import datetime

# Имя файла для сохранения данных
DATA_FILE = "data.json"


class WeatherEntry:
    """Класс для хранения одной записи о погоде."""

    def __init__(self, date, temperature, description, precipitation, humidity=None):
        self.date = date  # ГГГГ-ММ-ДД
        self.temperature = float(temperature)
        self.description = description
        self.precipitation = precipitation  # True/False
        self.humidity = int(humidity) if humidity else None

    def to_dict(self):
        """Преобразует запись в словарь для JSON."""
        return {
            "date": self.date,
            "temperature": self.temperature,
            "description": self.description,
            "precipitation": self.precipitation,
            "humidity": self.humidity
        }

    @classmethod
    def from_dict(cls, data):
        """Создаёт запись из словаря."""
        return cls(
            date=data["date"],
            temperature=data["temperature"],
            description=data["description"],
            precipitation=data["precipitation"],
            humidity=data.get("humidity")
        )

    def __str__(self):
        """Строковое представление записи для вывода."""
        precipitation_text = "Осадки" if self.precipitation else "Без осадков"
        humidity_text = f" | Влажность: {self.humidity}%" if self.humidity else ""
        return f"📅 {self.date} | {self.temperature}°C | {self.description} | {precipitation_text}{humidity_text}"


class WeatherDiary:
 """Основной класс приложения для работы с дневником погоды."""

    def __init__(self):
        self.entries = []
        self.load_data()

    def load_data(self):
        """Загружает данные из JSON-файла."""
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, "r", encoding="utf-8") as file:
                    data = json.load(file)
                    self.entries = [WeatherEntry.from_dict(entry) for entry in data.get("entries", [])]
                print(f"✅ Загружено {len(self.entries)} записей из {DATA_FILE}")
            except (json.JSONDecodeError, FileNotFoundError):
                print("⚠️ Файл повреждён. Начинаем с пустого дневника.")
                self.entries = []
        else:
            print("📝 Создан новый дневник погоды.")

    def save_data(self):
        """Сохраняет данные в JSON-файл."""
        data = {
            "entries": [entry.to_dict() for entry in self.entries]
        }
        with open(DATA_FILE, "w", encoding="utf-8") as file:
            json.dump(data, file, ensure_ascii=False, indent=4)
        print(f"💾 Данные сохранены в {DATA_FILE}")

    def add_entry(self):
        """Добавляет новую запись о погоде."""
        print("\n--- Добавление новой записи ---")

        # Ввод даты с проверкой
        while True:
            date = input("Введите дату (ГГГГ-ММ-ДД): ").strip()
            try:
                datetime.strptime(date, "%Y-%m-%d")
                break
            except ValueError:
                print("❌ Ошибка: неверный формат даты. Используйте ГГГГ-ММ-ДД")

        # Ввод температуры
        while True:
            try:
                temperature = float(input("Введите температуру (°C): "))
                break
            except ValueError:
                print("❌ Ошибка: введите число!")

        description = input("Введите описание погоды: ").strip()

        # Ввод осадков
        while True:
            precipitation_input = input("Осадки (да/нет): ").strip().lower()
            if precipitation_input in ["да", "yes", "y", "1", "true", "+"]:
                precipitation = True
                break
            elif precipitation_input in ["нет", "no", "n", "0", "false", "-"]:
                precipitation = False
                break
            else:
                print("❌ Ошибка: введите 'да' или 'нет'")

        # Ввод влажности (опционально)
        humidity = None
        humidity_input = input("Введите влажность (%) (необязательно, нажмите Enter): ").strip()
        if humidity_input:
            try:
                humidity = int(humidity_input)
            except ValueError:
                print("⚠️ Влажность не сохранена (введите число)")

        # Создаём и добавляем запись
        entry = WeatherEntry(date, temperature, description, precipitation, humidity)
        self.entries.append(entry)
        self.save_data()
        print(f"✅ Запись на {date} успешно добавлена!")

    def show_all_entries(self):
        """Показывает все записи."""
        if not self.entries:
            print("\n📭 Дневник пуст. Добавьте первую запись!")
            return

        print("\n" + "=" * 40)
        print("         Все записи о погоде")
        print("=" * 40)

        # Сортируем по дате
        sorted_entries = sorted(self.entries, key=lambda x: x.date)

        for entry in sorted_entries:
            print(entry)

        print("-" * 40)
        print(f"📊 Всего записей: {len(self.entries)}")

    def filter_by_date(self):
        """Фильтрует записи по дате."""
        if not self.entries:
            print("\n📭 Дневник пуст.")
            return

        date = input("\nВведите дату для поиска (ГГГГ-ММ-ДД): ").strip()

        found = [entry for entry in self.entries if entry.date == date]

        if found:
            print(f"\n📅 Записи за {date}:")
            for entry in found:
                print(entry)
            print(f"📊 Найдено записей: {len(found)}")
        else:
            print(f"❌ Записей на дату {date} не найдено.")

    def filter_by_temperature(self):
        """Фильтрует записи по температуре (выше порога)."""
        if not self.entries:
            print("\n📭 Дневник пуст.")
            return

        try:
            min_temp = float(input("\nВведите минимальную температуру: "))
        except ValueError:
            print("❌ Ошибка: введите число!")
            return

        filtered = [entry for entry in self.entries if entry.temperature >= min_temp]

        if filtered:
            print(f"\n🌡️ Записи с температурой ≥ {min_temp}°C:")
            for entry in filtered:
                print(entry)
            print(f"📊 Найдено записей: {len(filtered)}")
        else:
            print(f"❌ Записей с температурой ≥ {min_temp}°C не найдено.")

    def plot_temperature_graph(self):
        """Строит график температуры (если установлена matplotlib)."""
        if len(self.entries) < 2:
            print("\n⚠️ Для построения графика нужно минимум 2 записи.")
            return

        try:
            import matplotlib.pyplot as plt

            # Сортируем по дате
            sorted_entries = sorted(self.entries, key=lambda x: x.date)

            dates = [entry.date for entry in sorted_entries]
            temperatures = [entry.temperature for entry in sorted_entries]

            # Создаём график
            plt.figure(figsize=(10, 5))
            plt.plot(dates, temperatures, marker='o', linestyle='-', color='blue', linewidth=2)

            plt.title("График изменения температуры", fontsize=14, fontweight='bold')
            plt.xlabel("Дата", fontsize=12)
            plt.ylabel("Температура (°C)", fontsize=12)
            plt.xticks(rotation=45)
            plt.grid(True, alpha=0.3)

            # Добавляем значения на график
            for i, (date, temp) in enumerate(zip(dates, temperatures)):
                plt.annotate(f"{temp}°C", (date, temp), textcoords="offset points", xytext=(0, 10), ha='center')

            plt.tight_layout()
            plt.show()
            print("📈 График успешно построен!")

        except ImportError:
            print("\n❌ Библиотека matplotlib не установлена.")
            print("Установите её командой: pip install matplotlib")

    def show_statistics(self):
        """Показывает статистику по записям."""
        if not self.entries:
            print("\n📭 Нет данных для статистики.")
            return

        temperatures = [entry.temperature for entry in self.entries]
        precipitation_count = sum(1 for entry in self.entries if entry.precipitation)

        print("\n" + "=" * 40)
        print("         📊 Статистика")
        print("=" * 40)
        print(f"📝 Всего записей: {len(self.entries)}")
        print(f"🌡️ Средняя температура: {sum(temperatures) / len(temperatures):.1f}°C")
        print(f"📈 Максимальная температура: {max(temperatures)}°C")
        print(f"📉 Минимальная температура: {min(temperatures)}°C")
        print(f"☔ Дней с осадками: {precipitation_count}")
        print(f"☀️ Дней без осадков: {len(self.entries) - precipitation_count}")


def show_menu():
    """Отображает главное меню."""
    print("\n" + "=" * 40)
    print("    Weather Diary - Дневник погоды")
    print("=" * 40)
    print("1 — Добавить запись")
    print("2 — Показать все записи")
    print("3 — Фильтровать по дате")
    print("4 — Фильтровать по температуре")
    print("5 — Построить график температуры")
    print("6 — Показать статистику")
    print("7 — Выйти")
    print("=" * 40)


def main():
    """Главная функция программы."""
    print("\n🌦️ Добро пожаловать в Weather Diary!")

    diary = WeatherDiary()

    while True:
        show_menu()
        choice = input("Выберите действие (1-7): ").strip()

        if choice == "1":
            diary.add_entry()
        elif choice == "2":
            diary.show_all_entries()
        elif choice == "3":
            diary.filter_by_date()
        elif choice == "4":
            diary.filter_by_temperature()
        elif choice == "5":
            diary.plot_temperature_graph()
        elif choice == "6":
            diary.show_statistics()
        elif choice == "7":
            print("\n👋 До свидания! Данные сохранены.")
            break
        else:
            print("❌ Ошибка: выберите пункт от 1 до 7.")


if __name__ == "__main__":
    main()
