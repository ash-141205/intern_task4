# intern_task4
from flask import Flask, render_template, session, request, redirect, url_for, send_from_directory, jsonify
from flask_socketio import SocketIO, join_room, leave_room, emit
from flask_sqlalchemy import SQLAlchemy
from werkzeug.utils import secure_filename
from datetime import datetime
import os

app = Flask(__name__, template_folder='template chat')  # <== Pointing to your folder
app.config['SECRET_KEY'] = 'secretkey'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///chat.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['UPLOAD_FOLDER'] = 'uploads'

if not os.path.exists('uploads'):
    os.makedirs('uploads')

db = SQLAlchemy(app)
socketio = SocketIO(app)

online_users = {}  # room: [users]
typing_users = {}  # room: [typing_users]

# === MODELS ===
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), nullable=False)
    room = db.Column(db.String(50), nullable=False)
    content = db.Column(db.Text, nullable=False)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)

# === ROUTES ===
@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        session['username'] = request.form['username']
        session['room'] = request.form['room']
        return redirect(url_for('chat'))
    return render_template('index.html')

@app.route('/chat')
def chat():
    if 'username' not in session:
        return redirect(url_for('index'))
    messages = Message.query.filter_by(room=session['room']).all()
    users = online_users.get(session['room'], [])
    return render_template('chat.html', username=session['username'], room=session['room'], messages=messages, users=users)

@app.route('/upload', methods=['POST'])
def upload():
    file = request.files['file']
    if file:
        filename = secure_filename(file.filename)
        filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        file.save(filepath)
        return jsonify({'filename': filename}), 200
    return jsonify({'error': 'No file uploaded'}), 400

@app.route('/uploads/<filename>')
def uploaded_file(filename):
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)

# === SOCKET EVENTS ===
@socketio.on('send_message')
def handle_send_message_event(data):
    msg = Message(username=data['username'], room=data['room'], content=data['message'])
    db.session.add(msg)
    db.session.commit()
    emit('message', {
        'username': data['username'],
        'room': data['room'],
        'message': data['message'],
        'timestamp': msg.timestamp.strftime("%H:%M")
    }, room=data['room'])

@socketio.on('join_room')
def handle_join_room_event(data):
    join_room(data['room'])
    online_users.setdefault(data['room'], [])
    if data['username'] not in online_users[data['room']]:
        online_users[data['room']].append(data['username'])

    emit('user_list', online_users[data['room']], room=data['room'])
    emit('message', {
        'username': data['username'],
        'message': 'has joined the room.',
        'timestamp': datetime.utcnow().strftime('%H:%M')
    }, room=data['room'])

@socketio.on('leave_room')
def handle_leave_room_event(data):
    leave_room(data['room'])
    if data['room'] in online_users:
        if data['username'] in online_users[data['room']]:
            online_users[data['room']].remove(data['username'])
    emit('user_list', online_users.get(data['room'], []), room=data['room'])
    emit('message', {
        'username': data['username'],
        'message': 'has left the room.',
        'timestamp': datetime.utcnow().strftime('%H:%M')
    }, room=data['room'])

@socketio.on('typing')
def handle_typing(data):
    room = data['room']
    username = data['username']
    typing_users.setdefault(room, set()).add(username)
    emit('typing_status', list(typing_users[room]), room=room)

@socketio.on('stop_typing')
def handle_stop_typing(data):
    room = data['room']
    username = data['username']
    if room in typing_users and username in typing_users[room]:
        typing_users[room].remove(username)
    emit('typing_status', list(typing_users.get(room, [])), room=room)

# === RUN ===
if __name__ == '__main__':
    with app.app_context():
        db.drop_all()    # Drop any old tables
        db.create_all()  # Create new ones with updated schema
    socketio.run(app, debug=True)

#HTML(TEMP)

<!DOCTYPE html>
<html>
<head>
    <title>Chat Room</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.3.2/socket.io.min.js"></script>
    <style>
        body { font-family: Arial; padding: 20px; }
        #messages { height: 300px; border: 1px solid #ccc; overflow-y: scroll; padding: 10px; margin-bottom: 10px; background: #f9f9f9; }
        #users { background: #eef; padding: 5px; margin-bottom: 10px; }
        .msg { margin: 5px 0; }
        .file-link { color: purple; }
    </style>
</head>
<body>
    <h2>Chat Room: {{ room }}</h2>
    <p>Logged in as: {{ username }}</p>

    <b>Online Users:</b>
    <div id="users">
        {% for user in users %}
            <div>{{ user }}</div>
        {% endfor %}
    </div>

    <div id="messages">
        {% for msg in messages %}
            <div class="msg"><b>{{ msg.username }}:</b> {{ msg.content }}</div>
        {% endfor %}
    </div>

    <input id="message" placeholder="Type a message..." style="width: 80%;">
    <button onclick="sendMessage()">Send</button>

    <form id="uploadForm" enctype="multipart/form-data">
        <input type="file" name="file" id="fileInput">
        <button type="submit">Upload File</button>
    </form>

    <script>
        const socket = io();
        const username = "{{ username }}";
        const room = "{{ room }}";

        socket.emit("join_room", { username, room });

        socket.on("message", data => {
            const msgBox = document.getElementById("messages");
            const div = document.createElement("div");
            div.className = "msg";
            if (data.message.includes(".png") || data.message.includes(".jpg")) {
                div.innerHTML = `<b>${data.username}:</b> <a href="/uploads/${data.message}" class="file-link" target="_blank">${data.message}</a>`;
            } else {
                div.innerHTML = `<b>${data.username}:</b> ${data.message}`;
            }
            msgBox.appendChild(div);
            msgBox.scrollTop = msgBox.scrollHeight;
        });

        socket.on("user_list", users => {
            const userDiv = document.getElementById("users");
            userDiv.innerHTML = users.map(u => `<div>${u}</div>`).join('');
        });

        function sendMessage() {
            const input = document.getElementById("message");
            const message = input.value.trim();
            if (message !== "") {
                socket.emit("send_message", { username, room, message });
                input.value = "";
            }
        }

        document.getElementById("uploadForm").onsubmit = async (e) => {
            e.preventDefault();
            const fileInput = document.getElementById("fileInput");
            if (fileInput.files.length > 0) {
                const formData = new FormData();
                formData.append("file", fileInput.files[0]);

                const res = await fetch("/upload", { method: "POST", body: formData });
                const data = await res.json();

                if (data.filename) {
                    socket.emit("send_message", {
                        username, room, message: data.filename
                    });
                }
                fileInput.value = "";
            }
        };
    </script>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
  <title>Join Chat Room</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      padding: 20px;
      background: #f1f1f1;
    }
    form {
      background: white;
      padding: 30px;
      border-radius: 10px;
      width: 300px;
      margin: 50px auto;
      box-shadow: 0 0 10px #ccc;
    }
    input {
      width: 100%;
      padding: 10px;
      margin: 10px 0;
    }
    button {
      width: 100%;
      padding: 10px;
      background: #2196F3;
      color: white;
      border: none;
      border-radius: 5px;
    }
    h2 {
      text-align: center;
    }
  </style>
</head>
<body>
  <form method="POST">
    <h2>Join Chat Room</h2>
    <input type="text" name="username" placeholder="Your Name" required>
    <input type="text" name="room" placeholder="Room Name" required>
    <button type="submit">Enter</button>
  </form>
</body>
</html>
