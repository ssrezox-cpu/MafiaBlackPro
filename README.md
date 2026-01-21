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
