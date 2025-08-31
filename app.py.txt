# ---------- MONKEY PATCH POUR EVENTLET (doit être tout en haut) ----------
import eventlet
eventlet.monkey_patch()

# ---------- IMPORTS ----------
from flask import Flask, render_template, request, redirect, url_for, flash, session, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
from flask_socketio import SocketIO, emit
from functools import wraps
from urllib.parse import urlparse, urljoin
from datetime import datetime
import os
import cloudinary
import cloudinary.uploader
import cloudinary.api

# ---------- CONFIG ----------
app = Flask(__name__)
app.config['SECRET_KEY'] = "SuperCleSecrete2025!"
import os
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'instance', 'users.db')
app.config['MAX_CONTENT_LENGTH'] = 20 * 1024 * 1024
app.config['SESSION_COOKIE_SAMESITE'] = "Lax"
app.config['SESSION_COOKIE_HTTPONLY'] = True

# ---------- Cloudinary config remplacé avec tes vraies clés ----------
cloudinary.config(
    cloud_name='dqxhe93is',
    api_key='658421654799452',
    api_secret='mivqoYppyk_0JblAUuM1_amc-wc',
    secure=True
)

@app.before_request
def adapt_cookie_secure():
    if request.is_secure or request.headers.get('X-Forwarded-Proto', 'http') == 'https':
        app.config['SESSION_COOKIE_SECURE'] = True
    else:
        app.config['SESSION_COOKIE_SECURE'] = False

# ---------- DB & LOGIN ----------
db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'
socketio = SocketIO(app, cors_allowed_origins="*", async_mode='eventlet')

# ---------- MOT DE PASSE GLOBAL ----------
GLOBAL_PASSWORD = "7148"
ADMIN_USERNAME = "Big J"

def check_global_password(pwd: str) -> bool:
    return pwd == GLOBAL_PASSWORD

# ---------- MODELS ----------
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=True)
    avatar = db.Column(db.String(200), default=None)  # stocke URL Cloudinary
    online = db.Column(db.Boolean, default=False)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return self.password_hash and check_password_hash(self.password_hash, password)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    text = db.Column(db.String(1000), nullable=True)
    image = db.Column(db.String(200), nullable=True)  # stocke URL Cloudinary
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)
    recipient_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)

    @property
    def username(self):
        user = User.query.get(self.user_id)
        return user.username if user else 'Anonyme'

# ---------- INIT DB ----------
with app.app_context():
    db.create_all()

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# ---------- UTILS ----------
ALLOWED_EXT = {'png', 'jpg', 'jpeg', 'gif'}

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXT

def avatar_url(filename):
    if not filename:
        return "/static/avatars/default-avatar.png"
    return filename  # ici filename est une URL Cloudinary

def chat_img_url(filename):
    if not filename:
        return None
    return filename  # ici filename est une URL Cloudinary

def is_safe_url(target):
    ref_url = urlparse(request.host_url)
    test_url = urlparse(urljoin(request.host_url, target))
    return test_url.scheme in ('http', 'https') and ref_url.netloc == test_url.netloc

def make_user_list():
    users = User.query.order_by(User.username.asc()).all()
    return [{'username': u.username, 'online': bool(u.online), 'avatar': avatar_url(u.avatar)} for u in users]

def broadcast_user_list():
    socketio.emit('user_list_update', make_user_list(), to=None)

# ---------- DECORATORS ----------
def global_password_required(f):
    @wraps(f)
    def wrap(*args, **kwargs):
        if not session.get('global_authenticated'):
            flash("Mot de passe global requis", "error")
            return redirect(url_for('login', next=request.path))
        return f(*args, **kwargs)
    return wrap

def admin_required(f):
    @wraps(f)
    def wrap(*args, **kwargs):
        if not current_user.is_authenticated or current_user.username != ADMIN_USERNAME:
            flash("Accès refusé : uniquement pour l’admin.", "error")
            return redirect(url_for('chat'))
        return f(*args, **kwargs)
    return wrap

# ---------- SOCKET.IO ----------
user_sockets = {}

@socketio.on('connect')
def on_connect(auth):
    if current_user.is_authenticated:
        current_user.online = True
        db.session.commit()
        user_sockets[current_user.id] = request.sid
        socketio.emit('status_update', {'user_id': current_user.id, 'username': current_user.username, 'status': 'online'}, to=None)
        broadcast_user_list()
    print("✅ Utilisateur connecté (sid:", request.sid, ")")

@socketio.on('disconnect')
def on_disconnect():
    sid = request.sid
    uid_to_remove = None
    for uid, s in list(user_sockets.items()):
        if s == sid:
            uid_to_remove = uid
            break
    if uid_to_remove:
        user = User.query.get(uid_to_remove)
        if user:
            user.online = False
            db.session.commit()
            socketio.emit('status_update', {'user_id': user.id, 'username': user.username, 'status': 'offline'}, to=None)
        user_sockets.pop(uid_to_remove, None)
        broadcast_user_list()
    print("❌ Utilisateur déconnecté (sid:", sid, ")")

@socketio.on('request_user_list')
def handle_request_user_list():
    emit('user_list_update', make_user_list(), to=request.sid)

def emit_new_message(payload, recipient_id=None):
    if recipient_id:
        sid_recipient = user_sockets.get(recipient_id)
        sid_sender = user_sockets.get(current_user.id)
        if sid_recipient:
            socketio.emit('new_message', payload, to=sid_recipient)
        if sid_sender:
            socketio.emit('new_message', payload, to=sid_sender)
    else:
        socketio.emit('new_message', payload, to=None)

def emit_avatar_update(user_id, username, url):
    socketio.emit('avatar_updated', {'user_id': user_id, 'username': username, 'new_avatar': avatar_url(url)}, to=None)
    broadcast_user_list()

# ---------- ROUTES ----------
@app.route('/login', methods=['GET', 'POST'])
def login():
    next_url = request.args.get('next') or url_for('chat')
    if request.method == 'POST':
        global_pwd = request.form.get('global_password', '').strip()
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()
        next_url_form = request.form.get('next') or url_for('chat')

        if not check_global_password(global_pwd):
            flash("Mot de passe global incorrect.", "error")
            return render_template('login_global.html', next=next_url)

        if username == ADMIN_USERNAME and password == GLOBAL_PASSWORD:
            user = User.query.filter_by(username=ADMIN_USERNAME).first()
            if not user:
                user = User(username=ADMIN_USERNAME)
                user.set_password(GLOBAL_PASSWORD)
                db.session.add(user)
                db.session.commit()
                broadcast_user_list()
            login_user(user)
            session['global_authenticated'] = True
            flash('Connexion admin réussie !', 'success')
            return redirect(next_url_form if is_safe_url(next_url_form) else url_for('chat'))

        user = User.query.filter_by(username=username).first()
        if user and user.check_password(password):
            login_user(user)
            session['global_authenticated'] = True
            flash('Connexion réussie !', 'success')
            return redirect(next_url_form if is_safe_url(next_url_form) else url_for('chat'))

        flash('Nom d’utilisateur ou mot de passe incorrect', 'error')

    return render_template('login_global.html', next=next_url)

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        global_pwd = request.form.get('global_password', '').strip()
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()

        if not check_global_password(global_pwd):
            flash("Mot de passe global incorrect.", "error")
            return redirect(url_for('register'))

        if username == ADMIN_USERNAME:
            flash("Ce nom est réservé.", "error")
            return redirect(url_for('register'))

        if not username or not password:
            flash('Veuillez remplir tous les champs.', 'error')
            return redirect(url_for('register'))
        if User.query.filter_by(username=username).first():
            flash('Ce nom existe déjà.', 'error')
            return redirect(url_for('register'))

        user = User(username=username)
        user.set_password(password)
        db.session.add(user)
        db.session.commit()
        broadcast_user_list()

        login_user(user)
        session['global_authenticated'] = True
        flash('Compte créé avec succès !', 'success')
        return redirect(url_for('chat'))

    return render_template('register.html')

@app.route('/', methods=['GET', 'POST'])
@global_password_required
@login_required
def chat():
    if request.method == 'POST':
        text = request.form.get('text', '').strip()
        file = request.files.get('image')
        filename = None

        if file and file.filename != '' and allowed_file(file.filename):
            try:
                result = cloudinary.uploader.upload(
                    file,
                    folder="chat_app/messages",
                    public_id=f"{int(datetime.utcnow().timestamp()*1000)}_{secure_filename(file.filename)}",
                    resource_type="image"
                )
                print("Upload chat result:", result)  # <-- debug
                filename = result['secure_url']
            except Exception as e:
                flash(f"Erreur upload image: {str(e)}", "error")
                filename = None

        recipient_id = None
        if text.startswith("@"):
            parts = text.split(" ", 1)
            username_target = parts[0][1:]
            user_target = User.query.filter_by(username=username_target).first()
            if user_target:
                recipient_id = user_target.id
                text = parts[1] if len(parts) > 1 else ""

        if text or filename:
            msg = Message(user_id=current_user.id, text=text, image=filename, recipient_id=recipient_id)
            db.session.add(msg)
            db.session.commit()
            payload = {
                'username': current_user.username,
                'text': msg.text,
                'image': chat_img_url(msg.image),
                'avatar': avatar_url(current_user.avatar),
                'timestamp': msg.timestamp.strftime("%Y-%m-%d %H:%M:%S"),
                'private': bool(recipient_id)
            }

            emit_new_message(payload, recipient_id)
            return jsonify({'success': True})

    all_msgs = Message.query.order_by(Message.timestamp.asc()).all()
    msgs = [m for m in all_msgs if (m.recipient_id is None) or (m.user_id == current_user.id) or (m.recipient_id == current_user.id)]
    users = User.query.all()
    profiles = {u.username: avatar_url(u.avatar) for u in users}
    return render_template('chat.html', messages=msgs, users=users, profiles=profiles)

@app.route('/logout')
@login_required
def logout():
    user_id = current_user.id
    username = current_user.username
    try:
        current_user.online = False
        db.session.commit()
    except Exception:
        db.session.rollback()

    user_sockets.pop(user_id, None)
    socketio.emit('status_update', {'user_id': user_id, 'username': username, 'status': 'offline'}, to=None)
    broadcast_user_list()

    logout_user()
    session.pop('global_authenticated', None)
    flash('Déconnexion réussie.', 'success')
    return redirect(url_for('login'))

# ---------- UPLOAD AVATAR ----------
@app.route('/upload_avatar', methods=['POST'])
@login_required
@global_password_required
def upload_avatar():
    file = request.files.get('avatar')
    if file and file.filename != '' and allowed_file(file.filename):
        try:
            result = cloudinary.uploader.upload(
                file,
                folder="chat_app/avatars",
                public_id=str(current_user.id),
                overwrite=True,
                resource_type="image"
            )
            print("Upload avatar result:", result)  # <-- debug
            current_user.avatar = result['secure_url']
            db.session.commit()
            flash('Avatar mis à jour.', 'success')
            emit_avatar_update(current_user.id, current_user.username, current_user.avatar)
        except Exception as e:
            flash(f'Erreur upload Cloudinary: {str(e)}', 'error')
    else:
        flash('Fichier invalide.', 'error')
    return redirect(url_for('chat'))

# ---------- PURGES ----------
@app.route('/purge_messages', methods=['POST'])
@login_required
@admin_required
def purge_messages():
    Message.query.delete()
    db.session.commit()
    flash("Tous les messages ont été purgés.", "success")
    return redirect(url_for('chat'))

@app.route('/purge_users', methods=['POST'])
@login_required
@admin_required
def purge_users():
    User.query.filter(User.username != ADMIN_USERNAME).delete()
    db.session.commit()
    flash("Tous les comptes utilisateurs ont été purgés.", "success")
    return redirect(url_for('chat'))

@app.route('/purge_images', methods=['POST'])
@login_required
@admin_required
def purge_images():
    try:
        cloudinary.api.delete_resources_by_prefix("chat_app/messages")
        cloudinary.api.delete_resources_by_prefix("chat_app/avatars")
        flash("Toutes les images du chat ont été purgées.", "success")
    except Exception as e:
        flash(f"Erreur purge images: {str(e)}", "error")
    return redirect(url_for('chat'))

# ---------- RUN ----------
if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=5000, debug=True)
