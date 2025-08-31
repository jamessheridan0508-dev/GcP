import sqlite3

# Connexion à la base (créera le fichier si inexistant)
conn = sqlite3.connect('users.db')
c = conn.cursor()

# Création de la table utilisateurs
c.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
)
''')

# Sauvegarde et fermeture
conn.commit()
conn.close()

print("Base de données initialisée avec succès !")
