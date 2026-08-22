# Expense Tracker

A simple personal expense tracker built with Flask. Log in, record your expenses, and keep tabs on your spending.

## Status

This project is under active development. Currently implemented:

- Landing page
- Registration page
- Login page

Coming soon:

- Logout
- Profile page
- Add / edit / delete expenses

## Tech Stack

- [Flask](https://flask.palletsprojects.com/) — web framework
- [pytest](https://pytest.org/) + [pytest-flask](https://pytest-flask.readthedocs.io/) — testing

## Getting Started

### Prerequisites

- Python 3.9+

### Setup

```bash
# Clone the repo
git clone <repo-url>
cd expense-tracker

# Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### Running the app

```bash
python3 app.py
```

The app will be available at [http://127.0.0.1:5001](http://127.0.0.1:5001).

### Running tests

```bash
pytest
```

## Project Structure

```
expense-tracker/
├── app.py              # Flask app and routes
├── database/           # Database access layer
├── static/              # CSS and JS assets
├── templates/           # Jinja2 HTML templates
└── requirements.txt
```
