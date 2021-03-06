---
title: "Урок 13. Опросы v2.0"
type: docs
slug: "lesson_13"
BookToC: 3
---

## Введение

Добро пожаловать в 2020! В последний раз мы рассматривали нововведения Bot API аж в далёком 2017 году, когда появилось [удаление сообщений и ограничения в чатах](https://mastergroosha.github.io/telegram-tutorial/docs/lesson_10). С тех пор вышло много чего интересного и, возможно, о чём-то стоит выпустить отдельные уроки. 

А сегодня мы познакомимся с [опросами 2.0](https://telegram.org/blog/polls-2-0-vmq), точнее, с новой сущностью: викторинами (quiz). Викторина – это именно то, что вы и предположили; тест с одним правильными вариантом ответа и ещё N неправильными. 

Поставим себе задачу сделать бота, который умеет:
1. принимать от пользователя только викторины;
2. запоминать их содержимое и записывать к себе в память;
3. предлагать викторины в инлайн-запросе и отправляет их в группу;
4. получать новые ответы и сохранять ID правильно ответивших;
5. останавливать викторину после **двух** правильных ответов и поздравлять победителей.

Задач много, придётся вспомнить, что такое колбэки, инлайн-режим и классы. Но и это не самое главное…

## Пятиминутка ненависти к telebot или Привет, aiogram!
Как вы знаете, во всех предыдущих уроках использовалась библиотека [pyTelegramBotAPI](https://github.com/eternnoir/pyTelegramBotAPI), именуемая в коде telebot. В 2015-2017 годах, возможно, она ещё была актуальна, но прогресс не стоит на месте. А telebot, увы, стоит. Кривая реализация поллинга, проблемный next_step_handler, медленная поддержка новых версий Bot API и т.д.

В течение 2019 года я постепенно переносил своих ботов на другой фреймворк, который по многим пунктам превосходит pyTelegramBotAPI, и имя ему – [aiogram](https://github.com/aiogram/aiogram). «Почему?», спросит меня уважаемый читатель. Что ж, приведу следующие аргументы:

* это полноценный фреймворк, т.е. позволяет сделать больше полезных вещей;
* асинхронный, что делает его быстрее в некоторых задачах;
* поддерживается Python 3.7+ и выше, что сподвигнет обновить свой старенький интерпретатор и использовать новые возможности языка;
* множество встроенных «помощников» (синтаксический «сахар»), улучшающих читабельность кода;
* оперативные обновления (поддержка новых опросов появилась в тот же день, что и в самом Bot API);
* русскоязычный [чат](https://t.me/aiogram_ru) поддержки и обсуждений, где сидит, в том числе, и сам разработчик фреймворка;
* мой любимый пункт: нормально работающий поллинг.

Прокомментирую последний пункт: в настоящий момент почти все мои боты работают на aiogram-ном поллинге и не падают ежедневно, как в случае с pyTelegramBotAPI.   
**Важный момент**: чтобы использовать aiogram в Windows, нужны C++ Build Tools, которые входят в поставку Visual Studio (не VSCode!). В Linux таких «проблем» нет, к слову.

Введение получилось очень большим, поэтому давайте уже перейдём к делу.

## Плацдарм для бота
Напишем элементарного эхо-бота на aiogram с поллингом, чтобы бегло ознакомиться с фреймворком. Прежде всего, добавим нужные импорты (предполагается, что мы используем Virtual Environment, подробнее о нём – в [уроке №0](https://mastergroosha.github.io/telegram-tutorial/docs/lesson_00/)):

{{< highlight none >}}
#!venv/bin/python
import logging
from aiogram import Bot, Dispatcher, executor, types
logging.basicConfig(level=logging.INFO)
{{< /highlight >}}

Теперь создадим объект бота. А за хэндлеры здесь отвечает специальный Диспетчер:

{{< highlight none >}}
bot = Bot(token="12345678:AABcdeFGhIJkXyZ")
dp = Dispatcher(bot)
{{< /highlight >}}

Далее напишем простейший хэндлер, повторяющий текстовые сообщения:

{{< highlight none >}}
@dp.message_handler()
async def echo(message: types.Message):
    await message.reply(message.text)
{{< /highlight >}}

Началась магия.  
Во-первых, как я написал чуть выше, за хэндлеры отвечает диспетчер (dp).   
Во-вторых, подхэндлерные функции в aiogram асинхронные (async def), вызовы Bot API тоже асинхронные, поэтому необходимо использовать ключевое слово `await`.   
В-третьих, вместо `bot.send_message` можно для удобства использовать `message.reply( )` без указания `chat_id` и `message.id`, чтобы бот сделал «ответ» (reply), либо аналог `message.answer( )`, чтобы просто отправить в тот же чат, не создавая «ответ». Само выражение в хэндлере пустое, т.к. нас устроят любые текстовые сообщения.

Наконец, запуск!  
{{< highlight none >}}
if __name__ == "__main__":
    executor.start_polling(dp, skip_updates=True)
{{< /highlight >}}

Параметр `skip_updates=True` позволяет пропустить накопившиеся входящие сообщения, если они нам не важны.  
Запускаем код, убеждаемся в его работоспособности, после чего удаляем хэндлер вместе с функцией echo, нам они больше не понадобятся, в отличие от остального кода.

## Запрашиваем викторину у пользователя
В BotAPI 4.6 появилась новая кнопка для обычной (не инлайн) клавиатуры с типом [KeyboardButtonPollType](https://core.telegram.org/bots/api#keyboardbuttonpolltype). При нажатии на неё в приложении Telegram появляется окно для создания опроса. В самой кнопке можно выставить ограничение по типу создаваемого объекта: опрос, викторина или что угодно. Опросы нас пока не интересуют, поэтому напишем обработчик команды `/start`, выводящий приветственное сообщение и обычную клавиатуру с двумя кнопками: “Создать викторину” и “Отмена”, причём вторая отправляет [ReplyKeyboardRemove](https://core.telegram.org/bots/api#replykeyboardremove), удаляя первую клавиатуру.

{{< highlight none >}}
# Хэндлер на команду /start
@dp.message_handler(commands=["start"])
async def cmd_start(message: types.Message):
    poll_keyboard = types.ReplyKeyboardMarkup(resize_keyboard=True)
    poll_keyboard.add(types.KeyboardButton(text="Создать викторину",
                                           request_poll=types.KeyboardButtonPollType(type=types.PollType.QUIZ)))
    poll_keyboard.add(types.KeyboardButton(text="Отмена"))
    await message.answer("Нажмите на кнопку ниже и создайте викторину!", reply_markup=poll_keyboard)

# Хэндлер на текстовое сообщение с текстом “Отмена”
@dp.message_handler(lambda message: message.text == "Отмена")
async def action_cancel(message: types.Message):
    remove_keyboard = types.ReplyKeyboardRemove()
    await message.answer("Действие отменено. Введите /start, чтобы начать заново.", reply_markup=remove_keyboard)
{{< /highlight >}}

{{% img "l13_1.png" %}}Клавиатура с кнопками{{% /img %}}

## Сохраняем и предлагаем
В [11-м уроке](https://mastergroosha.github.io/telegram-tutorial/docs/lesson_11/) я использовал библиотеку [Vedis](https://github.com/coleifer/vedis-python) для сохранения состояний в файле, чтобы те не сбрасывались после перезагрузки бота. В этот раз мы будем сохранять всё в памяти, а выбор постоянного хранилища останется за читателем, чтобы не навязывать то или иное решение. Разумеется, данные в памяти сотрутся при остановке бота, но для примера так даже лучше.

Наше хранилище будет основано на стандартных питоновских словарях (dict), причём их будет два: первый словарь содержит пары ("id пользователя", "массив сохранённых викторин"), а второй — пары ("id викторины", "id автора викторины"). Зачем два словаря? В дальнейшем нам нужно будет по идентификатору викторины получать некоторую информацию о ней. Необходимые нам сведения лежат в первом словаре, но в виде значений, а не ключей. Поэтому нам пришлось бы проходиться по всем возможным парам ключ-значение, чтобы найти нужную викторину. 

Для ускорения поиска мы заведём второй словарь, чтобы по идентификатору викторины сразу же найти идентификатор её автора, который, в свою очередь, является ключом в первом словаре. А дальше проход по небольшому массиву и вуаля! Наши данные получены. На словах звучит сложно, но на практике реализуется довольно быстро и с минимальной избыточностью. Если придумаете решение лучше — пишите, буду рад исправить текст.

Помимо определения викторины, нам нужно хранить некоторую дополнительную информацию. Поэтому давайте создадим файл `quizzer.py`, опишем наш класс **Quiz** со всеми нужными полями в конструкторе класса (обратите внимание, в конструктор передаются не все поля, т.к. часть из них будет заполнена позднее):

{{< highlight none >}}
from typing import List

class Quiz:
    type: str = "quiz"

    def __init__(self, quiz_id, question, options, correct_option_id, owner_id):
        # Используем подсказки типов, чтобы было проще ориентироваться.
        self.quiz_id: str = quiz_id   # ID викторины. Изменится после отправки от имени бота
        self.question: str = question  # Текст вопроса
        self.options: List[str] = [*options] # "Распакованное" содержимое массива m_options в массив options
        self.correct_option_id: int = correct_option_id  # ID правильного ответа
        self.owner: int = owner_id  # Владелец опроса
        self.winners: List[int] = []  # Список победителей
        self.chat_id: int = 0  # Чат, в котором опубликована викторина
        self.message_id: int = 0  # Сообщение с викториной (для закрытия)
{{< /highlight >}}

Если вы раньше не сталкивались с подсказками типов (type hints), код вида “chat_id: int = 0” может ввести в замешательство. Здесь `chat_id` — это имя переменной, далее через двоеточие `int` — её тип (число), а дальше инициализация числом 0. Python по-прежнему является языком с динамической типизацией, отсюда и название “подсказка типа”. В реальности это влияет только на восприятие кода и предупреждения в полноценных IDE типа PyCharm. Никто не мешает вам написать `quiz_id: int = "чемодан"`, но зачем так делать?
Вернёмся в наш основной файл (я его далее буду называть `bot.py`) и импортируем наш класс: `from quizzer import Quiz`. Также добавим в начале файла под определением бота два пустых словаря:

{{< highlight none >}}
quizzes_database = {}  # здесь хранится информация о викторинах
quizzes_owners = {}  # здесь хранятся пары "id викторины <—> id её создателя"
{{< /highlight >}}

Теперь будем отлавливать викторины, приходящие в бота. Как только прилетает что-то, похожее на неё, извлекаем информацию и создаём две записи. В первом словаре храним параметры викторины, чтобы потом её воспроизвести, а во втором просто создаём пару викторина-создатель. Идентификаторы, составляющие ключ словаря, конвертируем в строки методом `str()`:

{{< gist MasterGroosha 2f8659f3623c6fe83c1f3167335e9c7f "04_content_types_poll.py" >}}

Раз уж мы сохраняем викторины, давайте теперь позволим пользователям их отправлять, причём через инлайн-режим. Есть одна загвоздка: в BotAPI через инлайн-режим нельзя напрямую отправлять опросы (нет объекта InlineQueryResultPoll), поэтому придётся доставать костыли. Будем возвращать обычное сообщение с URL-кнопкой вида https://t.me/нашбот?startgroup=id_викторины. Параметры startgroup и start — это т.н. "глубокие ссылки" ([Deep Linking](https://core.telegram.org/bots/#deep-linking)). Когда пользователь нажмёт на кнопку, он перейдёт по указанной выше ссылке, что, в свою очередь, благодаря параметру `startgroup` перекинет его к выбору группы, а затем, уже после подтверждения выбора, бот будет добавлен в группу с вызовом команды `/start id_викторины`.

Начнём разбираться с инлайн-режимом (не забудьте включить его у [@BotFather](https://t.me/botfather)). Когда пользователь вызывает нашего бота через инлайн, показываем все созданные им викторины, плюс кнопку "Создать новую". Если ничего нет, то только кнопку.

{{< gist MasterGroosha 2f8659f3623c6fe83c1f3167335e9c7f "05_inline_handler.py" >}}

Очень важно выставить флаг `is_personal` равным **True** (ответ на запрос будет уникален для каждого Telegram ID) и указать небольшое значение параметра `cache_time`, чтобы кэш инлайн-ответов оперативно обновлялся по мере появления новых викторин.  
Теперь при вызове бота через инлайн мы увидим наши сохранённые викторины, а при выборе одной из них — сообщение с кнопкой, по нажатию на которую нам предложат выбрать группу для отправки сообщения. Как только группа будет выбрана, в неё будет автоматически добавлен бот с сообщением вида `/start@имя_бота`. Но ничего не происходит! Сейчас разберёмся.

## Отправляем викторину и получаем ответы
Помните наш простой обработчик команды `/start`, возвращающий сообщение с кнопкой? Настало время переписать этот хэндлер. Первым делом, будем проверять, куда отправлено сообщение -- в диалог с ботом или нет. Если в диалог, то всё остаётся по-прежнему: приветствие (на этот раз укажем, что викторина принудительно будет сделана неанонимной) и кнопка для создания викторины.

А вот если сообщение отправлено в группу, то применяем следующую логику: проверяем количество “слов” в сообщении. Одно всегда есть (команда `/start`), но может быть и второе, невидимое в интерфейсе приложения Telegram -- параметр, переданный в качестве параметра `startgroup`, в нашем случае это ID викторины. Если второго слова нет (количество слов = 1), то показываем сообщение с предложением перейти в личку к боту с принудительным показом кнопки `/start`.

В случае, если второе слово есть, то считаем его идентификатором и пробуем отправить викторину в ту же группу. При этом мы, по сути, воспроизводим её [викторину] заново, просто от своего имени: повторяем вопрос, варианты ответов и отключаем анонимный режим, т.к. нам нужно знать, кто победитель.

**Очень важный момент**: при отправке викторины, в объекте `Message` будет записан уже новый её идентификатор, который нужно подставить в наши словари. Далее по этому новому ID мы будем смотреть и считать ответы. Побочным эффектом такого подхода будет возможность использования конкретной викторины лишь однажды и в одном чате, если отправить сообщение из инлайна в другой чат, то зашитый в ссылке инлайн-кнопки ID будет недействительным.

{{< gist MasterGroosha 2f8659f3623c6fe83c1f3167335e9c7f "02_new_start_handler.py" >}}

Далее необходимо научиться как-то обрабатывать новые ответы. В свежем обновлении API добавилось два новых типа обновлений (updates, т.е. входящие события): `PollAnswer` и просто `Poll`. Первый срабатывает при получении новых ответов в викторинах и опросах, в последнем случае ещё и при отзыве голоса (массив голосов от пользователя будет пустой). Второй срабатывает при изменении состояния опроса в целом, т.е. не только при получении новых ответов/голосов, но и при смене состояния "открыт/закрыт" и др. Опять-таки, в обучающих целях мы задействуем хэндлеры на оба типа событий. 

Начнём с `PollAnswer`. Когда прилетает событие с новым ответом на викторину, прежде всего достаём её ID, по ней ищем автора во втором словаре. Если находим, то гуляем по всем викторинам этого пользователя и ищем совпадение по ID самой викторины, т.е. в точности обратное действие, только уже в первом словаре. Когда обнаружится нужная викторина, то проверяем, верный ответ или нет (сравниваем с `correct_option_id`), и если да, то записываем ID пользователя в список победителей. Если количество победителей при этом достигает двух, то останавливаем викторину.

Остановка викторины (метод [stop_poll( )](https://core.telegram.org/bots/api#stoppoll)) вызовет срабатывание хэндлера на тип обновлений `Poll` с условием `is_closed is True`. Снова извлекаем нужный нам экземпляр класса **Quiz**, вытаскиваем ID победителей и вызываем метод [get_chat_member](https://core.telegram.org/bots/api#getchatmember), после чего, используя aiogram-ный вспомогательный метод `get_mention`, формируем ссылку на каждого из победителей в HTML-разметке и создаём поздравительное сообщение. Викторины у нас одноразовые, поэтому подчищаем за собой словари, дабы не раздувать объекты в памяти.

{{< gist MasterGroosha 2f8659f3623c6fe83c1f3167335e9c7f "03_poll_handlers.py" >}}

Код готов. Закинем викторину в группу и попросим друзей правильно ответить, а сами ответим неправильно.
После первого правильного ответа:

{{% img "l13_2.png" %}}2 ответа, только один правильный{{% /img %}}

После второго правильного ответа:

{{% img "l13_3.png" %}}3 ответа, 2 правильных, опрос закрыт{{% /img %}}

На этом всё! Если у вас возникли вопросы, не стесняйтесь задавать их в [нашем чатике](https://t.me/joinchat/ABtnIE7H7Q3TRRh8n8uNww), а если вы нашли ошибку/опечатку, либо есть чем дополнить материал, то добро пожаловать на [GitHub](https://github.com/MasterGroosha/telegram-tutorial/issues) (ну, или всё так же в чате). Полный код урока можно найти [здесь](https://github.com/MasterGroosha/telegram-tutorial/tree/master/lesson_13).

{{< btn_left relref="/docs/lesson_12" >}}Урок №12{{< /btn_left >}}
{{< btn_right relref="/docs/lesson_14" >}}Урок №14{{< /btn_right >}}