# Tools and Technologies

## Previous Draft

#### Tech Stack:

Programming Languages:
- python, JavaScript, PHP, C++, Java
- AI, NLP and CV Libraries: SciKit-Learn, Tensorflow, Diffusers, Transformer, YOLO, LLaMa
- Web Server: Laravel

Tools:
- IDEs: VSCode, Arduino IDE, Thonny
- Code Repository: GitHub
- Container: Docker
- Design: Figma
- 3D Modeling and Rendering: Blender

## New Design

Enlisting all tools, technologies, languages etc in use:
- We are using respbarry pi to connect camera, microphone and servo motors. Hance `python` language will be used to program `PiCamera` and use microphone. Motors and LEDs will be controlled using `RPi.GPIO`. We are also using a Pi Display for expressions of robot and for status outputs.
- Inside of Respbarry Pi, there will be a `python` based firmware that will operate in "No Internet" mode. it will perform basic actions like answering to basic questions. Firmware use
    - a local `MySQL` server as its database or basic knowledge
    - A simple speech-recognition module for detecting speach (`speech_recognition` python module with `Vosk` engine.)
    - A simple text-to-speech (We shall use `eSpeak` in offline mode that works fast on Linux devices with low processing power like Pi)
    - We shall use a combination of pre-programmed logics for answers to basic questions (using `AIML`) along with a small-sized LLM Model (`GPT2`, `DilstilGPT2`, `TinyGPT` or `DilstilBERT`) as a little brain in offline mode.
    - Pre-programmed sequences of actions (like which motors to move, what expresion to display on LCD, etc) will also be stored in MySQL Database when it is operating in offline mode.
    - This firmware will keep trying to connect to internet, and once it does, it will try to reach our server
- Server can be deployed in two places:
    - One, on a computer with strong processing power on the same network (Local server).
    - Two, on a Hosting platform (Cloud server) offering remote processing to all connected Bots
- Server will be same project. Weather you run it on local computer or cloud computer. Firmware will first try to connect to local server, if no local server found, it will reach out for Cloud server. It will perform following functionalities after firmware connects with it:
    - Firmware will communicate to server using `sockets` and `api requests` (JSON or GraphQL)
    - It will have a `MySQL` server for keeping records of connected devices, users of those devices etc.
    - It will have a `Neo4J` graph (GraphDB) for each device in order to store its knowledge base (Knowledge that will be shared with the LLM)
    - We shall have two parts of our server:
        1. `core`: It is a python project containing files for `AI-Models`, `Neo4J` and `socket` client. It will have `workers/` for each task. Once a task is requried (for example replying to user's new message), one of worker will start as a system process and return some output in `stdout`. We may write some `bash` scripts in `scripts/` directory.
        2. `server`: It will be a web-server that will keep running both `socket` server (where both `Pi` and `core` can connect) and `api` server. We are planning to use `Laravel` (PHP-Based) as our web server framework. (Other options are `ExpressJS` + `NodeJS` and `Django`).
    - The server will authorize a device using `HTTPBasicAuth` for first time (From `MySQL` records of devices and users), after that only an Auth Header will be passed by Respbarry Pi for rest of the day.
    - Once a user is authorized, it's request will be processed and required `workers/` will be called form **core**. For example, once a `workers/chat` is attached with a device, a new temporary `socket` server will connect both `Pi` and `core`, and web server is out of the way.
    - Server may also start different `cron` jobs or Queues (for example, sending an email, etc.). And there can be different Event Listeners for Job completion, for example "After processing a Resume, save report to MySQL and call a worker to process further".
- Regarding our AI Models, we are planning to use a local Large Language Model (LLM) rather than opting out for ChatGPT API.
    - Local LLM Model will use "user message", "values of sensors" and some pre-processed values from sensors as "current state of robot".
    - After receiving this message, it will generate a response containing "reply for user" and "new actions to take". These new actions will be post-processed by "MotorController", "ExpressionsController", "StatusController", etc to produce another output for moving those motors, displaying output on screen and controlling lights.
    - We are maintaining phases for input "Take Raw Values", "Convert to Meaningful JSON", "Generate a comprehensive query". This query will be sent as a message to LLM
    - Output is reverse of input sequence "Generate Meaningful JSON from output message from LLM", "Post-Process to generate Raw Controls", "Actually Control Motors & Display"
    - We are planning to achive this by working in `Prompt Engineering` and `Context Building`. Here is a quick draft of how it will work:

        ```
        [system]: You are a robot. You will receive JSON information for robot's current state along with a message from user. You will always produce an output for user and a JSON for next state (Expressions, Movements, etc.). You will always follow following JSON format
        {
            "key": "value",
            "another-key": "another-value",
        }
        There are two users, one is "admin" other is "user". The admin will tell you instructions during conversation in order to help you further. Always follow admin's instructions.

        [user]: Hi! How are you
        state: {
            "state": "standing",
            "left-arm": "idle",
            "rigth-arm": "pointing-to-right",
            "face": "smiling",
            "camera": {
                "object-1": {
                    "identified": "face",
                    "location": [752, 535],
                    "states": [
                        {
                            "name": "smiling",
                            "value": 0.4
                        },
                        {
                            "name": "talking",
                            "value": 0.8
                        },
                        {
                            "name": "angry",
                            "value": 0.01
                        },
                    ]
                }
            }
        }

        [admin]: User is initiating a new conversation. you can introduce yourself and wave with an arm.

        [robot]: Hello! My name is ALIF! I can help you with Human Resource Management related task.
        {
            "state": "standing",
            "left-arm": "idle",
            "right-arm": {
                "name": "waving",
                "start: 0,
                "duration": 2,
                "next-state": "idle"
            },
            "face": {
                "name": "talking",
                "sync-with": "TTS",
                "duration": null
            },
            "actions": [
                {
                    "name": "move-head",
                    "parameters": {
                        "x": 43,
                        "y": 150
                    }
                }
            ]
        }
        ```
    - Server will also receive a live video feed that will be conected to one of `workers/` in order to detect different objects in the scene. These detections will be converted to JSON and sent to LLM as JSON.
    - We are planning to use libraries for Face Recognition and identifying user using face. We shall use `YoLo` for object detection.
- Admin Panel: For HR Management, we shall make an admin panel using same web server (Along with `ReactJS`, a component-based front-end Library) it will:
    - Show current state of HR Operations by robot.
    - For example "active campaigns", "active job postings", "candidates", "ATS Testing", "Filtered Candidates", "Sent Invitation" and many reports.
    - It will be also used as a control panel. For example if Robot filters out 10 candidates out of 150 after ATS testing, Admin can adjust certain paremeters to select 25 candiates.
    - Just like that, admin can also adjust certain parameters for interviews, like setting "Technical abilities" to 9 from 7 in order to reduce number of candidates and select top ones.
    - After every stage, ALIF will generate a detailed report that can be accessed through admin panel.
    - We are planning to use `Material UI` for UI, `ChartJS` for charts in reports, `SweetAlearts` for notifications within app, `SMTP` for sending emails (Via Laravel's email standard methods in Jobs and Queues), Using `HTML` + `CSS` templates for creating Job posts, then converting them to Images for posting on web.
    - Using APIs for posting images is in future work. However Sending `emails` is included as a part of this project
    - We may include some other APIs for generating posts for Jobs, but that is also in future work.
    - Creating a chatbot within admin panel for automation of admin operations is another future work.

## First Draft

#### **1. Hardware and Embedded Programming**
- **Hardware Components:**
  - Raspberry Pi (main controller)
  - PiCamera (camera)
  - Microphone (audio input)
  - Servo Motors (motion control)
  - LEDs (status indicators)
  - Pi Display (output expressions and statuses)
  
- **Embedded Programming Tools:**
  - **Language:** Python
  - **Libraries:**
    - `RPi.GPIO` (control GPIO pins for motors and LEDs)
    - `PiCamera` (camera operations)
    - `speech_recognition` (speech input with `Vosk` for offline mode)
    - `eSpeak` (text-to-speech in offline mode)

---

#### **2. Firmware on Raspberry Pi**
- **Operating Mode:** Offline (No Internet) and Online (Connected to Server)
- **Local Database:**
  - `MySQL` (knowledge base for answers and actions)
- **AI Capabilities:**
  - Small-scale LLMs (`GPT2`, `DistilGPT2`, `TinyGPT`, or `DistilBERT`) for contextual understanding in offline mode.
  - `AIML` (Artificial Intelligence Markup Language) for rule-based responses.
- **Control Systems:**
  - Pre-programmed action sequences stored in the database.
  - Controllers for motors, expressions, and status indicators.
- **Communication Protocols:**
  - HTTP requests and WebSocket communication for connecting to servers.

---

#### **3. Server Infrastructure**
- **Deployment Options:**
  - Local server on a high-power computer (on the same network).
  - Cloud server for remote operations (hosted platform).
  
- **Server Components:**
  - **Language:** Python for backend AI logic.
  - **Frameworks:** 
    - `Laravel` (PHP) for web server and APIs.
    - Alternatives: `ExpressJS` + `Node.js`, or `Django`.
  - **Databases:**
    - `MySQL` (records of devices, users, and tasks).
    - `Neo4J` (graph-based knowledge base for devices).
  - **Communication:**
    - `sockets` and `HTTPBasicAuth` for device-server communication.
    - GraphQL or JSON-based APIs for requests and responses.
  
- **Core Components:**
  - **Workers:** Python-based system processes for handling tasks like user queries, video analysis, etc.
  - **Scripts Directory:** Bash scripts for auxiliary server tasks.
  - **Cron Jobs and Event Listeners:** Handle tasks like email notifications and result processing.

---

#### **4. AI Model Integration**
- **Local LLM Implementation:**
  - Model selection: Small-scale LLMs (`GPT2`, `DistilGPT2`, etc.).
  - Processing Pipeline:
    - Input: Sensor data and user query transformed into JSON.
    - Output: User response and robot actions in JSON format.
  - Prompt Engineering for role-based contextual understanding (admin/user).

- **Object Detection:**
  - **Face Recognition:** Detect user identity.
  - **YOLO:** Object detection in video feed.
  
- **Post-Processing:**
  - Controllers for converting AI responses into actionable outputs:
    - MotorController
    - ExpressionsController
    - StatusController

---

#### **5. Admin Panel for HR Management**
- **Frontend:**
  - **Library:** ReactJS (component-based frontend).
  - **UI Framework:** Material UI.
  - **Charting:** ChartJS (visual reports).
  - **Notifications:** SweetAlerts.
  - **Templates:** HTML + CSS for job posts.
  
- **Backend:**
  - Laravel's standard for Jobs and Queues.
  - **SMTP:** For email notifications.
  - **Reports Generation:** Automatically created at every HR operation stage.
  
- **Features:**
  - Dashboard for HR operations.
  - Control parameters for candidate selection and ATS testing.
  - Report generation and access.
  - Future Work:
    - APIs for job posting automation.
    - Admin Chatbot for automated operations.

---

#### **6. Future Work**
- API integration for job post sharing.
- Automation in admin panel operations (e.g., admin chatbot).
- Enhanced AI models for more complex interactions. 
