from flask import Flask
from flask_httpauth import HTTPBasicAuth
import os

app = Flask(__name__)
auth = HTTPBasicAuth()

# On récupère le mot de passe depuis la variable d'environnement
GLOBAL_PASSWORD = os.getenv("GLOBAL_PASSWORD", "changeme")

users = {
    "admin": GLOBAL_PASSWORD
}

@auth.verify_password
def verify(username, password):
    if username in users and users[username] == password:
        return username

@app.route("/")
@auth.login_required
def index():
    return "Bienvenue ! Auth globale OK ✅"

if __name__ == "__main__":
    app.run(debug=True)
