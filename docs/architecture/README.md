# BookCode architecture

BookCode lets people practise the coding exercises from their textbooks, LeetCode-style. A user uploads a book's PDF; an AI extracts the code exercises, writes test cases and a harness for each, and stores them in one shared catalogue. Anyone signed in can then pick a question, write code, and run it against the tests.

This folder holds the planning diagrams (PlantUML) and the log of every design decision behind them.

| File | What it shows |
|---|---|
| [`component-diagram.puml`](component-diagram.puml) | The runtime pieces: React SPA, Express API, MongoDB, Gemini, Judge0, and how they talk |
| [`class-diagram.puml`](class-diagram.puml) | The domain model, i.e. what is stored in MongoDB, plus the few non-stored types the flows rely on |
| [`sequence-sign-up.puml`](sequence-sign-up.puml) | Creating an account |
| [`sequence-sign-in.puml`](sequence-sign-in.puml) | Admin seeding at start-up, signing in, session checks, signing out |
| [`sequence-add-book.puml`](sequence-add-book.puml) | Uploading a PDF through to a finished (or failed) edition |
| [`sequence-attempt-question.puml`](sequence-attempt-question.puml) | Opening a question, keeping a draft, running code |

## Rendering the diagrams

- **VS Code:** the PlantUML extension (`jebbs.plantuml`) previews `.puml` files. It needs Java, and Graphviz for the component and class diagrams.
- **Command line:** `java -jar plantuml.jar -tsvg docs/architecture/*.puml` writes an SVG next to each file.
- The add-book diagram is taller than PlantUML's default PNG limit of 4096 px. Render it as SVG, or add `-DPLANTUML_LIMIT_SIZE=16384` after `java` for PNG.

## Decision log

Nothing in the diagrams should be an unstated decision. Every choice they depend on is listed here with its status, and the diagrams cite these IDs (for example `(P4)`) where a choice shows up.

| Status | Meaning |
|---|---|
| **D** Decided | Michael decided it. |
| **A** Assumed | Claude stated it as an assumption during planning and Michael didn't object. |
| **P** Proposed | Claude's proposal, drawn in the diagrams, **waiting for Michael's confirmation**. |
| **O** Open | Not decided yet. Doesn't change these diagrams. |

### Decided

**Users and sign-in**

| ID | Decision |
|---|---|
| D1 | Two roles: User and Admin. Signing up always creates a User. |
| D2 | The admin account is created when the API starts, from environment variables, and signs in through the normal sign-in form. |
| D3 | Sign-in is by email and password. |
| D4 | Sessions are stored server-side in MongoDB; the browser only holds an httpOnly session cookie. |

**Books and editions**

| ID | Decision |
|---|---|
| D5 | "Adding a book" means uploading a PDF for the AI to process. There is no personal library: every processed book is shared. |
| D6 | An edition that has been processed successfully is never processed again, because AI output isn't deterministic. Questions belong to a specific edition. |
| D7 | An edition is identified by main title + authors + edition number. The main title is the text before the first colon, compared ignoring case and punctuation. Authors are compared as a set of surnames, ignoring order. A book with no stated edition number is the 1st edition. |
| D8 | The uploader approves the detected title, authors and edition before processing continues. |
| D9 | An upload that hasn't been approved within 24 hours is discarded. |
| D10 | The PDF is deleted once processing ends. Questions are stored word for word: this is a non-commercial proof of concept, to be revisited before any production use. |
| D11 | A question that appears in several editions is stored as a separate copy in each, for now. |

**Processing an upload (ingestion)**

| ID | Decision |
|---|---|
| D12 | Ingestion runs inside the Express process. |
| D13 | An edition is locked while it's being processed, and the lock expires if the job stops making progress. A failed job doesn't lock the edition: it can be retried, and a retry starts over. |
| D14 | If someone uploads an edition that's already being processed, they get a message and nothing else happens. |
| D15 | There is no "my uploads" page. A book being processed appears like any other book, marked as work in progress, with live progress over Server-Sent Events (SSE). |
| D16 | A book's questions become usable only when the whole book has finished processing. |
| D17 | The LLM is Gemini on the free tier, sent the PDF pages directly rather than extracted text. Chosen knowing that Gemini's terms allow only paid services for apps made available to users in the UK, EEA or Switzerland, and that free-tier daily quotas are low. |
| D18 | AI-generated questions go live without review. Users report problems; an admin fixes them later, for example by changing a test case. |
| D19 | The AI doesn't write or check a reference solution. |
| D20 | Abstract (non-code) questions are discarded. |

**Questions and tests**

| ID | Decision |
|---|---|
| D21 | Supported languages: Python, JavaScript, Java. Books or questions in any other language are ignored. |
| D22 | The language is pinned per question. The version is pinned if the book specifies one; otherwise the latest version we support is used. If a pinned version isn't supported, the question is discarded. |
| D23 | Questions are function-signature style. The AI writes a harness for each question that converts inputs and outputs. |
| D24 | Each question has 2–3 visible test cases; the rest are hidden. When a test fails it is shown to the user, even if it's hidden. |
| D25 | Outputs must match exactly, in both value and type: `4` ≠ `4.0`, `'cat'` ≠ `' cat'`. This applies wherever the language can tell the types apart (Python and Java; JavaScript can't distinguish `4` from `4.0`). There is no float tolerance. Order is ignored where it doesn't matter. |
| D26 | The harness only prints outputs with their types; the comparison happens in our backend. The AI sets an "order doesn't matter" flag per question, which admins can change. |

**Running code**

| ID | Decision |
|---|---|
| D27 | There is a single Run button and no submission history. Only completion is tracked. |
| D28 | Each Run is one Judge0 submission. All tests run, with expected outputs kept out of the sandbox; the backend compares and shows only the first failing test. |
| D29 | The user's code is saved in localStorage so a refresh doesn't lose it. It is cleared when the question is passed, and after 24 hours. |
| D30 | Code runs on self-hosted Judge0 on a VPS (Ubuntu 24.04, booted with cgroup v1), using a custom image with current Python, Java and Node.js versions. Judge0's stock image only ships Python 3.8, Java 13 and Node.js 12. |

**Scope and stack**

| ID | Decision |
|---|---|
| D31 | In v1: reporting a question. Not in v1: submission history, server-side drafts, discussion, reference solutions, hints. |
| D32 | Admins can create, edit and delete books, editions, questions and test cases, edit AI output, manage users, and handle reports. |
| D33 | Portfolio/demo scale. Stack: TypeScript, React, Express, MongoDB. |
| D34 | Diagrams: one component diagram, one class diagram, and four sequence diagrams (sign up, sign in, add a book, attempt a question), written in PlantUML and kept in this repo. |

### Assumed (stated during planning, not objected to)

| ID | Assumption |
|---|---|
| A1 | No email verification and no password reset in v1. |
| A2 | Any signed-in user can upload a book. Admins can upload too, and can also write questions by hand. |
| A3 | You must be signed in both to browse the catalogue and to run code. |
| A4 | The catalogue is structured Book → Edition → Chapter → Question. |
| A5 | "Latest version" means the latest version our Judge0 image offers for that language. |
| A6 | Completion is a per-user, per-question flag, set when every test passes. It stays set if an admin later changes that question's tests. |
| A7 | Any signed-in user can report a question, optionally pointing at one test case, with a category and free text. A report's status goes from open to resolved or dismissed. Reporters aren't notified, and nothing is hidden automatically. |

### Proposed: needs confirmation

| ID | Proposal | Shows up in |
|---|---|---|
| P1 | Signing up signs you straight in. | sign-up |
| P2 | On the approval screen the uploader can correct the detected title, authors and edition before approving, or cancel the upload. The screen's URL contains the job ID, so a refresh returns to it. | add-book |
| P3 | The duplicate check and the lock happen together at approval, as one atomic update on the edition (unique on book + edition number). | add-book |
| P4 | The lock is refreshed after every LLM call and counts as expired after 10 minutes without a refresh. Every Gemini call has a timeout shorter than that. When the API starts, any job still marked as processing is failed immediately, because an in-process job can't survive a restart. | add-book |
| P5 | When a job fails, its chapters and questions are deleted, the PDF is deleted, and the edition is marked failed. The book page says processing failed; anyone can retry by uploading the PDF again. | add-book, class |
| P6 | Every signed-in user can see a work-in-progress book and its live progress, but can't open its questions until it's ready. | add-book |
| P7 | Processing works in three steps. **(1)** Gemini reads the first 15 pages to find the title, authors and edition. **(2)** After approval, Gemini reads the whole PDF and lists the chapters with their page ranges. **(3)** For each chapter, we cut out its pages and Gemini returns that chapter's code questions: one request per chapter, chapters in sequence. If Gemini hits a per-minute rate limit, we wait and retry. If the daily quota runs out, or Gemini keeps returning output we can't parse, the job fails (see P5). A single question that doesn't match the expected format is discarded. | add-book |
| P8 | Drafts are stored in localStorage as `{code, savedAt}`, keyed by user and question. Entries older than 24 hours are deleted when the app loads. (An LRU cache evicts by count, not age, so a timestamp sweep is what D29 actually describes.) | attempt-question |
| P9 | Harness contract. Judge0 receives the user's code and the harness as one source file; for Java, that's the user's `Solution` class plus the harness's `Main` class. stdin carries only the test inputs, as JSON. The harness prints one marked line per test, containing the typed result or the exception. Any unmarked output is the user's own printing and is shown to them as stdout. | attempt-question |
| P10 | The API calls Judge0 synchronously (`wait=true`). CPU-time and memory limits are set per language, not per question. | attempt-question |
| P11 | The backend talks to Gemini and Judge0 through interfaces (`LlmClient`, `CodeRunner`), so either can be replaced. | component |
| P12 | Questions written by an admin fill in the same fields as AI-generated ones, including the harness and the test cases. | class |
| P13 | Report categories: wrong test case, wrong statement, broken harness, other. | class |
| P14 | SSE progress events go through an in-process event bus. This means only one API instance can run, which already follows from D12. | component, add-book |
| P15 | Time-limit and runtime errors are reported against the test that was running: the first test with no result line. | attempt-question |
| P16 | The session cookie is `HttpOnly; Secure; SameSite=Lax`. SameSite is our CSRF protection, which requires the SPA and the API to be served from the same site. | sign-up, sign-in |

### Open: doesn't change these diagrams

| ID | Question |
|---|---|
| O1 | Which Gemini model on the free tier? A September 2026 forum post reports about 20 requests/day for Gemini 3.8 Flash and about 500 for Flash-Lite. A book needs one request per chapter plus two (P7). With 20 a day, most books can't finish within one day's quota, and a failed job starts over. |
| O2 | Judge0 for local development: a local Ubuntu 24.04 VM (free, uses about 4 GB of RAM) or the VPS itself. |
| O3 | Where the Express API and MongoDB are hosted. |
| O4 | Upload limits: maximum file size, uploads per user per day. |
| O5 | The encoding for typed values in tests and harness output, and whether "order doesn't matter" also applies to nested lists (e.g. "return all subsets"). To settle when we write the harness contract. |
| O6 | The actual per-language time and memory limits. |
| O7 | Password rules, session lifetime, rate limiting on sign-in. |
