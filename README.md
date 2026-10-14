# Interview AI

Interview AI is a full-stack application that turns a job description and a
candidate profile into a personalised interview-preparation plan. A user can
register, upload a resume or provide a self-description, and receive an
AI-generated report containing a match score, technical questions, behavioural
questions, skill gaps, and a day-by-day preparation roadmap.

The application is organised as two independently runnable packages:

- `Frontend/` — React client application served by Vite.
- `Backend/` — Express API with MongoDB persistence and Google Gemini
  integration.

## Features

- User registration, login, session restoration, and logout.
- Cookie-based authentication using JSON Web Tokens.
- Protected interview-planning and report routes.
- Job-description input with a 5,000-character client-side limit.
- Resume upload with a client-side PDF/DOCX selector.
- Optional self-description for users who do not have a resume available.
- Resume text extraction on the server.
- AI-generated interview report with:
  - Profile-to-job match score.
  - Technical interview questions, interviewer intent, and suggested answers.
  - Behavioural interview questions, interviewer intent, and suggested answers.
  - Skill gaps with low, medium, or high severity.
  - A day-wise preparation plan.
- Recent report history for the authenticated user.
- Expandable report sections for technical and behavioural questions.
- AI-generated, downloadable, ATS-oriented resume PDF tailored to the report.

## Technology stack

### Frontend

- React 19
- React Router 7
- Vite
- Axios
- Sass
- ESLint

### Backend

- Node.js with Express 5
- MongoDB with Mongoose
- Google Gemini via `@google/genai`
- JWT and `bcryptjs` for authentication
- `cookie-parser` and `cors`
- Multer for in-memory file uploads
- `pdf-parse` for extracting resume text
- Puppeteer for rendering generated resume HTML as a PDF
- Zod and `zod-to-json-schema` for the Gemini response schema

## How the application works

1. The frontend sends the job description, self-description, and resume file to
   `POST /api/interview/`.
2. The backend extracts text from the uploaded resume.
3. The backend sends the candidate information and job description to Gemini.
4. The structured Gemini response is validated through the configured Zod
   schema and stored in MongoDB as an interview report.
5. The frontend displays the saved report and its preparation sections.
6. When requested, the backend asks Gemini for tailored resume HTML and uses
   Puppeteer to convert it to a downloadable PDF.

## Prerequisites

- Node.js and npm.
- A running MongoDB instance or MongoDB Atlas connection.
- A Google Gemini API key with access to the configured model.

The backend currently uses the `gemini-3-flash-preview` model. Model
availability and billing depend on the Google AI account and API configuration.

## Configuration

Create `Backend/.env` with:

```env
MONGO_URI=mongodb://127.0.0.1:27017/interview-ai
JWT_SECRET=replace-with-a-long-random-secret
GOOGLE_GENAI_API_KEY=replace-with-your-google-genai-key
```

Do not commit this file. `Backend/.gitignore` already excludes `.env`.

The current source code uses these development URLs:

- Frontend: `http://localhost:5173`
- Backend: `http://localhost:3000`

The frontend API clients and backend CORS configuration currently contain these
URLs directly in source code. For a deployed environment, replace them with
environment-based configuration and update CORS accordingly.

## Installation and local development

Install dependencies in both packages:

```bash
cd Backend
npm install

cd ../Frontend
npm install
```

Start MongoDB, then run the backend in one terminal:

```bash
cd Backend
npm run dev
```

Run the frontend in a second terminal:

```bash
cd Frontend
npm run dev
```

Open the Vite URL shown in the terminal, normally
`http://localhost:5173`.

The backend's `dev` script uses `nodemon`. The backend does not currently
define a production `start` script; for a simple non-watch run, use:

```bash
cd Backend
node server.js
```

## Available scripts

### Frontend

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the Vite development server |
| `npm run build` | Create a production frontend build |
| `npm run preview` | Preview the production build locally |
| `npm run lint` | Run ESLint |

### Backend

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the API with Nodemon |
| `node server.js` | Start the API without watch mode |
| `npm test` | Currently exits with an intentional “no test specified” error |

## API overview

All interview routes require the authentication cookie issued during login or
registration.

### Authentication

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/auth/register` | Create an account and issue a token cookie |
| `POST` | `/api/auth/login` | Authenticate and issue a token cookie |
| `GET` | `/api/auth/logout` | Clear the cookie and blacklist the token |
| `GET` | `/api/auth/get-me` | Return the current authenticated user |

### Interviews and reports

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/interview/` | Generate and save a report; expects multipart field `resume` plus `jobDescription` and `selfDescription` |
| `GET` | `/api/interview/` | List the current user's reports |
| `GET` | `/api/interview/report/:interviewId` | Fetch one report owned by the current user |
| `POST` | `/api/interview/resume/pdf/:interviewReportId` | Generate a tailored resume PDF |

## Project structure

```text
.
├── Backend/
│   ├── server.js
│   └── src/
│       ├── config/          # MongoDB connection
│       ├── controllers/     # Auth and interview request handlers
│       ├── middlewares/     # Authentication and file upload middleware
│       ├── models/          # User, blacklist, and report schemas
│       ├── routes/          # Express route definitions
│       └── services/        # Gemini and Puppeteer integrations
├── Frontend/
│   └── src/
│       ├── features/auth/       # Auth context, pages, API, and guard
│       └── features/interview/  # Form, report view, state, and API
└── README.md
```

## Current shortcomings and risks

This is a functional prototype rather than a production-ready service.

### Correctness and product limitations

- The UI advertises PDF or DOCX uploads, but the backend passes the file to
  `pdf-parse`; DOCX files are not supported by the current extraction path.
- The UI says the maximum upload size is 5 MB, while Multer enforces a 3 MB
  limit. The client and server should use one consistent limit.
- The backend does not currently enforce a PDF MIME type or file extension and
  does not clearly validate that at least one of resume or self-description was
  supplied.
- The frontend does not provide robust form validation, upload-progress
  feedback, or user-facing error messages. Several API errors are only logged
  to the console, and failed requests can lead to undefined data being used.
- There are no automated backend or frontend tests, and the backend test script
  is still the default placeholder.
- There is no pagination, report deletion, retry mechanism, rate limiting, or
  usage/cost control for expensive AI requests.
- Gemini output is assumed to be valid JSON after the API call. Additional
  runtime validation and explicit error handling would make failures safer.
- Puppeteer requires a compatible Chromium environment, which may require
  additional configuration in some production or container deployments.

### Security and privacy

- Auth cookies are created without explicit `httpOnly`, `secure`, and
  `sameSite` options. These should be configured before production use.
- The resume-PDF controller looks up a report by ID without also constraining
  it to `req.user.id`. An authenticated user who obtains another report ID may
  therefore be able to request a PDF generated from another user's stored
  resume data.
- Uploaded resumes and self-descriptions contain personal information and are
  stored in MongoDB. The project does not currently document retention,
  deletion, encryption, or privacy-consent policies.
- The blacklist stores complete JWT values and has no cleanup or expiration
  strategy, so it can grow indefinitely.
- There is no CSRF protection for cookie-authenticated state-changing requests.
- The Gemini prompt directly includes user-supplied resume and job-description
  text. Production deployments should consider prompt-injection handling,
  sensitive-data minimisation, provider retention settings, and request limits.

### Deployment and maintainability

- API URLs and the allowed frontend origin are hardcoded to localhost.
- There is no centralized error-handling middleware, request logging, health
  endpoint, schema migration strategy, or production process configuration.
- The frontend and backend do not share generated API types, so response-shape
  changes can fail at runtime.
- The application currently has no documented deployment, observability,
  backup, or secret-rotation process.

## Suggested next steps

1. Add ownership checks to every report operation, especially resume PDF
   generation.
2. Move API URLs, CORS origins, model names, and upload limits to validated
   environment configuration.
3. Add secure cookie flags, CSRF protection, rate limiting, and file
   validation.
4. Support DOCX explicitly or change the UI to advertise PDF only.
5. Add request validation, centralized error responses, and visible frontend
   error states.
6. Add unit/integration tests for authentication, ownership boundaries, report
   generation, and upload failures.
7. Add privacy controls, report deletion/retention, and operational
   documentation before handling real candidate data.

