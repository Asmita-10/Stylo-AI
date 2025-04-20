# Stylo-AI
# StyloAI - Fashion Style Recommendation App

StyloAI is a fashion style recommendation application built with Flask backend and vanilla JavaScript frontend. It provides fashion recommendations through four main features: text input for style recommendations, image upload for style analysis, voice input for spoken style preferences, and digital wardrobe management.

## Project Structure

```
styloai/
├── app.py
├── main.py
├── models.py
├── static/
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── script.js
│   └── images/
│       └── (place your images here)
├── templates/
│   ├── index.html
│   ├── style-suggestions.html
│   ├── wardrobe.html
│   ├── about.html
│   └── contact.html
└── requirements.txt
```

## Requirements

```
flask
flask-cors
flask-login
flask-sqlalchemy
numpy
opencv-python
pillow
psycopg2-binary
gunicorn
email-validator
```

## File Contents

### main.py
```python
from app import app

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### app.py
```python
from flask import Flask, request, jsonify, render_template
from flask_cors import CORS
import numpy as np
import base64
import io
from PIL import Image
import os
import logging
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy.orm import DeclarativeBase
from flask_login import LoginManager, UserMixin

# Set up logging
logging.basicConfig(level=logging.DEBUG)

# Create database structure
class Base(DeclarativeBase):
    pass

db = SQLAlchemy(model_class=Base)

# Initialize Flask app
app = Flask(__name__)
CORS(app)
app.secret_key = os.environ.get("SESSION_SECRET", "styloai-secret-key")

# Configure database
app.config["SQLALCHEMY_DATABASE_URI"] = os.environ.get("DATABASE_URL", "sqlite:///styloai.db")
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
db.init_app(app)

# Initialize login manager
login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = 'login'

# Import models after app is initialized
with app.app_context():
    from models import User, WardrobeItem, OutfitCollection, OutfitItem
    db.create_all()

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# Mock speech function since pyttsx3 might have issues
def speak(text):
    logging.info(f"Text-to-speech would say: {text}")
    # We'll just log the text that would be spoken

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/style-suggestions')
def style_suggestions():
    return render_template('style-suggestions.html')

@app.route('/wardrobe')
def wardrobe():
    return render_template('wardrobe.html')

@app.route('/about')
def about():
    return render_template('about.html')

@app.route('/contact')
def contact():
    return render_template('contact.html')

@app.route("/api/styloai", methods=["POST"])
def get_suggestions():
    try:
        data = request.json
        input_text = data.get("input", "")
        
        if not input_text:
            return jsonify({"error": "No input text provided"}), 400

        # Mock TensorFlow output for demo
        suggestion = f"Based on '{input_text}', we recommend a modern casual outfit with denim and sneakers."
        speak(suggestion)

        return jsonify({"response": suggestion})
    except Exception as e:
        logging.error(f"Error in get_suggestions: {e}")
        return jsonify({"error": "Failed to process your request"}), 500

@app.route("/api/image-upload", methods=["POST"])
def analyze_image():
    try:
        if 'image' not in request.files:
            return jsonify({"error": "No image provided"}), 400
            
        file = request.files["image"]
        if file.filename == '':
            return jsonify({"error": "No image selected"}), 400
            
        # Instead of using TensorFlow model, we'll use a simple color analysis for demo
        image = Image.open(file.stream).convert("RGB")
        
        # Simple algorithm to determine style based on image brightness
        image_np = np.array(image)
        brightness = np.mean(image_np)
        
        # Simple classification based on brightness (for demo purposes)
        if brightness < 85:  # Darker colors often used in formal wear
            predicted_label = "Formal"
            confidence = 0.75
        elif brightness > 170:  # Brighter colors often used in sporty wear
            predicted_label = "Sporty"
            confidence = 0.82
        else:  # Medium brightness often casual
            predicted_label = "Casual"
            confidence = 0.88
        
        # Generate appropriate recommendations based on style
        recommendations = {
            "Casual": "Try pairing jeans with a t-shirt and sneakers for a relaxed look.",
            "Formal": "A suit or blazer with dress shoes would elevate this outfit.",
            "Sporty": "Athletic wear like joggers and performance tops would complement this style."
        }

        return jsonify({
            "style": predicted_label, 
            "confidence": float(confidence), 
            "recommendation": recommendations[predicted_label]
        })
    except Exception as e:
        logging.error(f"Error in analyze_image: {e}")
        return jsonify({"error": "Failed to analyze image"}), 500

@app.route("/api/voice-upload", methods=["POST"])
def voice_to_text():
    try:
        if 'audio' not in request.files:
            return jsonify({"error": "No audio file provided"}), 400
            
        audio_file = request.files['audio']
        if audio_file.filename == '':
            return jsonify({"error": "No audio selected"}), 400
            
        # For now, we'll simulate speech recognition
        # In a real application, we'd process the audio file with a speech-to-text service
        
        # Mock transcription text
        sample_texts = [
            "I need a casual outfit for the weekend",
            "Looking for a formal outfit for a business meeting",
            "What would look good for a workout session",
            "I'm going to a party tonight, need something stylish"
        ]
        import random
        text = random.choice(sample_texts)
        
        # Process the recognized text for style suggestions
        suggestion = f"Based on your voice input, we recommend a stylish outfit with {text.lower()} elements."
        
        return jsonify({"transcript": text, "suggestion": suggestion})
    except Exception as e:
        logging.error(f"Error in voice_to_text: {e}")
        return jsonify({"error": "Failed to process audio"}), 500

@app.route("/api/save-wardrobe", methods=["POST"])
def save_wardrobe():
    try:
        data = request.json
        
        if not data or "user_id" not in data or "wardrobe" not in data:
            return jsonify({"error": "Invalid request data"}), 400
            
        user_id = data["user_id"]
        wardrobe_items = data["wardrobe"]
        
        # Find the user
        user = User.query.filter_by(id=user_id).first()
        
        if not user:
            # For demo purposes, create a mock user if not found
            user = User(id=user_id, username=f"user{user_id}", email=f"user{user_id}@example.com")
            db.session.add(user)
            db.session.commit()
        
        # Delete existing wardrobe items for this user
        WardrobeItem.query.filter_by(user_id=user_id).delete()
        
        # Add new wardrobe items
        from datetime import datetime
        for item in wardrobe_items:
            wardrobe_item = WardrobeItem(
                user_id=user_id,
                category=item.get("category", "Other"),
                style=item.get("style", "Casual"),
                color=item.get("color", ""),
                description=item.get("description", ""),
                date_added=datetime.now()
            )
            db.session.add(wardrobe_item)
        
        db.session.commit()
        return jsonify({"message": "Wardrobe saved successfully."})
    except Exception as e:
        logging.error(f"Error in save_wardrobe: {e}")
        db.session.rollback()
        return jsonify({"error": "Failed to save wardrobe"}), 500

@app.route("/api/get-wardrobe", methods=["GET"])
def get_wardrobe():
    try:
        user_id = request.args.get("user_id")
        
        if not user_id:
            return jsonify({"error": "User ID is required"}), 400
        
        # Get all wardrobe items for this user
        items = WardrobeItem.query.filter_by(user_id=user_id).all()
        
        # Convert to JSON serializable format
        wardrobe = []
        for item in items:
            wardrobe.append({
                "category": item.category,
                "style": item.style,
                "color": item.color,
                "description": item.description,
                "date_added": item.date_added.isoformat() if item.date_added else None
            })
            
        return jsonify({"wardrobe": wardrobe})
    except Exception as e:
        logging.error(f"Error in get_wardrobe: {e}")
        return jsonify({"error": "Failed to retrieve wardrobe"}), 500
```

### models.py
```python
from app import db
from flask_login import UserMixin
from datetime import datetime

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256))
    
    # Additional fields for fashion preferences
    style_preferences = db.Column(db.String(500))
    favorite_brands = db.Column(db.String(500))
    
    # Relationships
    wardrobe_items = db.relationship('WardrobeItem', backref='owner', lazy='dynamic')
    outfit_collections = db.relationship('OutfitCollection', backref='owner', lazy='dynamic')
    
    # User preferences
    preferred_colors = db.Column(db.String(200))
    seasonal_preferences = db.Column(db.String(200))
    body_type = db.Column(db.String(50))
    
    # Timestamps
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)
    
    def __repr__(self):
        return f'<User {self.username}>'

class WardrobeItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    category = db.Column(db.String(50), nullable=False)  # tops, bottoms, shoes, accessories
    style = db.Column(db.String(50))  # casual, formal, sporty
    color = db.Column(db.String(50))
    description = db.Column(db.String(200))
    
    # Additional useful fields
    brand = db.Column(db.String(100))
    size = db.Column(db.String(20))
    season = db.Column(db.String(30))  # winter, summer, fall, spring, all-season
    occasion = db.Column(db.String(50))  # casual, work, formal, sports
    
    # Image storage - could be a URL or file path
    image_url = db.Column(db.String(500))
    
    # Timestamps
    date_added = db.Column(db.DateTime, default=datetime.utcnow)
    date_modified = db.Column(db.DateTime, onupdate=datetime.utcnow)
    
    # Relationships
    outfits = db.relationship('OutfitItem', backref='item', lazy='dynamic')
    
    def __repr__(self):
        return f'<WardrobeItem {self.category} - {self.color}>'

class OutfitCollection(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    name = db.Column(db.String(100), nullable=False)
    description = db.Column(db.String(500))
    
    # Type of outfit (casual, formal, work, etc.)
    outfit_type = db.Column(db.String(50))
    
    # Season this outfit is appropriate for
    season = db.Column(db.String(30))
    
    # Is this a favorite outfit?
    is_favorite = db.Column(db.Boolean, default=False)
    
    # Timestamps
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, onupdate=datetime.utcnow)
    
    # Relationships
    items = db.relationship('OutfitItem', backref='outfit', lazy='dynamic')
    
    def __repr__(self):
        return f'<OutfitCollection {self.name}>'

class OutfitItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    outfit_id = db.Column(db.Integer, db.ForeignKey('outfit_collection.id'), nullable=False)
    wardrobe_item_id = db.Column(db.Integer, db.ForeignKey('wardrobe_item.id'), nullable=False)
    
    # Position or layer in the outfit (base layer, top layer, accessory, etc.)
    position = db.Column(db.String(50))
    
    # Timestamps
    added_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<OutfitItem {self.id}>'
```

### templates/index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>StyloAI - Home</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <header>
    <div class="logo">👗 StyloAI</div>
    <nav>
      <a href="{{ url_for('index') }}">Home</a>
      <a href="{{ url_for('style_suggestions') }}">Style Suggestions</a>
      <a href="{{ url_for('wardrobe') }}">Wardrobe</a>
      <a href="{{ url_for('about') }}">About</a>
      <a href="{{ url_for('contact') }}">Contact</a>
    </nav>
    <div class="user-actions">
      <span>🔍</span>
      <span>❤️</span>
      <span>🛒</span>
      <span>👤</span>
    </div>
  </header>
  <main>
    <h1>StyloAI - Your Personal Style Assistant</h1>
    
    <section>
      <h2>Get Style Recommendations</h2>
      <p>Choose any method to get personalized fashion suggestions:</p>
      
      <div>
        <h3>Text Input</h3>
        <form id="textInputForm">
          <input type="text" id="styleInput" placeholder="Describe what you're looking for (e.g., casual summer outfit)">
          <button type="submit" class="btn">Get Suggestions</button>
        </form>
        <div id="textResults"></div>
      </div>
      
      <div>
        <h3>Image Upload</h3>
        <p>Upload an image to analyze the style:</p>
        <input type="file" id="imageUpload" accept="image/*">
        <div id="imageResults"></div>
      </div>
      
      <div>
        <h3>Voice Input</h3>
        <p>Speak your style preferences:</p>
        <button id="recordVoice" class="btn">Record Voice</button>
        <div id="voiceResults"></div>
      </div>
    </section>
    
    <section class="hero">
      <img src="{{ url_for('static', filename='images/Screenshot 2025-04-20 090632.png') }}" alt="Fashion Display">
    </section>
  </main>
  <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

### templates/style-suggestions.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Style Suggestions</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <header>
    <a href="{{ url_for('index') }}" class="logo">👗 StyloAI</a>
    <nav>
      <a href="{{ url_for('index') }}">Home</a>
      <a href="{{ url_for('style_suggestions') }}">Style Suggestions</a>
      <a href="{{ url_for('wardrobe') }}">Wardrobe</a>
      <a href="{{ url_for('about') }}">About</a>
      <a href="{{ url_for('contact') }}">Contact</a>
    </nav>
  </header>
  <main>
    <h1>Style Suggestions</h1>
    <p>Explore tailored fashion tips and outfits curated just for you!</p>
    <button class="btn" id="getSuggestions">Get New Suggestions</button>
    <div id="suggestionResults"></div>
  </main>
  <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

### templates/wardrobe.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Wardrobe</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <header>
    <a href="{{ url_for('index') }}" class="logo">👗 StyloAI</a>
    <nav>
      <a href="{{ url_for('index') }}">Home</a>
      <a href="{{ url_for('style_suggestions') }}">Style Suggestions</a>
      <a href="{{ url_for('wardrobe') }}">Wardrobe</a>
      <a href="{{ url_for('about') }}">About</a>
      <a href="{{ url_for('contact') }}">Contact</a>
    </nav>
  </header>
  <main>
    <h1>My Wardrobe</h1>
    <p>Upload and manage your wardrobe collection.</p>
    
    <form id="addWardrobeForm">
      <h2>Add New Item</h2>
      <select id="category" required>
        <option value="">Select Category</option>
        <option value="Tops">Tops</option>
        <option value="Bottoms">Bottoms</option>
        <option value="Shoes">Shoes</option>
        <option value="Accessories">Accessories</option>
        <option value="Outerwear">Outerwear</option>
      </select>
      
      <select id="style">
        <option value="">Select Style</option>
        <option value="Casual">Casual</option>
        <option value="Formal">Formal</option>
        <option value="Sporty">Sporty</option>
        <option value="Business">Business</option>
      </select>
      
      <input type="text" id="color" placeholder="Color (e.g., Blue, Red)">
      <input type="text" id="description" placeholder="Description (e.g., Cotton t-shirt)">
      
      <button type="submit" class="btn">Add to Wardrobe</button>
    </form>
    
    <h2>Your Wardrobe Items</h2>
    <div id="wardrobeContainer" class="wardrobe-container">
      <!-- Items will be loaded here -->
    </div>
  </main>
  <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

### templates/about.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>About StyloAI</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <header>
    <a href="{{ url_for('index') }}" class="logo">👗 StyloAI</a>
    <nav>
      <a href="{{ url_for('index') }}">Home</a>
      <a href="{{ url_for('style_suggestions') }}">Style Suggestions</a>
      <a href="{{ url_for('wardrobe') }}">Wardrobe</a>
      <a href="{{ url_for('about') }}">About</a>
      <a href="{{ url_for('contact') }}">Contact</a>
    </nav>
  </header>
  <main>
    <h1>About StyloAI</h1>
    <p>StyloAI is a real-time multilingual fashion stylist powered by AI that offers users personalized outfit recommendations through text, voice, and image inputs. Using cutting-edge tools like TensorFlow, OpenCV, and Groq-based inference, it analyzes your wardrobe and fashion sense to provide stylish suggestions and inspiration.</p>
  </main>
  <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

### templates/contact.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Contact Us - StyloAI</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
  <header>
    <a href="{{ url_for('index') }}" class="logo">👗 StyloAI</a>
    <nav>
      <a href="{{ url_for('index') }}">Home</a>
      <a href="{{ url_for('style_suggestions') }}">Style Suggestions</a>
      <a href="{{ url_for('wardrobe') }}">Wardrobe</a>
      <a href="{{ url_for('about') }}">About</a>
      <a href="{{ url_for('contact') }}">Contact</a>
    </nav>
  </header>
  <main>
    <h1>Contact Us</h1>
    <form>
      <input type="text" placeholder="Your Name" required>
      <input type="email" placeholder="Your Email" required>
      <textarea placeholder="Your Message"></textarea>
      <button type="submit" class="btn">Send</button>
    </form>
  </main>
  <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

### static/css/style.css
```css
body {
  font-family: Arial, sans-serif;
  margin: 0;
  background-color: #FFC397;
  color: #333;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background-color: #FFA998;
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
  color: #18ACBA;
  text-decoration: none;
}

nav a {
  margin: 0 1rem;
  text-decoration: none;
  color: #18ACBA;
}

.user-actions span {
  margin: 0 0.5rem;
  cursor: pointer;
}

.hero img {
  display: block;
  max-width: 100%;
  margin: 2rem auto;
  border-radius: 10px;
}

main {
  padding: 2rem;
  background-color: #F76566;
  color: white;
  border-radius: 10px;
  margin: 1rem;
}

form input, form textarea, form button, form select {
  display: block;
  width: 100%;
  margin: 0.5rem 0;
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
}

form button, .btn {
  background-color: #18ACBA;
  color: white;
  font-weight: bold;
  cursor: pointer;
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  margin: 1rem 0;
}

.wardrobe-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
  margin-top: 2rem;
}

.wardrobe-item {
  background-color: rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  padding: 1rem;
}

#suggestionResults, #imageResults, #voiceResults, #textResults {
  margin-top: 1rem;
  padding: 1rem;
  background-color: rgba(255, 255, 255, 0.2);
  border-radius: 8px;
}

section {
  margin-bottom: 2rem;
}

h1, h2, h3 {
  color: white;
}
```

### static/js/script.js
```javascript
document.addEventListener('DOMContentLoaded', function() {
    // Text Input Functionality
    const textForm = document.getElementById('textInputForm');
    if (textForm) {
        textForm.addEventListener('submit', async function(e) {
            e.preventDefault();
            const input = document.getElementById('styleInput').value;
            if (!input) return;
            
            try {
                document.getElementById('textResults').innerHTML = 'Getting suggestions...';
                
                const response = await fetch('/api/styloai', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({ input: input }),
                });
                
                if (!response.ok) {
                    throw new Error('Server error');
                }
                
                const data = await response.json();
                document.getElementById('textResults').innerHTML = data.response;
            } catch (error) {
                console.error('Error:', error);
                document.getElementById('textResults').innerHTML = 'Failed to get suggestions. Please try again.';
            }
        });
    }
    
    // Image Upload Functionality
    const imageUpload = document.getElementById('imageUpload');
    if (imageUpload) {
        imageUpload.addEventListener('change', async function(e) {
            const file = e.target.files[0];
            if (!file) return;
            
            try {
                document.getElementById('imageResults').innerHTML = 'Analyzing image...';
                
                const formData = new FormData();
                formData.append('image', file);
                
                const response = await fetch('/api/image-upload', {
                    method: 'POST',
                    body: formData,
                });
                
                if (!response.ok) {
                    throw new Error('Server error');
                }
                
                const data = await response.json();
                document.getElementById('imageResults').innerHTML = `
                    <p><strong>Style: </strong>${data.style} (${Math.round(data.confidence * 100)}% confident)</p>
                    <p><strong>Recommendation: </strong>${data.recommendation}</p>
                `;
            } catch (error) {
                console.error('Error:', error);
                document.getElementById('imageResults').innerHTML = 'Failed to analyze image. Please try again.';
            }
        });
    }
    
    // Voice Recording Functionality
    const recordButton = document.getElementById('recordVoice');
    if (recordButton) {
        let mediaRecorder;
        let audioChunks = [];
        let recording = false;
        
        recordButton.addEventListener('click', async function() {
            if (!recording) {
                // Start recording
                audioChunks = [];
                try {
                    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                    mediaRecorder = new MediaRecorder(stream);
                    
                    mediaRecorder.ondataavailable = e => {
                        audioChunks.push(e.data);
                    };
                    
                    mediaRecorder.onstop = async () => {
                        const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
                        const formData = new FormData();
                        formData.append('audio', audioBlob);
                        
                        try {
                            document.getElementById('voiceResults').innerHTML = 'Processing your voice input...';
                            
                            const response = await fetch('/api/voice-upload', {
                                method: 'POST',
                                body: formData,
                            });
                            
                            if (!response.ok) {
                                throw new Error('Server error');
                            }
                            
                            const data = await response.json();
                            document.getElementById('voiceResults').innerHTML = `
                                <p><strong>Transcript: </strong>${data.transcript}</p>
                                <p><strong>Suggestion: </strong>${data.suggestion}</p>
                            `;
                        } catch (error) {
                            console.error('Error:', error);
                            document.getElementById('voiceResults').innerHTML = 'Failed to process voice. Please try again.';
                        }
                    };
                    
                    mediaRecorder.start();
                    recording = true;
                    recordButton.textContent = 'Stop Recording';
                } catch (error) {
                    console.error('Error accessing microphone:', error);
                    document.getElementById('voiceResults').innerHTML = 'Failed to access microphone. Please check permissions.';
                }
            } else {
                // Stop recording
                mediaRecorder.stop();
                recording = false;
                recordButton.textContent = 'Record Voice';
            }
        });
    }
    
    // Wardrobe Functionality
    const wardrobeContainer = document.getElementById('wardrobeContainer');
    if (wardrobeContainer) {
        loadWardrobe();
    }
    
    const addWardrobeForm = document.getElementById('addWardrobeForm');
    if (addWardrobeForm) {
        addWardrobeForm.addEventListener('submit', function(e) {
            e.preventDefault();
            const category = document.getElementById('category').value;
            const style = document.getElementById('style').value;
            const color = document.getElementById('color').value;
            const description = document.getElementById('description').value;
            
            if (!category) return;
            
            // Add to wardrobe and save
            const wardrobe = JSON.parse(localStorage.getItem('wardrobe') || '[]');
            wardrobe.push({
                category,
                style,
                color,
                description,
                date_added: new Date().toISOString()
            });
            
            localStorage.setItem('wardrobe', JSON.stringify(wardrobe));
            saveWardrobeToServer(wardrobe);
            
            // Clear form and reload display
            addWardrobeForm.reset();
            displayWardrobeItems(wardrobe);
        });
    }
    
    // Get suggestions button
    const getSuggestionsBtn = document.getElementById('getSuggestions');
    if (getSuggestionsBtn) {
        getSuggestionsBtn.addEventListener('click', async function() {
            try {
                document.getElementById('suggestionResults').innerHTML = 'Getting suggestions...';
                
                // Random suggestion for demo purposes
                const suggestions = [
                    "For your casual style, try pairing a white t-shirt with blue jeans and white sneakers.",
                    "A navy blazer with khaki pants and brown loafers would look great for your business casual needs.",
                    "For a night out, consider a black dress with statement earrings and heels.",
                    "Your athletic style would benefit from moisture-wicking tops and comfortable running shoes."
                ];
                
                const randomSuggestion = suggestions[Math.floor(Math.random() * suggestions.length)];
                document.getElementById('suggestionResults').innerHTML = randomSuggestion;
                
            } catch (error) {
                console.error('Error:', error);
                document.getElementById('suggestionResults').innerHTML = 'Failed to get suggestions. Please try again.';
            }
        });
    }
});

async function loadWardrobe() {
    try {
        // For demo, user_id is hardcoded as 'user123'
        const userId = 'user123';
        
        const response = await fetch(`/api/get-wardrobe?user_id=${userId}`);
        
        if (!response.ok) {
            throw new Error('Failed to load wardrobe data');
        }
        
        const data = await response.json();
        const wardrobe = data.wardrobe;
        
        // Save to local storage for easy access
        localStorage.setItem('wardrobe', JSON.stringify(wardrobe));
        
        // Display wardrobe items
        displayWardrobeItems(wardrobe);
    } catch (error) {
        console.error('Error loading wardrobe:', error);
    }
}

function displayWardrobeItems(wardrobe) {
    const container = document.getElementById('wardrobeContainer');
    if (!container) return;
    
    if (!wardrobe || wardrobe.length === 0) {
        container.innerHTML = '<p>Your wardrobe is empty. Add some items!</p>';
        return;
    }
    
    container.innerHTML = '';
    wardrobe.forEach((item, index) => {
        const itemDiv = document.createElement('div');
        itemDiv.className = 'wardrobe-item';
        itemDiv.innerHTML = `
            <h3>${item.category}</h3>
            <p><strong>Style:</strong> ${item.style}</p>
            <p><strong>Color:</strong> ${item.color}</p>
            <p><strong>Description:</strong> ${item.description}</p>
            <button onclick="removeWardrobeItem(${index})">Remove</button>
        `;
        container.appendChild(itemDiv);
    });
}

function removeWardrobeItem(index) {
    const wardrobe = JSON.parse(localStorage.getItem('wardrobe') || '[]');
    wardrobe.splice(index, 1);
    localStorage.setItem('wardrobe', JSON.stringify(wardrobe));
    displayWardrobeItems(wardrobe);
    saveWardrobeToServer(wardrobe);
}

async function saveWardrobeToServer(wardrobe) {
    try {
        // For demo, user_id is hardcoded as 'user123'
        const userId = 'user123';
        
        const response = await fetch('/api/save-wardrobe', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                user_id: userId,
                wardrobe: wardrobe
            }),
        });
        
        if (!response.ok) {
            throw new Error('Failed to save wardrobe');
        }
    } catch (error) {
        console.error('Error saving wardrobe:', error);
    }
}
```

### requirements.txt
```
flask
flask-cors
flask-login
flask-sqlalchemy
numpy
opencv-python
pillow
psycopg2-binary
gunicorn
email-validator
```

## Running the Application

1. Clone this repository to your local machine
2. Install the requirements: `pip install -r requirements.txt`
3. Set up the environment variables (optional):
   ```
   export DATABASE_URL=postgresql://username:password@localhost:5432/styloai
   export SESSION_SECRET=your_secret_key
   ```
4. Run the application:
   ```
   python main.py
   ```
5. Access the application in your browser at http://localhost:5000

## Deployment

For production, you'll want to use a proper WSGI server like Gunicorn:

```
gunicorn --bind 0.0.0.0:5000 main:app
```

## Database Migrations

If you need to update the database schema:

1. Delete the existing database (during development only)
2. The tables will be recreated automatically when the application starts
