[taskpro_admin (1).zip](https://github.com/user-attachments/files/21689523/taskpro_admin.1.zip)
pip install flask openai google-api-python-client twilio
from flask import Flask, jsonify, request
import openai
from googleapiclient.discovery import build
from google_auth_oauthlib.flow import InstalledAppFlow
import os
from twilio.rest import Client

# Initialize Flask app
app = Flask(__name__)

# Set OpenAI API Key
openai.api_key = 'your_openai_api_key'

# Set up Twilio client
twilio_client = Client('your_twilio_account_sid', 'your_twilio_auth_token')

# Google Calendar API setup
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']
creds = None

def google_authenticate():
    """Authenticate Google API"""
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                'credentials.json', SCOPES)
            creds = flow.run_local_server(port=0)
        with open('token.json', 'w') as token:
            token.write(creds.to_json())
    return build('calendar', 'v3', credentials=creds)

@app.route("/ai", methods=["POST"])
def ai_assistant():
    data = request.json
    user_message = data.get("message")
    response = openai.Completion.create(
        model="gpt-3.5-turbo",
        prompt=f"You are an assistant. Respond to: {user_message}",
        max_tokens=150
    )
    return jsonify({"response": response.choices[0].text.strip()})

@app.route("/schedule", methods=["GET"])
def get_calendar_events():
    service = google_authenticate()
    events_result = service.events().list(calendarId='primary', timeMin='2023-08-01T00:00:00Z', timeMax='2023-08-31T23:59:59Z', singleEvents=True, orderBy='startTime').execute()
    events = events_result.get('items', [])
    event_list = []
    if not events:
        event_list.append('No upcoming events found.')
    for event in events:
        event_list.append(f"{event['summary']} at {event['start']['dateTime']}")
    return jsonify({"events": event_list})

@app.route("/make_call", methods=["POST"])
def make_call():
    data = request.json
    to_number = data.get("to")
    from_number = data.get("from")
    message = data.get("message")
    call = twilio_client.calls.create(
        to=to_number,
        from_=from_number,
        twiml=f"<Response><Say>{message}</Say></Response>"
    )
    return jsonify({"status": "call initiated", "sid": call.sid})

if __name__ == "__main__":
    app.run(debug=True)
