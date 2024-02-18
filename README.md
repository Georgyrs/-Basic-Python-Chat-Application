# -Basic-Python-Chat-Application
# The Basic Python Chat Application is a simple chat platform built using Python. It allows users to join a chat room, send text messages, and view messages from other  users in real-time. The application features a straightforward interface where users can enter their name, type messages, and see who else is currently online.
import asyncio
import time
from pywebio import start_server
from pywebio.input import *
from pywebio.output import *
from pywebio.session import run_async
from cryptography.fernet import Fernet

key = Fernet.generate_key()
cipher = Fernet(key)

chat_msgs = []
online_users = set()

MAX_MESSAGES_COUNT = 100
chat_password = "1"

user_actions = {}

def encrypt_message(message):
    return cipher.encrypt(message.encode())

def decrypt_message(encrypted_message):
    return cipher.decrypt(encrypted_message).decode()

async def main():
    global chat_msgs, online_users, user_actions

    put_markdown("## 🧊 Добро пожаловать в чат!\nМы храним ваши данные в безопасности")
    online_user_count = put_text("Онлайн: 0", position="before")

    password = await input("Введите пароль для входа в чат", type=PASSWORD)

    if password != chat_password:
        put_markdown("Неверный пароль! Попробуйте снова.")
        return

    put_markdown("Пароль принят. Добро пожаловать!")

    nickname = await input("Войти в чат", required=True, placeholder="Ваше имя",
                           validate=lambda n: "Такой ник уже используется!" if n in online_users or n == '📢' else None)
    online_users.add(nickname)
    user_actions[nickname] = 'join'  # Записываем действие в словарь
    put_text("Онлайн: {}".format(len(online_users)), position="before")

    refresh_task = run_async(refresh_msg(nickname, online_user_count))

    while True:
        data = await input_group("💭 Новое сообщение", [
            input(placeholder="Текст сообщения ...", name="msg"),
            actions(name="cmd", buttons=["Отправить", {'label': "Выйти из чата", 'type': 'cancel'}]),
        ], validate=lambda m: ('msg', "Введите текст сообщения!") if m["cmd"] == "Отправить" and not m['msg'] else None)

        if data is None:
            break

        msg_time = time.strftime('%H:%M:%S', time.localtime())
        if data['msg']:
            encrypted_msg = encrypt_message(data['msg'])
            put_markdown(f"`{nickname}` ({msg_time}): {data['msg']}")
            chat_msgs.append((nickname, encrypted_msg, msg_time))

    refresh_task.close()

    online_users.remove(nickname)
    put_text("Онлайн: {}".format(len(online_users)), position="before")
    toast("Вы вышли из чата!")
    user_actions[nickname] = 'leave'
    del user_actions[nickname]

async def refresh_msg(nickname, online_user_count):
    global chat_msgs, online_users, user_actions
    last_idx = len(chat_msgs)

    while True:
        await asyncio.sleep(1)

        actions_to_display = set()
        for user, action in user_actions.items():
            actions_to_display.add((action, user))

        for action, user in actions_to_display:
            if action == 'join' and user == nickname:
                put_markdown(f'📢 Пользователь `{user}` присоединился к чату!')
                del user_actions[user]
            elif action == 'leave' and user == nickname:
                put_markdown(f'📢 Пользователь `{user}` покинул чат!')
                del user_actions[user]
        put_text("Онлайн: {}".format(len(online_users)), position="before")

        if len(chat_msgs) > MAX_MESSAGES_COUNT:
            chat_msgs = chat_msgs[len(chat_msgs) // 2:]

        last_idx = len(chat_msgs)

if __name__ == "__main__":
    start_server(main, debug=True, port=8080)
