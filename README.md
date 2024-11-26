import sqlite3

class DatabaseManager:
    def __init__(self, db_name):
        self.db_name = db_name
        self.connection = None

    def open_connection(self):
        """Открывает соединение с базой данных."""
        if not self.connection:
            self.connection = sqlite3.connect(self.db_name)

    def close_connection(self):
        """Закрывает соединение с базой данных."""
        if self.connection:
            self.connection.close()
            self.connection = None

    def execute_query(self, query, params=None, fetch_one=False, fetch_all=False):
        """
        Выполняет SQL-запрос.
        :param query: SQL-запрос
        :param params: Параметры для запроса
        :param fetch_one: Возвращать одну запись
        :param fetch_all: Возвращать все записи
        :return: Результаты запроса, если применимо
        """
        self.open_connection()
        cursor = self.connection.cursor()
        try:
            if params:
                cursor.execute(query, params)
            else:
                cursor.execute(query)

            if fetch_one:
                return cursor.fetchone()
            if fetch_all:
                return cursor.fetchall()

            self.connection.commit()
        except sqlite3.Error as e:
            self.connection.rollback()
            print(f"ошибка выполнениее запрса: {e}")
        finally:
            cursor.close()

    def find_user_by_name(self, username):
        """Находит пользователя по имени в таблице users."""
        query = "SELECT * FROM users WHERE name = ?"
        return self.execute_query(query, (username,), fetch_one=True)


class User:
    def __init__(self, db_manager):
        self.db_manager = db_manager

    def add_user(self, name, age):
        """Добавляет нового пользователя в таблицу users."""
        query = "INSERT INTO users (name, age) VALUES (?, ?)"
        self.db_manager.execute_query(query, (name, age))

    def get_user_by_id(self, user_id):
        """Получает данные пользователя по ID."""
        query = "SELECT * FROM users WHERE id = ?"
        return self.db_manager.execute_query(query, (user_id,), fetch_one=True)

    def delete_user(self, user_id):
        """Удаляет пользователя по ID."""
        query = "DELETE FROM users WHERE id = ?"
        self.db_manager.execute_query(query, (user_id,))


class Admin(User):
    def add_admin(self, name, privileges):
        """Добавляет нового администратора."""
        query = "INSERT INTO admins (name, privileges) VALUES (?, ?)"
        self.db_manager.execute_query(query, (name, privileges))

    def get_admin_by_id(self, admin_id):
        """Получает данные администратора по ID."""
        query = "SELECT * FROM admins WHERE id = ?"
        return self.db_manager.execute_query(query, (admin_id,), fetch_one=True)


class Customer(User):
    def add_customer(self, name, email):
        """Добавляет нового клиента."""
        query = "INSERT INTO customers (name, email) VALUES (?, ?)"
        self.db_manager.execute_query(query, (name, email))

    def get_customer_by_id(self, customer_id):
        """Получает данные клиента по ID."""
        query = "SELECT * FROM customers WHERE id = ?"
        return self.db_manager.execute_query(query, (customer_id,), fetch_one=True)


class DatabaseManagerWithTransactions(DatabaseManager):
    def perform_transaction(self, operations):
        """
        Выполняет несколько операций в одной транзакции.
        :param operations: Список функций с параметрами, которые нужно выполнить
        """
        try:
            self.open_connection()
            for operation in operations:
                operation()
            self.connection.commit()
        except sqlite3.Error as e:
            self.connection.rollback()
            print(f"ошибка транзакеци: {e}")

if __name__ == "__main__":
    db_manager = DatabaseManagerWithTransactions("example.db")
    user_manager = User(db_manager)
    admin_manager = Admin(db_manager)
    customer_manager = Customer(db_manager)

    db_manager.execute_query("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            age INTEGER
        )
    """)
    db_manager.execute_query("""
        CREATE TABLE IF NOT EXISTS admins (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            privileges TEXT
        )
    """)
    db_manager.execute_query("""
        CREATE TABLE IF NOT EXISTS customers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT
        )
    """)

    
    user_manager.add_user("Alice", 30)
    print(user_manager.get_user_by_id(1))
    user_manager.delete_user(1)
