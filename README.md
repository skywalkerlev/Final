Нотатка — це невеликий і простий веб-додаток, створений за допомогою фреймворків початкового завантаження (інтерфейс) і флеш-системи (бекенд). Це допомагає користувачам робити нотатки, редагувати або видаляти їх.

Детальніше про проект я напишу в наступних розділах.
каталог 'templates'

Цей містить усі шаблони (файли HTML), які відтворить веб-програма.

Для створення шаблонів використовувався механізм створення шаблонів Jinja (який є механізмом створення шаблонів за замовчуванням, інтегрованим із flask). Усі шаблони розширюють шаблон "layout.html". Я використовував деякі компоненти початкового завантаження, такі як навігаційна панель, картки, кнопки, спадне меню, модальний режим і сповіщення. Я також додав деякі власні стилі за допомогою css.
«статичний» каталог

Він містить таблиці стилів (файли CSS), сценарії (файли JavaScript) і зображення.
style.css

Ця таблиця стилів містить усі спеціальні стилі, застосовані до HTML-сторінок.

Існують набори правил (блоки властивостей: пари значень «також відомі як delcarations»), які керують макетом сторінки за допомогою flexbox, замінюють переповнений текст еліпсом, додають ефекти наведення курсора на елементи та змінюють кольори фону та тексту елементів у темному режимі. Щоб зробити колір тексту заповнювача полів введення світлішим у темному режимі, я використав псевдоелемент : :placeholder pseudo-element.


script.js

Сценарій відповідає за ввімкнення спливаючих підказок початкового завантаження, здатність приховувати сповіщення, підсвічувати активний елемент панелі навігації та перемикати темну/світлу тему.

Реалізація темного/світлого режиму

У «script.js» ми визначаємо властивість (змінна всередині об’єкта в JavaScript) об’єкта localStorage, який зберігає поточну тему (або "light", або "dark").

if (!localStorage.getItem("currentTheme")) {
  localStorage.setItem("currentTheme", "light");
}

ми спочатку перевіряємо, чи властивість 'currentTheme' вже встановлено, і якщо це не так, ми встановлюємо його значення на "light".

Далі ми створюємо функцію слухача, яка викликається, коли натискається кнопка '.switch-theme-btn',

function switchTheme() {
  if (localStorage.getItem("currentTheme") === "light") {
    localStorage.setItem("currentTheme", "dark");
  } else {
    localStorage.setItem("currentTheme", "light");
  }

  location.reload();
}

const switchThemeBtn = document.querySelector(".switch-theme-btn");
switchThemeBtn.addEventListener("click", switchTheme);

все, що він робить, це змінює значення властивості "currentTheme" на "light" або "dark" залежно від поточного значення, а потім перезавантажує сторінку.

Нарешті, після вибору елементів DOM, які нам потрібно буде оновити, ми перевіряємо, яку тему вибрав користувач, і відповідно застосовуємо стилі до цих елементів

if (localStorage.getItem("currentTheme") === "dark") {
.
.
.
}


app.py

Це основний файл програми. Він містить більшість програмного коду та функцій.

Файл починається з імпорту всіх необхідних модулів.

Потім переходить до створення програми та її налаштування.

По-перше, він створює екземпляр програми Flask для поточного файлу, що робиться в рядку 9 таким чином:

app = Flask(__name__)

Тоді це дозволяє додатку отримувати та надсилати запити та відповіді між джерелами.

CORS(app)

Потім він налаштовує програму на використання файлів cookie, що дозволить їй запам’ятовувати інформацію навіть після закриття веб-програми, наприклад (у нашому випадку) ідентифікатор користувача, який увійшов у систему.

app.config["SESSION_PERMANENT"] = False
app.config["SESSION_TYPE"] = "файлова система"
Session(app)

Ми додаємо спеціальний фільтр jinja 'created_since', який використовуватиметься для визначення того, як давно була створена нотатка (про це пізніше).

app.jinja_env.filters["created_since"] = created_since

Тепер, після створення та налаштування програми, ми створюємо та налаштовуємо базу даних, яка буде зберігати всі дані користувачів.

У цьому рядку коду ми створюємо з’єднання з файлом бази даних notesapp.db

connection = sqlite3.connect("notesapp.db", check_same_thread=False)

Ми також створюємо об’єкт курсора, щоб мати можливість виконувати запити SQL

cursor = connection.cursor()

Ось як ми можемо використовувати цей об’єкт для створення нової таблиці

cursor.execute("""CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            hash TEXT NOT NULL
        )""")

Код створення таблиць вкладено в блок try/except, щоб уникнути виникнення будь-яких винятків (зокрема, винятку "table already exists") під час повторного запуску сервера.

Ми створюємо два індекси для стовпців «id» у таблицях «users» і «notes», щоб пришвидшити транзакції.

cursor.execute("CREATE UNIQUE INDEX 'user_id_index' ON 'users' ('id')")
cursor.execute("CREATE UNIQUE INDEX 'note_id_index' ON 'notes' ('id')")

Нарешті, ми закриваємо зв'язок з

connection.close()

ми відкриємо його знову, коли нам це знадобиться (перед обробкою запиту).

Далі створюємо дві функції:

    1) open_db_conn() виконується перед обробкою запиту (це робиться шляхом прикрашання функції @before_request). Він відкриває з’єднання з базою даних, створює об’єкт курсора та робить їх доступними глобально, додаючи їх до глобального об’єкта g, наданого flask.
    2) close_db_conn() виконується після обробки запиту (це робиться шляхом прикрашання функції @after_request). Він закриває підключення до бази даних.

Решта файлу містить усі визначення маршрутів та їхні функції перегляду. Функції перегляду містять код, який виконує операції CRUD з базою даних, перевірку на стороні сервера або просто рендерить шаблон. Маршрути, які вимагають авторизації користувача, прикрашаються декоратором @login_required. Це підсумовує цю частину файлу, нам не потрібно буде вдаватися в деталі кожного визначення маршруту та функції перегляду.


helpers.py

Цей файл містить допоміжні функції.

Він також починається з імпортування модулів, які він використовуватиме.

Потім ми визначаємо деякі функції, які абстрагують код для виконання операцій CRUD, які ми будемо виконувати в базі даних. Усі функції get_username(), add_note(), get_notes(), get_note(id), edit_note(id, title, content), delete_note(id) виконують операції CRUD і завжди виконуються під час обробки запиту.

Далі ми визначаємо функцію login_required(f), яка використовуватиметься для оформлення маршрутів, які вимагають від користувача входу в систему. Більше можна знайти в документації.

Нарешті, ми визначаємо функцію created_since(note_id), яка приймає ідентифікатор нотатки та повертає час тому, як ця нотатка була створена, у вигляді рядка у форматі: «[amount of time] [days|months|years] тому». Формат відрізняється залежно від кількості часу, наприклад, «менше за день» повертається, якщо кількість часу менша за добу.


requirements.txt

Містить назви та версії пакетів, які використовуються в проекті.
