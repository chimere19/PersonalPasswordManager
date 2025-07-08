# PersonalPasswordManager
# 🔐 Personal Password Manager

A beginner-friendly encrypted password manager built with Python. This project helps users securely store and manage their passwords using AES encryption. It includes a master password system and an easy-to-use graphical interface built with Tkinter.

---

## ✨ Features

* Master password login system
* AES-encrypted password storage using `cryptography`
* GUI built with `tkinter`
* Add, view, search, update, delete entries
* Clipboard password copy (auto-clears after 15 seconds)
* Auto-lock timeout after 5 minutes of inactivity
* Export/import passwords to and from a backup file
* Password strength checker (optional)
* Optional HaveIBeenPwned integration for breach check

---

## 📦 Dependencies

Install with pip:

```bash
pip install cryptography pyperclip
```

If using breach checker:

```bash
pip install requests
```

---

## 🚀 How to Run

1. Clone this repository or download the files
2. Run `password_gui.py`
3. On first run, you'll be prompted to create a master password
4. After login, use the GUI to manage your entries

---

## 📁 Project Structure

```
├── password_gui.py           # The graphical user interface
├── main_internship.py        # Core logic and encryption functions
├── breach_checker.py         # Optional: check if password is breached
├── key.py                    # Generates and loads encryption key
├── master_password.py        # Handles master password hashing
├── passwords.txt             # Encrypted password entries (auto-created)
├── key.key                   # Encryption key (auto-created)
└── README.md                 # This file
```

---

## 🔐 Security Notes

* All passwords are encrypted before saving.
* Master password is stored as a SHA-256 hash.
* Auto-lock clears clipboard and exits the app after 5 minutes.

---

## 📌 Author

**Chimeremeze Anumiheoma**
Student, Louisiana Tech University
[LinkedIn]www.linkedin.com/in/chimeremeze-anumiheoma-a45474205
