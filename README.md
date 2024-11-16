import requests
from flask import Flask, render_template_string, request, session, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
from collections import Counter
from datetime import datetime

# Flask app setup
app = Flask(_name_)
app.secret_key = 'your_secret_key_here'  # Required for sessions
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///career_advisor.db'
db = SQLAlchemy(app)

# Database Models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password = db.Column(db.String(120), nullable=False)
    education = db.Column(db.String(200))
    skills = db.Column(db.String(500))
    experience = db.Column(db.Integer)
    interests = db.Column(db.String(500))  # Interests field

# Fetch job data from an open-source API (e.g., Indeed)
def get_job_data(query, location="remote"):
    # Replace with your Indeed API key and Publisher ID
    indeed_url = "https://api.indeed.com/ads/apisearch"
    api_key = 'YOUR_INDEED_API_KEY'
    publisher_id = 'YOUR_INDEED_PUBLISHER_ID'

    params = {
        'q': query,
        'l': location,
        'format': 'json',
        'v': '2',
        'userip': '1.2.3.4',
        'useragent': 'Mozilla/5.0',
        'api_key': api_key,
        'publisher': publisher_id
    }
    
    response = requests.get(indeed_url, params=params)
    if response.status_code != 200:
        return []
    
    return response.json()['results']

# Suggest the required skills based on job descriptions
def extract_skills_from_description(description):
    # Example: A list of common skills (could be expanded with an external source or ML model)
    common_skills = ["Python", "Machine Learning", "Data Analysis", "JavaScript", "React", "Node.js", "AWS", "Azure", "Cybersecurity", "SQL"]
    description = description.lower()
    skills_found = [skill for skill in common_skills if skill.lower() in description]
    return skills_found

# Suggest learning platforms based on the required skills
def suggest_learning_platforms(skills):
    platform_dict = {
        "Python": ["Coursera", "Udemy", "EdX", "DataCamp"],
        "Machine Learning": ["Coursera", "Udemy", "EdX", "Kaggle"],
        "Data Analysis": ["Coursera", "Udemy", "DataCamp", "LinkedIn Learning"],
        "JavaScript": ["Udemy", "Coursera", "freeCodeCamp"],
        "React": ["Udemy", "Codecademy", "freeCodeCamp"],
        "AWS": ["AWS Training", "Udemy", "LinkedIn Learning"],
        "Azure": ["Microsoft Learn", "Udemy", "Pluralsight"],
        "Cybersecurity": ["Udemy", "Coursera", "LinkedIn Learning", "Cybrary"],
        "SQL": ["Udemy", "Coursera", "LinkedIn Learning", "SQLZoo"]
    }
    
    platforms = set()
    for skill in skills:
        if skill in platform_dict:
            platforms.update(platform_dict[skill])
    
    return list(platforms)

# Home route
@app.route('/')
def home():
    return render_template_string('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>AI Career Advisor</title>
        <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
    </head>
    <body class="bg-gray-100">
        <nav class="bg-blue-600 text-white p-4">
            <div class="container mx-auto flex justify-between items-center">
                <h1 class="text-2xl font-bold">AI Career Advisor</h1>
                <div>
                    <a href="/login" class="px-4 py-2 bg-blue-500 rounded hover:bg-blue-400">Login</a>
                    <a href="/register" class="px-4 py-2 bg-green-500 rounded hover:bg-green-400">Register</a>
                </div>
            </div>
        </nav>

        <div class="container mx-auto mt-10 p-4">
            <h2 class="text-3xl font-bold mb-4">Find Your Dream Job</h2>
            <p class="text-gray-600 mb-4">Analyze your skills, education, and experience to match you with the perfect job.</p>
            <a href="/trending-jobs" class="px-4 py-2 bg-orange-500 text-white rounded hover:bg-orange-400">See Trending Jobs</a>
        </div>
    </body>
    </html>
    ''')

# Login Route
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        user = User.query.filter_by(username=username).first()
        
        if user and user.password == password:
            session['user_id'] = user.id
            return redirect(url_for('dashboard'))
        else:
            return "Invalid credentials", 401
    
    return render_template_string('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Login - AI Career Advisor</title>
        <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
    </head>
    <body class="bg-gray-100">
        <nav class="bg-blue-600 text-white p-4">
            <div class="container mx-auto flex justify-between items-center">
                <h1 class="text-2xl font-bold">AI Career Advisor</h1>
                <a href="/register" class="px-4 py-2 bg-green-500 rounded hover:bg-green-400">Register</a>
            </div>
        </nav>

        <div class="container mx-auto mt-10 p-4">
            <h2 class="text-3xl font-bold mb-4">Login</h2>
            <form method="POST">
                <input type="text" name="username" placeholder="Username" class="w-full p-2 mb-4" required>
                <input type="password" name="password" placeholder="Password" class="w-full p-2 mb-4" required>
                <button type="submit" class="px-4 py-2 bg-blue-500 text-white rounded">Login</button>
            </form>
        </div>
    </body>
    </html>
    ''')

# Register Route
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        education = request.form['education']
        skills = request.form['skills']
        experience = int(request.form['experience'])
        interests = request.form['interests']
        
        # Save new user to database
        new_user = User(username=username, password=password, education=education,
                        skills=skills, experience=experience, interests=interests)
        db.session.add(new_user)
        db.session.commit()
        
        return redirect(url_for('login'))
    
    return render_template_string('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Register - AI Career Advisor</title>
        <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
    </head>
    <body class="bg-gray-100">
        <nav class="bg-blue-600 text-white p-4">
            <div class="container mx-auto flex justify-between items-center">
                <h1 class="text-2xl font-bold">AI Career Advisor</h1>
                <a href="/login" class="px-4 py-2 bg-green-500 rounded hover:bg-green-400">Login</a>
            </div>
        </nav>

        <div class="container mx-auto mt-10 p-4">
            <h2 class="text-3xl font-bold mb-4">Register</h2>
            <form method="POST">
                <input type="text" name="username" placeholder="Username" class="w-full p-2 mb-4" required>
                <input type="password" name="password" placeholder="Password" class="w-full p-2 mb-4" required>
                <input type="text" name="education" placeholder="Education" class="w-full p-2 mb-4" required>
                <input type="text" name="skills" placeholder="Skills" class="w-full p-2 mb-4" required>
                <input type="number" name="experience" placeholder="Experience (in years)" class="w-full p-2 mb-4" required>
                <input type="text" name="interests" placeholder="Interests" class="w-full p-2 mb-4" required>
                <button type="submit" class="px-4 py-2 bg-green-500 text-white rounded">Register</button>
            </form>
        </div>
    </body>
    </html>
    ''')

# Dashboard Route
@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        return redirect(url_for('login'))

    user = User.query.get(session['user_id'])
    
    if user:
        job_data = get_job_data(user.skills, location="remote")
        return render_template_string('''
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Dashboard - AI Career Advisor</title>
            <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
        </head>
        <body class="bg-gray-100">
            <nav class="bg-blue-600 text-white p-4">
                <div class="container mx-auto flex justify-between items-center">
                    <h1 class="text-2xl font-bold">AI Career Advisor</h1>
                    <a href="/logout" class="px-4 py-2 bg-red-500 rounded hover:bg-red-400">Logout</a>
                </div>
            </nav>

            <div class="container mx-auto mt-10 p-4">
                <h2 class="text-3xl font-bold mb-4">Welcome, {{ user.username }}!</h2>
                <p class="text-lg mb-4">Here are some job opportunities that match your skills:</p>

                <ul>
                    {% for job in job_data %}
                        <li class="mb-4 p-4 bg-white shadow-lg rounded">
                            <h3 class="text-xl font-semibold">{{ job['jobtitle'] }}</h3>
                            <p>{{ job['company'] }}</p>
                            <p>{{ job['snippet'] }}</p>
                        </li>
                    {% endfor %}
                </ul>
            </div>
        </body>
        </html>
        ''', user=user, job_data=job_data)
    return redirect(url_for('login'))

# Logout Route
@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect(url_for('login'))

if _name_ == '_main_':
    with app.app_context():
     db.create_all()  # Creates the database tables
    app.run(debug=True)
