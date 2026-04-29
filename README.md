import re
import math
import os

class Checker:
    def __init__(self, f="passwords.txt"):
        self.bad_list = self.load_bad(f) # переменная, где будут храниться эти пароли
        print("Загружено ненадежных паролей:", len(self.bad_list))

    def load_bad(self, f): # загружает пароли из файла
        bad = set()
        if os.path.exists(f): # проверяет, есть ли файл с таким именем
            try:
                with open(f, 'r', encoding='utf-8') as file: # кодировка, чтобы русские буквы читались
                    for line in file:
                        p = line.strip().lower()
                        if p:
                            bad.add(p)  # добавляет пароль в множество
                print("Файл", f, "успешно загружен")
                return bad

            except Exception as e:
                print("Ошибка при чтении файла:", e)
                print("Беру запасной список ")
                return self.get_default()
        else:
            print("Файл", f, "не найден, беру запасной список")
            return self.get_default()

    def get_default(self): # запасной список паролей, на случай если основного нет
        return {'123456', 'password', 'qwerty', 'admin', 'пароль',
                '12345678', '111111', '0000', 'ertyu', 'love',
                 '12345', 'welcome', 'abc123'}

    def len_check(self, p): # проверка длины пароля и подсчет баллов
        l = len(p)
        if l < 8:
            return 0, "Слишком короткий", f"{l} символов меньше 8"
        elif l >= 8 and l <= 11:
            return 1, "Средний", f"{l} символов 8-11"
        else:
            return 2, "Хороший", f"{l} символов 12 и больше"

    def sym_check(self, p): # проверка типов символов в пароле
        a = re.search(r'[a-z]', p)  # ищет строчные латинские буквы
        b = re.search(r'[A-Z]', p)  # ищет заглавные латинские буквы
        c = re.search(r'\d', p)  # ищет цифры
        d = re.search(r'[!@#$%^&*()]', p)  # ищет спецсимволы

        sc = 0  # счетчик
        if a:
            sc +=1
        if b:
            sc += 1
        if c:
            sc += 1
        if d:
            sc += 1

        det = {
            'a': ('строчные буквы (a-z)', a),
            'b': ('заглавные буквы (A-Z)', b),
            'c': ('цифры (0-9)', c),
            'd': ('спецсимволы (!@#...)', d)
        } # ключ 'a' или 'b' или 'c' или 'd' - результат подсчета
        return sc, det

    def calc_ent(self, p): # расчет энтропии
        rz = 0 # сколько разных символов можно использовать
        if re.search(r'[a-z]', p):
            rz += 26
        if re.search(r'[A-Z]', p):
            rz += 26
        if re.search(r'\d', p):
            rz += 10
        if re.search(r'[!@#$%^&*()]', p):
            rz += 33

        if rz == 0:  # если вдруг пароль пустой
            return 0, 0

        l = len(p)
        e = l * math.log2(rz)
        return round(e, 1), rz

    def check_bad(self, p): # проверка пароля в словаре ненадежных паролей
        return p.lower() in self.bad_list

    def check_all(self, p): # сборка результата
        if p == "":
            return {'err': 'пустой пароль'}
        # запуск проверок
        ls, lst, lsd = self.len_check(p)  # ls - баллы длины, lst - статус, lsd - описание
        cs, cdt = self.sym_check(p)  # cs - баллы символов, cdt - детали
        is_bad = self.check_bad(p)  # is_bad - есть ли в словаре
        ent, rz = self.calc_ent(p)  # ent - энтропия, rz - разные символы

        if ent >= 60: # перевод энтропии в баллы от 0 до 2
            ent_b = 2
        elif ent >= 30:
            ent_b = 1
        else:
            ent_b = 0

        dict_b = 0 # проверка словаря, от 0 до 1 балла
        if is_bad == False:
            dict_b = 1

        total = ls + cs + dict_b + ent_b #итог
        proc = round((total / 9) * 100, 1)

        if proc < 50:
            grade = "Крайне ненадежный"
        elif proc < 80:
            grade = "Слабый"
        elif proc < 90:
            grade = "Средний"
        else:
            grade = "Надежный"

        return {
            'pwd': p,  # сам пароль
            'len': {'sc': ls, 'st': lst, 'desc': lsd},  # длины
            'char': {'sc': cs, 'det': cdt},  # символы
            'bad': is_bad,
            'ent': {'val': ent, 'size': rz},  # энтропия
            'proc': proc,  # проценты
            'grade': grade  # итог
        }


import tkinter as tk # интерфейс
from tkinter import scrolledtext # модуль для создания текстового поля с полосой прокрутки
from tkinter import messagebox # модуль для всплывающих окошек с сообщениями
class AppWindow(tk.Tk):
    def __init__(self):
        super().__init__() # конструктор родительского класса
        self.title("AppWindow") # заголовок
        self.geometry("650x550") # размер окна
        self.ch = Checker("passwords.txt") # объект класса Checker
        self.show_var = tk.BooleanVar(value=False)
        self.draw() # вызываем функцию, которая создает все кнопки и поля

    def draw(self): # создает все элементы интерфейса
        t = tk.Label(self, text="Анализатор паролей", font=("Arial", 14))
        t.pack(pady=10)  # размещение элемента в окне с отступом
        f = tk.LabelFrame(self, text="Введите пароль") # рамка с заголовком
        f.pack(fill="x", padx=20, pady=10) # растяжение по горизонтали

        self.ent = tk.Entry(f, show="*", width=40) # поле для ввода текста
        self.ent.pack(side="left", padx=5, pady=5) # прижимает к левому краю
        cb = tk.Checkbutton(f, text="Показать", variable=self.show_var, command=self.show_pwd)
        # галочка показать пароль
        cb.pack(side="right")
        bf = tk.Frame(self) # рамка для группировки
        bf.pack(pady=10)

        # кнопка "Проверить"
        b1 = tk.Button(bf, text="Проверить", command=self.check_pwd, bg="green", fg="white")
        b1.pack(side="left", padx=5)

        # кнопка "Очистить"
        b2 = tk.Button(bf, text="Очистить", command=self.clear_all, bg="red", fg="white")
        b2.pack(side="left", padx=5)

        rf = tk.LabelFrame(self, text="Результат") # поле результатов
        rf.pack(fill="both", expand=True, padx=20, pady=10)

        self.out = scrolledtext.ScrolledText(rf, wrap=tk.WORD, height=15)
        self.out.pack(fill="both", expand=True)
        self.out.tag_config("g", foreground="green")  # зеленый хорошо
        self.out.tag_config("r", foreground="red")  # красный плохо
        self.out.tag_config("o", foreground="orange")  # оранжевый внимание
        self.out.tag_config("h", foreground="blue")  # синий заголовки

    def show_pwd(self): # показывает или скрывает пароль при нажатии
        if self.show_var.get():
            self.ent.config(show="")
        else:
            self.ent.config(show="*")

    def clear_all(self): # очистка поля
        self.ent.delete(0, tk.END)  # очистка поле ввода
        self.out.delete(1.0, tk.END)  # очистка поле результатов

    def check_pwd(self): # проверяет пароль и выводит результат
        p = self.ent.get()
        if p == "":
            messagebox.showwarning("Ошибка", "Введите пароль.")
            return
        res = self.ch.check_all(p) # вызов проверки

        if 'err' in res:
            self.out.insert(tk.END, "Ошибка\n", "r")
            return

        # очищает старые результаты перед выводом новых
        self.out.delete(1.0, tk.END)
        # вывод заголовка
        self.out.insert(tk.END, "Результат проверки\n", "h")

        ld = res['len']  # вывод длины
        if ld['sc'] == 2:
            col = "g"  # зеленый
        elif ld['sc'] == 1:
            col = "o"  # оранжевый
        else:
            col = "r"  # красный

        self.out.insert(tk.END, "Длина: ", "h")
        self.out.insert(tk.END, ld['st'] + "\n", col)
        self.out.insert(tk.END, "   " + ld['desc'] + "\n\n")

        cd = res['char']  # вывод символов
        self.out.insert(tk.END, "Символы: " + str(cd['sc']) + "/4\n")
        for key, val in cd['det'].items():# key - это 'a', 'b', 'c', 'd'
            if val[1]:
                self.out.insert(tk.END, "   + " + val[0] + "\n", "g")
            else:
                self.out.insert(tk.END, "   - " + val[0] + "\n", "r")
        self.out.insert(tk.END, "\n")

        if res['bad']:
            self.out.insert(tk.END, "Словарь: пароль найден в списке ненадежных!\n", "r")
        else:
            self.out.insert(tk.END, "Словарь: не найден\n", "g")

        ed = res['ent']
        self.out.insert(tk.END, "\nЭнтропия: " + str(ed['val']) + " бит\n")
        self.out.insert(tk.END, "   (алфавит: " + str(ed['size']) + " символов)\n")

        self.out.insert(tk.END, "\n" + " " * 45 + "\n", "h")
        self.out.insert(tk.END, "Оценка: " + str(res['proc']) + "%\n", "h")

        # цвет итога
        if res['grade'] == "Надежный":
            colg = "g"
        elif res['grade'] == "Средний":
            colg = "o"  # оранжевый
        else:
            colg = "r"  # красный

        self.out.insert(tk.END, "\nИтог: ", "h")
        self.out.insert(tk.END, res['grade'] + "\n", colg)


if __name__ == "__main__": # запуск программы
    app = AppWindow()
    app.mainloop()



if __name__ == "__main__": # запуск программы
    app = AppWindow()
    app.mainloop()
