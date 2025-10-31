rom flask import Flask, render_template, request, redirect, url_for, session, flash
import pandas as pd
import pickle
from flask_caching import Cache
from flask_sqlalchemy import SQLAlchemy

# ---------------------------
# Flask App Setup
# ---------------------------
app = Flask(__name__)
app.secret_key = "supersecretkey"
app.config['CACHE_TYPE'] = 'SimpleCache'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

cache = Cache(app)
db = SQLAlchemy(app)

# ---------------------------
# Load ML Model
# ---------------------------
with open("model.pkl", "rb") as f:
    model = pickle.load(f)

# ---------------------------
# Database Model
# ---------------------------
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), unique=True, nullable=False)
    password = db.Column(db.String(100), nullable=False)

    def __repr__(self):
        return f"<User {self.username}>"

# ---------------------------
# Helper Functions
# ---------------------------
def get_cleaned_data(form_data):
    try:
        gestation = float(form_data['gestation'])
        parity = int(form_data['parity'])
        age = float(form_data['age'])
        height = float(form_data['height'])
        weight = float(form_data['weight'])
        smoke = float(form_data['smoke'])
        return {
            "gestation": [gestation],
            "parity": [parity],
            "age": [age],
            "height": [height],
            "weight": [weight],
            "smoke": [smoke]
        }
    except (KeyError, ValueError):
        return None

def convert_oz_to_kg(oz):
    return round(oz * 0.0283495, 2)

# ---------------------------
# Authentication Routes
# ---------------------------
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form.get("username").strip()
        password = request.form.get("password").strip()

        existing_user = User.query.filter_by(username=username).first()
        if existing_user:
            flash("❌ Username already exists!", "error")
            return redirect(url_for("register"))

        new_user = User(username=username, password=password)
        db.session.add(new_user)
        db.session.commit()

        flash("✅ Registration successful! Please login.", "success")
        return redirect(url_for("login"))

    return render_template("register.html")


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form.get("username").strip()
        password = request.form.get("password").strip()

        user = User.query.filter_by(username=username).first()

        if not user:
            flash("❌ Username does not exist!", "error")
            return redirect(url_for("login"))

        if user.password != password:
            flash("❌ Wrong password!", "error")
            return redirect(url_for("login"))

        session["username"] = username
        flash(f"✅ Welcome {username}!", "success")
        return redirect(url_for("home"))

    return render_template("login.html")


@app.route("/logout")
def logout():
    session.pop("username", None)
    flash("✅ Logged out successfully.", "success")
    return redirect(url_for("login"))

# ---------------------------
# Home / Prediction Page
# ---------------------------
@app.route("/")
def home():
    if "username" not in session:
        return redirect(url_for("login"))
    return render_template("index.html")

@app.route("/predict", methods=["POST"])
def predict_weight():
    if "username" not in session:
        return redirect(url_for("login"))

    form_data = request.form
    cleaned_data = get_cleaned_data(form_data)
    if not cleaned_data:
        return "Invalid input data", 400

    df = pd.DataFrame(cleaned_data)
    prediction_oz = round(float(model.predict(df)[0]), 2)  # <- fixed
    prediction_kg = convert_oz_to_kg(prediction_oz)

    if prediction_kg < 2.5:
        health_status = "Unhealthy (Low Birth Weight)"
    elif 2.5 <= prediction_kg <= 4.0:
        health_status = "Healthy"
    else:
        health_status = "Overweight (High Birth Weight)"

    return render_template(
        "index.html",
        prediction_oz=prediction_oz,
        prediction_kg=prediction_kg,
        health_status=health_status
    )


# ---------------------------
# About Us & Contact Us
# ---------------------------
@app.route("/about")
def about():
    return render_template("about.html")

@app.route("/contact")
def contact():
    return render_template("contact.html")

# ---------------------------
# Run App
# ---------------------------
if __name__ == "__main__":
    with app.app_context():
        db.create_all()  # Ensure users table exists
    app.run(debug=True)
