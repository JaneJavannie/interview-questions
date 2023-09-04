# Go Interview Questions
Идея данного репозитория возникла во время подготовки к собеседованиям на позицию Backend разработчика на языке Go.  
В данном Readme собраны частые и не очень вопросы и ответы к ним. Они условно делятся на общие вопросы и вопросы непосредственно по самому языку Go.  

Настоятельно рекомендую, прежде чем открывать ответ на вопрос попробовать ответить на него самостоятельно.

p.s Если вы заметили не точность или не соответствие информации - напишите не стесняйтесь.  
p.s.s Вы так же можете предложить свои вопросы, я обязательно их добавлю.

---

## Questions

***Общие вопросы***
1. [В чем отличие протоколов TCP и UDP](#tcp-udp)
2. [Удаление пользователя](#del-user)
3. [Создание сегмента](#create-seg)
4. [Удаление сегмента](#del-seg)
5. [Добавление/Удаление сегментов](#add-remove)
6. [Получение списка сегментов](#seg-list)
7. [Получение истории пользователя](#user-history)
8. [Вопросы во время разработки](#decisions)

---

***Вопросы по языку GO***
1. [Создание пользователя](#create-user)
2. [Удаление пользователя](#del-user)
3. [Создание сегмента](#create-seg)
4. [Удаление сегмента](#del-seg)

---

## Answers

### Вопрос: В чем отличие протоколов TCP и UDP <a name="tcp-udp"></a>

<details>
  <summary>Ответ <- Click</summary>
  - TCP (Transmission Control Protocol):
      - Ориентирован на установление надежного соединения.
      - Ошибки корректируются; потерянные или поврежденные пакеты пересылаются.
      - Поддерживает управление потоком и перегрузкой.
      - Нормально работает в условиях высокой задержки.
  - UDP (User Datagram Protocol):
      - Безусловный протокол, не устанавливает соединение.
      - Ошибки не корректируются; потерянные пакеты не восстанавливаются.
      - Не поддерживает управление потоком и перегрузкой.
      - Обычно быстрее, чем TCP.
  - Когда UDP предпочтительнее:
      - Потоковое медиа, онлайн-игры, VoIP — там, где задержка критична и потеря пакетов допустима.
</details>
