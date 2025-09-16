# Virtual Recruiter
Installation steps and code for a virtual recruiter on Windows (Python 3.13.4)
This solution creates a virtual recruiter that interacts with candidates over the phone, transcribes their speech, and performs sentiment analysis on their responses. The process is fully automated and runs on a local Windows machine. 
Technologies used:
Twilio: Provides the phone number and handles incoming calls.
Rasa: An open-source framework for building conversational AI. It manages the dialogue flow and understands the user's intent.
Flask: A Python web framework that creates the webhook to connect Twilio and Rasa.
TextBlob: A simple library for performing natural language processing tasks, including sentiment analysis.
SpeechRecognition and pydub: Libraries for speech-to-text transcription.
gTTS: Converts the bot's text responses back into speech. 
Step 1: Install prerequisites and set up the Rasa project
Install Python 3.13.4: Download and install the latest 64-bit version of Python for Windows from the official website. Check the box "Add python.exe to PATH" during installation.
Install FFmpeg: Download FFmpeg for Windows from the gyan.dev website. Extract the zip file and add the bin folder's path to your system's PATH environment variable.
# Create and activate a virtual environment:
sh
mkdir rasa_recruiter_313
cd rasa_recruiter_313
python -m venv venv
venv\Scripts\activate

# Install libraries:
sh
pip install rasa Flask twilio textblob gTTS SpeechRecognition pydub
python -m textblob.download_corpora


# Initialize Rasa:
sh
rasa init --no-prompt


Update Rasa configuration: Customize the screening process by editing the Rasa files. This example asks for a name, experience, and availability.
# domain.yml: Defines the conversation's structure.
yaml
version: "3.1"
intents:
  - greet
  - provide_name
  - provide_experience
  - provide_availability
  - goodbye

entities:
  - name
  - experience
  - availability

responses:
  utter_greet:
    - text: "Hello! I am a virtual recruiter for Acme Corp. May I please have your name?"
  utter_ask_experience:
    - text: "Thank you, {name}. How many years of experience do you have with Python?"
  utter_ask_availability:
    - text: "Thank you. When are you available to start?"
  utter_goodbye:
    - text: "Thank you for your time. We will be in touch shortly."

actions:
  - action_log_and_analyze

# data/rules.yml: Defines the conversation's flow.
yaml
version: "3.1"
rules:
- rule: Greet and ask for name
  steps:
  - intent: greet
  - action: utter_greet

- rule: Ask for experience after name
  steps:
  - intent: provide_name
  - action: utter_ask_experience

- rule: Ask for availability after experience
  steps:
  - intent: provide_experience
  - action: utter_ask_availability

- rule: Log and analyze after availability
  steps:
  - intent: provide_availability
  - action: action_log_and_analyze


# actions.py: Includes a custom action to log the conversation and analyze the candidate's sentiment.
python
from rasa_sdk import Action, Tracker
from rasa_sdk.executor import CollectingDispatcher
from textblob import TextBlob

class ActionLogAndAnalyze(Action):
    def name(self) -> str:
        return "action_log_and_analyze"

    def run(self, dispatcher: CollectingDispatcher, tracker: Tracker, domain: dict) -> list:
        # Retrieve conversation details and slots
        name = tracker.get_slot("name")
        experience = tracker.get_slot("experience")
        availability = tracker.get_slot("availability")

        # Get the full conversation transcript
        transcript = ""
        for event in tracker.events:
            if event.get("event") == "user":
                transcript += f"Candidate: {event.get('text')}\n"
            elif event.get("event") == "bot":
                transcript += f"Recruiter: {event.get('text')}\n"

        # Perform sentiment analysis on the transcript
        blob = TextBlob(transcript)
        sentiment_score = blob.sentiment.polarity

        # Create a log file
        with open("screening_results.log", "a") as f:
            f.write(f"--- Candidate Screening: {name} ---\n")
            f.write(f"Experience: {experience}\n")
            f.write(f"Availability: {availability}\n")
            f.write(f"Transcript:\n{transcript}\n")
            f.write(f"Sentiment Analysis Score: {sentiment_score:.2f}\n")
            f.write("-" * 30 + "\n")

        # Provide the final bot response
        dispatcher.utter_message(text="Thank you for your time. The screening process is complete.")
        return []


# Train Rasa:
sh
rasa train


# Run Rasa servers: Open two Command Prompt windows and run these commands in your virtual environment:
Action Server: rasa run actions
Core Server: rasa run --enable-api --cors "*" 

# Step 2: Set up the Flask application
Install ngrok: Download and configure ngrok to expose your local Flask server to the internet.
app.py: Create a Flask application that acts as the webhook between Twilio and Rasa.
python
import os
import requests
import tempfile
from flask import Flask, request, send_file
from twilio.twiml.voice_response import VoiceResponse, Gather
from gtts import gTTS
from pydub import AudioSegment
import speech_recognition as sr

app = Flask(__name__)

RASA_API_URL = "http://localhost:5005/webhooks/rest/webhook"

def text_to_speech(text):
    """Converts text to speech and saves it as a temporary MP3 file."""
    tts = gTTS(text=text, lang='en')
    filepath = os.path.join(tempfile.gettempdir(), "response.mp3")
    tts.save(filepath)
    return filepath

def speech_to_text(audio_url):
    """Downloads Twilio audio, converts it to text, and cleans up files."""
    try:
        mp3_path = os.path.join(tempfile.gettempdir(), "twilio_audio.mp3")
        wav_path = os.path.join(tempfile.gettempdir(), "twilio_audio.wav")

        # Download audio
        audio_content = requests.get(audio_url).content
        with open(mp3_path, 'wb') as f:
            f.write(audio_content)

        # Convert MP3 to WAV
        audio = AudioSegment.from_mp3(mp3_path)
        audio.export(wav_path, format="wav")

        # Transcribe audio
        r = sr.Recognizer()
        with sr.AudioFile(wav_path) as source:
            audio_data = r.record(source)

        text = r.recognize_google(audio_data)

        # Clean up temporary files
        os.remove(mp3_path)
        os.remove(wav_path)
        return text

    except sr.UnknownValueError:
        return ""
    except Exception as e:
        print(f"Error in speech-to-text conversion: {e}")
        return ""

@app.route("/voice", methods=['POST'])
def voice():
    """Main entry point for incoming calls from Twilio."""
    response = VoiceResponse()
    gather = Gather(input='speech', action='/gather', method='POST', timeout=5)

    # Start the conversation with Rasa
    rasa_payload = {"sender": request.values.get('CallSid'), "message": "/greet"}
    rasa_response = requests.post(RASA_API_URL, json=rasa_payload).json()
    bot_response_text = rasa_response[0]['text'] if rasa_response else "Hello, welcome to the recruiter line."

    audio_file = text_to_speech(bot_response_text)
    gather.play(f"http://{request.host}/audio/{os.path.basename(audio_file)}")
    response.append(gather)

    return str(response)

@app.route("/gather", methods=['POST'])
def gather():
    """Handles the user's speech input."""
    response = VoiceResponse()
    call_sid = request.values.get('CallSid')

    # Transcribe user's speech
    user_speech = request.values.get('SpeechResult')

    # Send user input to Rasa
    rasa_payload = {"sender": call_sid, "message": user_speech}
    rasa_response = requests.post(RASA_API_URL, json=rasa_payload).json()

    if rasa_response:
        bot_response_text = rasa_response[0]['text']
        audio_file = text_to_speech(bot_response_text)
        response.play(f"http://{request.host}/audio/{os.path.basename(audio_file)}")

        # Continue gathering input
        gather = Gather(input='speech', action='/gather', method='POST', timeout=5)
        response.append(gather)
    else:
        response.say("Sorry, I could not process your request. Goodbye.")

    return str(response)

@app.route("/audio/<filename>")
def get_audio(filename):
    """Serves the generated audio files."""
    return send_file(os.path.join(tempfile.gettempdir(), filename), mimetype="audio/mpeg")

if __name__ == "__main__":
    app.run(port=5000, debug=True)


 
# Step 3: Connect Twilio and test
Start ngrok: In a new Command Prompt window, run ngrok http 5000. Copy the public forwarding URL.
Configure Twilio: Log in to your Twilio Console, navigate to your phone number settings, and set the "A Call Comes In" webhook to your ngrok URL with /voice appended (e.g., https://abcdef12345.ngrok.io/voice).
Test the recruiter: With the Rasa action server, Rasa web server, and Flask app all running, call your Twilio number. The bot will guide you through the screening process, and the results will be logged to screening_results.log.


To install and configure ngrok on Windows to expose your local Flask server, follow these steps. Ngrok creates a secure tunnel from a public internet address to a specific port on your local machine, allowing services like Twilio to send webhooks to your application. 
Step 1: Download ngrok
Navigate to the official ngrok download page at https://ngrok.com/download.
Log in or sign up for a free account. A free account is sufficient for testing.
On the download page, select Windows and download the ZIP file. You can also install it via the Microsoft Store for automatic updates. 

Step 2: Extract and set up the executable
Locate the downloaded ngrok-v<version>-stable-windows-<architecture>.zip file, and extract its contents.
Move the extracted ngrok.exe file to a dedicated folder on your system, for example, C:\ngrok. 

Step 3: Add ngrok to your system's PATH (optional but recommended)
Adding ngrok to your system's PATH allows you to run it from any Command Prompt window without having to navigate to its specific folder. 
Open the Start menu and search for "environment variables."
Click on "Edit the system environment variables".
In the "System Properties" window, click the "Environment Variables..." button.
Under "System variables," find and select the Path variable, then click "Edit...".
Click "New" and add the path to your ngrok folder (e.g., C:\ngrok).
Click "OK" on all open windows to save the changes. 

Step 4: Connect your ngrok account
Go to your ngrok dashboard.
In the "Your Authtoken" section, copy the unique command containing your authentication token.
Open a Command Prompt window (or a new one if you just changed the PATH).
Paste the command and press Enter. This will save your auth token to the ngrok configuration file. 
Step 5: Start the ngrok tunnel
Make sure your Flask server is running locally on the specified port (e.g., 5000).
In your Command Prompt, run the following command to create a public tunnel to your local server:
sh
ngrok http 5000
Use code with caution.

Ngrok will initialize a secure tunnel and display information, including "Forwarding" URLs. Copy the https URL, as this is the public address that Twilio will send webhooks to. 

Step 6: Configure Twilio
In your Twilio Console, navigate to your phone number settings.
Set the webhook for "A Call Comes In" to the ngrok forwarding URL you copied, appending the endpoint path for your Flask app (e.g., https://your-ngrok-url.ngrok.io/voice).
Ensure the webhook method is set to POST. 
