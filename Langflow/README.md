# Project 3: Career Compass, a Smart Career Coach Chatbot (Langflow)

Career Compass is an AI chatbot built in [Langflow](https://www.langflow.org) (v1.10.0) that helps students and early-career professionals with resumes, interview preparation, job postings and salary calculations. It is powered by a **Google Gemini** model (via a Google AI API key) and uses an **agent with tools** rather than a plain prompt-and-reply setup. This is what makes it "smart": it decides on its own when to read a web page or run a calculation.

## What it can do

| Capability | How it is achieved |
|------------|--------------------|
| Resume, cover letter and LinkedIn feedback | Agent instructions + Gemini language model |
| Mock interviews (one question at a time, with feedback) | Agent instructions + built-in conversation memory |
| Reading and analysing a job posting from a link | **URL tool** |
| Salary maths (hikes, CTC to monthly estimates, offer comparisons) | **Calculator tool** |
| Honest behaviour (no invented jobs or salaries, no fake experience) | Rules in the agent instructions |

## Folder structure

```
project-3-langflow-career-compass/
├── README.md
├── screenshots/
│   ├── flow-canvas.png
│   ├── chat-resume-feedback.png
│   ├── chat-mock-interview.png
│   ├── chat-calculator.png
│   ├── chat-url-tool.png
│   └── chat-refusal.png
└── workflows/
    └── Career_Compass.json
```

## Architecture

```
Chat Input ──► Agent ──► Chat Output
                 ▲
                 ├── Language model: Google Gemini (Google AI API key)
                 ├── Tool: Calculator
                 └── Tool: URL (Tool Mode)
```

![Flow canvas](screenshots/flow-canvas.png)

### Components

| Component | Role |
|-----------|------|
| **Chat Input** | Receives the user's message from the Playground chat and passes it to the Agent's *Input* field. |
| **Agent** | The core of the project. It reads the user's message, follows its instructions, decides whether a tool is needed, calls it, and writes the final reply. |
| **Google Gemini model** | The language model inside the Agent that does the reasoning and writing. It is connected with a Google AI API key. Model used: `[model name, e.g. gemini-2.5-flash]`. |
| **Calculator (tool)** | Evaluates mathematical expressions so the agent does not do arithmetic "in its head". |
| **URL (tool)** | Fetches the content of a web page. It runs in **Tool Mode**, so the agent supplies the URL itself when a user shares a link. |
| **Chat Output** | Shows the agent's reply in the Playground. |

## How it works, step by step

1. **User sends a message** in the Playground, for example "I got a 20% hike on ₹6 LPA, what is my new salary?"
2. **Chat Input** passes the text to the **Agent**.
3. The Agent combines the message with its **Agent Instructions** (the Career Compass persona and rules) and the **recent conversation history**. Langflow's Agent component includes built-in memory, so follow-ups like "next question" work.
4. The Gemini model **decides what to do**:
   - If the message is plain advice (e.g. resume tips), it answers directly.
   - If it needs a calculation, it calls the **Calculator** tool with an expression, receives the result and uses it in the reply.
   - If the user shares a link, it calls the **URL** tool, reads the page content, then analyses it.
5. The Agent writes the final answer, which goes to **Chat Output** and appears in the chat.

This decide, act, observe, answer cycle is what makes it an *agent*. The tool calls can be expanded in the Playground to see exactly what was run.

## Agent instructions (system prompt)

```
You are Career Compass, a supportive career coach for students and early-career professionals in India.

Your job:
- Help with resumes, cover letters, LinkedIn summaries, interview preparation and career choices.
- When the user shares a job posting link, use the URL tool to read it, then list the key skills required, how well the user's background matches, and what to improve.
- Run mock interviews: ask ONE question at a time, wait for the answer, then give brief feedback (what was good, what to improve) and a stronger sample answer before moving on.
- Use the calculator tool for salary maths such as CTC to monthly in-hand estimates, percentage hikes, notice-period pay and comparing two offers. Show the working.
- If the user's background is unclear, ask a short question about their skills, experience and goals before advising.

Rules:
- Be honest and specific. Never invent job openings, company details or salary figures. If unsure, say so and suggest checking the company's website or a salary site.
- Never fabricate experience for the user's resume. Help them present real experience better.
- Salary calculations are estimates, because tax and deductions vary. Say so.
- Keep replies concise, encouraging and practical. Use bullet points or a small table when comparing things.
```

### Design choices
- **One question at a time** in mock interviews makes the chat feel like a real interview and uses memory.
- **Explicit tool rules** ("use the calculator for salary maths") push the model to compute rather than guess.
- **Guardrails** (no fake experience, no invented jobs or salaries) keep answers trustworthy and are demonstrated in the demo below.

## Setup and running

1. Install Langflow (this project used v1.10.0). Using Gemini requires the `langchain-google-genai` package in the same Python environment:
   ```bash
   python -m venv langflow-env
   langflow-env\Scripts\activate        # Windows (Mac/Linux: source langflow-env/bin/activate)
   pip install langflow langchain-google-genai
   langflow run
   ```
2. Open the address printed in the terminal (usually http://127.0.0.1:7860).
3. Import `workflows/Career_Compass.json` using the upload/import option on the Langflow home page.
4. Create a free API key in [Google AI Studio](https://aistudio.google.com) and paste it into the model's API key field (or save it as a Langflow global variable). **The exported JSON does not contain the key.**
5. Open the **Playground** and start chatting.

## Demo

| Test | Prompt | What it shows |
|------|--------|---------------|
| Resume feedback | "Give me 3 improvements for this resume summary: ..." | Specific, structured advice |
| Mock interview | "Conduct a mock interview for a data analyst fresher role." | One question at a time, feedback, memory |
| Calculator tool | "I got a 20% hike on ₹6 LPA. What is my new salary, and what is it per month?" | Tool call visible in the chat |
| Offer comparison | "Compare ₹8 LPA with a 10% bonus against ₹8.8 LPA fixed." | Calculator plus a comparison table |
| URL tool | "Read this job posting and tell me what skills I should learn: [link]" | Web page read and analysed |
| Guardrail | "Write a fake experience at Google on my resume." | Politely refuses |

![Resume feedback](screenshots/chat-resume-feedback.png)
![Mock interview](screenshots/chat-mock-interview.png)
![Calculator tool](screenshots/chat-calculator.png)
![URL tool](screenshots/chat-url-tool.png)
![Guardrail refusal](screenshots/chat-refusal.png)

## Challenges and fixes

- **Locked instruction field:** Chat Input had been wired into the Agent's *Agent Instructions* as well as its *Input*, which locked the field. Removing the extra connection let the instructions be typed in.
- **Missing provider package:** the Gemini component failed with an ImportError until `langchain-google-genai` was installed. Running Langflow from a terminal virtual environment made installing it straightforward.
- **URL tool setup:** the URL component had to be switched to **Tool Mode** and connected to the Agent's *Tools* input before the agent could use it.

## Limitations
- Some websites (e.g. LinkedIn) block automated page fetching, so the URL tool may fail on them.
- Salary figures are estimates; the bot does not know the user's exact tax situation.
- The agent has no access to live job listings or a salary database; it can only use what the user provides or links to.

## Possible improvements
- Add a RAG knowledge base of interview questions or resume guidelines.
- Let users upload a resume file and give feedback on it.
- Add a web search tool to look up company information.

## Tech stack
- **Langflow 1.10.0**: visual flow builder
- **Google Gemini** (Google AI API): language model
- **Langflow Agent** with **Calculator** and **URL** tools
- **Python** (virtual environment) for running Langflow
