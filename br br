import asyncio, json, os, base64
from datetime import datetime

FILE, MENU_FILE, STOCK_FILE = "pizza_orders.json", "menu.json", "stock.json"
CIPHER_PAYLOAD, CIPHER_KEY = "e3tx", 73

RECIPES = {
    "🍕 Мини-пицца Миньон":      {"тесто": 1, "сыр": 2, "томатный соус": 1},
    "🥤 Коктейль Бр-бр потопим":  {"молоко": 1, "сироп": 1, "лёд": 1},
    "🧀 Сырные шарики":          {"сыр": 2, "мука": 1, "масло": 1},
    "🍕 Пицца Пепперони":        {"тесто": 1, "сыр": 2, "томатный соус": 1, "пепперони": 2},
    "🥛 Молочный коктейль":      {"молоко": 2, "мороженое": 1},
    "🌭 Корн-дог":               {"тесто": 1, "сосиска": 1, "масло": 1},
    "🥤 Кола":                   {"кола-сироп": 1, "лёд": 1, "вода": 1},
    "🍕 Пицца 4 сыра":           {"тесто": 1, "сыр": 4, "томатный соус": 1},
    "🧁 Капкейк":                {"мука": 1, "шоколад": 1, "сахар": 1, "масло": 1},
}

MENUS = {
    "c": {"🍕 Мини-пицца Миньон": (350, 5), "🥤 Коктейль Бр-бр потопим": (150, 2), "🧀 Сырные шарики": (200, 3)},
    "t": {"🍕 Пицца Пепперони": (380, 5), "🥛 Молочный коктейль": (250, 2), "🌭 Корн-дог": (150, 3)},
    "a": {"🥤 Кола": (100, 1), "🍕 Пицца 4 сыра": (580, 6), "🧁 Капкейк": (100, 2)},
}

DEFAULT_STOCK = {
    "тесто": 10, "сыр": 20, "томатный соус": 10, "пепперони": 10,
    "молоко": 10, "сироп": 5, "лёд": 20, "мука": 10, "масло": 10,
    "мороженое": 5, "сосиска": 5, "кола-сироп": 5, "вода": 20,
    "шоколад": 5, "сахар": 10,
}

BANNER = """
██████  ██  ██████  ██████   ████
██      ██     ██      ██   ██  ██
████    ██    ██     ██    ██████
██      ██   ██     ██     ██  ██
██      ██  ██████  ██████  ██  ██
"""


def init():
    # меню всегда берём из кода — чтобы названия совпадали с RECIPES
    json.dump(MENUS, open(MENU_FILE, "w", encoding="utf-8"), ensure_ascii=False, indent=4)
    # заказы и склад создаём только если их нет
    for path, data in ((FILE, []), (STOCK_FILE, DEFAULT_STOCK)):
        if not os.path.exists(path):
            json.dump(data, open(path, "w", encoding="utf-8"), ensure_ascii=False, indent=4)


def load(path, fb):
    try:
        return json.load(open(path, encoding="utf-8"))
    except Exception:
        json.dump(fb, open(path, "w", encoding="utf-8"), ensure_ascii=False, indent=4)
        return fb


def save(path, data):
    json.dump(data, open(path, "w", encoding="utf-8"), ensure_ascii=False, indent=4)


async def check_admin(pwd):
    raw = base64.b64decode(CIPHER_PAYLOAD)
    return await asyncio.to_thread(lambda: pwd == "".join(chr(b ^ CIPHER_KEY) for b in raw))


def can_cook(dish, stock):
    recipe = RECIPES.get(dish)
    if recipe is None:
        return False, "нет рецепта"
    for ing, need in recipe.items():
        if stock.get(ing, 0) < need:
            return False, ing
    return True, None


async def cook(gid, dish, sec, stock):
    print(f"   👨‍🍳 Готовим «{dish}» для гостя {gid} ({sec} сек)...")
    await asyncio.sleep(sec)
    print(f"   ✅ «{dish}» для гостя {gid} готово!")


def show_history():
    h = load(FILE, [])
    print("\n🔐 ИСТОРИЯ ЗАКАЗОВ")
    if not h:
        print("   📭 История пуста.")
        return
    print(f"   Всего заказов: {len(h)} | Выручка: {sum(o['total_price'] for o in h)} руб.\n")
    for i, o in enumerate(h, 1):
        print(f"   🧾 Заказ №{i} | {o.get('time', '—')} | {o['payment_method']} | {o['total_price']} руб.")
        for g in o["guests"]:
            print(f"      • Гость {g['guest_id']} ({g['age']} лет): {', '.join(g['chosen_items']) or 'ничего не выбрано'}")
        print()


def show_stock(stock):
    print("\n📦 СКЛАД:")
    for ing, qty in sorted(stock.items()):
        bar = "█" * min(qty, 20)
        print(f"   {ing:<15} {qty:>3} шт.  {bar}")
    print()


def admin_panel():
    stock = load(STOCK_FILE, DEFAULT_STOCK)
    while True:
        print("\n🔐 АДМИН-ПАНЕЛЬ")
        print("   1. История заказов")
        print("   2. Склад (остатки)")
        print("   3. Пополнить ингредиенты")
        print("   0. Выход из админки")
        c = input("   Выберите пункт: ").strip()

        if c == "1":
            show_history()
        elif c == "2":
            show_stock(stock)
        elif c == "3":
            show_stock(stock)
            ings = list(stock.keys())
            print("   Доступные ингредиенты:")
            for n, ing in enumerate(ings, 1):
                print(f"   {n}. {ing} (сейчас {stock[ing]})")
            while True:
                try:
                    n = int(input("   Номер ингредиента (0 — назад): "))
                    if n == 0:
                        break
                    if 1 <= n <= len(ings):
                        qty = int(input(f"   Сколько добавить «{ings[n-1]}»? "))
                        if qty > 0:
                            stock[ings[n - 1]] += qty
                            save(STOCK_FILE, stock)
                            print(f"   ✔ Добавлено +{qty} «{ings[n-1]}» (теперь {stock[ings[n-1]]})")
                        else:
                            print("   ⚠️ Нужно положительное число.")
                    else:
                        print("   ⚠️ Нет такого номера.")
                except ValueError:
                    print("   ⚠️ Введите число.")
        elif c == "0":
            save(STOCK_FILE, stock)
            print("   👋 Вышли из админки.")
            return
        else:
            print("   ⚠️ Нет такого пункта.")


async def make_order():
    menus = load(MENU_FILE, MENUS)
    stock = load(STOCK_FILE, DEFAULT_STOCK)

    while True:
        try:
            people = int(input("\n   👥 Сколько вас человек? "))
            if people > 0: break
        except ValueError: pass
        print("   ⚠️ Введите целое число больше нуля.")

    guests, total, tasks = [], 0, []
    for i in range(1, people + 1):
        print(f"\n   🙋 ГОСТЬ №{i}")
        while True:
            try:
                age = int(input("   🎂 Сколько вам лет? "))
                if 0 <= age <= 120: break
            except ValueError: pass
            print("   ⚠️ Введите реальный возраст (0–120).")

        menu = dict(menus["c"])
        if age >= 12: menu.update(menus["t"])
        if age >= 18: menu.update(menus["a"])

        items = list(menu)
        print("\n   📋 Доступное меню:")
        for n, k in enumerate(items, 1):
            ok, bad = can_cook(k, stock)
            mark = "" if ok else f"  ❌ не хватает: {bad}"
            print(f"   {n}. {k} — {menu[k][0]} руб. (готовится {menu[k][1]} сек){mark}")

        chosen = []
        while True:
            c = input("   Номер блюда (0 — закончить): ").strip()
            if c == "0": break
            if c.isdigit() and 1 <= int(c) <= len(items):
                dish = items[int(c) - 1]
                ok, bad = can_cook(dish, stock)
                if not ok:
                    print(f"   ⚠️ На кухне не хватает «{bad}».")
                    continue
                for ing, need in RECIPES[dish].items():
                    stock[ing] -= need
                save(STOCK_FILE, stock)
                chosen.append(dish); total += menu[dish][0]
                tasks.append(cook(i, dish, menu[dish][1], stock))
                print(f"   ✔ Добавлено: {dish} ({menu[dish][0]} руб.)")
            else:
                print("   ⚠️ Неверный номер блюда.")
        guests.append({"guest_id": i, "age": age, "chosen_items": chosen})

    print(f"\n   💰 Итого к оплате: {total} руб.")
    while True:
        pay = input("   💳 Способ оплаты (карта / наличные): ").strip().lower()
        if pay in ("карта", "наличные"): break
        print("   ⚠️ Введите 'карта' или 'наличные'.")

    if tasks:
        print("\n   👨‍🍳 Заказ передан на кухню, готовим...\n")
        await asyncio.gather(*tasks)
        print("\n   🎉 Все блюда готовы!")

    h = load(FILE, [])
    h.append({"time": datetime.now().strftime("%d.%m %H:%M"), "payment_method": pay, "total_price": total, "guests": guests})
    save(FILE, h)
    print("\n   🧾 Чек сохранён в историю. Спасибо за заказ!\n")


async def main():
    init()
    print(BANNER)
    print("   🍕 Добро пожаловать в пиццерию FIZZA! 🍕\n")

    while True:
        role = input("   Введите 'admin' для админки, 'заказ' для заказа или 'выход': ").strip().lower()

        if role == "admin":
            if await check_admin(input("   🔑 Введите пароль администратора: ").strip()):
                admin_panel()
            else:
                print("   ❌ Неверный пароль! Доступ запрещён.")
        elif role in ("заказ", "order", ""):
            await make_order()
            again = input("   Сделать ещё один заказ? (да/нет): ").strip().lower()
            if again not in ("да", "д", "yes", "y"):
                print("   👋 До свидания!")
                return
        elif role == "выход":
            print("   👋 До свидания!")
            return
        else:
            print("   ⚠️ Напишите 'admin', 'заказ' или 'выход'.")


if __name__ == "__main__":
    asyncio.run(main())
