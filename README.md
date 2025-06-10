import tkinter as tk
from tkinter import ttk, messagebox
import random

class Environment:
    def __init__(self):
        self.temperature = 20
        self.light = 70

class Plant:
    def __init__(self, name, soil, min_temp, max_temp, min_light, max_light, group, original = None):
        self.name = name
        self.progress = 0
        self.stage = "Семя"
        self.soil = soil
        self.min_temp = min_temp
        self.max_temp = max_temp
        self.min_light = min_light
        self.max_light = max_light
        self.group = group
        self.vitamins = False
        self.vitamins_active_days = 0
        self.moisture = 100
        self.last_vitamins_day = -8
        self.wilt_days = 0
        self.transplanted = False
        self.bad_temp_days = 0
        self.days_at_zero_moisture = 0
        self.clone_count = 0
        self.original = original

        if original:
            self.clone_number = original.clone_count + 1
        else:
            self.clone_number = 0

    def update(self, environment):
        if self.stage == "Засохло":
            return

        if self.min_temp <= environment.temperature <= self.max_temp and self.min_light <= environment.light <= self.max_light:
            growth = random.uniform(2.0, 5.0)
        else:
            growth = random.uniform(0.5, 1.5)

        if self.vitamins:
            growth *= 2

        if self.moisture < 20:
            growth *= 0.5
            self.wilt_days += 1
        elif self.moisture > 100:
            growth = -abs(growth) * 2
        else:
            self.wilt_days = 0

        if self.moisture < 20:
            self.days_at_zero_moisture += 1
        else:
            self.days_at_zero_moisture = 0

        if self.days_at_zero_moisture >= 7:
            growth = -1

        self.progress = min(100, max(0, self.progress + growth))

        normal = (self.min_temp <= environment.temperature <= self.max_temp) and (self.min_light <= environment.light <= self.max_light) and (20 <= self.moisture <= 100)

        if self.group in ["Деревья", "Кустарники"]:
            if self.progress < 20:
                new_stage = "Семя"
            elif self.progress < 40:
                new_stage = "Росток"
            else:
                new_stage = "Взрослое растение"
        else:
            if self.progress < 20:
                new_stage = "Семя"
            elif self.progress < 40:
                new_stage = "Росток"
            elif self.progress < 70:
                new_stage = "Взрослое растение"
            elif self.progress < 90:
                new_stage = "Цветение"
            else:
                new_stage = "Увядание"

        temp_ok = self.min_temp <= environment.temperature <= self.max_temp
        if not temp_ok:
            self.bad_temp_days += 1
            if self.bad_temp_days == 3:
                self.stage = "Увядает"
            elif self.bad_temp_days >= 4:
                self.stage = "Засохло"
        else:
            self.bad_temp_days = 0

        if self.wilt_days >= 3:
            self.stage = "Увядает"
        if self.wilt_days >= 4:
            self.stage = "Засохло"

        if self.stage == "Засохло":
            return

        if normal:
            self.stage = new_stage

class PlantSimulationApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Биологическая лаборатория")
        self.root.geometry("1400x900")
        self.root.resizable(False, False)
        self.root.configure(bg="#C0C4CE")

        self.environment = Environment()
        self.plants = []
        self.day_counter = 0
        self.transplanted_plants = []

        main_frame = tk.Frame(root, bg="#C0C4CE")
        main_frame.pack(fill="both", expand=True)

        top_panel = tk.Frame(main_frame, bg="#C0C4CE")
        top_panel.pack(side="top", fill="x")

        env_frame = tk.LabelFrame(top_panel, text="Параметры среды0", bg="#C0C4CE", font=("Arial", 10))
        env_frame.pack(side="left", fill="both", expand=True, padx=10, pady=5)
        tk.Label(env_frame, text="Температура (°C)", bg="#C0C4CE",).pack(anchor="w")
        self.temp_scale = tk.Scale(env_frame, from_=0, to=40, orient="horizontal", command=self.update_temperature, bg="#C0C4CE", highlightbackground="#C0C4CE")
        self.temp_scale.set(self.environment.temperature)
        self.temp_scale.pack(fill="x")
        tk.Label(env_frame, text="Освещение (%)", bg="#C0C4CE").pack(anchor="w")
        self.light_scale = tk.Scale(env_frame, from_=0, to=100, orient="horizontal", command=self.update_light, bg="#C0C4CE", highlightbackground="#C0C4CE")
        self.light_scale.set(self.environment.light)
        self.light_scale.pack(fill="x")

        control_frame = tk.LabelFrame(top_panel, text="Управление", bg="#C0C4CE", font=("Arial", 10))
        control_frame.pack(side="left", fill="both", expand=True, padx=10, pady=5)
        self.day_label = tk.Label(control_frame, text="Текущий день: 0", bg="#C0C4CE", font=("Arial", 18, "bold"))
        self.day_label.pack(pady=5)
        tk.Button(control_frame, text="Новый день", command=self.new_day, relief='ridge', font=("Arial", 14)).pack(fill="x", pady=5)
        tk.Button(control_frame, text="Добавить растение", command=self.add_plant, relief='ridge', font=("Arial", 14)).pack(fill="x", pady=5)

        plant_frame_container = tk.Frame(main_frame, bg="#C0C4CE")
        plant_frame_container.pack(side="top", fill="both", expand=True, padx=10, pady=10)

        self.canvas = tk.Canvas(plant_frame_container, bg="#C0C4CE", highlightbackground="#C0C4CE")
        self.canvas.pack(side="left", fill="both", expand=True)
        scrollbar = ttk.Scrollbar(plant_frame_container, orient="vertical", command=self.canvas.yview)
        scrollbar.pack(side="right", fill="y")
        self.canvas.configure(yscrollcommand=scrollbar.set)
        self.plant_frame = tk.Frame(self.canvas, bg="#C0C4CE")
        self.canvas.create_window((0, 0), window=self.plant_frame, anchor="nw")
        self.plant_frame.bind("<Configure>", lambda event: self.canvas.configure(scrollregion=self.canvas.bbox("all")))
        self.create_sample_plants()
        self.display_plants()

        def on_frame_configure(event=None):
            self.canvas.update_idletasks()
            bbox = self.canvas.bbox("all")
            canvas_height = self.canvas.winfo_height()
            if bbox:
                bottom = bbox[3] if bbox[3] > canvas_height else canvas_height
                self.canvas.configure(scrollregion=(0, 0, bbox[2], bottom))

        self.plant_frame.bind("<Configure>", on_frame_configure)

    def delete_plant(self, plant):
        if plant in self.plants:
            self.plants.remove(plant)
            self.display_plants()

    def create_sample_plants(self):
        p1 = Plant("Дуб", "Чернозём", 10, 30, 50, 90, "Деревья")
        p1.progress = 0
        p1.stage = "Семя"

        p2 = Plant("Береза", "Песчаная", 10, 30, 50, 90, "Деревья")
        p2.progress = 0
        p2.stage = "Семя"

        p3 = Plant("Кустарник", "Глинистая", 15, 28, 40, 80, "Кустарники")
        p3.progress = 0
        p3.stage = "Семя"

        p4 = Plant("Ромашка", "Чернозём", 12, 25, 60, 95, "Травы")
        p4.progress = 0
        p4.stage = "Семя"

        p5 = Plant("Лаванда", "Песчаная", 10, 25, 70, 100, "Травы")
        p5.progress = 0
        p5.stage = "Семя"

        self.plants.extend([p1, p2, p3, p4, p5])

    def update_temperature(self, value):
        self.environment.temperature = int(value)

    def update_light(self, value):
        self.environment.light = int(value)

    def clone_plant(self, plant):
        if plant.stage != "Взрослое растение" or plant.group != "Травы":
            messagebox.showerror("Ошибка", "Растение не готово для клонирования! Оно должно быть в стадии 'Взрослое растение'.")
            return

        original = plant.original if plant.original else plant

        if original.clone_count >= 3:
            messagebox.showerror("Ошибка", f"Достигнут лимит клонов для этого растения (максимум 3)! Уже создано: {original.clone_count}")
            return

        clone_name = f"[Клон {original.clone_count + 1}] {original.name}"
        new_plant = Plant(clone_name, plant.soil, plant.min_temp, plant.max_temp, plant.min_light, plant.max_light, plant.group, original)
        new_plant.progress = plant.progress * 0.3
        self.plants.append(new_plant)
        original.clone_count += 1
        self.display_plants()

    def add_plant(self):
        add_window = tk.Toplevel(self.root)
        add_window.title("Добавить растение")
        add_window.geometry("600x600")
        add_window.resizable(False, False)
        add_window.configure(bg="#94C4CA")

        def validate_name(new_value):
            return new_value == "" or len(new_value) <= 39

        def validate_temperature(new_value):
            if new_value == "" or new_value == "-":
                return True
            return new_value.lstrip('-').isdigit()

        def validate_light(new_value):
            if new_value == "":
                return True
            return new_value.isdigit()

        validate_name_cmd = add_window.register(validate_name)
        validate_temp_cmd = add_window.register(validate_temperature)
        validate_light_cmd = add_window.register(validate_light)

        tk.Label(add_window, text="Название растения:", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        name_entry = ttk.Entry(add_window, validate="key", validatecommand=(validate_name_cmd, '%P'), font=("Arial", 11))
        name_entry.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Группа растений:", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        group_box = ttk.Combobox(add_window, values=["Деревья", "Кустарники", "Травы"], state="readonly", font=("Arial", 11))
        group_box.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Почва:", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        soil_box = ttk.Combobox(add_window, values=["Песчаная", "Глинистая", "Чернозём"], state="readonly", font=("Arial", 11))
        soil_box.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Минимальная температура:", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        min_temp_entry = ttk.Entry(add_window, validate="key", validatecommand=(validate_temp_cmd, '%P'), font=("Arial", 11))
        min_temp_entry.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Максимальная температура:", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        max_temp_entry = ttk.Entry(add_window, validate="key", validatecommand=(validate_temp_cmd, '%P'), font=("Arial", 11))
        max_temp_entry.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Минимальное освещение (%):", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        min_light_entry = ttk.Entry(add_window, validate="key", validatecommand=(validate_light_cmd, '%P'), font=("Arial", 11))
        min_light_entry.pack(fill="x", padx=10, pady=5)

        tk.Label(add_window, text="Максимальное освещение (%):", bg="#94C4CA", font=("Arial", 11)).pack(anchor="w", padx=10, pady=(10, 0))
        max_light_entry = ttk.Entry(add_window, validate="key", validatecommand=(validate_light_cmd, '%P'), font=("Arial", 11))
        max_light_entry.pack(fill="x", padx=10, pady=5)

        def save_plant():
            name = name_entry.get().strip()
            group = group_box.get()
            soil = soil_box.get()
            try:
                min_temp = int(min_temp_entry.get())
                max_temp = int(max_temp_entry.get())
                min_light = int(min_light_entry.get())
                max_light = int(max_light_entry.get())
            except ValueError:
                messagebox.showerror("Ошибка", "Введите корректные числовые значения!")
                return

            if min_temp < -25:
                messagebox.showerror("Ошибка", "Минимальная температура не может быть меньше -25°C!")
                return
            if max_temp > 40:
                messagebox.showerror("Ошибка", "Максимальная температура не может быть больше 40°C!")
                return
            if min_light < 0:
                messagebox.showerror("Ошибка", "Минимальное освещение не может быть меньше 0%!")
                return
            if max_light > 100:
                messagebox.showerror("Ошибка", "Максимальное освещение не может быть больше 100%!")
                return
            if max_temp < min_temp:
                messagebox.showerror("Ошибка", "Максимальная температура не может быть меньше минимальной!")
                return
            if max_light < min_light:
                messagebox.showerror("Ошибка", "Максимальное освещение не может быть меньше минимального!")
                return
            if name and group and soil:
                plant = Plant(name, soil, min_temp, max_temp, min_light, max_light, group)
                self.plants.append(plant)
                self.display_plants()
                add_window.destroy()
            else:
                messagebox.showerror("Ошибка", "Заполните все поля!")

        tk.Button(add_window, text="Сохранить", command=save_plant, relief='ridge').pack(pady=10, padx=10)

    def add_vitamins(self, plant):
        if plant.stage == "Засохло":
            messagebox.showerror("Ошибка", "Растение засохло. Добавление витаминов невозможно!")
            return
        if self.day_counter - plant.last_vitamins_day >= 8:
            plant.vitamins = True
            plant.vitamins_active_days = 5
            plant.last_vitamins_day = self.day_counter
            self.display_plants()

    def water_plant(self, plant):
        if plant.stage == "Засохло":
            messagebox.showerror("Ошибка", "Растение засохло. Полив невозможен!")
            return
        plant.moisture = min(200, plant.moisture + 30)
        self.display_plants()

    def transplant_plant(self, plant):
        if plant.stage == "Засохло":
            messagebox.showerror("Ошибка", "Нельзя пересаживать засохшее растение!")
            return

        plant.transplanted = True
        if plant in self.plants:
            self.plants.remove(plant)
            self.transplanted_plants.append(plant)
            messagebox.showinfo("Пересадка", f"Растение {plant.name} было пересажено на постоянное место.")
            self.display_plants()

    def display_plants(self):
        for widget in self.plant_frame.winfo_children():
            widget.destroy()

        columns = 3

        for index, plant in enumerate(self.plants):
            row = index // columns
            col = index % columns
            card_frame = tk.Frame(self.plant_frame, bg="#A3C6CA", bd=1, relief="solid",padx=10, pady=10, width=450, height=250)
            card_frame.grid(row=row, column=col, padx=1, pady=3, sticky="n")
            card_frame.pack_propagate(False)

            tk.Label(card_frame, text=f"{plant.name} ({plant.stage})", bg="#A3C6CA").pack(anchor="w")
            tk.Label(card_frame, text=f"Почва: {plant.soil}", bg="#A3C6CA").pack(anchor="w")
            tk.Label(card_frame, text=f"Влажность: {plant.moisture:.0f}%", bg="#A3C6CA").pack(anchor="w")
            tk.Label(card_frame, text=f"Группа: {plant.group}", bg="#A3C6CA").pack(anchor="w")
            tk.Label(card_frame, text=f"Процент развития: {plant.progress:.0f}%", bg="#A3C6CA").pack(anchor="w")

            if plant.stage != "Засохло":
                if plant.moisture <= 20:
                    tk.Label(card_frame, text="Рекомендация: Полить растение (влажность ≤ 20%)", foreground="red", bg="#A3C6CA").pack(anchor="w")
                if not (plant.min_temp <= self.environment.temperature <= plant.max_temp):
                    tk.Label(card_frame, text=f"Рекомендация: Температура должна быть {plant.min_temp}–{plant.max_temp}°C", foreground="orange", bg="#A3C6CA").pack(anchor="w")
                if not (plant.min_light <= self.environment.light <= plant.max_light):
                    tk.Label(card_frame, text=f"Рекомендация: Освещение должно быть {plant.min_light}–{plant.max_light}%", foreground="orange", bg="#A3C6CA").pack(anchor="w")

            buttons_frame = tk.Frame(card_frame, bg="#A3C6CA")
            buttons_frame.pack(side="bottom", fill="x", pady=(5, 0))

            btn_row1 = tk.Frame(buttons_frame, bg="#A3C6CA")
            btn_row1.pack(side="top", fill="x", pady=2)
            tk.Button(btn_row1, text="Информация", command=lambda p=plant: self.show_plant_info(p)).pack(side="left", padx=5, pady=2)
            if plant.stage != "Засохло":
                if self.day_counter - plant.last_vitamins_day >= 8:
                    tk.Button(btn_row1, text="Добавить витамины", command=lambda p=plant: self.add_vitamins(p)).pack(side="left", padx=5, pady=2)
                else:
                    remaining = 8 - (self.day_counter - plant.last_vitamins_day)
                    tk.Label(btn_row1, text=f"Витамины через {remaining} дн.", bg="#A3C6CA").pack(side="left", padx=5, pady=2)
                tk.Button(btn_row1, text="Полить растение", command=lambda p=plant: self.water_plant(p)).pack(side="left", padx=5, pady=2)

            btn_row2 = tk.Frame(buttons_frame, bg="#A3C6CA")
            btn_row2.pack(side="top", fill="x", pady=2)
            if plant.stage != "Засохло":
                if plant.group in ["Деревья", "Кустарники"] and not plant.transplanted:
                    tk.Button(btn_row2, text="Пересадить", command=lambda p=plant: self.transplant_plant(p)).pack(side="left", padx=5, pady=2)
                if plant.group == "Травы" and plant.stage == "Взрослое растение":
                    original = plant.original if plant.original else plant
                    if original.clone_count < 3:
                        tk.Button(btn_row2, text="Клонировать", command=lambda p=plant: self.clone_plant(p)).pack(side="left", padx=5, pady=2)
                    else:
                        tk.Label(btn_row2, text="Лимит клонов", bg="#A3C6CA").pack(side="left", padx=5, pady=2)
            tk.Button(btn_row2, text="Удалить растение", command=lambda p=plant: self.delete_plant(p)).pack(side="left", padx=5, pady=2)

    def show_plant_info(self, plant):
        info_window = tk.Toplevel(self.root)
        info_window.title(f"Информация о растении: {plant.name}")
        info_window.geometry("400x300")
        info_window.resizable(False, False)
        info_text = (
            f"Название: {plant.name}\n"
            f"Почва: {plant.soil}\n"
            f"Группа: {plant.group}\n"
            f"Стадия: {plant.stage}\n"
            f"Процент развития: {plant.progress:.0f}%\n"
            f"Влажность: {plant.moisture:.0f}%\n"
            f"Оптимальная температура: {plant.min_temp} - {plant.max_temp} °C\n"
            f"Оптимальное освещение: {plant.min_light} - {plant.max_light} %\n"
            f"Витамины: {'Да' if plant.vitamins else 'Нет'}\n"
            f"Дней действия витаминов: {plant.vitamins_active_days}\n"
            f"Дней без влаги: {plant.days_at_zero_moisture}\n"
        )
        tk.Label(info_window, text=info_text, anchor="w", justify="left").pack(padx=10, pady=10)

    def new_day(self):
        self.day_counter += 1
        self.day_label.config(text=f"Текущий день: {self.day_counter}")
        to_transplant = []

        for plant in self.plants:
            if plant.stage == "Засохло":
                continue

            temp = self.environment.temperature
            min_percent = 0.1 + (temp / 40) * 0.4
            max_percent = 0.1 + (temp / 40) * 0.7
            base_reduction = random.uniform(min_percent, max_percent)

            soil_multiplier = {
                "Песчаная": 2.0,
                "Глинистая": 1.8,
                "Чернозём": 1.5
            }.get(plant.soil, 1.0)

            reduction_percent = base_reduction * soil_multiplier
            plant.moisture = max(0, plant.moisture * (1 - reduction_percent))

            if plant.vitamins:
                plant.vitamins_active_days -= 1
                if plant.vitamins_active_days <= 0:
                    plant.vitamins = False

            plant.update(self.environment)

            if plant.group in ["Деревья", "Кустарники"] and plant.stage == "Взрослое растение" and not plant.transplanted:
                to_transplant.append(plant)

        for plant in to_transplant:
            self.transplant_plant(plant)

        self.display_plants()

if __name__ == "__main__":
    root = tk.Tk()
    app = PlantSimulationApp(root)
    root.mainloop()
