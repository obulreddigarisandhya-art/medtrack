from flask import Flask, render_template, request, redirect, session
import sqlite3
import uuid
import logging
import os

app = Flask(__name__)
app.secret_key = "secret123"  # change in production

# ----------------------------
# Logging Setup
# ----------------------------
logging.basicConfig(
    filename='app.log',
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ----------------------------
# Database Setup (SQLite)
# ----------------------------
DB_NAME = "medtrack.db"

def get_db():
    return sqlite3.connect(DB_NAME)

def init_db():
    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        email TEXT PRIMARY KEY,
        name TEXT,
        password TEXT,
        role TEXT,
        login_count INTEGER
    )
    """)

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS appointments (
        appointment_id TEXT PRIMARY KEY,
        patient_email TEXT,
        doctor_email TEXT,
        date TEXT,
        time TEXT,
        status TEXT,
        diagnosis TEXT
    )
    """)

    conn.commit()
    conn.close()

init_db()

# ----------------------------
# Home
# ----------------------------
@app.route("/")
def home():
    return render_template("index.html")

# ----------------------------
# Register
# ----------------------------
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        conn = get_db()
        cursor = conn.cursor()

        cursor.execute("""
        INSERT INTO users (email, name, password, role, login_count)
        VALUES (?, ?, ?, ?, ?)
        """, (
            request.form["email"],
            request.form["name"],
            request.form["password"],
            request.form["role"],
            0
        ))

        conn.commit()
        conn.close()

        logging.info("New user registered")
        return redirect("/login")

    return render_template("register.html")

# ----------------------------
# Login
# ----------------------------
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        conn = get_db()
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM users WHERE email=?", (request.form["email"],))
        user = cursor.fetchone()

        if user and user[2] == request.form["password"]:
            session["user"] = user[0]
            session["role"] = user[3]

            cursor.execute("""
            UPDATE users SET login_count = login_count + 1 WHERE email=?
            """, (user[0],))

            conn.commit()
            conn.close()

            logging.info(f"{user[0]} logged in")

            if session["role"] == "doctor":
                return redirect("/doctor_dashboard")
            else:
                return redirect("/patient_dashboard")

        return "Invalid Credentials"

    return render_template("login.html")

# ----------------------------
# Logout
# ----------------------------
@app.route("/logout")
def logout():
    session.clear()
    return redirect("/login")

# ----------------------------
# Dashboards
# ----------------------------
@app.route("/doctor_dashboard")
def doctor_dashboard():
    if "user" not in session:
        return redirect("/login")
    return render_template("doctor_dashboard.html")

@app.route("/patient_dashboard")
def patient_dashboard():
    if "user" not in session:
        return redirect("/login")
    return render_template("patient_dashboard.html")

# ----------------------------
# Book Appointment
# ----------------------------
@app.route("/book_appointment", methods=["GET", "POST"])
def book_appointment():
    if "user" not in session:
        return redirect("/login")

    if request.method == "POST":
        conn = get_db()
        cursor = conn.cursor()

        appointment_id = str(uuid.uuid4())

        cursor.execute("""
        INSERT INTO appointments 
        (appointment_id, patient_email, doctor_email, date, time, status)
        VALUES (?, ?, ?, ?, ?, ?)
        """, (
            appointment_id,
            session["user"],
            request.form["doctor_email"],
            request.form["date"],
            request.form["time"],
            "Scheduled"
        ))

        conn.commit()
        conn.close()

        logging.info("Appointment booked")
        return redirect("/view_appointment_patient")

    return render_template("book_appointment.html")

# ----------------------------
# View Doctor Appointments
# ----------------------------
@app.route("/view_appointment_doctor")
def view_appointment_doctor():
    if "user" not in session:
        return redirect("/login")

    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
    SELECT * FROM appointments WHERE doctor_email=?
    """, (session["user"],))

    appointments = cursor.fetchall()
    conn.close()

    return render_template("view_appointment_doctor.html", appointments=appointments)

# ----------------------------
# View Patient Appointments
# ----------------------------
@app.route("/view_appointment_patient")
def view_appointment_patient():
    if "user" not in session:
        return redirect("/login")

    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
    SELECT * FROM appointments WHERE patient_email=?
    """, (session["user"],))

    appointments = cursor.fetchall()
    conn.close()

    return render_template("view_appointment_patient.html", appointments=appointments)

# ----------------------------
# Submit Diagnosis
# ----------------------------
@app.route("/submit_diagnosis", methods=["GET", "POST"])
def submit_diagnosis():
    if "user" not in session:
        return redirect("/login")

    if request.method == "POST":
        conn = get_db()
        cursor = conn.cursor()

        cursor.execute("""
        UPDATE appointments
        SET diagnosis=?, status=?
        WHERE appointment_id=?
        """, (
            request.form["diagnosis"],
            "Completed",
            request.form["appointment_id"]
        ))

        conn.commit()
        conn.close()

        logging.info("Diagnosis submitted")
        return redirect("/view_appointment_doctor")

    appointment_id = request.args.get("appointment_id")
    return render_template("submit_diagnosis.html", appointment_id=appointment_id)

# ----------------------------
# Search Appointment
# ----------------------------
@app.route("/search", methods=["POST"])
def search():
    conn = get_db()
    cursor = conn.cursor()

    cursor.execute("""
    SELECT * FROM appointments WHERE date=?
    """, (request.form["date"],))

    results = cursor.fetchall()
    conn.close()

    return render_template("search_results.html", appointments=results)

# ----------------------------
# Health Check
# ----------------------------
@app.route("/health")
def health():
    return {"status": "Application Running"}, 200

# ----------------------------
# Run App
# ----------------------------
if __name__ == "__main__":
    app.run(debug=True)
