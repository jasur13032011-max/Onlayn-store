# Onlayn-store
team jasur
kod 
from flask import Flask, request, jsonify
from flask_jwt_extended import (
    JWTManager, create_access_token, jwt_required, get_jwt_identity
)
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)

# JWT maxfiy kalitini sozlash
app.config["JWT_SECRET_KEY"] = "super-secret-key-change-this"
jwt = JWTManager(app)

# Ma'lumotlar bazasi o'rnida ishlatiladigan vaqtinchalik xotira (In-memory DB)
users_db = {}  # { username: password_hash }
products_db = [
    {"id": 1, "title": "Noutbuk", "price": 1200.0},
    {"id": 2, "title": "Smartfon", "price": 800.0},
    {"id": 3, "title": "Quloqchin", "price": 150.0}
]
carts_db = {}  # { username: [ {product_id, quantity}, ... ] }


# 1. Ro'yxatdan o'tish (Register)
@app.route('/api/register', methods=['POST'])
def register():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    if not username or not password:
        return jsonify({"message": "Username va password kiritilishi shart"}), 400

    if username in users_db:
        return jsonify({"message": "Bu foydalanuvchi nomi band"}), 400

    # Parolni xavfsiz heshlash
    users_db[username] = generate_password_hash(password)
    carts_db[username] = []  # Yangi foydalanuvchiga bo'sh savat yaratish

    return jsonify({"message": "Foydalanuvchi muvaffaqiyatli ro'yxatdan o'tdi"}), 201


# 2. Login
@app.route('/api/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    user_hash = users_db.get(username)

    if not user_hash or not check_password_hash(user_hash, password):
        return jsonify({"message": "Username yoki parol noto'g'ri"}), 401

    # JWT token yaratish
    access_token = create_access_token(identity=username)
    return jsonify({"access_token": access_token}), 200


# 3. Mahsulotlar ro'yxatini olish (Ochiq endpoint)
@app.route('/api/products', methods=['GET'])
def get_products():
    return jsonify({"products": products_db}), 200


# 4. Savatga mahsulot qo'shish (Faqat token bilan)
@app.route('/api/cart', methods=['POST'])
@jwt_required()
def add_to_cart():
    # Tokendan foydalanuvchi shaxsini aniqlash
    current_user = get_jwt_identity()
    data = request.get_json()
    
    product_id = data.get('product_id')
    quantity = data.get('quantity', 1)

    if not product_id:
        return jsonify({"message": "product_id ko'rsatilishi kerak"}), 400

    # Mahsulot mavjudligini tekshirish
    product = next((p for p in products_db if p["id"] == product_id), None)
    if not product:
        return jsonify({"message": "Mahsulot topilmadi"}), 404

    # Savatga qo'shish
    user_cart = carts_db.setdefault(current_user, [])
    user_cart.append({
        "product_id": product_id,
        "title": product["title"],
        "price": product["price"],
        "quantity": quantity
    })

    return jsonify({
        "message": "Mahsulot savatga qo'shildi",
        "cart": user_cart
    }), 200


if __name__ == '__main__':
    app.run(debug=True)

body json
{
  "username": "ali",
  "password": "secretpassword123"
}
{
  "username": "ali",
  "password": "secretpassword123"
}
{
  "access_token": "eyJhbGciOiJIUzI1Ni..."
}
mahsulotlar royhati token tala qimidi
savat
Authorization: Bearer <Siz olgan access_token>
body
{
  "product_id": 1,
  "quantity": 2
}
get delete
# 5. Savat tarkibini ko'rish (Faqat token bilan)
@app.route('/api/cart', methods=['GET'])
@jwt_required()
def get_cart():
    current_user = get_jwt_identity()
    user_cart = carts_db.get(current_user, [])
    
    # Umumiy summani hisoblash
    total_price = sum(item['price'] * item['quantity'] for item in user_cart)
    
    return jsonify({
        "cart": user_cart,
        "total_items": len(user_cart),
        "total_price": total_price
    }), 200


# 6. Savatdan mahsulotni o'chirish yoki savatni to'liq tozalash (Faqat token bilan)
@app.route('/api/cart', methods=['DELETE'])
@jwt_required()
def delete_from_cart():
    current_user = get_jwt_identity()
    data = request.get_json(silent=True) or {}
    
    product_id = data.get('product_id')

    # Agar product_id berilmagan bo'lsa, butun savatni tozalaymiz
    if not product_id:
        carts_db[current_user] = []
        return jsonify({"message": "Savat to'liq tozalandi"}), 200

    # Bitta mahsulotni o'chirish
    user_cart = carts_db.get(current_user, [])
    updated_cart = [item for item in user_cart if item['product_id'] != product_id]

    if len(user_cart) == len(updated_cart):
        return jsonify({"message": "Savatda bunday mahsulot topilmadi"}), 404

    carts_db[current_user] = updated_cart
    return jsonify({
        "message": "Mahsulot savatdan o'chirildi",
        "cart": updated_cart
    }), 200

json
{
  "cart": [
    {
      "product_id": 1,
      "title": "Noutbuk",
      "price": 1200.0,
      "quantity": 2
    }
  ],
  "total_items": 1,
  "total_price": 2400.0
}

{
  "product_id": 1
}
pip install Flask Flask-JWT-Extended Werkzeug Flask-SQLAlchemy

from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import (
    JWTManager, create_access_token, jwt_required, get_jwt_identity
)
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)

# Konfiguratsiyalar
app.config["JWT_SECRET_KEY"] = "super-secret-key-change-this"
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///app.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)
jwt = JWTManager(app)


# ================= MA'LUMOTLAR BAZASI MODELLARI =================

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(200), nullable=False)
    cart_items = db.relationship('CartItem', backref='user', lazy=True, cascade="all, delete-orphan")

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    price = db.Column(db.Float, nullable=False)

class CartItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    product_id = db.Column(db.Integer, db.ForeignKey('product.id'), nullable=False)
    quantity = db.Column(db.Integer, default=1)
    
    product = db.relationship('Product')


# ================= ENDPOINTLAR =================

# 1. Ro'yxatdan o'tish (Register)
@app.route('/api/register', methods=['POST'])
def register():
    data = request.get_json() or {}
    username = data.get('username')
    password = data.get('password')

    if not username or not password:
        return jsonify({"message": "Username va password kiritilishi shart"}), 400

    if User.query.filter_by(username=username).first():
        return jsonify({"message": "Bu foydalanuvchi nomi band"}), 400

    hashed_pw = generate_password_hash(password)
    new_user = User(username=username, password_hash=hashed_pw)
    
    db.session.add(new_user)
    db.session.commit()

    return jsonify({"message": "Foydalanuvchi muvaffaqiyatli ro'yxatdan o'tdi"}), 201


# 2. Login
@app.route('/api/login', methods=['POST'])
def login():
    data = request.get_json() or {}
    username = data.get('username')
    password = data.get('password')

    user = User.query.filter_by(username=username).first()

    if not user or not check_password_hash(user.password_hash, password):
        return jsonify({"message": "Username yoki parol noto'g'ri"}), 401

    # JWT tokenga user.id biriktiriladi
    access_token = create_access_token(identity=str(user.id))
    return jsonify({"access_token": access_token}), 200


# 3. Mahsulotlar ro'yxatini olish (Ochiq)
@app.route('/api/products', methods=['GET'])
def get_products():
    products = Product.query.all()
    product_list = [
        {"id": p.id, "title": p.title, "price": p.price} for p in products
    ]
    return jsonify({"products": product_list}), 200


# 4. Savatga mahsulot qo'shish (Token talab qilinadi)
@app.route('/api/cart', methods=['POST'])
@jwt_required()
def add_to_cart():
    user_id = int(get_jwt_identity())
    data = request.get_json() or {}
    
    product_id = data.get('product_id')
    quantity = data.get('quantity', 1)

    if not product_id:
        return jsonify({"message": "product_id ko'rsatilishi kerak"}), 400

    # Mahsulot bazada borligini tekshirish
    product = Product.query.get(product_id)
    if not product:
        return jsonify({"message": "Mahsulot topilmadi"}), 404

    # Savatda ushbu mahsulot allaqachon bor-yo'qligini tekshirish
    cart_item = CartItem.query.filter_by(user_id=user_id, product_id=product_id).first()
    
    if cart_item:
        cart_item.quantity += quantity
    else:
        cart_item = CartItem(user_id=user_id, product_id=product_id, quantity=quantity)
        db.session.add(cart_item)

    db.session.commit()
    return jsonify({"message": "Mahsulot savatga qo'shildi"}), 200


# 5. Savat tarkibini ko'rish (Token talab qilinadi)
@app.route('/api/cart', methods=['GET'])
@jwt_required()
def get_cart():
    user_id = int(get_jwt_identity())
    cart_items = CartItem.query.filter_by(user_id=user_id).all()

    result = []
    total_price = 0.0

    for item in cart_items:
        item_total = item.product.price * item.quantity
        total_price += item_total
        result.append({
            "product_id": item.product_id,
            "title": item.product.title,
            "price": item.product.price,
            "quantity": item.quantity
        })

    return jsonify({
        "cart": result,
        "total_items": len(result),
        "total_price": total_price
    }), 200


# 6. Savatdan mahsulotni o'chirish yoki savatni to'liq tozalash (Token talab qilinadi)
@app.route('/api/cart', methods=['DELETE'])
@jwt_required()
def delete_from_cart():
    user_id = int(get_jwt_identity())
    data = request.get_json(silent=True) or {}
    product_id = data.get('product_id')

    # Agar product_id berilmagan bo'lsa, foydalanuvchining butun savatini tozalaymiz
    if not product_id:
        CartItem.query.filter_by(user_id=user_id).delete()
        db.session.commit()
        return jsonify({"message": "Savat to'liq tozalandi"}), 200

    # Bitta mahsulotni o'chirish
    cart_item = CartItem.query.filter_by(user_id=user_id, product_id=product_id).first()
    if not cart_item:
        return jsonify({"message": "Savatda bunday mahsulot topilmadi"}), 404

    db.session.delete(cart_item)
    db.session.commit()

    return jsonify({"message": "Mahsulot savatdan o'chirildi"}), 200


# ================= BAZANI DASTLABKI TO'LDIRISH VA ISHGA TUSHIRISH =================

def init_db():
    """Baza jadvallarini yaratish va boshlang'ich mahsulotlarni qo'shish."""
    db.create_all()
    if Product.query.count() == 0:
        sample_products = [
            Product(title="Noutbuk", price=1200.0),
            Product(title="Smartfon", price=800.0),
            Product(title="Quloqchin", price=150.0)
        ]
        db.session.bulk_save_objects(sample_products)
        db.session.commit()

if __name__ == '__main__':
    with app.app_context():
        init_db()
    app.run(debug=True)

User, Product, va CartItem obyektlari SQLite ma'lumotlar bazasida saqlanadi (app.db fayli avtomasiyalashgan holda hosil bo'ladi
Dastlabki ma'lumotlar (Seed Data): Ilova birinchi marta ishga tushganda init_db() funksiyasi jadvallarni avtomatik yaratib, sinov uchun bir nechta mahsulot qo'shadi.
Foreign Keys va Relationship: Foydalanuvchi va mahsulotlar o'rtasidagi bog'lanish CartItem modeli orqali amalga oshirilgan.
JWT Identity: JWT tokenda foydalanuvchining unikal user.id si saqlanadi, bu esa qidiruv unumdorligini oshiradi.
JWT autentifikatsiyasi sozlandi:
Foydalanuvchi ro'yxatdan o'tishi (/api/register) va tizimga kirishi (/api/login) uchun endpointlar yaratildi.
Parollar xavfsiz holda heshlandi (Werkzeug orqali).
Login qilgan foydalanuvchiga xavfsiz JWT token berilishi ta'minlandi.
Mahsulotlar va Savat boshqaruvi qo'shildi:
Mahsulotlar ro'yxati (GET /api/products): Ochiq endpoint, barcha mahsulotlarni JSON formatida qaytaradi.
Savatga qo'shish (POST /api/cart): Faqat JWT token yuborgan autentifikatsiyadan o'tgan foydalanuvchilar uchun ishlaydi.
Savatni ko'rish va o'chirish (GET /api/cart, DELETE /api/cart): Savatdagi mahsulotlarni, umumiy summani ko'rish hamda alohida mahsulotni yoki butun savatni tozalash funksiyalari qo'shildi.
Vaqtinchalik xotiradan haqiqiy Ma'lumotlar Bazasiga o'tildi:
Dastlabki in-memory (vaqtinchalik xotira) o'rniga Flask-SQLAlchemy va SQLite bazasi ulandi.
User, Product va CartItem obyektlari o'rtasida relatsion bog'liqliklar yaratildi.
Natijada tayyor, xavfsiz va SQLite ma'lumotlar bazasiga ulangan REST API hosil bo'ldi:
Dastur ishga tushganda app.db bazasi avtomatik yaratiladi va namuna mahsulotlar bilan to'ldiriladi.
Har bir foydalanuvchi o'z login/paroli bilan kirib, unikal token oladi va faqat o'z savatiga mahsulot joylaydi hamda uni boshqaradi.
Loyiha kodi to'liq REST API standartlariga va JSON formatidagi ma'lumot almashinuviga mos keladi.
Team JASUR
JASUR RIZAMATOV
Aziz Sobirjonov
Dilshoda Zakirjanova
Shodiya Rajjabova
