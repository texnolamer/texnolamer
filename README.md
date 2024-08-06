import logging
from telethon import TelegramClient, events, types
import asyncio
from datetime import datetime, timedelta
from openpyxl import Workbook
import os
import json

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

api_id = '17021123'  # Замените на ваш API ID
api_hash = 'fc7d39e5a991f779992924cbeed1145e'  # Замените на ваш API Hash
phone_number = '+375445363309'  # Замените на ваш номер телефона
group_id = -1002152268453  # ID группы для уведомлений
client = TelegramClient('unique_session_name', api_id, api_hash)

tracked_accounts = {}

def load_tracked_accounts():
    global tracked_accounts
    if os.path.exists('tracked_accounts.json'):
        with open('tracked_accounts.json', 'r') as file:
            tracked_accounts = json.load(file)
            for key, value in tracked_accounts.items():
                value['online_since'] = datetime.fromisoformat(value['online_since']) if value['online_since'] else None
                value['offline_since'] = datetime.fromisoformat(value['offline_since']) if value['offline_since'] else None
                value['total_online_time'] = timedelta(seconds=float(value['total_online_time']))
                value['total_offline_time'] = timedelta(seconds=float(value['total_offline_time']))
                value['online_sessions'] = [timedelta(seconds=float(duration)) for duration in value['online_sessions']]
                value['offline_sessions'] = [timedelta(seconds=float(duration)) for duration in value['offline_sessions']]

def save_tracked_accounts():
    with open('tracked_accounts.json', 'w') as file:
        json.dump(tracked_accounts, file, default=str)

load_tracked_accounts()

@client.on(events.NewMessage(pattern='/1'))
async def command_handler(event):
    try:
        await client.send_message(group_id, 'Работаю')
        logger.info('Команда /1 выполнена: отправлено сообщение "Работаю"')
    except Exception as e:
        logger.error(f'Ошибка при выполнении команды /1: {e}')

@client.on(events.NewMessage(pattern='/add'))
async def add_account(event):
    try:
        message_parts = event.message.text.split()
        if len(message_parts) != 3:
            await event.reply('Использование: /add <номер телефона> <имя контакта>')
            return

        phone_number_to_track = message_parts[1]
        contact_name = message_parts[2]
        tracked_accounts[phone_number_to_track] = {
            'name': contact_name,
            'online_since': None,
            'offline_since': None,
            'total_online_time': timedelta(),
            'total_offline_time': timedelta(),
            'online_sessions': [],
            'offline_sessions': [],
            'last_online': None,
            'last_offline': None
        }
        save_tracked_accounts()
        await event.reply(f'{contact_name} добавлен для отслеживания.')
        logger.info(f'{contact_name} ({phone_number_to_track}) добавлен для отслеживания.')
    except Exception as e:
        logger.error(f'Ошибка при добавлении аккаунта: {e}')

@client.on(events.NewMessage(pattern='/remove'))
async def remove_account(event):
    try:
        message_parts = event.message.text.split()
        if len(message_parts) != 2:
            await event.reply('Использование: /remove <номер телефона>')
            return

        phone_number_to_track = message_parts[1]
        if phone_number_to_track in tracked_accounts:
            del tracked_accounts[phone_number_to_track]
            save_tracked_accounts()
            await event.reply(f'Контакт удален из отслеживания.')
            logger.info(f'Контакт ({phone_number_to_track}) удален из отслеживания.')
        else:
            await event.reply(f'Контакт не найден.')
    except Exception as e:
        logger.error(f'Ошибка при удалении аккаунта: {e}')

@client.on(events.NewMessage(pattern='/list'))
async def list_accounts(event):
    try:
        if not tracked_accounts:
            await event.reply('Нет отслеживаемых контактов.')
            return

        message = 'Отслеживаемые контакты:\n'
        for info in tracked_accounts.values():
            message += f'- {info["name"]}\n'
        await event.reply(message)
    except Exception as e:
        logger.error(f'Ошибка при выводе списка аккаунтов: {e}')

def format_duration(duration):
    seconds = int(duration.total_seconds())
    periods = [
        ('ч', 3600),
        ('м', 60),
        ('с', 1),
    ]
    parts = []
    for period_name, period_seconds in periods:
        if seconds >= period_seconds:
            period_value, seconds = divmod(seconds, period_seconds)
            parts.append(f'{period_value}{period_name}')
    return ' '.join(parts)
async def track_user_status():
    while True:
        try:
            tracked_accounts_copy = tracked_accounts.copy()
            for phone_number_to_track, info in tracked_accounts_copy.items():
                try:
                    user = await client.get_entity(phone_number_to_track)
                    current_status = user.status
                    current_time = datetime.now().strftime('%d.%m.%Y %H:%М:%С')  # Измененный формат даты

                    if current_status != info.get('previous_status'):
                        if isinstance(current_status, types.UserStatusOnline):
                            if info['offline_since']:
                                offline_duration = datetime.now() - info['offline_since']
                                info['total_offline_time'] += offline_duration
                                info['offline_sessions'].append(offline_duration)
                                formatted_offline_duration = format_duration(offline_duration)
                                await client.send_message(group_id, f'🟢 {info["name"]} в сети\n<small>{current_time}</small>\nБыл офлайн: {formatted_offline_duration}', parse_mode='html')
                                logger.info(f'{info["name"]} в сети ({current_time}). Был офлайн {formatted_offline_duration}.')
                            info['online_since'] = datetime.now()
                            info['offline_since'] = None
                            info['last_online'] = current_time
                        elif isinstance(current_status, types.UserStatusOffline):
                            if info['online_since']:
                                online_duration = datetime.now() - info['online_since']
                                info['total_online_time'] += online_duration
                                info['online_sessions'].append(online_duration)
                                formatted_online_duration = format_duration(online_duration)
                                await client.send_message(group_id, f'🔴 {info["name"]} вышел из сети\n<small>{current_time}</small>\nБыл в сети: {formatted_online_duration}', parse_mode='html')
                                logger.info(f'{info["name"]} вышел из сети ({current_time}). Был в сети {formatted_online_duration}.')
                            info['offline_since'] = datetime.now()
                            info['online_since'] = None
                            info['last_offline'] = current_time
                        elif isinstance(current_status, types.UserStatusRecently):
                            await client.send_message(group_id, f'🕒 {info["name"]} недавно был в сети\n<small>{current_time}</small>', parse_mode='html')
                            logger.info(f'{info["name"]} недавно был в сети ({current_time})')

                        info['previous_status'] = current_status

                except Exception as e:
                    logger.error(f'Ошибка при отслеживании статуса пользователя {phone_number_to_track}: {e}')

            save_tracked_accounts()
            await asyncio.sleep(10)
        except Exception as e:
            logger.error(f'Ошибка в функции track_user_status: {e}')
            await asyncio.sleep(10)

async def create_and_send_report():
    while True:
        try:
            wb = Workbook()
            ws = wb.active
            ws.title = "Статистика"

            ws.append(["Имя", "Статус", "Общее время в сети", "Общее время офлайн", "Последний вход в сеть", "Последний выход из сети", "Средняя продолжительность онлайн", "Средняя продолжительность офлайн", "Максимальная продолжительность онлайн", "Минимальная продолжительность онлайн", "Максимальная продолжительность офлайн", "Минимальная продолжительность офлайн"])

            for phone_number_to_track, info in tracked_accounts.items():
                total_online_time = format_duration(info['total_online_time'])
                total_offline_time = format_duration(info['total_offline_time'])
                last_online = info['last_online'] if info['last_online'] else "N/A"
                last_offline = info['last_offline'] if info['last_offline'] else "N/A"
                avg_online_duration = format_duration(sum(info['online_sessions'], timedelta()) / len(info['online_sessions'])) if info['online_sessions'] else "N/A"
                avg_offline_duration = format_duration(sum(info['offline_sessions'], timedelta()) / len(info['offline_sessions'])) if info['offline_sessions'] else "N/A"
                max_online_duration = format_duration(max(info['online_sessions'], default=timedelta())) if info['online_sessions'] else "N/A"
                min_online_duration = format_duration(min(info['online_sessions'], default=timedelta())) if info['online_sessions'] else "N/A"
                max_offline_duration = format_duration(max(info['offline_sessions'], default=timedelta())) if info['offline_sessions'] else "N/A"
                min_offline_duration = format_duration(min(info['offline_sessions'], default=timedelta())) if info['offline_sessions'] else "N/A"
                ws.append([info['name'], "В сети" if info['online_since'] else "Оффлайн", total_online_time, total_offline_time, last_online, last_offline, avg_online_duration, avg_offline_duration, max_online_duration, min_online_duration, max_offline_duration, min_offline_duration])

            file_name = f'statistics_{datetime.now().strftime("%Y%m%d_%H%M%S")}.xlsx'
            wb.save(file_name)

            await client.send_file(group_id, file_name, caption="Статистика активности")
            os.remove(file_name)

            await asyncio.sleep(18000)
        except Exception as e:
            logger.error(f'Ошибка при создании и отправке отчета: {e}')
            await asyncio.sleep(60)

async def main():
    while True:
        try:
            await client.start(phone=phone_number)
            client.loop.create_task(track_user_status())
            client.loop.create_task(create_and_send_report())
            await client.run_until_disconnected()
        except Exception as e:
            logger.error(f'Ошибка в основном цикле: {e}')
            await asyncio.sleep(10)

try:
    client.loop.run_until_complete(main())
except (KeyboardInterrupt, SystemExit):
    logger.info("Бот остановлен")
finally:
    client.disconnect()

