#!/usr/bin/env python3
"""
QuickBite: Food Delivering System  -  Console Prototype
HCI 101 - Group 6

A text-based version of the QuickBite Figma prototype (UM Campus Canteen).
Students browse the canteen menu, build a cart, place an order in advance,
track its status, and claim it at the canteen when it is ready.

Screens (same as the Figma prototype):
    Home  |  Menu  |  Search  |  Cart/Checkout  |  Order Confirmation
    My Orders  |  Order Details  |  Account

Run it with:
    python quickbite_console.py

Navigation (works on every screen, like the bottom tab bar):
    H = Home   M = Menu   C = Cart   O = Orders   A = Account   Q = Quit
Add an item by typing its code, e.g.  M1   or   M1 2  (two of M1).
"""

import os
import re
import sys
from datetime import datetime, timedelta


# SETTINGS
# WElcome to my kainan
WIDTH = 64
PREP_SECONDS = int(os.environ.get("QUICKBITE_PREP_SECONDS", 60))  # "Preparing" time (short, for demos)
MAX_QTY = 10           # maximum quantity of a single item in the cart
CATEGORIES = ["Meals", "Snacks", "Drinks", "Desserts"]
ORDER_FILTERS = ["All", "Preparing", "Completed"]
PICKUP_POINTS = [
    "Main Counter",
    "Meals Stall Counter",
    "Drinks & Desserts Counter",
]

# TERMINAL HELPERS (colors, symbols, clearing the screen)

USE_COLOR = sys.stdout.isatty() and "NO_COLOR" not in os.environ
if USE_COLOR and os.name == "nt":
    os.system("")  # turns on ANSI colors in the Windows console


def _paint(text, code):
    return f"\033[{code}m{text}\033[0m" if USE_COLOR else text


def maroon(text):
    return _paint(text, "1;31")


def green(text):
    return _paint(text, "32")


def red(text):
    return _paint(text, "31")


def yellow(text):
    return _paint(text, "33")


def bold(text):
    return _paint(text, "1")


def _can_print(text):
    """True if the console can display this text (falls back otherwise)."""
    try:
        text.encode(sys.stdout.encoding or "ascii")
        return True
    except (UnicodeEncodeError, LookupError):
        return False


PESO = "\u20b1" if _can_print("\u20b1") else "PHP "
CHECK = "\u2714" if _can_print("\u2714") else "OK"


def money(amount):
    return f"{PESO}{amount:,.2f}"


def clear_screen():
    if sys.stdout.isatty():
        os.system("cls" if os.name == "nt" else "clear")
    else:
        print("\n" + "~" * WIDTH)


class QuitProgram(Exception):
    """Raised when the user quits (Q) or closes the input (Ctrl+C / Ctrl+D)."""


def ask(prompt="> "):
    try:
        return input(prompt).strip()
    except (EOFError, KeyboardInterrupt):
        print()
        raise QuitProgram
# DATA: MENU
def _item(code, name, category, price, available=True, popular=False):
    return {
        "code": code,
        "name": name,
        "category": category,
        "price": price,
        "available": available,
        "popular": popular,
    }
MENU = [
    # Meals
    _item("M1", "Chicken Adobo Rice", "Meals", 65, popular=True),
    _item("M2", "Siomai Rice", "Meals", 45, popular=True),
    _item("M3", "Pancit Canton", "Meals", 45, popular=True),
    _item("M4", "Lumpia Rice", "Meals", 50, available=False, popular=True),
    _item("M5", "Fried Chicken Rice", "Meals", 75),
    _item("M6", "Longganisa Silog", "Meals", 70),
    # Snacks
    _item("S1", "Siomai (4 pcs)", "Snacks", 30),
    _item("S2", "French Fries", "Snacks", 35),
    _item("S3", "Banana Cue", "Snacks", 15),
    _item("S4", "Turon", "Snacks", 20, available=False),
    # Drinks
    _item("D1", "Iced Tea", "Drinks", 25),
    _item("D2", "Lemonade", "Drinks", 30),
    _item("D3", "Bottled Water", "Drinks", 15),
    _item("D4", "Iced Coffee", "Drinks", 40, available=False),
    # Desserts
    _item("T1", "Halo-Halo", "Desserts", 45),
    _item("T2", "Leche Flan", "Desserts", 35),
    _item("T3", "Buko Pandan", "Desserts", 30, available=False),
    _item("T4", "Ice Cream Cup", "Desserts", 25),
]
def find_item(code):
    for item in MENU:
        if item["code"] == code:
            return item
    return None



# SESSION STATE

session = {
    "user": None,            # dict: name, student_id, email, pickup, notifications
    "cart": {},              # item code -> quantity
    "orders": [],            # all orders placed in this run (every user)
    "next_order_no": 123,
    "category": "Meals",
    "order_filter": "All",
    "visible_orders": [],
    "viewing": None,         # order reference shown on the Order Details screen
    "last_order": None,
    "flash": [],             # short messages shown at the top of the next screen
    "previous": "home",
}


def flash(message):
    session["flash"].append(message)


def show_flash():
    if session["flash"]:
        print()
        for message in session["flash"]:
            print(" >> " + message)
        session["flash"].clear()


def first_name(user):
    return user["name"].split()[0]


def make_email(name, student_id):
    """pablo.e.125@umindanao.edu.ph from 'Pablo E.' and '2026-00125'."""
    words = [re.sub(r"[^a-z]", "", w) for w in name.lower().split()]
    words = [w for w in words if w] or ["student"]
    return ".".join(words) + "." + student_id[-3:] + "@umindanao.edu.ph"


# --------------------------------------------------------------------------
# CART HELPERS
# --------------------------------------------------------------------------
def cart_total():
    return sum(find_item(code)["price"] * qty for code, qty in session["cart"].items())


def cart_count():
    return sum(session["cart"].values())


CODE_PATTERN = re.compile(r"^(?:add\s+)?([a-z]\d+)(?:\s+(?:x\s*)?(\d+))?$", re.I)



def try_add(raw):
    """Handle 'M1', 'M1 3' or 'add M1 3'. Returns True if raw looked like an item code."""
    match = CODE_PATTERN.match(raw)
    if not match:
        return False
    code = match.group(1).upper()
    qty = int(match.group(2)) if match.group(2) else 1
    item = find_item(code)
    if item is None:
        flash(f"No item with code {code}. Use the code beside the item name.")
    elif not item["available"]:
        flash(f"Sorry, {item['name']} is sold out right now.")
    elif qty < 1:
        flash("Quantity must be at least 1.")
    else:
        have = session["cart"].get(code, 0)
        if have + qty > MAX_QTY:
            flash(f"You can order at most {MAX_QTY} of one item.")
        else:
            session["cart"][code] = have + qty
            flash(f"Added {qty} x {item['name']} to your cart.")
    return True


def change_qty(code, delta):
    cart = session["cart"]
    item = find_item(code)
    new_qty = cart.get(code, 0) + delta
    if new_qty <= 0:
        cart.pop(code, None)
        flash(f"Removed {item['name']} from your cart.")
    elif new_qty > MAX_QTY:
        flash(f"You can order at most {MAX_QTY} of one item.")
    else:
        cart[code] = new_qty


# --------------------------------------------------------------------------
# ORDER HELPERS
# --------------------------------------------------------------------------
def new_order_ref():
    ref = f"QB-{session['next_order_no']:05d}"
    session["next_order_no"] += 1
    return ref


def make_order(user, lines, placed_at, ready_at, notified=False):
    """lines = list of (name, qty, unit_price)."""
    items = [{"name": n, "qty": q, "price": p} for n, q, p in lines]
    return {
        "ref": new_order_ref(),
        "student_id": user["student_id"],
        "items": items,
        "total": sum(i["qty"] * i["price"] for i in items),
        "placed_at": placed_at,
        "ready_at": ready_at,
        "pickup": user["pickup"],
        "notified": notified,
    }


def place_order():
    user = session["user"]
    now = datetime.now()
    lines = []
    for code, qty in session["cart"].items():
        item = find_item(code)
        lines.append((item["name"], qty, item["price"]))
    order = make_order(user, lines, now, now + timedelta(seconds=PREP_SECONDS))
    session["orders"].append(order)
    session["cart"].clear()
    session["last_order"] = order


def seed_demo_orders(user):
    """Give a new account two sample orders so My Orders is not empty."""
    if any(o["student_id"] == user["student_id"] for o in session["orders"]):
        return
    now = datetime.now()
    yesterday = now - timedelta(days=1)
    session["orders"].append(make_order(
        user, [("Siomai Rice", 2, 45)],
        yesterday, yesterday + timedelta(minutes=10), notified=True))
    just_placed = now - timedelta(seconds=PREP_SECONDS // 4)
    session["orders"].append(make_order(
        user, [("Chicken Adobo Rice", 1, 65)],
        just_placed, just_placed + timedelta(seconds=PREP_SECONDS)))


def order_status(order):
    return "Completed" if datetime.now() >= order["ready_at"] else "Preparing"


def my_orders():
    uid = session["user"]["student_id"]
    mine = [o for o in session["orders"] if o["student_id"] == uid]
    return sorted(mine, key=lambda o: o["placed_at"], reverse=True)


def date_text(moment):
    return f"{moment:%b} {moment.day}, {moment.year}"


def friendly_wait(delta):
    secs = max(0, int(delta.total_seconds()))
    if secs >= 60:
        return f"{secs // 60} min {secs % 60:02d} sec"
    return f"{secs} sec"


def check_ready_notifications():
    """Tell the student when one of their orders has just become ready."""
    user = session["user"]
    if not user:
        return
    for order in session["orders"]:
        if (order["student_id"] == user["student_id"]
                and not order["notified"]
                and order_status(order) == "Completed"):
            order["notified"] = True
            if user["notifications"]:
                flash(f"Order {order['ref']} is ready! Head to the "
                      f"{order['pickup']} to claim it.")

# SHARED LAYOUT: header, tab bar, item list
def header(title=None):
    clear_screen()
    check_ready_notifications()
    print("=" * WIDTH)
    left_plain = "QuickBite  |  UM CAMPUS CANTEEN"
    right = f"[C] Cart ({cart_count()})"
    gap = WIDTH - len(left_plain) - len(right)
    print(maroon("QuickBite") + "  |  UM CAMPUS CANTEEN" + " " * gap + right)
    print("=" * WIDTH)
    if title:
        print()
        print(maroon(title))
    show_flash()
    print()


def tab_bar(current):
    print("-" * WIDTH)
    tabs = [("home", "H", "Home"), ("menu", "M", "Menu"),
            ("orders", "O", "Orders"), ("account", "A", "Account")]
    parts = []
    for key, letter, label in tabs:
        text = f"[{letter}] {label}"
        parts.append(maroon(text) if key == current else text)
    print("   ".join(parts) + "   [Q] Quit")


def box(lines):
    inner = WIDTH - 8
    print("  +" + "-" * (inner + 2) + "+")
    for line in lines:
        print("  | " + line.ljust(inner) + " |")
    print("  +" + "-" * (inner + 2) + "+")


def print_items(items):
    if not items:
        print("  Nothing to show here.")
        return
    for item in items:
        tag = green("[Available]") if item["available"] else red("[Sold Out]")
        print(f"  {item['code']:<3} {item['name']:<22} {money(item['price']):>9}   {tag}")


def category_tabs(selected=None):
    parts = []
    for i, name in enumerate(CATEGORIES, 1):
        text = f"[{i}] {name}"
        parts.append(maroon(text) if name == selected else text)
    return "   ".join(parts)


NAV_KEYS = {
    "h": "home", "home": "home",
    "m": "menu", "menu": "menu",
    "c": "cart", "cart": "cart",
    "o": "orders", "orders": "orders",
    "a": "account", "account": "account",
    "q": "quit", "quit": "quit",
}


def handle_common(raw, current, allow_add=True):
    """Tab-bar navigation and 'type an item code to add it', shared by all screens."""
    key = raw.lower()
    if key in NAV_KEYS:
        return NAV_KEYS[key]
    if raw == "":
        return current
    if allow_add and try_add(raw):
        return current
    flash(f"Sorry, '{raw}' is not an option on this screen.")
    return current


# SCREEN: LOGIN
def screen_login():
    clear_screen()
    print("=" * WIDTH)
    print(maroon("QuickBite".center(WIDTH)))
    print("UM CAMPUS CANTEEN".center(WIDTH))
    print("Food Delivering System  -  Skip the line, enjoy the bite".center(WIDTH))
    print("=" * WIDTH)
    show_flash()
    print()
    print("  Log in with your University of Mindanao student account.")
    print("  (Just press Enter to use the demo account: Pablo E. / 2026-00125)")
    print()
    name = ask("  Full name: ")
    if name:
        while True:
            student_id = ask("  Student ID (format 2026-00125): ")
            if re.match(r"^\d{4}-\d{5}$", student_id):
                break
            print("  Please use the format YYYY-NNNNN, for example 2026-00125.")
    else:
        name, student_id = "Pablo E.", "2026-00125"

    session["user"] = {
        "name": name,
        "student_id": student_id,
        "email": make_email(name, student_id),
        "pickup": PICKUP_POINTS[0],
        "notifications": True,
    }
    session["cart"].clear()
    seed_demo_orders(session["user"])
    return "home"


# --------------------------------------------------------------------------
# SCREEN: HOME
# --------------------------------------------------------------------------
def screen_home():
    user = session["user"]
    header()
    print(maroon(f"Good day, {first_name(user)}!"))
    print()
    print("  [S] Search  -  What are you craving?")
    print()
    box(["LUNCH BREAK",
         f"Time to recharge, {first_name(user)}!",
         "[T] Today's specials  ->"])
    print()
    print(bold("Find your favourite"))
    print("  " + category_tabs())
    print()
    print(bold("Popular for lunch") + "                           [V] View menu")
    print_items([i for i in MENU if i["popular"]])
    print()
    print("  Type an item code to add it to your cart (e.g. M1, or M1 2).")
    tab_bar("home")
    raw = ask("> ")
    low = raw.lower()

    if low in ("1", "2", "3", "4"):
        session["category"] = CATEGORIES[int(low) - 1]
        return "menu"
    if low == "s":
        return "search"
    if low == "t":
        session["category"] = "Meals"
        return "menu"
    if low == "v":
        return "menu"
    return handle_common(raw, "home")


# --------------------------------------------------------------------------
# SCREEN: MENU
# --------------------------------------------------------------------------
def screen_menu():
    header("Our Menu")
    print("  [S] Search  -  What are you craving?")
    print()
    print(bold("Find your favourite"))
    print("  " + category_tabs(session["category"]))
    print()
    print(bold(session["category"]))
    print_items([i for i in MENU if i["category"] == session["category"]])
    print()
    print("  Type an item code to add it (e.g. M1, or M1 2). 1-4 switches category.")
    tab_bar("menu")
    raw = ask("> ")
    low = raw.lower()

    if low in ("1", "2", "3", "4"):
        session["category"] = CATEGORIES[int(low) - 1]
        return "menu"
    if low == "s":
        return "search"
    return handle_common(raw, "menu")


# --------------------------------------------------------------------------
# SCREEN: SEARCH
# --------------------------------------------------------------------------
def screen_search():
    back = session["previous"] if session["previous"] in ("home", "menu") else "home"
    query = None
    while True:
        header("Search")
        if query is None:
            raw = ask("  What are you craving? (type a keyword, Enter to go back): ")
            if raw == "":
                return back
            query = raw
            continue

        needle = query.lower()
        results = [i for i in MENU
                   if needle in i["name"].lower() or needle in i["category"].lower()]
        print(f'  Results for "{query}":')
        print()
        if results:
            print_items(results)
        else:
            print("  No matches. Try a keyword like 'rice', 'tea' or 'snacks'.")
        print()
        print("  Type an item code to add it.  [S] New search   [B] Back")
        tab_bar("menu")
        raw = ask("> ")
        low = raw.lower()

        if low == "s":
            query = None
        elif low in ("b", ""):
            return back
        elif low in NAV_KEYS:
            return NAV_KEYS[low]
        elif not try_add(raw):
            flash(f"Sorry, '{raw}' is not an option on this screen.")


# --------------------------------------------------------------------------
# SCREEN: CART / CHECKOUT
# --------------------------------------------------------------------------
def screen_cart():
    header("Checkout")
    print(bold("My Cart"))
    cart = session["cart"]
    if not cart:
        print()
        print("  Your cart is empty.")
        print("  Open the [M] Menu and add something tasty!")
    else:
        print()
        for n, (code, qty) in enumerate(cart.items(), 1):
            item = find_item(code)
            print(f"  {n}. {item['name']:<22} {money(item['price']):>9}   [-] {qty} [+]")
        print()
        print(bold("Order Summary"))
        for code, qty in cart.items():
            item = find_item(code)
            label = f"{item['name']} x {qty}"
            print(f"  {label:<38}{money(item['price'] * qty):>12}")
        print("  " + "-" * 50)
        print(bold(f"  {'Total':<38}{money(cart_total()):>12}"))
        print()
        print("  [+ n] one more   [- n] one less   [R n] remove item n")
        print("  [CLEAR] empty cart              [P] Place Order ->")
    tab_bar("cart")
    raw = ask("> ")
    low = raw.lower()

    match = re.match(r"^(\+|-|r|rm|remove)\s*(\d+)$", low)
    if match and cart:
        position = int(match.group(2))
        if 1 <= position <= len(cart):
            code = list(cart)[position - 1]
            action = match.group(1)
            if action == "+":
                change_qty(code, 1)
            elif action == "-":
                change_qty(code, -1)
            else:
                name = find_item(code)["name"]
                del cart[code]
                flash(f"Removed {name} from your cart.")
        else:
            flash(f"There is no item number {position} in your cart.")
        return "cart"

    if low == "clear" and cart:
        cart.clear()
        flash("Your cart is now empty.")
        return "cart"

    if low in ("p", "place", "place order"):
        if not cart:
            flash("Your cart is empty. Add something first!")
            return "cart"
        answer = ask(f"  Place this order for {money(cart_total())}? (y/n): ").lower()
        if answer in ("y", "yes"):
            place_order()
            return "confirmation"
        flash("Order not placed. Your cart has been kept.")
        return "cart"

    return handle_common(raw, "cart")


# --------------------------------------------------------------------------
# SCREEN: ORDER CONFIRMATION
# --------------------------------------------------------------------------
def screen_confirmation():
    order = session["last_order"]
    if order is None:
        return "home"
    header()
    print(green(f"( {CHECK} )".center(WIDTH)))
    print()
    print(bold("Order Placed!".center(WIDTH)))
    print("Your order has been successfully placed.".center(WIDTH))
    print()
    print("-" * WIDTH)
    print("ORDER REFERENCE".center(WIDTH))
    print(maroon(order["ref"].center(WIDTH)))
    print()
    print(f"  {'Total Amount':<40}{money(order['total']):>20}")
    print(f"  {'Pickup point':<30}{order['pickup']:>30}")
    print(f"  Estimated prep time: about {friendly_wait(order['ready_at'] - order['placed_at'])}")
    print()
    print("  [1] View My Orders")
    print("  [2] Back to Home")
    tab_bar("orders")
    raw = ask("> ")
    if raw == "1":
        return "orders"
    if raw == "2":
        return "home"
    return handle_common(raw, "confirmation", allow_add=False)


# --------------------------------------------------------------------------
# SCREEN: MY ORDERS + ORDER DETAILS
# --------------------------------------------------------------------------
def screen_orders():
    header("My Orders")
    current = session["order_filter"]
    tabs = []
    for i, name in enumerate(ORDER_FILTERS, 1):
        text = f"[{i}] {name}"
        tabs.append(maroon(text) if name == current else text)
    print("  " + "   ".join(tabs))
    print()
    print(bold("Orders"))

    orders = [o for o in my_orders() if current == "All" or order_status(o) == current]
    session["visible_orders"] = orders
    if not orders:
        print()
        print("  No orders to show here yet.")
    for n, order in enumerate(orders, 1):
        status = order_status(order)
        count = sum(i["qty"] for i in order["items"])
        title = order["items"][0]["name"]
        extra = len(order["items"]) - 1
        if extra:
            title += f" +{extra} more"
        badge = yellow("[Preparing]") if status == "Preparing" else green("[Completed]")
        status_line = (yellow("Preparing your order") if status == "Preparing"
                       else green("Order Completed"))
        print()
        print("  " + f"{n}. {title}".ljust(46) + badge)
        print(f"     {order['ref']} | {count} item{'s' if count != 1 else ''} "
              f"| {date_text(order['placed_at'])}")
        print(f"     Total: {bold(money(order['total']))}")
        print(f"     Order Status: {status_line}")
    print()
    print("  [V n] View details of order n     [Enter] Refresh status")
    tab_bar("orders")
    raw = ask("> ")
    low = raw.lower()

    if low in ("1", "2", "3"):
        session["order_filter"] = ORDER_FILTERS[int(low) - 1]
        return "orders"
    match = re.match(r"^(?:v|view)\s*(\d+)$", low)
    if match:
        position = int(match.group(1))
        if 1 <= position <= len(orders):
            session["viewing"] = orders[position - 1]["ref"]
            return "order_details"
        flash(f"There is no order number {position} in this list.")
        return "orders"
    return handle_common(raw, "orders", allow_add=False)


def screen_order_details():
    order = next((o for o in session["orders"] if o["ref"] == session["viewing"]), None)
    if order is None:
        return "orders"
    header("Order Details")
    placed = order["placed_at"]
    status = order_status(order)
    print(f"  Order reference : {bold(order['ref'])}")
    print(f"  Placed          : {date_text(placed)}, {placed:%I:%M %p}")
    print(f"  Pickup point    : {order['pickup']}")
    print()
    for item in order["items"]:
        label = f"{item['name']} x {item['qty']}"
        print(f"  {label:<38}{money(item['price'] * item['qty']):>12}")
    print("  " + "-" * 50)
    print(bold(f"  {'Total':<38}{money(order['total']):>12}"))
    print()
    if status == "Preparing":
        remaining = friendly_wait(order["ready_at"] - datetime.now())
        print("  Order Status: " + yellow("Preparing your order"))
        print(f"  Ready in about {remaining}. We'll notify you when it's done.")
    else:
        print("  Order Status: " + green("Order Completed"))
        print(f"  Show {order['ref']} at the {order['pickup']} to claim your food.")
    print()
    print("  [Enter] Refresh   [B] Back to My Orders")
    tab_bar("orders")
    raw = ask("> ")
    if raw.lower() == "b":
        return "orders"
    return handle_common(raw, "order_details", allow_add=False)


# --------------------------------------------------------------------------
# SCREEN: ACCOUNT (+ sub-screens)
# --------------------------------------------------------------------------
def screen_account():
    user = session["user"]
    header("Account")
    print(bold(f"  {user['name']}"))
    print(f"  {user['email']}")
    print(f"  Student ID: {user['student_id']}")
    print()
    rows = [
        ("Personal Information", "Manage your profile"),
        ("My Orders", "View your order history"),
        ("Pickup Information", "View your pickup preferences"),
        ("Notifications", "Manage notifications"),
        ("Help & Support", "Get assistance"),
    ]
    for i, (title, subtitle) in enumerate(rows, 1):
        print(f"  [{i}] {title:<22} {subtitle}")
    print()
    print("  [6] Log Out")
    tab_bar("account")
    raw = ask("> ")
    destinations = {"1": "personal", "2": "orders", "3": "pickup",
                    "4": "notifications", "5": "help"}
    if raw in destinations:
        return destinations[raw]
    if raw == "6":
        session["user"] = None
        session["cart"].clear()
        flash("You have been logged out.")
        return "login"
    return handle_common(raw, "account", allow_add=False)


def screen_personal():
    user = session["user"]
    header("Personal Information")
    print(f"  Name        : {user['name']}")
    print(f"  Email       : {user['email']}")
    print(f"  Student ID  : {user['student_id']}")
    print()
    print("  [E] Edit name   [B] Back")
    tab_bar("account")
    raw = ask("> ")
    if raw.lower() == "e":
        new_name = ask("  New name (Enter to keep current): ")
        if new_name:
            user["name"] = new_name
            user["email"] = make_email(new_name, user["student_id"])
            flash("Name updated.")
        return "personal"
    if raw.lower() in ("b", ""):
        return "account"
    return handle_common(raw, "personal", allow_add=False)


def screen_pickup():
    user = session["user"]
    header("Pickup Information")
    print("  Where would you like to claim your orders?")
    print()
    for i, point in enumerate(PICKUP_POINTS, 1):
        mark = maroon("  (selected)") if point == user["pickup"] else ""
        print(f"  [{i}] {point}{mark}")
    print()
    print("  Type a number to change it, or [B] to go back.")
    tab_bar("account")
    raw = ask("> ")
    if raw.isdigit() and 1 <= int(raw) <= len(PICKUP_POINTS):
        user["pickup"] = PICKUP_POINTS[int(raw) - 1]
        flash(f"Pickup point set to {user['pickup']}.")
        return "pickup"
    if raw.lower() in ("b", ""):
        return "account"
    return handle_common(raw, "pickup", allow_add=False)


def screen_notifications():
    user = session["user"]
    header("Notifications")
    state = green("ON") if user["notifications"] else red("OFF")
    print(f"  Order status alerts: {state}")
    print("  You'll be told when your order is ready to claim.")
    print()
    print("  [T] Turn on/off   [B] Back")
    tab_bar("account")
    raw = ask("> ")
    if raw.lower() == "t":
        user["notifications"] = not user["notifications"]
        flash("Notifications are now " + ("ON." if user["notifications"] else "OFF."))
        return "notifications"
    if raw.lower() in ("b", ""):
        return "account"
    return handle_common(raw, "notifications", allow_add=False)


def screen_help():
    header("Help & Support")
    print("  How do I order?")
    print("    1. Browse the Menu or use Search to find your food.")
    print("    2. Type an item code (e.g. M1) to add it to your cart.")
    print("    3. Open the Cart, review your total, then Place Order.")
    print("    4. Track it under Orders, and claim it at the canteen")
    print("       using your order reference (e.g. QB-00125).")
    print()
    print("  Why can't I add an item?")
    print("    Items tagged [Sold Out] can't be ordered right now.")
    print()
    print("  Need more help? Visit the UM Campus Canteen counter.")
    print()
    print("  [B] Back")
    tab_bar("account")
    raw = ask("> ")
    if raw.lower() in ("b", ""):
        return "account"
    return handle_common(raw, "help", allow_add=False)


# --------------------------------------------------------------------------
# MAIN LOOP
# --------------------------------------------------------------------------
SCREENS = {
    "login": screen_login,
    "home": screen_home,
    "menu": screen_menu,
    "search": screen_search,
    "cart": screen_cart,
    "confirmation": screen_confirmation,
    "orders": screen_orders,
    "order_details": screen_order_details,
    "account": screen_account,
    "personal": screen_personal,
    "pickup": screen_pickup,
    "notifications": screen_notifications,
    "help": screen_help,
}


def main():
    current = "login"
    try:
        while current != "quit":
            next_screen = SCREENS[current]()
            if next_screen != current:
                session["previous"] = current
            current = next_screen
    except QuitProgram:
        pass
    print()
    print("Thanks for using QuickBite! Skip the line, enjoy the bite.")


if __name__ == "__main__":
    main()
