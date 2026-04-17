from flask import Flask, render_template_string, request, redirect, url_for, session, flash
import sqlite3
import hashlib
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'wedding_gold_premium_2024'

DB_NAME = 'wedding_database.db'

# ==========================================
# 1. БАЗА ДАННЫХ (Расширенная)
# ==========================================
def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    # Пользователи с расширенными полями
    c.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            role TEXT NOT NULL,
            phone TEXT,
            email TEXT,
            city TEXT,
            experience TEXT,
            about TEXT,
            avatar TEXT,
            rating REAL DEFAULT 5.0,
            reviews_count INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Услуги с портфолио
    c.execute('''
        CREATE TABLE IF NOT EXISTS services (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            title TEXT NOT NULL,
            description TEXT,
            price TEXT,
            category TEXT NOT NULL,
            portfolio TEXT,
            video_url TEXT,
            views INTEGER DEFAULT 0,
            bookings_count INTEGER DEFAULT 0,
            is_active INTEGER DEFAULT 1,
            FOREIGN KEY(user_id) REFERENCES users(id)
        )
    ''')
    
    # Заказы с деталями
    c.execute('''
        CREATE TABLE IF NOT EXISTS bookings (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            buyer_id INTEGER,
            service_id INTEGER,
            event_date TEXT,
            event_location TEXT,
            guest_count INTEGER,
            budget TEXT,
            message TEXT,
            status TEXT DEFAULT 'new',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(buyer_id) REFERENCES users(id),
            FOREIGN KEY(service_id) REFERENCES services(id)
        )
    ''')
    
    # Отзывы
    c.execute('''
        CREATE TABLE IF NOT EXISTS reviews (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            service_id INTEGER,
            buyer_id INTEGER,
            rating INTEGER,
            comment TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(service_id) REFERENCES services(id),
            FOREIGN KEY(buyer_id) REFERENCES users(id)
        )
    ''')
    
    # Сообщения (чат)
    c.execute('''
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            sender_id INTEGER,
            receiver_id INTEGER,
            message TEXT,
            is_read INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    conn.commit()
    conn.close()
    print("✅ База данных инициализирована с расширенными таблицами!")

# ==========================================
# 2. HTML ШАБЛОНЫ (Премиум дизайн)
# ==========================================
BASE_TEMPLATE = """
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>💍 Golden Wedding Premium | Организация Свадьб</title>
    <link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;600;700&family=Lato:wght@300;400;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <style>
        :root {
            --gold: #D4AF37;
            --gold-light: #F4E5B2;
            --gold-dark: #B8860B;
            --gold-gradient: linear-gradient(135deg, #D4AF37 0%, #F4E5B2 50%, #B8860B 100%);
            --bg: #FFFEF5;
            --bg-dark: #F9F6EE;
            --text: #2C2C2C;
            --text-light: #666;
            --white: #FFFFFF;
            --shadow: 0 10px 40px rgba(212, 175, 55, 0.15);
            --shadow-hover: 0 20px 60px rgba(212, 175, 55, 0.25);
        }
        
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        body {
            font-family: 'Lato', sans-serif;
            background: var(--bg);
            color: var(--text);
            line-height: 1.6;
        }
        
        h1, h2, h3, h4, h5 {
            font-family: 'Playfair Display', serif;
            color: var(--gold-dark);
        }
        
        a { text-decoration: none; color: inherit; transition: 0.3s; }
        
        /* Навигация */
        nav {
            background: var(--white);
            padding: 15px 50px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            box-shadow: 0 2px 20px rgba(212, 175, 55, 0.1);
            position: sticky;
            top: 0;
            z-index: 1000;
        }
        
        .logo {
            font-size: 28px;
            font-weight: bold;
            background: var(--gold-gradient);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        
        .nav-links {
            display: flex;
            align-items: center;
            gap: 25px;
        }
        
        .nav-links a {
            color: var(--text);
            font-weight: 600;
            padding: 8px 16px;
            border-radius: 25px;
        }
        
        .nav-links a:hover {
            background: var(--gold-light);
            color: var(--gold-dark);
        }
        
        .btn-gold {
            background: var(--gold-gradient);
            color: white;
            padding: 12px 30px;
            border-radius: 30px;
            border: none;
            cursor: pointer;
            font-weight: bold;
            font-size: 14px;
            box-shadow: 0 4px 15px rgba(212, 175, 55, 0.3);
            transition: all 0.3s;
        }
        
        .btn-gold:hover {
            transform: translateY(-3px);
            box-shadow: 0 8px 25px rgba(212, 175, 55, 0.4);
        }
        
        .btn-outline {
            background: transparent;
            color: var(--gold);
            border: 2px solid var(--gold);
        }
        
        .btn-outline:hover {
            background: var(--gold);
            color: white;
        }
        
        /* Контейнер */
        .container {
            max-width: 1200px;
            margin: 0 auto;
            padding: 40px 20px;
            min-height: 80vh;
        }
        
        /* Hero секция */
        .hero {
            text-align: center;
            padding: 100px 20px;
            background: linear-gradient(rgba(255,255,255,0.95), rgba(255,255,255,0.9)),
                        url('https://images.unsplash.com/photo-1519741497674-611481863552?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80');
            background-size: cover;
            background-position: center;
            margin-bottom: 60px;
            border-radius: 0 0 30px 30px;
            box-shadow: var(--shadow);
        }
        
        .hero h1 {
            font-size: 56px;
            margin-bottom: 20px;
            background: var(--gold-gradient);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        
        .hero p {
            font-size: 20px;
            color: var(--text-light);
            max-width: 700px;
            margin: 0 auto 40px;
        }
        
        /* Формы */
        .form-box {
            background: var(--white);
            padding: 50px;
            border-radius: 20px;
            box-shadow: var(--shadow);
            max-width: 600px;
            margin: 0 auto;
            border-top: 5px solid var(--gold);
        }
        
        .form-box h2 {
            text-align: center;
            margin-bottom: 30px;
            font-size: 32px;
        }
        
        .form-group {
            margin-bottom: 20px;
        }
        
        .form-group label {
            display: block;
            margin-bottom: 8px;
            font-weight: 600;
            color: var(--text);
        }
        
        input, select, textarea {
            width: 100%;
            padding: 15px;
            border: 2px solid #eee;
            border-radius: 10px;
            font-family: 'Lato', sans-serif;
            font-size: 15px;
            transition: 0.3s;
        }
        
        input:focus, select:focus, textarea:focus {
            border-color: var(--gold);
            outline: none;
            box-shadow: 0 0 10px rgba(212, 175, 55, 0.2);
        }
        
        /* Карточки услуг */
        .filters {
            background: var(--white);
            padding: 30px;
            border-radius: 15px;
            margin-bottom: 40px;
            box-shadow: var(--shadow);
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
            align-items: center;
        }
        
        .filters select, .filters input {
            flex: 1;
            min-width: 200px;
        }
        
        .grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
            gap: 30px;
        }
        
        .card {
            background: var(--white);
            border-radius: 15px;
            overflow: hidden;
            box-shadow: var(--shadow);
            transition: all 0.4s;
            position: relative;
        }
        
        .card:hover {
            transform: translateY(-10px);
            box-shadow: var(--shadow-hover);
        }
        
        .card-image {
            height: 220px;
            background: var(--gold-light);
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 60px;
            color: var(--gold);
        }
        
        .card-badge {
            position: absolute;
            top: 15px;
            right: 15px;
            background: var(--gold-gradient);
            color: white;
            padding: 6px 15px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: bold;
        }
        
        .card-body {
            padding: 25px;
        }
        
        .card-title {
            font-size: 22px;
            margin-bottom: 10px;
            color: var(--text);
        }
        
        .card-rating {
            color: var(--gold);
            margin-bottom: 15px;
        }
        
        .card-rating i {
            margin-right: 3px;
        }
        
        .card-price {
            font-size: 24px;
            color: var(--gold-dark);
            font-weight: bold;
            display: block;
            margin: 15px 0;
        }
        
        .card-meta {
            display: flex;
            justify-content: space-between;
            color: var(--text-light);
            font-size: 13px;
            margin-top: 15px;
            padding-top: 15px;
            border-top: 1px solid #eee;
        }
        
        /* Статистика */
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin-bottom: 40px;
        }
        
        .stat-card {
            background: var(--white);
            padding: 30px;
            border-radius: 15px;
            text-align: center;
            box-shadow: var(--shadow);
            border-left: 4px solid var(--gold);
        }
        
        .stat-number {
            font-size: 36px;
            font-weight: bold;
            color: var(--gold-dark);
            font-family: 'Playfair Display', serif;
        }
        
        .stat-label {
            color: var(--text-light);
            margin-top: 10px;
        }
        
        /* Таблицы */
        .table-container {
            background: var(--white);
            border-radius: 15px;
            overflow: hidden;
            box-shadow: var(--shadow);
        }
        
        table {
            width: 100%;
            border-collapse: collapse;
        }
        
        th {
            background: var(--gold-light);
            padding: 15px;
            text-align: left;
            color: var(--gold-dark);
            font-weight: 600;
        }
        
        td {
            padding: 15px;
            border-bottom: 1px solid #eee;
        }
        
        tr:hover {
            background: #FFFEF5;
        }
        
        /* Сообщения */
        .flash {
            padding: 20px;
            border-radius: 10px;
            margin-bottom: 20px;
            text-align: center;
            font-weight: 500;
        }
        
        .success {
            background: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }
        
        .error {
            background: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }
        
        .info {
            background: #d1ecf1;
            color: #0c5460;
            border: 1px solid #bee5eb;
        }
        
        /* Footer */
        footer {
            background: var(--gold-dark);
            color: white;
            text-align: center;
            padding: 40px;
            margin-top: 60px;
        }
        
        footer a {
            color: var(--gold-light);
        }
        
        /* Анимации */
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }
        
        .card, .form-box, .stat-card {
            animation: fadeIn 0.6s ease-out;
        }
        
        /* Responsive */
        @media (max-width: 768px) {
            nav { padding: 15px 20px; flex-direction: column; gap: 15px; }
            .hero h1 { font-size: 36px; }
            .grid { grid-template-columns: 1fr; }
            .filters { flex-direction: column; }
        }
    </style>
</head>
<body>
    <nav>
        <a href="/" class="logo">💍 Golden Wedding</a>
        <div class="nav-links">
            {% if session.get('user_id') %}
                <a href="/"><i class="fas fa-home"></i> Главная</a>
                <a href="/catalog"><i class="fas fa-search"></i> Каталог</a>
                <a href="/dashboard"><i class="fas fa-user"></i> Кабинет</a>
                <a href="/messages"><i class="fas fa-envelope"></i> Сообщения</a>
                <a href="/logout" class="btn-gold" style="background:#ccc; color:#333;"><i class="fas fa-sign-out-alt"></i> Выйти</a>
            {% else %}
                <a href="/"><i class="fas fa-home"></i> Главная</a>
                <a href="/catalog"><i class="fas fa-search"></i> Каталог</a>
                <a href="/login"><i class="fas fa-sign-in-alt"></i> Войти</a>
                <a href="/register" class="btn-gold"><i class="fas fa-user-plus"></i> Регистрация</a>
            {% endif %}
        </div>
    </nav>
    
    <div class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="flash {{ category }}">
                        {% if category == 'success' %}✅{% elif category == 'error' %}❌{% else %}ℹ️{% endif %}
                        {{ message }}
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        {% block content %}{% endblock %}
    </div>
    
    <footer>
        <p>© 2024 Golden Wedding Premium. Все права защищены.</p>
        <p style="margin-top: 10px; font-size: 14px; opacity: 0.8;">
            <a href="#">Политика конфиденциальности</a> | 
            <a href="#">Условия использования</a> | 
            <a href="#">Поддержка</a>
        </p>
    </footer>
</body>
</html>
"""

# ==========================================
# 3. МАРШРУТЫ (Расширенные)
# ==========================================

@app.route('/')
def index():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    # Топ услуг по рейтингу
    c.execute("SELECT * FROM services WHERE is_active = 1 ORDER BY views DESC LIMIT 6")
    featured = c.fetchall()
    
    # Статистика платформы
    c.execute("SELECT COUNT(*) FROM users WHERE role = 'tamada'")
    tamada_count = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM users WHERE role = 'business'")
    business_count = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM bookings")
    booking_count = c.fetchone()[0]
    
    conn.close()
    
    content_html = """
    <div class="hero">
        <h1>Свадьба вашей мечты начинается здесь</h1>
        <p>Единая платформа для поиска лучших тамад, организаторов и декораторов. 
           Более {} специалистов готовы сделать ваш праздник незабываемым.</p>
        <div style="display: flex; gap: 15px; justify-content: center; flex-wrap: wrap;">
            <a href="/catalog" class="btn-gold" style="font-size: 18px; padding: 15px 40px;">
                <i class="fas fa-search"></i> Найти исполнителя
            </a>
            <a href="/register" class="btn-gold btn-outline" style="font-size: 18px; padding: 15px 40px;">
                <i class="fas fa-user-plus"></i> Стать исполнителем
            </a>
        </div>
    </div>
    
    <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 60px;">
        <div class="stat-card">
            <div class="stat-number">🎤 {}</div>
            <div class="stat-label">Профессиональных тамад</div>
        </div>
        <div class="stat-card">
            <div class="stat-number">💼 {}</div>
            <div class="stat-label">Организаторов праздников</div>
        </div>
        <div class="stat-card">
            <div class="stat-number">🎉 {}</div>
            <div class="stat-label">Успешных свадеб</div>
        </div>
    </div>
    
    <h2 style="margin-bottom: 30px; text-align: center;">
        <i class="fas fa-star" style="color: var(--gold);"></i> 
        Популярные исполнители
        <i class="fas fa-star" style="color: var(--gold);"></i>
    </h2>
    
    <div class="grid">
    """.format(tamada_count + business_count, tamada_count, business_count, booking_count)
    
    for s in featured:
        rating = 5.0  # Можно добавить реальный рейтинг из БД
        stars = ''.join(['<i class="fas fa-star"></i>' for _ in range(int(rating))])
        category_name = "🎤 Тамада" if s[5] == 'tamada' else "💼 Организатор"
        btn = '<a href="/service/{}" class="btn-gold" style="display:block;text-align:center;margin-top:15px;">Подробнее</a>'.format(s[0]) if session.get('role') == 'buyer' else '<a href="/service/{}" class="btn-gold btn-outline" style="display:block;text-align:center;margin-top:15px;">Подробнее</a>'.format(s[0])
        
        content_html += """
        <div class="card">
            <div class="card-badge">{}</div>
            <div class="card-image"><i class="fas fa-microphone-alt"></i></div>
            <div class="card-body">
                <h3 class="card-title">{}</h3>
                <div class="card-rating">{}</div>
                <p style="color: #666; margin-bottom: 15px;">{}</p>
                <span class="card-price">{}</span>
                {}
                <div class="card-meta">
                    <span><i class="fas fa-eye"></i> {} просмотров</span>
                    <span><i class="fas fa-calendar-check"></i> {} заказов</span>
                </div>
            </div>
        </div>
        """.format(category_name, s[1], stars, s[2][:100] + '...' if len(s[2]) > 100 else s[2], s[3], btn, s[6], s[7])
    
    content_html += """
    </div>
    
    <div style="text-align: center; margin-top: 60px; padding: 60px; background: var(--white); border-radius: 20px; box-shadow: var(--shadow);">
        <h2 style="margin-bottom: 20px;">Готовы создать незабываемую свадьбу?</h2>
        <p style="color: #666; margin-bottom: 30px; max-width: 600px; margin-left: auto; margin-right: auto;">
            Присоединяйтесь к тысячам счастливых пар, которые уже нашли своих идеальных исполнителей через Golden Wedding.
        </p>
        <a href="/register" class="btn-gold" style="font-size: 18px; padding: 15px 40px;">Начать сейчас</a>
    </div>
    """
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/catalog')
def catalog():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    # Получаем фильтры из запроса
    category = request.args.get('category', 'all')
    city = request.args.get('city', '')
    min_price = request.args.get('min_price', '')
    max_price = request.args.get('max_price', '')
    
    query = "SELECT * FROM services WHERE is_active = 1"
    params = []
    
    if category != 'all':
        query += " AND category = ?"
        params.append(category)
    
    c.execute(query, params)
    services = c.fetchall()
    conn.close()
    
    content_html = """
    <h2 style="margin-bottom: 30px;"><i class="fas fa-search"></i> Каталог исполнителей</h2>
    
    <div class="filters">
        <form method="GET" style="display: flex; gap: 15px; flex-wrap: wrap; flex: 1;">
            <select name="category" style="flex: 1; min-width: 150px;">
                <option value="all">Все категории</option>
                <option value="tamada" {}>🎤 Тамады</option>
                <option value="business" {}>💼 Организаторы</option>
            </select>
            <input type="text" name="city" placeholder="🏙 Город" value="{}" style="flex: 1; min-width: 150px;">
            <input type="number" name="min_price" placeholder="💰 От цены" value="{}" style="flex: 1; min-width: 120px;">
            <input type="number" name="max_price" placeholder="До цены" value="{}" style="flex: 1; min-width: 120px;">
            <button type="submit" class="btn-gold"><i class="fas fa-filter"></i> Применить</button>
        </form>
    </div>
    
    <div class="grid">
    """.format(
        'selected' if category == 'tamada' else '',
        'selected' if category == 'business' else '',
        city,
        min_price,
        max_price
    )
    
    for s in services:
        rating = 5.0
        stars = ''.join(['<i class="fas fa-star"></i>' for _ in range(int(rating))])
        category_name = "🎤 Тамада" if s[5] == 'tamada' else "💼 Организатор"
        btn = '<a href="/service/{}" class="btn-gold" style="display:block;text-align:center;margin-top:15px;">Подробнее</a>'.format(s[0]) if session.get('role') == 'buyer' else '<a href="/service/{}" class="btn-gold btn-outline" style="display:block;text-align:center;margin-top:15px;">Подробнее</a>'.format(s[0])
        
        content_html += """
        <div class="card">
            <div class="card-badge">{}</div>
            <div class="card-image"><i class="fas fa-microphone-alt"></i></div>
            <div class="card-body">
                <h3 class="card-title">{}</h3>
                <div class="card-rating">{}</div>
                <p style="color: #666; margin-bottom: 15px;">{}</p>
                <span class="card-price">{}</span>
                {}
                <div class="card-meta">
                    <span><i class="fas fa-eye"></i> {} просмотров</span>
                    <span><i class="fas fa-calendar-check"></i> {} заказов</span>
                </div>
            </div>
        </div>
        """.format(category_name, s[1], stars, s[2][:100] + '...' if len(s[2]) > 100 else s[2], s[3], btn, s[6], s[7])
    
    content_html += """
    </div>
    
    <div style="text-align: center; margin-top: 40px;">
        <p style="color: #666;">Найдено услуг: {}</p>
    </div>
    """.format(len(services))
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/service/<int:service_id>')
def service_detail(service_id):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    # Увеличиваем счетчик просмотров
    c.execute("UPDATE services SET views = views + 1 WHERE id = ?", (service_id,))
    conn.commit()
    
    c.execute("SELECT * FROM services WHERE id = ?", (service_id,))
    service = c.fetchone()
    
    if not service:
        conn.close()
        flash('Услуга не найдена', 'error')
        return redirect(url_for('catalog'))
    
    c.execute("SELECT * FROM users WHERE id = ?", (service[1],))
    provider = c.fetchone()
    
    c.execute("SELECT * FROM reviews WHERE service_id = ?", (service_id,))
    reviews = c.fetchall()
    
    conn.close()
    
    category_name = "🎤 Тамада" if service[5] == 'tamada' else "💼 Организатор"
    stars = ''.join(['<i class="fas fa-star"></i>' for _ in range(5)])
    
    book_form = ""
    if session.get('role') == 'buyer':
        book_form = """
        <div style="background: var(--bg-dark); padding: 30px; border-radius: 15px; margin-top: 30px;">
            <h3 style="margin-bottom: 20px;"><i class="fas fa-calendar-alt"></i> Заказать услугу</h3>
            <form method="POST" action="/book/{}">
                <div class="form-group">
                    <label>Дата мероприятия</label>
                    <input type="date" name="event_date" required>
                </div>
                <div class="form-group">
                    <label>Место проведения</label>
                    <input type="text" name="event_location" placeholder="Ресторан, адрес..." required>
                </div>
                <div class="form-group">
                    <label>Количество гостей</label>
                    <input type="number" name="guest_count" placeholder="50" required>
                </div>
                <div class="form-group">
                    <label>Ваш бюджет</label>
                    <input type="text" name="budget" placeholder="{}">
                </div>
                <div class="form-group">
                    <label>Пожелания</label>
                    <textarea name="message" rows="4" placeholder="Расскажите подробнее о вашем мероприятии..."></textarea>
                </div>
                <button type="submit" class="btn-gold" style="width: 100%; font-size: 18px;">
                    <i class="fas fa-check-circle"></i> Отправить заявку
                </button>
            </form>
        </div>
        """.format(service_id, service[3])
    elif not session.get('user_id'):
        book_form = """
        <div style="background: var(--bg-dark); padding: 30px; border-radius: 15px; margin-top: 30px; text-align: center;">
            <p style="margin-bottom: 20px; font-size: 18px;">Чтобы заказать эту услугу, необходимо войти в систему</p>
            <a href="/login" class="btn-gold"><i class="fas fa-sign-in-alt"></i> Войти</a>
            <a href="/register" class="btn-gold btn-outline" style="margin-left: 10px;"><i class="fas fa-user-plus"></i> Регистрация</a>
        </div>
        """
    
    reviews_html = ""
    if reviews:
        reviews_html = "<h3 style=\"margin: 40px 0 20px;\"><i class=\"fas fa-comments\"></i> Отзывы ({})</h3>".format(len(reviews))
        for r in reviews:
            review_stars = ''.join(['<i class="fas fa-star"></i>' for _ in range(r[2])])
            reviews_html += """
            <div style="background: var(--white); padding: 20px; border-radius: 10px; margin-bottom: 15px; box-shadow: var(--shadow);">
                <div style="display: flex; justify-content: space-between; margin-bottom: 10px;">
                    <span style="font-weight: bold;">Покупатель #{}</span>
                    <span style="color: var(--gold);">{}</span>
                </div>
                <p style="color: #666;">{}</p>
                <small style="color: #999;">{}</small>
            </div>
            """.format(r[1], review_stars, r[3], r[4])
    else:
        reviews_html = "<p style=\"color: #666; margin-top: 30px;\">Пока нет отзывов. Будьте первым!</p>"
    
    content_html = """
    <div style="display: grid; grid-template-columns: 2fr 1fr; gap: 30px;">
        <div>
            <div style="background: var(--white); padding: 40px; border-radius: 15px; box-shadow: var(--shadow);">
                <span style="background: var(--gold-gradient); color: white; padding: 6px 15px; border-radius: 20px; font-size: 12px; font-weight: bold;">{}</span>
                <h1 style="margin: 20px 0; font-size: 36px;">{}</h1>
                <div style="color: var(--gold); font-size: 20px; margin-bottom: 20px;">{}</div>
                <p style="color: #666; line-height: 1.8; font-size: 16px;">{}</p>
                
                <div style="margin-top: 30px; padding-top: 30px; border-top: 2px solid var(--gold-light);">
                    <h3><i class="fas fa-user"></i> Об исполнителе</h3>
                    <p style="color: #666; margin-top: 10px;"><strong>Имя:</strong> {}</p>
                    <p style="color: #666;"><strong>Город:</strong> {}</p>
                    <p style="color: #666;"><strong>Опыт:</strong> {}</p>
                    <p style="color: #666;"><strong>Рейтинг:</strong> {} ⭐</p>
                    <p style="color: #666;"><strong>На платформе с:</strong> {}</p>
                </div>
            </div>
            
            {}
        </div>
        
        <div>
            <div style="background: var(--white); padding: 30px; border-radius: 15px; box-shadow: var(--shadow); position: sticky; top: 100px;">
                <h3 style="margin-bottom: 20px;">Стоимость</h3>
                <div style="font-size: 32px; color: var(--gold-dark); font-weight: bold; margin-bottom: 20px;">{}</div>
                
                <div style="border-top: 1px solid #eee; padding-top: 20px;">
                    <p style="display: flex; justify-content: space-between; margin-bottom: 10px;">
                        <span style="color: #666;">Просмотры</span>
                        <span style="font-weight: bold;">{}</span>
                    </p>
                    <p style="display: flex; justify-content: space-between; margin-bottom: 10px;">
                        <span style="color: #666;">Заказы</span>
                        <span style="font-weight: bold;">{}</span>
                    </p>
                    <p style="display: flex; justify-content: space-between;">
                        <span style="color: #666;">Рейтинг</span>
                        <span style="font-weight: bold; color: var(--gold);">⭐ {}</span>
                    </p>
                </div>
                
                {}
                
                <div style="margin-top: 30px; padding-top: 20px; border-top: 1px solid #eee;">
                    <p style="color: #666; font-size: 14px;"><i class="fas fa-shield-alt"></i> Безопасная сделка</p>
                    <p style="color: #666; font-size: 14px;"><i class="fas fa-check-circle"></i> Проверенный исполнитель</p>
                </div>
            </div>
        </div>
    </div>
    
    {}
    """.format(
        category_name, service[1], stars, service[2],
        provider[1], provider[6] or 'Не указан', provider[7] or 'Не указан', provider[9], provider[11],
        book_form,
        service[3], service[6], service[7], provider[9],
        '<a href="/chat/{}" class="btn-gold btn-outline" style="width: 100%; margin-top: 20px;"><i class="fas fa-comment"></i> Написать сообщение</a>'.format(provider[0]) if session.get('user_id') else '',
        reviews_html
    )
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = hashlib.sha256(request.form['password'].encode()).hexdigest()
        role = request.form['role']
        phone = request.form['phone']
        email = request.form.get('email', '')
        city = request.form.get('city', '')
        experience = request.form.get('experience', '')
        about = request.form.get('about', '')
        
        try:
            conn = sqlite3.connect(DB_NAME)
            c = conn.cursor()
            c.execute("""
                INSERT INTO users (username, password, role, phone, email, city, experience, about) 
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            """, (username, password, role, phone, email, city, experience, about))
            conn.commit()
            conn.close()
            flash('✅ Регистрация успешна! Теперь войдите в систему.', 'success')
            return redirect(url_for('login'))
        except Exception as e:
            flash('❌ Такое имя пользователя уже занято.', 'error')
    
    content_html = """
    <div class="form-box">
        <h2><i class="fas fa-user-plus"></i> Регистрация</h2>
        <p style="text-align: center; color: #666; margin-bottom: 30px;">
            Присоединяйтесь к платформе Golden Wedding Premium
        </p>
        <form method="POST">
            <div class="form-group">
                <label>Ваше имя / Название компании *</label>
                <input type="text" name="username" required placeholder="Иван Иванов или ООО Праздник">
            </div>
            
            <div class="form-group">
                <label>Email</label>
                <input type="email" name="email" placeholder="example@mail.ru">
            </div>
            
            <div class="form-group">
                <label>Пароль *</label>
                <input type="password" name="password" required placeholder="Минимум 6 символов">
            </div>
            
            <div class="form-group">
                <label>Телефон для связи *</label>
                <input type="text" name="phone" required placeholder="+7 (999) 000-00-00">
            </div>
            
            <div class="form-group">
                <label>Город</label>
                <input type="text" name="city" placeholder="Москва">
            </div>
            
            <div class="form-group">
                <label>Кто вы? *</label>
                <select name="role" id="roleSelect" onchange="toggleFields()">
                    <option value="buyer">👰 Покупатель (Жених/Невеста)</option>
                    <option value="tamada">🎤 Тамада (Ведущий)</option>
                    <option value="business">💼 Предприниматель (Организатор/Декор)</option>
                </select>
            </div>
            
            <div id="providerFields" style="display: none;">
                <div class="form-group">
                    <label>Опыт работы</label>
                    <input type="text" name="experience" placeholder="Например: 5 лет">
                </div>
                <div class="form-group">
                    <label>О себе</label>
                    <textarea name="about" rows="4" placeholder="Расскажите о себе, своем стиле работы..."></textarea>
                </div>
            </div>
            
            <button type="submit" class="btn-gold" style="width: 100%; font-size: 16px; margin-top: 10px;">
                <i class="fas fa-check"></i> Зарегистрироваться
            </button>
        </form>
        <p style="text-align: center; margin-top: 25px; color: #666;">
            Уже есть аккаунт? <a href="/login" style="color: var(--gold); font-weight: bold;">Войти</a>
        </p>
    </div>
    
    <script>
    function toggleFields() {
        var role = document.getElementById('roleSelect').value;
        var providerFields = document.getElementById('providerFields');
        if (role === 'tamada' || role === 'business') {
            providerFields.style.display = 'block';
        } else {
            providerFields.style.display = 'none';
        }
    }
    </script>
    """
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = hashlib.sha256(request.form['password'].encode()).hexdigest()
        
        conn = sqlite3.connect(DB_NAME)
        c = conn.cursor()
        c.execute("SELECT * FROM users WHERE username = ? AND password = ?", (username, password))
        user = c.fetchone()
        conn.close()
        
        if user:
            session['user_id'] = user[0]
            session['username'] = user[1]
            session['role'] = user[3]
            flash('🎉 Добро пожаловать, {}!'.format(user[1]), 'success')
            return redirect(url_for('dashboard'))
        else:
            flash('❌ Неверный логин или пароль', 'error')
    
    content_html = """
    <div class="form-box">
        <h2><i class="fas fa-sign-in-alt"></i> Вход в систему</h2>
        <p style="text-align: center; color: #666; margin-bottom: 30px;">
            Войдите для доступа к личному кабинету
        </p>
        <form method="POST">
            <div class="form-group">
                <label>Имя пользователя</label>
                <input type="text" name="username" required>
            </div>
            <div class="form-group">
                <label>Пароль</label>
                <input type="password" name="password" required>
            </div>
            <button type="submit" class="btn-gold" style="width: 100%; font-size: 16px; margin-top: 10px;">
                <i class="fas fa-sign-in-alt"></i> Войти
            </button>
        </form>
        <p style="text-align: center; margin-top: 25px; color: #666;">
            Нет аккаунта? <a href="/register" style="color: var(--gold); font-weight: bold;">Зарегистрироваться</a>
        </p>
        <p style="text-align: center; margin-top: 15px; color: #999; font-size: 14px;">
            <a href="#" style="color: var(--gold);">Забыли пароль?</a>
        </p>
    </div>
    """
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        flash('⚠️ Пожалуйста, войдите в систему', 'error')
        return redirect(url_for('login'))
    
    role = session['role']
    user_id = session['user_id']
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    content_html = ""
    
    if role == 'buyer':
        # Статистика покупателя
        c.execute("SELECT COUNT(*) FROM bookings WHERE buyer_id = ?", (user_id,))
        total_orders = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM bookings WHERE buyer_id = ? AND status = 'new'", (user_id,))
        new_orders = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM bookings WHERE buyer_id = ? AND status = 'confirmed'", (user_id,))
        confirmed_orders = c.fetchone()[0]
        
        content_html = """
        <h2 style="margin-bottom: 30px;"><i class="fas fa-user"></i> Кабинет Покупателя</h2>
        
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Всего заказов</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Новые заявки</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Подтверждено</div>
            </div>
        </div>
        
        <h3 style="margin: 40px 0 20px;"><i class="fas fa-list"></i> Мои заказы</h3>
        <div class="table-container">
            <table>
                <thead>
                    <tr>
                        <th>Услуга</th>
                        <th>Тип</th>
                        <th>Дата</th>
                        <th>Статус</th>
                        <th>Действия</th>
                    </tr>
                </thead>
                <tbody>
        """.format(total_orders, new_orders, confirmed_orders)
        
        c.execute("""
            SELECT bookings.id, services.title, services.category, bookings.event_date, bookings.status, bookings.created_at
            FROM bookings
            JOIN services ON bookings.service_id = services.id
            WHERE bookings.buyer_id = ?
            ORDER BY bookings.created_at DESC
        """, (user_id,))
        orders = c.fetchall()
        
        if orders:
            for o in orders:
                cat_name = "🎤 Тамада" if o[2] == 'tamada' else "💼 Организатор"
                status_color = "#28a745" if o[4] == 'confirmed' else "#ffc107" if o[4] == 'new' else "#dc3545"
                content_html += """
                <tr>
                    <td><strong>{}</strong></td>
                    <td>{}</td>
                    <td>{}</td>
                    <td><span style="background: {}; color: white; padding: 5px 15px; border-radius: 15px; font-size: 12px;">{}</span></td>
                    <td>
                        <a href="/service/{}" class="btn-gold btn-outline" style="padding: 5px 15px; font-size: 12px;">Просмотр</a>
                    </td>
                </tr>
                """.format(o[1], cat_name, o[3] or 'Не указана', status_color, o[4], o[0])
        else:
            content_html += """
                <tr>
                    <td colspan="5" style="text-align: center; padding: 40px; color: #666;">
                        У вас пока нет заказов. <a href="/catalog" style="color: var(--gold);">Перейти в каталог</a>
                    </td>
                </tr>
            """
        
        content_html += """
                </tbody>
            </table>
        </div>
        
        <div style="margin-top: 40px; text-align: center;">
            <a href="/catalog" class="btn-gold" style="font-size: 18px; padding: 15px 40px;">
                <i class="fas fa-search"></i> Найти еще услуги
            </a>
        </div>
        """
        
    else:
        # Статистика исполнителя
        c.execute("SELECT COUNT(*) FROM services WHERE user_id = ?", (user_id,))
        total_services = c.fetchone()[0]
        c.execute("SELECT SUM(views) FROM services WHERE user_id = ?", (user_id,))
        total_views = c.fetchone()[0] or 0
        c.execute("SELECT COUNT(*) FROM bookings JOIN services ON bookings.service_id = services.id WHERE services.user_id = ?", (user_id,))
        total_bookings = c.fetchone()[0]
        
        c.execute("SELECT * FROM services WHERE user_id = ?", (user_id,))
        my_services = c.fetchall()
        
        content_html = """
        <h2 style="margin-bottom: 30px;"><i class="fas fa-briefcase"></i> Кабинет Исполнителя</h2>
        
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Услуг в каталоге</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Просмотров</div>
            </div>
            <div class="stat-card">
                <div class="stat-number">{}</div>
                <div class="stat-label">Заказов</div>
            </div>
        </div>
        """.format(total_services, total_views, total_bookings)
        
        if my_services:
            content_html += """
            <h3 style="margin: 40px 0 20px;"><i class="fas fa-list"></i> Мои услуги</h3>
            <div class="table-container">
                <table>
                    <thead>
                        <tr>
                            <th>Название</th>
                            <th>Категория</th>
                            <th>Цена</th>
                            <th>Просмотры</th>
                            <th>Заказы</th>
                            <th>Действия</th>
                        </tr>
                    </thead>
                    <tbody>
            """
            
            for s in my_services:
                cat_name = "🎤 Тамада" if s[5] == 'tamada' else "💼 Организатор"
                content_html += """
                <tr>
                    <td><strong>{}</strong></td>
                    <td>{}</td>
                    <td>{}</td>
                    <td>{}</td>
                    <td>{}</td>
                    <td>
                        <a href="/service/{}" class="btn-gold btn-outline" style="padding: 5px 15px; font-size: 12px;">Просмотр</a>
                        <a href="/edit_service/{}" class="btn-gold btn-outline" style="padding: 5px 15px; font-size: 12px;">Редактировать</a>
                    </td>
                </tr>
                """.format(s[1], cat_name, s[3], s[6], s[7], s[0], s[0])
            
            content_html += """
                    </tbody>
                </table>
            </div>
            """
        else:
            content_html += """
            <div style="background: var(--white); padding: 40px; border-radius: 15px; box-shadow: var(--shadow); text-align: center; margin-top: 30px;">
                <h3 style="margin-bottom: 20px;"><i class="fas fa-plus-circle"></i> Добавьте свою первую услугу</h3>
                <p style="color: #666; margin-bottom: 30px; max-width: 600px; margin-left: auto; margin-right: auto;">
                    Создайте анкету, чтобы покупатели могли найти вас и заказать ваши услуги.
                </p>
                <form method="POST" action="/add_service">
                    <input type="hidden" name="category" value="{}">
                    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px; max-width: 600px; margin: 0 auto 20px;">
                        <input type="text" name="title" placeholder="Название услуги" required>
                        <input type="text" name="price" placeholder="Цена" required>
                    </div>
                    <textarea name="description" placeholder="Описание услуги" rows="4" style="max-width: 600px; margin: 0 auto 20px; display: block;"></textarea>
                    <button type="submit" class="btn-gold" style="width: auto; padding: 15px 40px;">
                        <i class="fas fa-check"></i> Опубликовать
                    </button>
                </form>
            </div>
            """.format(role)
    
    conn.close()
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/add_service', methods=['POST'])
def add_service():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        INSERT INTO services (user_id, title, description, price, category) 
        VALUES (?, ?, ?, ?, ?)
    """, (session['user_id'], request.form['title'], request.form['description'], request.form['price'], request.form['category']))
    conn.commit()
    conn.close()
    
    flash('✅ Услуга успешно опубликована!', 'success')
    return redirect(url_for('dashboard'))

@app.route('/book/<int:service_id>', methods=['POST'])
def book(service_id):
    if 'user_id' not in session or session['role'] != 'buyer':
        flash('⚠️ Только покупатели могут делать заказы', 'error')
        return redirect(url_for('login'))
    
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    
    # Обновляем счетчик заказов
    c.execute("UPDATE services SET bookings_count = bookings_count + 1 WHERE id = ?", (service_id,))
    
    c.execute("""
        INSERT INTO bookings (buyer_id, service_id, event_date, event_location, guest_count, budget, message)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        session['user_id'],
        service_id,
        request.form.get('event_date'),
        request.form.get('event_location'),
        request.form.get('guest_count'),
        request.form.get('budget'),
        request.form.get('message')
    ))
    conn.commit()
    conn.close()
    
    flash('✅ Заявка отправлена! Исполнитель свяжется с вами.', 'success')
    return redirect(url_for('dashboard'))

@app.route('/messages')
def messages():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    content_html = """
    <h2 style="margin-bottom: 30px;"><i class="fas fa-envelope"></i> Сообщения</h2>
    <div style="background: var(--white); padding: 40px; border-radius: 15px; box-shadow: var(--shadow); text-align: center;">
        <i class="fas fa-comments" style="font-size: 60px; color: var(--gold); margin-bottom: 20px;"></i>
        <h3>Чат с исполнителями</h3>
        <p style="color: #666; margin: 20px 0;">Здесь будут отображаться ваши диалоги с тамадами и организаторами</p>
        <a href="/catalog" class="btn-gold"><i class="fas fa-search"></i> Найти исполнителя</a>
    </div>
    """
    
    final_html = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', content_html)
    return render_template_string(final_html)

@app.route('/logout')
def logout():
    session.clear()
    flash('👋 Вы успешно вышли из системы', 'info')
    return redirect(url_for('index'))

if __name__ == '__main__':
    init_db()
    print("🚀 Сайт запущен: http://127.0.0.1:5000")
    print("📊 База данных: wedding_database.db")
    app.run(debug=True)
