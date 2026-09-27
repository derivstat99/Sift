# Sift - Research Assistant

A Chrome extension that summarizes selected webpage text using Google Gemini. Highlight anything on a page, click **Summarize**, and get a concise AI-generated summary — all without leaving your browser.

The extension talks to a Spring Boot backend that handles the Gemini API communication, keeping the API credential server-side.

---

## Features

- **In-browser summarization** — highlight text on any webpage and summarize it in one click from the Chrome side panel
- **Persistent research notes** — write and save notes directly in the side panel; they are preserved across browser sessions via `chrome.storage.local`
- **Chrome Side Panel UI** — the extension lives in Chrome's native side panel, so it stays open while you continue reading the page
- **Gemini-powered** — uses `gemini-3.5-flash` via the Gemini REST API for fast, concise summaries

---

## How It Works

```
User highlights text on a webpage
            |
            v
   Chrome Side Panel (Summarize button)
    chrome.scripting reads window.getSelection()
            |
            v
   POST /api/research/process
   { content: "...", operation: "summarize" }
            |
            v
   Spring Boot — ResearchService
    builds prompt, calls Gemini via WebClient
            |
            v
   Gemini API (gemini-3.5-flash)
            |
            v
   Spring Boot parses response, returns plain text
            |
            v
   Side panel displays the summary
```

1. Clicking the extension icon opens the side panel (`chrome.sidePanel` API).
2. When **Summarize** is clicked, the extension uses `chrome.scripting.executeScript` to read the selected text from the active tab.
3. That text is sent to the Spring Boot backend as a `POST` request with `operation: "summarize"`.
4. The backend constructs the appropriate prompt, forwards the request to the Gemini REST API using `WebClient`, and parses the JSON response into a clean text string.
5. The result is returned to the extension and displayed in the side panel.
6. Notes typed into the **Research Notes** area can be saved locally with the **Save Notes** button and will be available the next time the panel is opened.

---

## Tech Stack

| Component | Technology |
|---|---|
| Chrome Extension | Manifest V3, JavaScript, HTML, CSS |
| Backend | Java 21, Spring Boot 4.1.1 |
| REST Framework | Spring Web MVC |
| Outbound HTTP | Spring WebFlux `WebClient` |
| AI Model | Google Gemini (`gemini-3.5-flash`) |
| Response Parsing | Jackson (`ObjectMapper`, `@JsonIgnoreProperties`) |
| Boilerplate | Lombok (`@Data`, `@AllArgsConstructor`) |
| Build | Maven + Maven Wrapper |

---

## Project Structure

```
Project/
├── backend/
│   └── research-assistant/
│       ├── pom.xml
│       └── src/main/java/com/research/assistant/
│           ├── ResearchAssistantApplication.java  # entry point
│           ├── config/
│           │   └── WebClientConfig.java           # WebClient bean
│           ├── controller/
│           │   └── ResearchController.java        # POST /api/research/process
│           ├── dto/
│           │   ├── ResearchRequest.java            # incoming request shape
│           │   └── GeminiResponse.java             # Gemini JSON mapping (nested DTOs)
│           └── service/
│               └── ResearchService.java            # prompt building + Gemini call
└── extension/
    ├── manifest.json     # MV3 config, permissions, side panel setup
    ├── background.js     # registers panel-on-click behavior
    ├── sidepanel.html    # panel layout
    ├── sidepanel.js      # all extension logic
    └── sidepanel.css     # panel styling
```

---

## Backend

The backend follows a straightforward Controller → Service → External API structure.

**`ResearchController`** receives `POST /api/research/process` requests and delegates to the service layer.

**`ResearchService`** builds the prompt string based on the requested operation (`summarize` or `suggest`), assembles the Gemini API request body, and uses `WebClient` to call the Gemini endpoint. The response is parsed from JSON into a `GeminiResponse` DTO — which mirrors the Gemini API's nested `candidates → content → parts → text` structure — and the extracted text is returned to the controller.

The Gemini API key is injected at runtime via `@Value("${gemini.api.key}")` and supplied through the `GEMINI_KEY` environment variable. It is never embedded in source code.

---

## Chrome Extension

The extension is built to Manifest V3 and uses the **Side Panel API** (`chrome.sidePanel`) so the UI stays visible while the user browses.

`background.js` registers a single call — `chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true })` — so clicking the toolbar icon opens the panel automatically.

`sidepanel.js` handles all runtime logic:
- Uses `chrome.scripting.executeScript` to inject a one-liner into the active tab and retrieve whatever the user has selected (`window.getSelection().toString()`).
- `fetch()`-es the Spring Boot endpoint with the selected content.
- Displays the result in the panel.
- Reads from and writes to `chrome.storage.local` for note persistence.

Permissions used: `activeTab`, `scripting`, `storage`, `sidePanel`.

---

## API

### `POST /api/research/process`

The single endpoint that receives content from the extension and returns an AI-generated result.

**Request**

```http
POST http://localhost:8080/api/research/process
Content-Type: application/json
```

```json
{
  "content": "The text you want to process...",
  "operation": "summarize"
}
```

**Supported operations**

| Value | Behavior |
|---|---|
| `summarize` | Returns a concise summary of the provided text |
| `suggest` | Returns related topics and further reading suggestions |

**Response** — `200 OK`, plain text

```
A clear, concise summary of the provided text generated by Gemini.
```

---

## Setup

### Requirements

- Java 21+
- Google Chrome
- A [Google Gemini API key](https://aistudio.google.com/)
- Maven (or use the included `mvnw` wrapper — no separate install needed)

### Backend

1. Set your Gemini API key as an environment variable:

   **Windows (PowerShell)**
   ```powershell
   $env:GEMINI_KEY = "your_api_key"
   ```

   **macOS / Linux**
   ```bash
   export GEMINI_KEY=your_api_key
   ```

2. Start the Spring Boot application:

   ```bash
   cd backend/research-assistant
   ./mvnw spring-boot:run
   ```

   On Windows:
   ```cmd
   mvnw.cmd spring-boot:run
   ```

   The API will be available at `http://localhost:8080`.

### Chrome Extension

1. Open Chrome and go to `chrome://extensions/`
2. Enable **Developer mode** (top-right toggle)
3. Click **Load unpacked** and select the `extension/` folder
4. The **Research Assistant** icon will appear in your toolbar

---

## Usage

1. Start the backend (`mvnw spring-boot:run`).
2. Load the extension in Chrome as described above.
3. Open any webpage and highlight the text you want to summarize.
4. Click the **Research Assistant** icon to open the side panel.
5. Click **Summarize** — the result appears in the panel within a few seconds.
6. Optionally, write notes in the **Research Notes** area and click **Save Notes** to keep them for your next session.

---

## Configuration

The Gemini API key and endpoint are configured through `application.properties`:

```properties
gemini.api.url=https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash:generateContent
gemini.api.key=${GEMINI_KEY}
```

The key is read from the `GEMINI_KEY` environment variable at startup, so it is not present anywhere in the source code or extension bundle.

---

## Possible Extensions

- Surface the backend's `suggest` operation through a second button in the side panel
- Add a loading indicator while the Gemini request is in progress
- Render Gemini's markdown-formatted responses with proper styling
- Add input length validation before sending content to the API
- Sync notes across devices using `chrome.storage.sync`
