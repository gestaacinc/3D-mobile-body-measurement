```markdown
# 📋 User Stories & Technologies

Below are the end‑to‑end user stories, following your flow chart, with the tech stack for each:

| ID    | User Story                                                                 | Tech / Library                           |
|-------|-----------------------------------------------------------------------------|------------------------------------------|
| **US‑01** | **Login / Register**<br>As a new or returning user, I want to sign up or log in so I can access my measurements. | • **Flask‑Login**<br>• **Flask‑WTF** (WTForms)<br>• **Bcrypt** (password hashing) |
| **US‑02** | **Enter Height & Weight**<br>As an authenticated user, I want to enter my height (cm) and weight (kg) so the system can calibrate measurements. | • **Flask‑WTF** forms<br>• **Jinja2** templates |
| **US‑03** | **Open Camera & Capture Front Pose**<br>As a user, I want to open my device camera and snap a front‑view photo. | • **HTML5 getUserMedia**<br>• Vanilla **JS** + **Tailwind** |
| **US‑04** | **Capture Side Pose**<br>As a user, I want to turn sideways and snap my side‑view photo. | • **HTML5 getUserMedia**<br>• Vanilla **JS** + **Tailwind** |
| **US‑05** | **Detect Landmarks**<br>System runs MediaPipe Pose on each snapshot to extract 33 3D keypoints. | • **@mediapipe/pose** (JS) |
| **US‑06** | **Compute Measurements**<br>System converts pixel distances → cm (using height calibration) for: shoulder, chest, waist, hip widths; torso & thigh lengths. | • Custom **JS** logic (distance formula)<br>• Pixel→cm ratio algorithm |
| **US‑07** | **Validate & Retake**<br>If landmark confidence or rotation is off, notify user to adjust posture & retake. | • JS visibility/z‑value checks<br>• **Alpine.js** (optional) for UI state |
| **US‑08** | **Save Measurements**<br>User confirms and saves the computed values to their profile. | • **Fetch API** (POST JSON)<br>• **Flask‑REST** endpoint<br>• **SQLAlchemy** |
| **US‑09** | **3D Visualization**<br>Display a rotating 3D human model with CSS2D labels pinned at each measurement point. | • **Three.js** + **GLTFLoader**<br>• **CSS2DRenderer** |
| **US‑10** | **View History**<br>User views a paginated list of past measurements. | • **Flask** template & pagination logic |
| **US‑11** | **Export to Excel**<br>User clicks “Download XLSX” to get all measurements in Excel format. | • **pandas** / **openpyxl**<br>• Flask `send_file` |
| **US‑12** | **Deploy & Wrap**<br>Deploy to HTTPS hosting and wrap the live URL as an APK via Appgeyser. | • **Heroku** / **DigitalOcean App Platform**<br>• **Appgeyser** WebView APK |

---

# 🗄️ `models.py` (SQLAlchemy)

Put this in `app/models.py` before running migrations:

```python
from datetime import datetime
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()


class User(db.Model):
    __tablename__ = 'users'
    id             = db.Column(db.Integer,   primary_key=True)
    email          = db.Column(db.String(120), unique=True, nullable=False)
    password_hash  = db.Column(db.String(128), nullable=False)
    created_at     = db.Column(db.DateTime,   default=datetime.utcnow)
    updated_at     = db.Column(db.DateTime,   default=datetime.utcnow,
                                onupdate=datetime.utcnow)

    # relations
    profile        = db.relationship('Profile',    back_populates='user', uselist=False)
    measurements   = db.relationship('Measurement', back_populates='user', lazy='dynamic')

    def set_password(self, pw):
        self.password_hash = generate_password_hash(pw)

    def check_password(self, pw):
        return check_password_hash(self.password_hash, pw)


class Profile(db.Model):
    __tablename__ = 'profiles'
    id          = db.Column(db.Integer, primary_key=True)
    user_id     = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    height_cm   = db.Column(db.Float,   nullable=False)
    weight_kg   = db.Column(db.Float,   nullable=False)
    created_at  = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at  = db.Column(db.DateTime, default=datetime.utcnow,
                             onupdate=datetime.utcnow)

    user        = db.relationship('User', back_populates='profile')


class Measurement(db.Model):
    __tablename__       = 'measurements'
    id                  = db.Column(db.Integer,   primary_key=True)
    user_id             = db.Column(db.Integer,   db.ForeignKey('users.id'), nullable=False)
    shoulder_cm         = db.Column(db.Float,     nullable=False)
    chest_cm            = db.Column(db.Float,     nullable=False)
    waist_cm            = db.Column(db.Float,     nullable=False)
    hip_cm              = db.Column(db.Float,     nullable=False)
    torso_length_cm     = db.Column(db.Float,     nullable=False)
    thigh_length_cm     = db.Column(db.Float,     nullable=False)
    timestamp           = db.Column(db.DateTime,  default=datetime.utcnow)

    user                = db.relationship('User', back_populates='measurements')
```

---

# 🏗️ Project Structure

A clean, maintainable Flask layout:

```
mybodyapp/
├── app/
│   ├── __init__.py          # app factory, blueprints registration
│   ├── models.py            # SQLAlchemy models (above)
│   ├── auth/                # authentication blueprint
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   └── routes.py
│   ├── public/              # main user flows blueprint
│   │   ├── __init__.py
│   │   ├── forms.py
│   │   └── routes.py
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css
│   │   ├── js/
│   │   │   ├── pose.js
│   │   │   └── capture.js
│   │   └── models/
│   │       └── human.glb
│   └── templates/
│       ├── base.html
│       ├── auth/
│       │   ├── login.html
│       │   └── register.html
│       └── public/
│           ├── measure.html
│           ├── profile.html
│           ├── history.html
│           └── no_measurements.html
├── migrations/              # Flask‑Migrate / Alembic
├── tests/                   # pytest unit & integration tests
├── config.py                # configuration classes
├── requirements.txt         # pinned dependencies
└── run.py                   # entry point (Flask CLI)
```

- **`app/__init__.py`**: create_app factory, init extensions  
- **`auth` & `public`**: separated blueprints for clear concerns  
- **`static/js`**: your MediaPipe + Three.js scripts  
- **`migrations/`**: auto‑generated by Flask‑Migrate  
- **`tests/`**: start with basic smoke tests, add coverage over time  

With this roadmap, schema and structure, you’re ready to scaffold, migrate, and build each user story in turn. Good luck! 🚀
