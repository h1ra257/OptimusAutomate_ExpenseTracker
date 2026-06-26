# OptimusAutomate_ExpenseTracker
"""
============================================================
  OPTIMUS AUTOMATE — Virtual Internship Program
  Python Programming Internship
  Task 1: Expense Tracker CLI App

  Developer : Hira Sarwar
  Program   : Python Programming
============================================================
"""

import json
import os
import sqlite3
from datetime import datetime

DB_FILE = "expenses.db"

CATEGORIES = [
    "Food", "Transport", "Shopping", "Entertainment",
    "Health", "Education", "Utilities", "Other"
]


# ─── Database Setup ───────────────────────────────────────

def init_db():
    conn = sqlite3.connect(DB_FILE)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS expenses (
            id        INTEGER PRIMARY KEY AUTOINCREMENT,
            date      TEXT    NOT NULL,
            category  TEXT    NOT NULL,
            amount    REAL    NOT NULL,
            note      TEXT
        )
    """)
    conn.commit()
    conn.close()


def get_conn():
    return sqlite3.connect(DB_FILE)


# ─── Core CRUD ────────────────────────────────────────────

def add_expense(date: str, category: str, amount: float, note: str = ""):
    conn = get_conn()
    conn.execute(
        "INSERT INTO expenses (date, category, amount, note) VALUES (?, ?, ?, ?)",
        (date, category, amount, note)
    )
    conn.commit()
    conn.close()
    print(f"\n  ✓ Expense added: {category} — PKR {amount:.2f} on {date}")


def view_expenses(month: str | None = None):
    conn = get_conn()
    if month:
        rows = conn.execute(
            "SELECT id, date, category, amount, note FROM expenses WHERE date LIKE ? ORDER BY date",
            (f"{month}%",)
        ).fetchall()
    else:
        rows = conn.execute(
            "SELECT id, date, category, amount, note FROM expenses ORDER BY date"
        ).fetchall()
    conn.close()

    if not rows:
        print("\n  No expenses found.")
        return

    print(f"\n  {'ID':<5} {'Date':<12} {'Category':<15} {'Amount (PKR)':<15} Note")
    print("  " + "-" * 65)
    for r in rows:
        print(f"  {r[0]:<5} {r[1]:<12} {r[2]:<15} {r[3]:<15.2f} {r[4] or ''}")
    print(f"\n  Total: PKR {sum(r[3] for r in rows):.2f}  ({len(rows)} transaction(s))")


def delete_expense(expense_id: int):
    conn = get_conn()
    cur = conn.execute("DELETE FROM expenses WHERE id = ?", (expense_id,))
    conn.commit()
    conn.close()
    if cur.rowcount:
        print(f"\n  ✓ Expense #{expense_id} deleted.")
    else:
        print(f"\n  ✗ No expense with ID {expense_id}.")


def monthly_summary():
    conn = get_conn()
    rows = conn.execute("""
        SELECT substr(date, 1, 7) AS month, category, SUM(amount) AS total
        FROM expenses
        GROUP BY month, category
        ORDER BY month, category
    """).fetchall()
    conn.close()

    if not rows:
        print("\n  No data available for summary.")
        return

    current_month = None
    month_total = 0.0
    print()
    for month, category, total in rows:
        if month != current_month:
            if current_month:
                print(f"  {'─'*40}")
                print(f"  Month Total: PKR {month_total:.2f}\n")
                month_total = 0
            print(f"  📅  {month}")
            print(f"  {'Category':<20} {'Amount (PKR)'}")
            print(f"  {'─'*40}")
            current_month = month
        print(f"  {category:<20} {total:.2f}")
        month_total += total
    print(f"  {'─'*40}")
    print(f"  Month Total: PKR {month_total:.2f}")


# ─── Input Helpers ────────────────────────────────────────

def input_date() -> str:
    while True:
        raw = input("  Date (YYYY-MM-DD, blank = today): ").strip()
        if not raw:
            return datetime.today().strftime("%Y-%m-%d")
        try:
            datetime.strptime(raw, "%Y-%m-%d")
            return raw
        except ValueError:
            print("  ✗ Invalid format. Use YYYY-MM-DD.")


def input_category() -> str:
    print("  Categories:")
    for i, c in enumerate(CATEGORIES, 1):
        print(f"    {i}. {c}")
    while True:
        raw = input("  Choose category number: ").strip()
        if raw.isdigit() and 1 <= int(raw) <= len(CATEGORIES):
            return CATEGORIES[int(raw) - 1]
        print("  ✗ Enter a number from the list.")


def input_amount() -> float:
    while True:
        raw = input("  Amount (PKR): ").strip()
        try:
            val = float(raw)
            if val <= 0:
                raise ValueError
            return val
        except ValueError:
            print("  ✗ Enter a positive number.")


def input_id(prompt: str) -> int:
    while True:
        raw = input(prompt).strip()
        if raw.isdigit() and int(raw) > 0:
            return int(raw)
        print("  ✗ Enter a valid positive integer ID.")


def input_month() -> str:
    while True:
        raw = input("  Month (YYYY-MM, blank = all): ").strip()
        if not raw:
            return ""
        try:
            datetime.strptime(raw, "%Y-%m")
            return raw
        except ValueError:
            print("  ✗ Use YYYY-MM format.")


# ─── Menu ─────────────────────────────────────────────────

MENU = """
  ┌────────────────────────────────────┐
  │   💰  Expense Tracker — Hira Sarwar│
  ├────────────────────────────────────┤
  │  1. Add Expense                    │
  │  2. View All Expenses              │
  │  3. View by Month                  │
  │  4. Monthly Summary by Category    │
  │  5. Delete Expense                 │
  │  0. Exit                           │
  └────────────────────────────────────┘
"""


def main():
    init_db()
    print(MENU)
    while True:
        choice = input("  Select option: ").strip()

        if choice == "1":
            print("\n  — Add Expense —")
            date = input_date()
            category = input_category()
            amount = input_amount()
            note = input("  Note (optional): ").strip()
            add_expense(date, category, amount, note)

        elif choice == "2":
            print("\n  — All Expenses —")
            view_expenses()

        elif choice == "3":
            print("\n  — Expenses by Month —")
            month = input_month()
            view_expenses(month if month else None)

        elif choice == "4":
            print("\n  — Monthly Summary —")
            monthly_summary()

        elif choice == "5":
            print("\n  — Delete Expense —")
            view_expenses()
            eid = input_id("  Enter ID to delete: ")
            delete_expense(eid)

        elif choice == "0":
            print("\n  Goodbye, Hira! 👋\n")
            break

        else:
            print("  ✗ Invalid option. Choose 0–5.")

        print()
        input("  Press Enter to continue...")
        print(MENU)


if __name__ == "__main__":
    main()
