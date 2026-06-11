---
name: flask
description: Build server-rendered, full-stack Flask web applications — the application-factory pattern with Blueprints, class-based config, SQLAlchemy models and Flask-Migrate migrations, Jinja2 templates, WTForms/Flask-WTF forms with CSRF, Flask-Login session authentication with werkzeug password hashing, error handlers, pytest test-client setup, and Gunicorn deployment. For Flask backends that return JSON to clients rather than rendering HTML, use the `flask-apis` skill instead. Use when the user invokes `/flask`, asks to "build a Flask app", "add a Flask route/blueprint", "set up the application factory", "add a login form", "render a template", "add a WTForms form", or similar server-rendered Flask work.
---

# Flask Full-Stack Web Development

Comprehensive best practices for building Flask applications with templates, forms, databases, and authentication.

## Project Structure

Always use the **application factory pattern** with Blueprints. Never use the singleton `app = Flask(__name__)` anti-pattern.

```
project/
├── app/
│   ├── __init__.py          # create_app() factory
│   ├── extensions.py         # extension initialization (db, migrate, login, etc.)
│   ├── config.py             # Config classes (Base, Development, Production, Testing)
│   ├── templates/            # Jinja2 templates
│   ├── static/               # CSS, JS, images
│   └── modules/
│       ├── __init__.py
│       ├── admin/
│       │   ├── __init__.py   # admin = Blueprint('admin', __name__)
│       │   ├── routes.py
│       │   ├── forms.py
│       │   ├── models.py
│       │   └── errors.py
│       └── public/
│           ├── __init__.py
│           ├── routes.py
│           └── forms.py
├── migrations/               # Flask-Migrate Alembic migrations
├── tests/
│   ├── conftest.py           # fixtures (app, db, client, etc.)
│   └── test_*.py
├── requirements.txt
├── gunicorn.conf.py
└── run.py                    # dev entry point only
```

### Factory Pattern (`app/__init__.py`)

```python
def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # Initialize extensions (see extensions.py)
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)

    # Register blueprints
    from app.modules.admin import admin_bp
    from app.modules.public import public_bp
    app.register_blueprint(admin_bp, url_prefix='/admin')
    app.register_blueprint(public_bp)

    return app
```

### Extension Initialization (`app/extensions.py`)

Initialize extensions as **bare objects** (not bound to an app). Bind them via `init_app()` in the factory.

```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()
```

## Routing and Blueprints

- **One Blueprint per domain module**, not per route.
- Keep route handlers thin — delegate business logic to service functions or model methods.
- Use `@blueprint.errorhandler` for blueprint-specific error handling.
- Use `url_for()` for all internal URLs, never hard-code paths.

```python
@blueprint.route('/items/<int:item_id>', methods=['GET'])
def item_detail(item_id):
    item = get_or_404(Item, item_id)
    return render_template('items/detail.html', item=item)
```

## Configuration

Use class-based configuration with environment-variable overrides. Never commit secrets.

```python
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY')
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class Development(Config):
    DEBUG = True

class Production(Config):
    DEBUG = False

class Testing(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'
```

## Database (SQLAlchemy)

### Model conventions
- Use `db.Model` from the shared extension object.
- Define `__tablename__` explicitly.
- Use `db.Column` types with appropriate constraints (`nullable=False`, `unique=True`).
- Override `__repr__` for debugging.
- Use `relationship(..., backref='...')` or `back_populates` for associations.

```python
class User(db.Model):
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)

    posts = db.relationship('Post', backref='author', lazy='dynamic')

    def __repr__(self):
        return f'<User {self.username}>'
```

### Migrations
- Use **Flask-Migrate** (Alembic wrapper). Never alter models and expect the DB to follow.
- After every model change: `flask db migrate -m "description"` then `flask db upgrade`.
- Commit migration files to version control.

## Templates (Jinja2)

### Template hierarchy
```
templates/
├── base.html              # Master layout
├── errors/
│   ├── 404.html
│   └── 500.html
└── items/
    ├── list.html
    └── detail.html
```

### Base template pattern
```html
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}Site{% endblock %}</title>
    {% block head %}{% endblock %}
</head>
<body>
    <nav>{% block navigation %}{% endblock %}</nav>
    <main>
        {% with messages = get_flashed_messages(with_categories=true) %}
        {% for category, message in messages %}
            <div class="alert alert-{{ category }}">{{ message }}</div>
        {% endfor %}
        {% endwith %}
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

### Template best practices
- Extend `base.html` for all page templates.
- Use `{% block %}` for extensible sections.
- Keep logic out of templates — no complex conditionals or loops beyond simple iteration.
- Use **macros** for repeated UI components (form fields, cards, pagination).
- Always use `get_flashed_messages()` in the base template for flash messaging.
- Use `selectattr` and `rejectattr` filters for filtering in templates rather than doing it in views when appropriate.

## Forms (WTForms / Flask-WTF)

```python
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField
from wtforms.validators import DataRequired, Email, Length

class LoginForm(FlaskForm):
    email = StringField('Email', validators=[DataRequired(), Email()])
    password = StringField('Password', validators=[DataRequired(), Length(min=8)])
    submit = SubmitField('Sign In')
```

### Form best practices
- **Always use CSRF protection** — Flask-WTF enables it by default. Ensure `SECRET_KEY` is set.
- Validate on `POST` only: `if form.validate_on_submit():`
- Use field-level validators from `wtforms.validators`.
- Render forms with macros for consistent styling.
- Return form errors in templates by iterating `form.errors`.

## Authentication (Flask-Login)

### User loader
```python
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

### Login/logout
```python
@blueprint.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('public.index'))
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user and check_password_hash(user.password_hash, form.password.data):
            login_user(user)
            flash('Logged in successfully.', 'success')
            return redirect(url_for('public.index'))
        flash('Invalid credentials.', 'danger')
    return render_template('auth/login.html', form=form)
```

### Password hashing
- **Always use `werkzeug.security`** for password hashing.
- Hash: `generate_password_hash(password)`
- Verify: `check_password_hash(hash, password)`
- Never store plain-text passwords.

## Error Handling

### Global error handlers
```python
@app.errorhandler(404)
def not_found(e):
    return render_template('errors/404.html'), 404

@app.errorhandler(500)
def internal_error(e):
    db.session.rollback()
    return render_template('errors/500.html'), 500
```

### Custom exceptions
```python
from werkzeug.exceptions import HTTPException

class NotFound(HTTPException):
    code = 404

# In a view:
raise NotFound(f'Item {item_id} not found')
```

## Testing

### Test fixtures (`tests/conftest.py`)
```python
import pytest
from app import create_app
from app.extensions import db

@pytest.fixture
def app():
    app = create_app('config.Testing')
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def runner(app):
    return app.test_cli_runner()
```

### Test patterns
```python
def test_login_success(client, app):
    response = client.post('/login', data={
        'email': 'test@example.com',
        'password': 'password123'
    }, follow_redirects=True)
    assert response.status_code == 200
    assert b'Logged in' in response.data
```

### Testing checklist
- Test routes return correct status codes.
- Test redirects work as expected.
- Test form validation with invalid data.
- Test template rendering includes expected content.
- Use `follow_redirects=True` to assert post-redirect state.
- Test with the test client, not manual requests.

## Performance and Deployment

### Performance
- Use **Flask-Caching** for expensive queries or rendered templates.
- Enable compression with `Flask-Compress`.
- Use connection pooling (SQLAlchemy default pool is usually fine).
- Lazy-load relationships; use `joinedload()` when you know you need them.

### Deployment
- **Never use Flask's development server in production.** Use Gunicorn or uWSGI.
- Example Gunicorn command: `gunicorn -w 4 -b 0.0.0.0:8000 app:app`
- Set `SECRET_KEY` via environment variable.
- Set `FLASK_ENV=production` or use `FLASK_CONFIG=Production`.
- Use a reverse proxy (nginx) for static files and SSL termination.

## Common Pitfalls

1. **Circular imports**: Import blueprints inside `create_app()`, not at module level.
2. **Testing without `app.app_context()`**: Many extensions require an app context. Use `with app.app_context():` in tests.
3. **Forgetting `db.session.rollback()` in error handlers**: Leaves the session in a broken state.
4. **N+1 queries**: Use `joinedload()` or `selectinload()` for eager loading.
5. **Storing secrets in config files**: Always use environment variables for `SECRET_KEY`, database URIs, and API keys.
6. **Not using Blueprints**: Keeping all routes in one file becomes unmaintainable quickly.
7. **Missing CSRF on forms**: Flask-WTF handles this automatically, but only if `SECRET_KEY` is configured.
