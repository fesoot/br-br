banner = """
████████  ██  █████████  █████████     ██████   
██        ██         ██         ██    ████████  
██████    ██       ███        ███    ███    ███ 
██        ██     ███        ███     ███████████ 
██        ██  █████████  █████████  ███     ███ 
"""

print(banner)
import asyncio
import json
import os
import base64

FILE = "pizza_orders.json"
MENU_FILE = "menu.json"
CIPHER_PAYLOAD = "e3tx"
CIPHER_KEY = 73

DEFAULT_MENUS = {
    "c": {
        "пицца миньон": (350, 5),
        "коктейль бр бр потопим": (150, 2),
        "сырные шарики": (200, 3)
    },
    "t": {
        "пицца пеперони": (380, 5),
        "коктейль молочный": (250, 2),
        "корндог": (150, 3)
    },
    "a": {
        "кола": (100, 1),
        "пицца 4 сына": (580, 6),
        "капкейк": (100, 2)
    }
}


def init_files():
    if not os.path.exists(MENU_FILE):
        with open(MENU_FILE, "w", encoding="utf-8") as f:
            json.dump(DEFAULT_MENUS, f, ensure_ascii=False, indent=4)
    if not os.path.exists(FILE):
        with open(FILE, "w", encoding="utf-8") as f:
            json.dump([], f, ensure_ascii=False, indent=4)


def load_json(path, fallback):
    try:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        with open(path, "w", encoding="utf-8") as f:
            json.dump(fallback, f, ensure_ascii=False, indent=4)
        return fallback


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=4)


async def async_decrypt_and_verify(user_input: str) -> bool:
    def _crypto_worker():
        raw_bytes = base64.b64decode(CIPHER_PAYLOAD)
        decrypted_pass = "".join(chr(b ^ CIPHER_KEY) for b in raw_bytes)
        return user_input == decrypted_pass

    return await asyncio.to_thread(_crypto_worker)


async def cook_dish(guest_id, name, sec):
    print(f"  [Кухня] Начали готовить '{name}' для гостя {guest_id} ({sec} сек)...")
    await asyncio.sleep(sec)
    print(f"  [Готово!] '{name}' для гостя {guest_id} можно забирать!")


async def main():
    init_files()
    menus = load_json(MENU_FILE, DEFAULT_MENUS)

    print("ДОБРО ПОЖАЛОВАТЬ В ПИЦЦЕРИЮ")
    role = input("Напишите admin для входа в историю заказов или нажмите Enter для заказа: ").strip().lower()

    if role == "admin":
        entered_pass = input("Введите код доступа администратора: ").strip()
        is_valid = await async_decrypt_and_verify(entered_pass)

        if not is_valid:
            print("Неверный код! Доступ запрещён.")
            return

        print("\n--- ПАНЕЛЬ АДМИНИСТРАТОРА: ИСТОРИЯ ВСЕХ ЗАКАЗОВ ---")
        history = load_json(FILE, [])
        if not history:
            print("История заказов пуста.")
        else:
            for idx, order in enumerate(history, 1):
                print(f"\nЗаказ №{idx}")
                print(f"  Способ оплаты: {order['payment_method']}")
                print(f"  Итоговая стоимость: {order['total_price']} руб.")
                print("  Детали по гостям:")
                for g in order["guests"]:
                    items_str = ", ".join(g["chosen_items"]) or "ничего не выбрано"
                    print(f"    Гость №{g['guest_id']} (Возраст: {g['age']}): {items_str}")
        return

    while True:
        try:
            people = int(input("\nСколько вас будет? "))
            if people > 0: break
        except ValueError:
            pass
        print("Введите целое число больше нуля.")

    guests, total, tasks = [], 0, []

    for i in range(1, people + 1):
        print(f"\n-- Идентификация гостя {i} --")
        while True:
            try:
                age = int(input("Сколько вам лет? "))
                if 0 <= age <= 120: break
            except ValueError:
                pass
            print("Введите реальный возраст.")

        menu = dict(menus.get("c", {}))
        if age >= 12: menu.update(menus.get("t", {}))
        if age >= 18: menu.update(menus.get("a", {}))

        items = list(menu.keys())
        print("Доступное меню:")
        for n, k in enumerate(items, 1):
            price, sec = menu[k]
            print(f"  {n}. {k} — {price} руб. (готовится {sec} сек)")

        chosen = []
        while True:
            c = input("Номер блюда (0 - закончить): ").strip()
            if c == "0": break
            if c.isdigit() and 1 <= int(c) <= len(items):
                dish = items[int(c) - 1]
                price, sec = menu[dish]
                chosen.append(dish)
                total += price
                tasks.append(cook_dish(i, dish, sec))
                print(f"+ {dish} ({price} руб.)")
            else:
                print("Неверный номер блюда.")
        guests.append({"guest_id": i, "age": age, "chosen_items": chosen})

    print(f"\nИтого к оплате: {total} руб.")
    while True:
        pay = input("Способ оплаты (карта / наличные): ").strip().lower()
        if pay in ("карта", "наличные"): break
        print("Введите 'карта' или 'наличные'.")

    if tasks:
        print("\n--- ЗАКАЗ ПЕРЕДАН НА КУХНЮ ---")
        await asyncio.gather(*tasks)
        print("--- ВСЕ БЛЮДА ГОТОВЫ! ---")

    history = load_json(FILE, [])
    history.append({"payment_method": pay, "total_price": total, "guests": guests})
    save_json(FILE, history)

    print("Чек сохранён в историю. Спасибо за заказ!")


asyncio.run(main())
