# meta developer: @Androfon_AI
# meta name: MafiaBlackPro
# meta version: 2.1.0
# scope: hikka_min 1.3.0
# scope: hikka_only

import logging
import asyncio
import random
import re
import urllib.parse
from telethon.tl.types import Message
from .. import loader, utils

logger = logging.getLogger(__name__)

@loader.tds
class MafiaBlackProMod(loader.Module):
    """
    Авто-игра для @MafiaRuBlackBot. 
    Умеет: заходить в игру, играть за Комиссара (авто-проверка), 
    голосовать против вскрытой мафии.
    """

    strings = {
        "name": "MafiaBlackPro",
        "enabled": "✅ <b>MafiaPro включен.</b>\nОжидаю игру...",
        "disabled": "❌ <b>MafiaPro выключен.</b>",
        "status": (
            "📊 <b>Статус MafiaPro:</b>\n"
            "🟢 Состояние: {}\n"
            "🎭 Текущая роль: {}\n"
            "☠️ Известные враги (Мафия/Дон): {}\n"
            "⚡️ Задержка: {} сек"
        ),
    }

    def __init__(self):
        self.config = loader.ModuleConfig(
            loader.ConfigValue(
                "enabled",
                True,
                "Включен ли модуль",
                validator=loader.validators.Boolean()
            ),
            loader.ConfigValue(
                "delay_range",
                [1.5, 3.0],
                "Диапазон задержки перед нажатием кнопок (минимум, максимум)",
                validator=loader.validators.Series(loader.validators.Float())
            ),
            loader.ConfigValue(
                "target_bot_id",
                761250017, # ID True Mafia Black
                "ID бота для игры (по умолчанию True Mafia Black)",
                validator=loader.validators.Integer()
            ),
            loader.ConfigValue(
                "auto_join_keywords",
                ["присоединиться", "играть", "🌚", "🌝", "✅"],
                "Слова на кнопках для входа в лобби",
                validator=loader.validators.Series(loader.validators.String())
            ),
        )
        # Внутреннее состояние игры
        self.game_state = {
            "my_role": None,   # 'com', 'maf', 'don', 'citizen', etc.
            "enemies": set(),  # Имена выявленных мафиози
            "allies": set(),   # Имена союзников
            "in_game": False
        }

    async def client_ready(self, client, _):
        self._client = client

    @loader.command(ru_doc="Включить/выключить модуль")
    async def mafon(self, message: Message):
        """Переключает состояние модуля"""
        self.config["enabled"] = not self.config["enabled"]
        # Сброс состояния при выключении
        if not self.config["enabled"]:
            self._reset_game()
        
        status = self.strings("enabled") if self.config["enabled"] else self.strings("disabled")
        await utils.answer(message, status)

    @loader.command(ru_doc="Показать статус")
    async def mafstats(self, message: Message):
        """Показывает текущие данные об игре"""
        status_text = "Включен" if self.config["enabled"] else "Выключен"
        role = self.game_state["my_role"] or "Не определена / Не в игре"
        enemies = ", ".join(self.game_state["enemies"]) or "Нет данных"
        
        await utils.answer(
            message, 
            self.strings("status").format(
                status_text, 
                role, 
                enemies, 
                self.config["delay_range"]
            )
        )

    def _reset_game(self):
        """Сброс памяти о текущей катке"""
        self.game_state = {
            "my_role": None,
            "enemies": set(),
            "allies": set(),
            "in_game": False
        }

    async def _click_random_delay(self, button):
        """Нажатие с рандомной задержкой (чтобы не палиться и не ловить флуд)"""
        min_d, max_d = self.config["delay_range"]
        delay = random.uniform(min_d, max_d)
        await asyncio.sleep(delay)
        try:
            await button.click()
            return True
        except Exception as e:
            logger.error(f"MafiaPro Click Error: {e}")
            return False

    @loader.watcher(incoming=True)
    async def watcher(self, message: Message):
        if not self.config["enabled"]:
            return

        # Проверяем, что сообщение от бота или в чате с ботом
        sender = await message.get_sender()
        user_id = getattr(sender, 'id', None)
        
        # Основной ID бота
        target_bot = self.config["target_bot_id"]

        # --- ЛОГИКА В ЛИЧНЫХ СООБЩЕНИЯХ (Роли и Действия) ---
        if message.is_private and user_id == target_bot:
            txt = message.text or ""
            
            # 1. Определение роли
            if "Ты комиссар" in txt or "Ты Комиссар" in txt:
                self.game_state["my_role"] = "com"
                self.game_state["in_game"] = True
                logger.info("MafiaPro: Роль - КОМИССАР")
            elif "Ты Дон" in txt:
                self.game_state["my_role"] = "don"
                self.game_state["in_game"] = True
            elif "Ты Мафия" in txt:
                self.game_state["my_role"] = "maf"
                self.game_state["in_game"] = True
            
            # 2. Действие Комиссара: Нажать "Проверить"
            if self.game_state["my_role"] == "com" and message.buttons:
                for row in message.buttons:
                    for btn in row:
                        if "проверить" in btn.text.lower():
                            logger.info("MafiaPro: Нажимаю кнопку ПРОВЕРИТЬ")
                            await self._click_random_delay(btn)
                            return
            
            # 3. Выбор цели (после нажатия "Проверить" или ночью для мафии)
            # Если бот предлагает список игроков (обычно это просто имена или имена с цифрами)
            if self.game_state["in_game"] and message.buttons:
                # Простая эвристика: если кнопок много, скорее всего это выбор игрока
                # Исключаем кнопки меню типа "Назад", "Сдаться"
                ignore_btns = ["назад", "сдаться", "правила", "отмена"]
                valid_targets = []
                
                for row in message.buttons:
                    for btn in row:
                        if not btn.text: continue
                        if btn.text.lower() not in ignore_btns:
                            valid_targets.append(btn)
                
                if valid_targets:
                    # Если мы комиссар и только что нажали проверить (предыдущее сообщение было с кнопкой проверить)
                    # Либо это просто фаза ночи. Тут сложнее отследить контекст без сохранения истории.
                    # Но если мы комиссар и видим список людей - тыкаем.
                    if self.game_state["my_role"] == "com":
                        target = random.choice(valid_targets)
                        logger.info(f"MafiaPro: Комиссар проверяет -> {target.text}")
                        await self._click_random_delay(target)

        # --- ЛОГИКА В ГРУППОВОМ ЧАТЕ (Регистрация, Голосование, Итоги) ---
        if not message.is_private:
            # Если сообщение не от бота, проверяем текст на наличие итогов (кто умер, кто мафия)
            if user_id != target_bot and not getattr(sender, 'bot', False):
                return # Игнорим сообщения юзеров, кроме бота

            txt = message.text or ""
            
            # 1. Логика авто-входа (регистрации)
            join_phrases = ["Ведётся набор в игру", "Регистрация началась", "Набор в игру"]
            if any(p in txt for p in join_phrases) and message.buttons:
                for row in message.buttons:
                    for btn in row:
                        # Обычный вход или Deep-Link (🌚/🌝)
                        if any(k.lower() in btn.text.lower() for k in self.config["auto_join_keywords"]):
                            # Обработка DeepLink
                            if btn.url:
                                parsed = urllib.parse.urlparse(btn.url)
                                start_arg = urllib.parse.parse_qs(parsed.query).get('start', [None])[0]
                                if start_arg:
                                    bot_username = "MafiaRuBlackBot" # Жестко задаем или парсим
                                    await self._click_random_delay(btn) # Кликаем для вида (иногда боты требуют)
                                    # Отправляем старт в лс
                                    await self._client.send_message(bot_username, f"/start {start_arg}")
                                    logger.info(f"MafiaPro: DeepLink вход -> {start_arg}")
                                    self._reset_game() # Новая игра - сброс
                                    return
                            else:
                                # Обычная кнопка callback
                                await self._click_random_delay(btn)
                                self._reset_game()
                                return

            # 2. Анализ итогов (кто мафия)
            # Пример строки: "Вася Пупкин - 🤵🏼Мафия"
            if "— 🤵🏼Мафия" in txt or "— 🤵🏻Дон" in txt:
                # Регулярка для поиска имен
                # Ищет текст от начала строки или новой строки до тире и роли
                matches = re.findall(r"(?:^|\n)(.*?) — (?:🤵🏼Мафия|🤵🏻Дон)", txt)
                for name in matches:
                    clean_name = name.strip()
                    self.game_state["enemies"].add(clean_name)
                    logger.info(f"MafiaPro: Обнаружен враг -> {clean_name}")

            # 3. Голосование днем
            if "Голосование" in txt and message.buttons:
                # Ждем чуть дольше перед голосованием
                await asyncio.sleep(random.randint(2, 5))
                
                target_btn = None
                
                # Ищем кнопку с именем известного врага
                for row in message.buttons:
                    for btn in row:
                        btn_text = btn.text.strip()
                        # Если текст кнопки совпадает с именем врага
                        if any(enemy in btn_text for enemy in self.game_state["enemies"]):
                             target_btn = btn
                             break
                
                if target_btn:
                    logger.info(f"MafiaPro: Голосую против мафии -> {target_btn.text}")
                    await target_btn.click()
                elif self.game_state["my_role"] in ["maf", "don"]:
                    # Если я мафия, и врагов не найдено (или это мои союзники), 
                    # нужно голосовать против мирных. 
                    # (Сложная логика, пока пропускаем, чтобы не слить своих)
                    pass
