# migrate_postgres.py
from flask_sqlalchemy import SQLAlchemy
from flask import Flask
from sqlalchemy import Column, Integer, String, Boolean, text
import os

# --- CONFIG ---  
DATABASE_URL = os.environ.get("DATABASE_URL", "postgresql://user:password@localhost:5432/chatdb")

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = DATABASE_URL
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

# --- MODELE USER (PostgreSQL) ---
class User(db.Model):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    username = Column(String(80), unique=True, nullable=False)
    password_hash = Column(String(128), nullable=True)
    avatar = Column(String(200), default=None)
    online = Column(Boolean, default=False)

# --- CREATION / MIGRATION ---
with app.app_context():
    # Crée la table si elle n'existe pas
    db.create_all()
    print("✅ Table 'user' créée ou déjà existante.")

    # Vérifie si les colonnes existent (PostgreSQL)
    insp = db.inspect(db.engine)
    columns = [c['name'] for c in insp.get_columns('user')]

    if 'password_hash' not in columns:
        db.engine.execute(text('ALTER TABLE "user" ADD COLUMN password_hash VARCHAR(128)'))
        print("Colonne 'password_hash' ajoutée.")

    if 'avatar' not in columns:
        db.engine.execute(text('ALTER TABLE "user" ADD COLUMN avatar VARCHAR(200) DEFAULT NULL'))
        print("Colonne 'avatar' ajoutée.")

    print("Migration terminée ✅")
