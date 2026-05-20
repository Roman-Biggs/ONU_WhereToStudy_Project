# ONU_WhereToStudy_Project
Interactive Navigator for Open Audiences in University

**Project Name:** WhereToStudy  
**Main Problem:** During breaks students need to find a quiet place to study, make calls or discuss a project. Searching blindly for an empty audience is a waste of time, and department schedules are constantly changing.  
**Solution:** A web application that parses the official university schedule and displays a list of currently empty audiences in real time.  
**Target User:** Students

**User Stories:**  
1. As a student who has a "window" between classes, I want to see a list of available audiences on my current floor, indicating how long they will be available, so I can quickly find a quiet place to study and not waste time walking around the building.
2. As a student in an audience, I want to be able to check in with one click to report the current noise level and occupancy of the room, so that other students can choose a suitable location (for example, for a call or, conversely, for reading).
3. As a professor whose lecture ended early or was cancelled, I want to make the room available in the app so students can occupy it without waiting for the scheduled end of the class.

**Non-Functional Requirements:**  
- **Performance:** The response latency of the main endpoint for obtaining a list of free classrooms should not exceed **200 ms** under a peak load of up to 500 simultaneous requests per second during breaks between pairs.
- **Availability:** The web service availability rate must be at least **99.5%** during the academic semester (September–June) and operate from 8:00 AM to 10:00 PM. Scheduled maintenance and parser updates must be performed strictly at night.
- **Security:** All endpoints that change the system state (check-ins, manual audience status changes) must be protected by corporate token (JWT) authentication. Session token lifetime is limited to 12 hours, and the check-in rate per account is limited to 1 request every 5 minutes to prevent spam.

**Endpoint Contracts Design:**  
  1) **Getting a list of free audiences**
     - ***Request:*** `GET /api/v1/classrooms/free`
       
     - ***Query Parameters:***
       `building_id` *(integer, required)* — ID of the university building;
       `floor` *(integer, optional)* — Filter by a specific floor;
       `required_features` *(array of strings, optional)* — Audience requirements (e.g. `["sockets", "projector"]`).
       
     - ***Successful Response (200 OK):***
       ```json
       [
          {
            "classroom_id": 402,
            "room_number": "402а",
            "floor": 4,
            "features": ["sockets", "whiteboard"],
            "status": "free",
            "free_until": "14:20",
            "crowdsourced_status": {
              "noise_level": "silent",
              "occupancy_rate": "low",
              "last_update": "2026-05-20T14:05:00Z"
            }
          }
       ]
  2) **Submitting a Crowdsourced Check-in**
     - ***Request:*** `POST /api/v1/classrooms/{classroom_id}/checkin`
       
     - ***Headings:***
       Authorization: Bearer <token>
       
     - ***Request Body:***
       ```json
       {
         "noise_level": "moderate",
         "occupancy_rate": "medium"
       }
  
     - ***Successful Response (201 Created):***
       ```json
       {
        "status": "success",
        "message": "Отметка успешно принята",
        "updated_at": "2026-05-20T14:12:30Z"
       }
  
     - ***Bad response (401 Unauthorized):***
       ```json
       {
        "error": "AccessDenied",
        "message": "Для выполнения чекина необходима авторизация через корпоративный аккаунт."
       }

**Test Cases:**  
  ***1. Search for available rooms by schedule:***
  - **Given:** The database for Building 1 lists room 302. According to the official schedule, there are no classes between 12:00 and 14:00. The current system time is 12:30.
  - **When:** User sends request `GET /api/v1/classrooms/free?building_id=1&floor=3`.
  - **Then:** The system returns an array of objects that contains audience 302, its status is *free*, and the *free_until* field contains the value *13:30* (the start time of the next scheduled class).

  ***2. Dynamic noise level update via crowdsourcing:***
  - **Given:** Room 105 is displayed as "available" in the system, but the *noise_level* field is *unknown* (no recent student data). The user is logged in to the app.
  - **When:** User sends request `POST /api/v1/classrooms/105/checkin` with request body `{"noise_level": "silent", "occupancy_rate": "low"}`.
  - **Then:** The system returns a *201 Created* response, updates the audience status cache, and when the available seats list for audience 105 is retrieved again, the noise level status is *silent*.

  ***3. Protecting the system from unauthorized spam with statuses:***
  - **Given:** An anonymous web user (request headers are missing a valid Bearer token) is on audience page 210.
  - **When:** The user attempts to simulate clicking the check-in button and sends a request `POST /api/v1/classrooms/210/checkin`.
  - **Then:** The system blocks the transaction, returns HTTP status *401 Unauthorized*, and the global audience state 210 in the database remains unchanged.
