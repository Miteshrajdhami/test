import drivers
import requests
import serial
from gpiozero import OutputDevice, InputDevice
from time import sleep
from flask import Flask, render_template_string, Response, jsonify, redirect
import threading
from datetime import datetime
import json

# Initialize the LCD display
display = drivers.Lcd()

# Pin Definitions
IN1 = OutputDevice(14)  # Connect to motor IN1
IN2 = OutputDevice(15)  # Connect to motor IN2
IN3 = OutputDevice(18)  # Connect to motor IN3
IN4 = OutputDevice(23)  # Connect to motor IN4
IR_SENSOR_PIN = 17      # Pin connected to the IR sensor's OUT pin

# Step sequence for a 4-phase stepper motor
step_sequence = [
    [1, 0, 0, 0],  # Activate IN1
    [1, 1, 0, 0],  # Activate IN1 + IN2
    [0, 1, 0, 0],  # Activate IN2
    [0, 1, 1, 0],  # Activate IN2 + IN3
    [0, 0, 1, 0],  # Activate IN3
    [0, 0, 1, 1],  # Activate IN3 + IN4
    [0, 0, 0, 1],  # Activate IN4
    [1, 0, 0, 1]   # Activate IN4 + IN1
]

# Set up the IR sensor
ir_sensor = InputDevice(IR_SENSOR_PIN)

# Initialize Flask web server
app = Flask(__name__)

# Global variables to track motor state, server logs, and last GPS location
motor_running = False
vehicle_stopped = False
server_logs = ["System Ready"]
last_log_time = datetime.now()
last_gps_location = None  # New global variable to store the latest GPS URL

# Function to update the LCD display
def update_lcd(line1, line2=""):
    try:
        display.lcd_clear()
        display.lcd_display_string(line1, 1)
        display.lcd_display_string(line2, 2)
        print(f"LCD Updated: {line1} | {line2}")
        sleep(1)
    except Exception as e:
        print(f"LCD Error: {e}")

# Function to control the stepper motor
def set_step(w1, w2, w3, w4):
    IN1.value = w1
    IN2.value = w2
    IN3.value = w3
    IN4.value = w4

# Function to continuously rotate the motor
def step_motor_continuous(delay=100):
    global motor_running, server_logs, last_log_time
    motor_running = True
    update_lcd("Vehicle Status:", "Running")
    current_time = datetime.now()
    if (current_time - last_log_time).total_seconds() > 1:
        new_log = f"{current_time.strftime('%H:%M:%S')} - Vehicle has been started"
        server_logs.append(new_log)
        last_log_time = current_time
    print("Vehicle is now running continuously...")
    while motor_running:
        for step in step_sequence:
            if not motor_running:
                break
            set_step(*step)
            sleep(delay / 1000)
    stop_motor()

# Function to stop the motor and update the display
def stop_motor():
    global motor_running, vehicle_stopped, server_logs, last_log_time
    motor_running = False
    vehicle_stopped = True
    set_step(0, 0, 0, 0)
    update_lcd("Vehicle Status:", "Stopped")
    current_time = datetime.now()
    if (current_time - last_log_time).total_seconds() > 1:
        new_log = f"{current_time.strftime('%H:%M:%S')} - Vehicle has been stopped"
        server_logs.append(new_log)
        last_log_time = current_time
    print("Vehicle stopped!")

# Function to start the motor only when IR sensor detects a finger
def start_motor_with_ir():
    global motor_running, vehicle_stopped, server_logs, last_log_time
    if vehicle_stopped:
        print("Waiting for finger detection to start the vehicle...")
        update_lcd("Place finger to start", "Vehicle stopped")
        current_time = datetime.now()
        if (current_time - last_log_time).total_seconds() > 1:
            new_log = f"{current_time.strftime('%H:%M:%S')} - Waiting for finger detection"
            server_logs.append(new_log)
            last_log_time = current_time
        previous_finger_state = ir_sensor.is_active
        while True:
            current_finger_state = ir_sensor.is_active
            if current_finger_state != previous_finger_state:
                if current_finger_state:
                    print("Finger detected! Starting vehicle...")
                    update_lcd("Vehicle Status:", "Running")
                    current_time = datetime.now()
                    if (current_time - last_log_time).total_seconds() > 1:
                        new_log = f"{current_time.strftime('%H:%M:%S')} - Finger detected! Starting vehicle..."
                        server_logs.append(new_log)
                        last_log_time = current_time
                    threading.Thread(target=step_motor_continuous, args=(100,)).start()
                    vehicle_stopped = False
                    sleep(2)
                    break
            previous_finger_state = current_finger_state
            sleep(0.1)
    else:
        print(".!")
        current_time = datetime.now()
        if (current_time - last_log_time).total_seconds() > 1:
            new_log = f"{current_time.strftime('%H:%M:%S')} - ."
            server_logs.append(new_log)
            last_log_time = current_time

# Function to send SMS with links
def send_sms():
    website_url = 'http://192.168.10.86:5000'
    message = f"""
    Ã°Å¸Å¡â€” Vehicle Start Detected!!!

    The vehicle has been started. Control it here:
    {website_url}

    Thank you!
    """
    r = requests.post(
        "https://sms.aakashsms.com/sms/v3/send/",
        data={
            'auth_token': 'a5a492a8f95c9a887b799021d5a447890d9d7bf1e5eeb954b38ebfa9cf76b482',
            'to': '9818858567',
            'text': message
        }
    )
    if r.status_code == 200:
        print("SMS sent successfully!")
    else:
        print(f"Failed to send SMS: {r.text}")

# GPS Setup
gps_serial = serial.Serial('/dev/ttyUSB0', baudrate=9600, timeout=1)

def parse_gprmc(sentence):
    parts = sentence.split(',')
    if len(parts) < 12 or parts[2] != "A":
        return None
    lat = convert_to_decimal(parts[3], parts[4])
    lon = convert_to_decimal(parts[5], parts[6])
    return {"latitude": lat, "longitude": lon}

def convert_to_decimal(value, direction):
    if not value or not direction:
        return None
    if direction in ['N', 'S']:
        degrees = float(value[:2])
        minutes = float(value[2:]) / 60
    elif direction in ['E', 'W']:
        degrees = float(value[:3])
        minutes = float(value[3:]) / 60
    else:
        return None
    decimal = degrees + minutes
    if direction in ['S', 'W']:
        decimal *= -1
    return decimal

def get_location():
    global server_logs, last_log_time, last_gps_location
    while True:
        line = gps_serial.readline().decode('ascii', errors='replace').strip()
        if line.startswith("$GPRMC"):
            data = parse_gprmc(line)
            if data:
                maps_link = f"https://www.google.com/maps?q={data['latitude']},{data['longitude']}"
                last_gps_location = maps_link  # Store the latest GPS URL
                update_lcd("GPS Location:", "Fetched")
                current_time = datetime.now()
                if (current_time - last_log_time).total_seconds() > 1:
                    new_log = f"{current_time.strftime('%H:%M:%S')} - GPS Location fetched: {maps_link}"
                    server_logs.append(new_log)
                    last_log_time = current_time
                return maps_link
        sleep(1)

# SSE Route for real-time updates
def stream_logs():
    global server_logs
    yield f"data: {json.dumps({'logs': server_logs})}\n\n"
    while True:
        sleep(0.1)
        if len(server_logs) > len(initial_logs):
            yield f"data: {json.dumps({'logs': server_logs})}\n\n"
            initial_logs = server_logs.copy()

# Flask Routes
@app.route('/stop_vehicle', methods=['GET'])
def stop_motor_web():
    global server_logs
    print("Stopping Vehicle via web...")
    stop_motor()
    return jsonify({"logs": server_logs})

@app.route('/start_vehicle', methods=['GET'])
def start_motor_web():
    global server_logs
    print("Starting vehicle via web...")
    threading.Thread(target=start_motor_with_ir).start()
    sleep(2)
    return jsonify({"logs": server_logs})

@app.route('/send_location', methods=['GET'])
def send_location_web():
    global server_logs
    print("Fetching GPS location...")
    location = get_location()
    return jsonify({"logs": server_logs, "location": location})

@app.route('/redirect_to_map', methods=['GET'])
def redirect_to_map():
    global last_gps_location
    if last_gps_location:
        return redirect(last_gps_location)  # Redirect to the last fetched GPS URL
    else:
        return jsonify({"error": "No GPS location available yet"}), 400

@app.route('/stream')
def stream():
    return Response(stream_logs(), mimetype='text/event-stream')

@app.route('/')
def index():
    start_url = 'http://192.168.10.86:5000/start_vehicle'
    stop_url = 'http://192.168.10.86:5000/stop_vehicle'
    location_url = 'http://192.168.10.86:5000/send_location'
    map_redirect_url = 'http://192.168.10.86:5000/redirect_to_map'
    return render_template_string("""
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Vehicle Control Dashboard</title>
            <style>
                * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Arial', sans-serif; }
                body { 
                    background: #1a2535; 
                    min-height: 100vh; 
                    display: flex; 
                    justify-content: center; 
                    align-items: center; 
                    padding: 20px; 
                    color: #fff; 
                }
                .container { 
                    background: rgba(255, 255, 255, 0.1); 
                    backdrop-filter: blur(10px); 
                    border-radius: 15px; 
                    padding: 30px; 
                    width: 100%; 
                    max-width: 400px; 
                    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.3); 
                    border: 1px solid rgba(255, 255, 255, 0.1); 
                    text-align: center; 
                }
                h1 { 
                    font-size: 2.2em; 
                    color: #ff4d6d; 
                    margin-bottom: 20px; 
                    text-shadow: 0 0 10px rgba(255, 77, 109, 0.5); 
                }
                .button-group { 
                    display: flex; 
                    flex-direction: column; 
                    gap: 15px; 
                    margin-bottom: 20px; 
                }
                .control-btn { 
                    display: inline-block; 
                    padding: 15px; 
                    font-size: 1.1em; 
                    text-decoration: none; 
                    color: #fff; 
                    background: linear-gradient(90deg, #ff4d6d, #4a69bd); 
                    border: none; 
                    border-radius: 25px; 
                    width: 100%; 
                    cursor: pointer; 
                    transition: transform 0.3s, box-shadow 0.3s; 
                    box-shadow: 0 5px 15px rgba(255, 77, 109, 0.4); 
                }
                .control-btn:hover { 
                    transform: translateY(-5px); 
                    box-shadow: 0 8px 25px rgba(255, 77, 109, 0.6); 
                }
                .control-btn:active { 
                    transform: translateY(0); 
                    box-shadow: 0 3px 10px rgba(255, 77, 109, 0.3); 
                }
                .status-box { 
                    margin-top: 20px; 
                    padding: 15px; 
                    background: rgba(255, 255, 255, 0.05); 
                    border-radius: 10px; 
                    font-size: 0.9em; 
                    color: #a3bffa; 
                    max-height: 150px; 
                    overflow-y: auto; 
                    text-align: left; 
                    border: 1px solid rgba(255, 255, 255, 0.1); 
                }
                .status-log { 
                    margin-bottom: 10px; 
                    padding: 5px; 
                    background: rgba(255, 255, 255, 0.02); 
                    border-radius: 5px; 
                }
                .status-log.new { 
                    animation: fadeIn 1s; 
                }
                @keyframes fadeIn { 
                    from { opacity: 0; transform: translateY(10px); } 
                    to { opacity: 1; transform: translateY(0); } 
                }
                footer { 
                    margin-top: 20px; 
                    font-size: 0.9em; 
                    color: rgba(255, 255, 255, 0.6); 
                }
                .bottom-button { 
                    margin-top: 20px; 
                }
                @media (max-width: 480px) { 
                    .container { padding: 20px; } 
                    h1 { font-size: 1.8em; } 
                    .control-btn { font-size: 1em; padding: 12px; } 
                    .status-box { font-size: 0.8em; max-height: 120px; } 
                }
            </style>
        </head>
        <body>
            <div class="container">
                <h1>Vehicle Control</h1>
                <div class="button-group">
                    <a href="{{ start_url }}" class="control-btn" onclick="updateStatus(event, this)">Start Vehicle</a>
                    <a href="{{ stop_url }}" class="control-btn" onclick="updateStatus(event, this)">Stop Vehicle</a>
                    <a href="{{ location_url }}" class="control-btn" onclick="updateStatus(event, this)">View GPS Location</a>
                </div>
                <div class="status-box" id="status-box">
                    {% for log in server_logs %}
                        <div class="status-log">{{ log }}</div>
                    {% endfor %}
                </div>
                <div class="bottom-button">
                    <a href="{{ map_redirect_url }}" class="control-btn">Open Latest Map</a>
                </div>
                <footer>2025</footer>
            </div>
            <script>
                const eventSource = new EventSource('/stream');
                const statusBox = document.getElementById('status-box');
                let lastLogCount = {{ server_logs|length }};

                eventSource.onmessage = function(event) {
                    const data = JSON.parse(event.data);
                    const currentLogCount = data.logs.length;
                    if (currentLogCount > lastLogCount) {
                        statusBox.innerHTML = '';
                        data.logs.forEach(log => {
                            const logDiv = document.createElement('div');
                            logDiv.className = 'status-log' + (currentLogCount > lastLogCount ? ' new' : '');
                            logDiv.textContent = log;
                            statusBox.appendChild(logDiv);
                        });
                        lastLogCount = currentLogCount;
                        statusBox.scrollTop = statusBox.scrollHeight;
                    }
                };

                eventSource.onerror = function() {
                    console.error('EventSource failed');
                };

                function updateStatus(event, element) {
                    event.preventDefault();
                    const url = element.href;
                    fetch(url)
                        .then(response => response.json())
                        .then(data => {
                            if (data.location) {
                                window.open(data.location, '_blank');
                            }
                        })
                        .catch(error => {
                            console.error('Error:', error);
                            const logDiv = document.createElement('div');
                            logDiv.className = 'status-log new';
                            logDiv.textContent = 'Error occurred';
                            statusBox.appendChild(logDiv);
                            statusBox.scrollTop = statusBox.scrollHeight;
                        });
                }
            </script>
        </body>
        </html>
    """, start_url=start_url, stop_url=stop_url, location_url=location_url, map_redirect_url=map_redirect_url, server_logs=server_logs)

initial_logs = server_logs.copy()
def run_flask():
    app.run(host='0.0.0.0', port=5000, debug=False)

flask_thread = threading.Thread(target=run_flask)
flask_thread.daemon = True
flask_thread.start()

try:
    print("Waiting for Key detection...")
    update_lcd("System Ready", "Waiting for IR")
    current_time = datetime.now()
    if (current_time - last_log_time).total_seconds() > 1:
        new_log = f"{current_time.strftime('%H:%M:%S')} - Waiting for Key detection..."
        server_logs.append(new_log)
        last_log_time = current_time
    previous_finger_state = ir_sensor.is_active

    while True:
        current_finger_state = ir_sensor.is_active
        if current_finger_state != previous_finger_state:
            if current_finger_state and not motor_running and not vehicle_stopped:
                print("Key detected! Starting vehicle...")
                threading.Thread(target=step_motor_continuous, args=(100,)).start()
                send_sms()
                sleep(2)
        previous_finger_state = current_finger_state
        sleep(0.1)

except KeyboardInterrupt:
    print("Program stopped by user.")
finally:
    print("Cleaning up...")
    stop_motor()
    display.lcd_clear()
