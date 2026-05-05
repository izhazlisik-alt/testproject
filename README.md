"""
Book Tracker - Трекер прочитанных книг
GUI-приложение для управления списком прочитанных книг с фильтрацией и сохранением в JSON
"""

import tkinter as tk
from tkinter import ttk, messagebox
import json
import os
from datetime import datetime

# Имя файла для сохранения данных
DATA_FILE = "books.json"


class Book:
    """Класс для хранения информации о книге."""

    def __init__(self, title, author, genre, pages):
        self.title = title.strip()
        self.author = author.strip()
        self.genre = genre.strip()
        self.pages = int(pages)

    def to_dict(self):
        """Преобразует книгу в словарь для JSON."""
        return {
            "title": self.title,
            "author": self.author,
            "genre": self.genre,
            "pages": self.pages
        }

    @classmethod
    def from_dict(cls, data):
        """Создаёт книгу из словаря."""
        return cls(
            title=data["title"],
            author=data["author"],
            genre=data["genre"],
            pages=data["pages"]
        )


class BookTrackerApp:
    """Главный класс приложения Book Tracker."""

    def __init__(self, root):
        self.root = root
        self.root.title("📚 Book Tracker - Трекер прочитанных книг")
        self.root.geometry("900x600")
        self.root.resizable(True, True)
        self.root.configure(bg="#f5f5f5")

        # Список всех книг
        self.books = []

        # Загружаем данные из файла
        self.load_data()

        # Создаём интерфейс
        self.create_widgets()

        # Обновляем отображение
        self.refresh_book_list()

    def create_widgets(self):
        """Создание всех элементов интерфейса."""

        # ==================== ВЕРХНЯЯ ПАНЕЛЬ (ВВОД ДАННЫХ) ====================
        input_frame = tk.LabelFrame(
            self.root,
            text="📖 Добавление новой книги",
            font=("Arial", 12, "bold"),
            bg="#ffffff",
            fg="#333333",
            padx=15,
            pady=15
        )
        input_frame.pack(pady=15, padx=15, fill="x")

        # Поле для названия книги
        tk.Label(input_frame, text="📌 Название книги:", font=("Arial", 10), bg="#ffffff").grid(
            row=0, column=0, sticky="w", padx=(0, 10), pady=5
        )
        self.title_entry = tk.Entry(input_frame, width=25, font=("Arial", 10), relief="solid", bd=1)
        self.title_entry.grid(row=0, column=1, padx=(0, 20), pady=5)

        # Поле для автора
        tk.Label(input_frame, text="👤 Автор:", font=("Arial", 10), bg="#ffffff").grid(
            row=0, column=2, sticky="w", padx=(0, 10), pady=5
        )
        self.author_entry = tk.Entry(input_frame, width=20, font=("Arial", 10), relief="solid", bd=1)
        self.author_entry.grid(row=0, column=3, padx=(0, 20), pady=5)

        # Поле для жанра
        tk.Label(input_frame, text="🎭 Жанр:", font=("Arial", 10), bg="#ffffff").grid(
            row=1, column=0, sticky="w", padx=(0, 10), pady=5
        )
        self.genre_entry = tk.Entry(input_frame, width=25, font=("Arial", 10), relief="solid", bd=1)
        self.genre_entry.grid(row=1, column=1, padx=(0, 20), pady=5)

        # Поле для количества страниц
        tk.Label(input_frame, text="📄 Количество страниц:", font=("Arial", 10), bg="#ffffff").grid(
            row=1, column=2, sticky="w", padx=(0, 10), pady=5
        )
        self.pages_entry = tk.Entry(input_frame, width=20, font=("Arial", 10), relief="solid", bd=1)
        self.pages_entry.grid(row=1, column=3, padx=(0, 20), pady=5)

        # Кнопка добавления книги
        self.add_button = tk.Button(
            input_frame,
            text="➕ Добавить книгу",
            command=self.add_book,
            bg="#4CAF50",
            fg="white",
            font=("Arial", 10, "bold"),
            cursor="hand2",
            padx=10,
            pady=5
        )
        self.add_button.grid(row=2, column=0, columnspan=4, pady=15)

        # ==================== ПАНЕЛЬ ФИЛЬТРАЦИИ ====================
        filter_frame = tk.LabelFrame(
            self.root,
            text="🔍 Фильтрация книг",
            font=("Arial", 12, "bold"),
            bg="#ffffff",
            fg="#333333",
            padx=15,
            pady=10
        )
        filter_frame.pack(pady=(0, 15), padx=15, fill="x")

        # Фильтр по жанру
        tk.Label(filter_frame, text="Фильтр по жанру:", font=("Arial", 10), bg="#ffffff").grid(
            row=0, column=0, sticky="w", padx=(0, 10)
        )
        self.genre_filter_var = tk.StringVar(value="Все")
        self.genre_filter_combo = ttk.Combobox(
            filter_frame,
            textvariable=self.genre_filter_var,
            width=20,
            state="readonly"
        )
        self.genre_filter_combo.grid(row=0, column=1, padx=(0, 20))
        self.genre_filter_combo.bind("<<ComboboxSelected>>", lambda e: self.refresh_book_list())

        # Фильтр по количеству страниц
        tk.Label(filter_frame, text="Страниц больше:", font=("Arial", 10), bg="#ffffff").grid(
            row=0, column=2, sticky="w", padx=(0, 10)
        )
        self.pages_filter_var = tk.StringVar(value="0")
        self.pages_filter_entry = tk.Entry(
            filter_frame,
            textvariable=self.pages_filter_var,
            width=10,
            font=("Arial", 10),
            relief="solid",
            bd=1
        )
        self.pages_filter_entry.grid(row=0, column=3, padx=(0, 10))
        self.pages_filter_entry.bind("<KeyRelease>", lambda e: self.refresh_book_list())

        # Кнопка сброса фильтров
        self.reset_filter_button = tk.Button(
            filter_frame,
            text="🔄 Сбросить фильтры",
            command=self.reset_filters,
            bg="#FF9800",
            fg="white",
            font=("Arial", 9),
            cursor="hand2",
            padx=10
        )
        self.reset_filter_button.grid(row=0, column=4, padx=(20, 0))

        # ==================== ОБЛАСТЬ ОТОБРАЖЕНИЯ КНИГ ====================
        list_frame = tk.LabelFrame(
            self.root,
            text="📚 Список прочитанных книг",
            font=("Arial", 12, "bold"),
            bg="#ffffff",
            fg="#333333",
            padx=10,
            pady=10
        )
        list_frame.pack(pady=(0, 15), padx=15, fill="both", expand=True)

        # Создаём Treeview (таблицу) для отображения книг
        columns = ("Название", "Автор", "Жанр", "Страницы")
        self.tree = ttk.Treeview(list_frame, columns=columns, show="headings", height=12)

        # Настройка столбцов
        self.tree.heading("Название", text="📌 Название книги")
        self.tree.heading("Автор", text="👤 Автор")
        self.tree.heading("Жанр", text="🎭 Жанр")
        self.tree.heading("Страницы", text="📄 Страницы")

        self.tree.column("Название", width=250)
        self.tree.column("Автор", width=180)
        self.tree.column("Жанр", width=120)
        self.tree.column("Страницы", width=80, anchor="center")

        # Добавляем скроллбар
        scrollbar = ttk.Scrollbar(list_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)

        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Привязываем обработчик выбора книги
        self.tree.bind("<<TreeviewSelect>>", self.on_select_book)

        # ==================== НИЖНЯЯ ПАНЕЛЬ (КНОПКИ УПРАВЛЕНИЯ) ====================
        bottom_frame = tk.Frame(self.root, bg="#f5f5f5")
        bottom_frame.pack(pady=(0, 15), padx=15, fill="x")

        # Кнопка удаления книги
        self.delete_button = tk.Button(
            bottom_frame,
            text="🗑️ Удалить выбранную книгу",
            command=self.delete_book,
            bg="#f44336",
            fg="white",
            font=("Arial", 10),
            cursor="hand2",
            padx=10,
            pady=5
        )
        self.delete_button.pack(side="left", padx=5)

        # Кнопка очистки всех данных
        self.clear_all_button = tk.Button(
            bottom_frame,
            text="⚠️ Очистить все книги",
            command=self.clear_all_books,
            bg="#9E9E9E",
            fg="white",
            font=("Arial", 10),
            cursor="hand2",
            padx=10,
            pady=5
        )
        self.clear_all_button.pack(side="left", padx=5)

        # Статистика
        self.stats_label = tk.Label(
            bottom_frame,
            text="",
            font=("Arial", 10),
            bg="#f5f5f5",
            fg="#666666"
        )
        self.stats_label.pack(side="right", padx=10)

    def add_book(self):
        """Добавляет новую книгу."""
        # Получаем данные из полей ввода
        title = self.title_entry.get()
        author = self.author_entry.get()
        genre = self.genre_entry.get()
        pages = self.pages_entry.get()

        # Проверка на пустые поля
        if not title:
            messagebox.showwarning("Ошибка", "Пожалуйста, введите название книги!")
            return

        if not author:
            messagebox.showwarning("Ошибка", "Пожалуйста, введите автора!")
            return

        if not genre:
            messagebox.showwarning("Ошибка", "Пожалуйста, введите жанр!")
            return

        if not pages:
            messagebox.showwarning("Ошибка", "Пожалуйста, введите количество страниц!")
            return

        # Проверка, что количество страниц - число
        try:
            pages_int = int(pages)
            if pages_int <= 0:
                messagebox.showwarning("Ошибка", "Количество страниц должно быть больше 0!")
                return
        except ValueError:
            messagebox.showwarning("Ошибка", "Количество страниц должно быть целым числом!")
            return

        # Создаём книгу и добавляем в список
        book = Book(title, author, genre, pages_int)
        self.books.append(book)
        self.save_data()

        # Очищаем поля ввода
        self.title_entry.delete(0, tk.END)
        self.author_entry.delete(0, tk.END)
        self.genre_entry.delete(0, tk.END)
        self.pages_entry.delete(0, tk.END)

        # Обновляем список книг
        self.update_genre_filter()
        self.refresh_book_list()

        messagebox.showinfo("Успех", f"Книга '{title}' успешно добавлена!")

    def delete_book(self):
        """Удаляет выбранную книгу."""
        selected_item = self.tree.selection()

        if not selected_item:
            messagebox.showwarning("Предупреждение", "Пожалуйста, выберите книгу для удаления!")
            return

        # Получаем название книги из выбранной строки
        item = self.tree.item(selected_item[0])
        title = item["values"][0]

        # Подтверждение удаления
        result = messagebox.askyesno("Подтверждение", f"Вы уверены, что хотите удалить книгу '{title}'?")

        if result:
            # Находим и удаляем книгу
            self.books = [book for book in self.books if book.title != title]
            self.save_data()

            # Обновляем отображение
            self.update_genre_filter()
            self.refresh_book_list()

            messagebox.showinfo("Успех", f"Книга '{title}' удалена!")

    def clear_all_books(self):
        """Очищает все книги."""
        if not self.books:
            messagebox.showinfo("Информация", "Список книг уже пуст!")
            return

        result = messagebox.askyesno(
            "Подтверждение",
            "⚠️ ВНИМАНИЕ! Это действие удалит ВСЕ книги.\nВы уверены?"
        )

        if result:
            self.books = []
            self.save_data()
            self.update_genre_filter()
            self.refresh_book_list()
            messagebox.showinfo("Успех", "Все книги удалены!")

    def on_select_book(self, event):
        """Обработчик выбора книги из списка."""
        selected_item = self.tree.selection()
        if not selected_item:
            return

        item = self.tree.item(selected_item[0])
        title = item["values"][0]

        # Находим книгу и заполняем поля для редактирования
        for book in self.books:
            if book.title == title:
                self.title_entry.delete(0, tk.END)
                self.title_entry.insert(0, book.title)

                self.author_entry.delete(0, tk.END)
                self.author_entry.insert(0, book.author)

                self.genre_entry.delete(0, tk.END)
                self.genre_entry.insert(0, book.genre)

                self.pages_entry.delete(0, tk.END)
                self.pages_entry.insert(0, str(book.pages))
                break

    def update_genre_filter(self):
        """Обновляет список жанров в выпадающем списке."""
        genres = sorted(set(book.genre for book in self.books))
        genres.insert(0, "Все")
        self.genre_filter_combo["values"] = genres

        # Если текущее значение не в списке, сбрасываем
        if self.genre_filter_var.get() not in genres:
            self.genre_filter_var.set("Все")

    def refresh_book_list(self):
        """Обновляет отображение книг с учётом фильтров."""
        # Очищаем текущий список
        for item in self.tree.get_children():
            self.tree.delete(item)

        # Получаем отфильтрованный список книг
        filtered_books = self.get_filtered_books()

        # Добавляем книги в таблицу
        for book in filtered_books:
            self.tree.insert("", tk.END, values=(book.title, book.author, book.genre, book.pages))

        # Обновляем статистику
        total_books = len(self.books)
        filtered_count = len(filtered_books)

        if total_books == filtered_count:
            self.stats_label.config(text=f"📊 Всего книг: {total_books}")
        else:
            self.stats_label.config(text=f"📊 Показано: {filtered_count} из {total_books}")

    def get_filtered_books(self):
        """Возвращает отфильтрованный список книг."""
        filtered = self.books[:]

        # Фильтр по жанру
        selected_genre = self.genre_filter_var.get()
        if selected_genre != "Все":
            filtered = [book for book in filtered if book.genre == selected_genre]

        # Фильтр по количеству страниц
        try:
            pages_filter = int(self.pages_filter_var.get())
            if pages_filter > 0:
                filtered = [book for book in filtered if book.pages > pages_filter]
        except ValueError:
            pass

        return filtered

    def reset_filters(self):
        """Сбрасывает все фильтры."""
        self.genre_filter_var.set("Все")
        self.pages_filter_var.set("0")
        self.refresh_book_list()

    def save_data(self):
        """Сохраняет данные в JSON-файл."""
        data = {
            "books": [book.to_dict() for book in self.books],
            "last_updated": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        }

        with open(DATA_FILE, "w", encoding="utf-8") as file:
            json.dump(data, file, ensure_ascii=False, indent=4)

        print(f"💾 Данные сохранены в {DATA_FILE}")

    def load_data(self):
        """Загружает данные из JSON-файла."""
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, "r", encoding="utf-8") as file:
                    data = json.load(file)
                    self.books = [Book.from_dict(book_data) for book_data in data.get("books", [])]
                    last_updated = data.get("last_updated", "Неизвестно")
                    print(f"✅ Загружено {len(self.books)} книг. Последнее обновление: {last_updated}")
            except (json.JSONDecodeError, FileNotFoundError, KeyError) as e:
                print(f"⚠️ Ошибка загрузки данных: {e}")
                self.books = []


# ==================== ЗАПУСК ПРИЛОЖЕНИЯ ====================
if __name__ == "__main__":
    root = tk.Tk()
    app = BookTrackerApp(root)
    root.mainloop()
