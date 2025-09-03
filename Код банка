import streamlit as st
import sqlite3
import pandas as pd
import plotly.express as px
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash

# Database setup
def init_db():
    conn = sqlite3.connect('finance.db')
    c = conn.cursor()
    # Users table
    c.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL
        )
    ''')
    # Income table with user_id
    c.execute('''
        CREATE TABLE IF NOT EXISTS income (
            id INTEGER PRIMARY KEY,
            user_id INTEGER NOT NULL,
            month TEXT NOT NULL,
            salary REAL NOT NULL,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    ''')
    # Expenses table with user_id
    c.execute('''
        CREATE TABLE IF NOT EXISTS expenses (
            id INTEGER PRIMARY KEY,
            user_id INTEGER NOT NULL,
            month TEXT NOT NULL,
            category TEXT NOT NULL,
            amount REAL NOT NULL,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    ''')
    conn.commit()
    conn.close()

init_db()

# Function to register user
def register_user(username, password):
    hashed = generate_password_hash(password)
    conn = sqlite3.connect('finance.db')
    c = conn.cursor()
    try:
        c.execute('INSERT INTO users (username, password_hash) VALUES (?, ?)', (username, hashed))
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        return False
    finally:
        conn.close()

# Function to login user
def login_user(username, password):
    conn = sqlite3.connect('finance.db')
    c = conn.cursor()
    c.execute('SELECT id, password_hash FROM users WHERE username = ?', (username,))
    user = c.fetchone()
    conn.close()
    if user and check_password_hash(user[1], password):
        return user[0]
    return None

# Function to add income
def add_income(user_id, month, salary):
    conn = sqlite3.connect('finance.db')
    c = conn.cursor()
    c.execute('INSERT INTO income (user_id, month, salary) VALUES (?, ?, ?)', (user_id, month, salary))
    conn.commit()
    conn.close()

# Function to add expense
def add_expense(user_id, month, category, amount):
    conn = sqlite3.connect('finance.db')
    c = conn.cursor()
    c.execute('INSERT INTO expenses (user_id, month, category, amount) VALUES (?, ?, ?, ?)', (user_id, month, category, amount))
    conn.commit()
    conn.close()

# Function to get data for a month
def get_data(user_id, month):
    conn = sqlite3.connect('finance.db')
    income_df = pd.read_sql_query(f"SELECT * FROM income WHERE user_id = {user_id} AND month = '{month}'", conn)
    expenses_df = pd.read_sql_query(f"SELECT * FROM expenses WHERE user_id = {user_id} AND month = '{month}'", conn)
    conn.close()
    return income_df, expenses_df

# Function to suggest better distribution (50/30/20 rule)
def suggest_distribution(total_income, expenses_df):
    if total_income == 0:
        return "No income data available."
    
    categories = {
        'Needs': ['Квартплата', 'Еда', 'Транспорт'],  # Example needs
        'Wants': ['Развлечения', 'Одежда'],  # Example wants
        'Savings': ['Сбережения']  # Savings
    }
    
    current_spending = {'Needs': 0, 'Wants': 0, 'Savings': 0, 'Other': 0}
    for _, row in expenses_df.iterrows():
        cat = row['category']
        found = False
        for key, vals in categories.items():
            if cat in vals:
                current_spending[key] += row['amount']
                found = True
                break
        if not found:
            current_spending['Other'] += row['amount']
    
    total_expenses = expenses_df['amount'].sum()
    remaining = total_income - total_expenses
    
    suggested = {
        'Needs': total_income * 0.5,
        'Wants': total_income * 0.3,
        'Savings': total_income * 0.2
    }
    
    suggestion_text = f"Рекомендуемое распределение (50/30/20):\n- Needs: {suggested['Needs']:.2f} (текущие: {current_spending['Needs']:.2f})\n- Wants: {suggested['Wants']:.2f} (текущие: {current_spending['Wants']:.2f})\n- Savings: {suggested['Savings']:.2f} (текущие: {current_spending['Savings']:.2f})\nОстаток: {remaining:.2f}"
    return suggestion_text

# Streamlit app
st.title("Личный Финансовый Менеджер")

# Session state for authentication
if 'user_id' not in st.session_state:
    st.session_state.user_id = None

# Authentication
if st.session_state.user_id is None:
    tab1, tab2 = st.tabs(["Вход", "Регистрация"])

    with tab1:
        st.header("Вход")
        username_login = st.text_input("Имя пользователя (вход)")
        password_login = st.text_input("Пароль (вход)", type="password")
        if st.button("Войти"):
            user_id = login_user(username_login, password_login)
            if user_id:
                st.session_state.user_id = user_id
                st.success("Успешный вход!")
                st.rerun()
            else:
                st.error("Неверные данные")

    with tab2:
        st.header("Регистрация")
        username_reg = st.text_input("Имя пользователя (регистрация)")
        password_reg = st.text_input("Пароль (регистрация)", type="password")
        if st.button("Зарегистрироваться"):
            if register_user(username_reg, password_reg):
                st.success("Пользователь зарегистрирован! Теперь войдите.")
            else:
                st.error("Имя пользователя уже существует")
else:
    st.sidebar.button("Выход", on_click=lambda: st.session_state.clear())

    # Main app
    current_month = datetime.now().strftime("%Y-%m")
    month = st.text_input("Месяц (YYYY-MM)", current_month)

    # Input income
    salary = st.number_input("Заработная плата за месяц", min_value=0.0)
    if st.button("Добавить доход"):
        add_income(st.session_state.user_id, month, salary)
        st.success("Доход добавлен!")

    # Input expenses
    category = st.selectbox("Категория расхода", ["Еда", "Квартплата", "Транспорт", "Развлечения", "Одежда", "Сбережения", "Другое"])
    amount = st.number_input("Сумма расхода", min_value=0.0)
    if st.button("Добавить расход"):
        add_expense(st.session_state.user_id, month, category, amount)
        st.success("Расход добавлен!")

    # Display history
    st.header("История расходов")
    income_df, expenses_df = get_data(st.session_state.user_id, month)
    if not income_df.empty:
        st.write(f"Доход: {income_df['salary'].sum():.2f}")
    else:
        st.write("Нет данных о доходе.")

    if not expenses_df.empty:
        st.dataframe(expenses_df[['category', 'amount']])
        total_expenses = expenses_df['amount'].sum()
        st.write(f"Общие расходы: {total_expenses:.2f}")
    else:
        st.write("Нет расходов.")

    # Pie chart for expense distribution
    if not expenses_df.empty:
        st.header("Распределение расходов")
        fig_pie = px.pie(expenses_df, values='amount', names='category', title='Распределение расходов')
        st.plotly_chart(fig_pie)

    # Bar chart for monthly overview
    if not expenses_df.empty:
        st.header("Ежемесячный обзор")
        fig_bar = px.bar(expenses_df, x='category', y='amount', title='Расходы по категориям')
        st.plotly_chart(fig_bar)

    # Suggestions
    if not income_df.empty and not expenses_df.empty:
        st.header("Рекомендации по распределению")
        total_income = income_df['salary'].sum()
        suggestions = suggest_distribution(total_income, expenses_df)
        st.write(suggestions)
