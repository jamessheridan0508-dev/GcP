import os
import sqlite3
import cloudinary
import cloudinary.uploader
from pathlib import Path

# ---------------- CONFIG ----------------
DB_PATH = "instance/users.db"
CHAT_IMG_FOLDER = "static/chat_images"
AVATAR_FOLDER = "static/avatars"

# Cloudinary config
cloudinary.config(
    cloud_name="TON_CLOUD_NAME",
    api_key="TON_API_KEY",
    api_secret="TON_API_SECRET",
    secure=True
)

ADMIN_USERNAME = "Big J"

# ---------------- UTILITAIRES ----------------
def confirm(prompt="Confirmer ? (y/n) : "):
    while True:
        choice = input(prompt).lower()
        if choice in ['y', 'n']:
            return choice == 'y'

def delete_file_local(path):
    if os.path.exists(path):
        os.remove(path)
        print(f"✅ Supprimé local : {path}")

def delete_file_cloudinary(public_id):
    try:
        cloudinary.uploader.destroy(public_id)
        print(f"✅ Supprimé Cloudinary : {public_id}")
    except Exception as e:
        print(f"⚠️ Erreur Cloudinary ({public_id}) : {e}")

def list_folder(folder):
    return [f for f in os.listdir(folder) if os.path.isfile(os.path.join(folder, f))]

# ---------------- ACTIONS ----------------
def purge_messages():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("DELETE FROM message;")
    conn.commit()
    conn.close()
    print("✅ Tous les messages ont été supprimés.")

def purge_chat_images():
    local_files = list_folder(CHAT_IMG_FOLDER)
    if local_files:
        print("Images locales du chat :")
        for f in local_files:
            print(" -", f)
        if confirm("Supprimer toutes les images locales ? (y/n) "):
            for f in local_files:
                delete_file_local(os.path.join(CHAT_IMG_FOLDER, f))
    else:
        print("Pas d’images locales dans le chat.")

    # Supprimer sur Cloudinary
    cloud_files = cloudinary.api.resources(type="upload", prefix="chat_images")["resources"]
    if cloud_files:
        print("Images sur Cloudinary :")
        for f in cloud_files:
            print(" -", f['public_id'])
        if confirm("Supprimer toutes les images Cloudinary ? (y/n) "):
            for f in cloud_files:
                delete_file_cloudinary(f['public_id'])
    else:
        print("Pas d’images sur Cloudinary.")

def purge_users():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT username, avatar FROM user WHERE username != ?", (ADMIN_USERNAME,))
    users = cursor.fetchall()

    if users:
        print("Comptes utilisateurs :")
        for u in users:
            print(" -", u[0], "(avatar:", u[1], ")")
        if confirm("Supprimer tous ces utilisateurs et leurs avatars ? (y/n) "):
            for username, avatar in users:
                cursor.execute("DELETE FROM user WHERE username=?", (username,))
                if avatar and avatar != "default-avatar.png":
                    local_path = os.path.join(AVATAR_FOLDER, avatar)
                    delete_file_local(local_path)
                    delete_file_cloudinary(f"avatars/{Path(avatar).stem}")
            conn.commit()
            print("✅ Utilisateurs supprimés.")
    else:
        print("Pas d’utilisateurs à supprimer.")
    conn.close()

def delete_specific():
    print("Options de suppression spécifique :")
    print("1: Utilisateur")
    print("2: Image chat")
    choice = input("Choisir type : ")
    if choice == "1":
        username = input("Nom de l’utilisateur à supprimer : ").strip()
        if username == ADMIN_USERNAME:
            print("❌ Impossible de supprimer l’admin !")
            return
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT avatar FROM user WHERE username=?", (username,))
        res = cursor.fetchone()
        if res:
            if confirm(f"Supprimer l’utilisateur {username} ? (y/n) "):
                cursor.execute("DELETE FROM user WHERE username=?", (username,))
                conn.commit()
                avatar = res[0]
                if avatar and avatar != "default-avatar.png":
                    delete_file_local(os.path.join(AVATAR_FOLDER, avatar))
                    delete_file_cloudinary(f"avatars/{Path(avatar).stem}")
                print(f"✅ Utilisateur {username} supprimé.")
        else:
            print("Utilisateur non trouvé.")
        conn.close()
    elif choice == "2":
        fname = input("Nom de l’image à supprimer (ex: 12345_image.png) : ").strip()
        local_path = os.path.join(CHAT_IMG_FOLDER, fname)
        cloud_id = f"chat_images/{Path(fname).stem}"
        if os.path.exists(local_path) or confirm("Supprimer sur Cloudinary ? (y/n) "):
            if confirm(f"Supprimer {fname} ? (y/n) "):
                delete_file_local(local_path)
                delete_file_cloudinary(cloud_id)
    else:
        print("Choix invalide.")

# ---------------- MENU ----------------
def menu():
    while True:
        print("\n=== MENU DE GESTION ===")
        print("1: Supprimer tous les comptes (sauf admin) + avatars")
        print("2: Supprimer toutes les images du chat")
        print("3: Supprimer tous les messages")
        print("4: Tout proposer en même temps")
        print("5: Supprimer une donnée spécifique")
        print("0: Quitter")
        choice = input("Choisir une option : ")

        if choice == "1":
            purge_users()
        elif choice == "2":
            purge_chat_images()
        elif choice == "3":
            purge_messages()
        elif choice == "4":
            purge_users()
            purge_chat_images()
            purge_messages()
        elif choice == "5":
            delete_specific()
        elif choice == "0":
            print("Au revoir !")
            break
        else:
            print("Choix invalide.")

if __name__ == "__main__":
    menu()
