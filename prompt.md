# Build UN-BURY — The Unread Problem: What Did I Miss?

You are an expert full-stack developer, UI/UX designer, and software tester. Build a complete, polished, functional hackathon project called **UN-BURY**.

I have approximately 3 hours to complete this project. I am a beginner, so make sensible technical decisions, avoid unnecessary complexity, and prioritize a working, deployable application over ambitious features that might break.

**Do not merely explain how to build it. Create the actual project files in the connected GitHub repository.**

## 1. Problem statement

People return to hundreds of unread messages across college groups, project chats, and personal conversations. Important information gets buried in the noise.

They may miss:

* Assignment deadlines and deadline changes.
* Important announcements.
* Decisions made by a team.
* Tasks assigned directly to them.
* Meeting time or location changes.
* Questions or requests that need their response.

UN-BURY should answer one central question:

**“What did I miss, and what do I need to do about it?”**

The application must help users understand conversations quickly, prioritize what matters, and identify actionable information without reading every message.

## 2. Mandatory privacy requirement

The problem statement requires conversations, extracted data, and summaries to remain on the user's device.

Therefore:

* All message analysis must happen locally in the browser.
* Never send pasted conversation text to an external AI API, backend, analytics provider, or third party.
* Do not create a server that receives chat content.
* Do not add cloud-based summarization.
* Do not claim to use a local AI model unless one is actually implemented and running locally.
* Do not claim perfect privacy without verifying the implementation.

For the first working version, implement a transparent, deterministic, browser-based analysis engine using JavaScript or TypeScript. Make it modular so an actual local AI model can be added later if feasible.

The UI must clearly explain that analysis currently uses local text-processing rules, not a cloud generative AI model.

## 3. Technology and repository

Inspect the existing repository before making changes.

Use React, Vite, and TypeScript if practical. Otherwise, select the simplest reliable stack that supports the required features and static deployment.

Requirements:

* Create all necessary application files in the repository.
* Include a valid package.json and lockfile if the selected package manager generates one.
* Include all necessary source files and configuration.
* Include a README.md with installation, running, testing, privacy, and deployment instructions.
* Do not overwrite useful existing files without inspecting them.
* Do not commit secrets, API keys, credentials, or private conversation data.
* Avoid unnecessary dependencies, authentication, databases, and backend infrastructure.

The app must be suitable for deployment as a static website on Vercel, Netlify, or a comparable static host.

## 4. Product design

Build a premium, modern, professional dashboard branded exclusively as **UN-BURY**.

Never use the name MIA or Missed Information Assistant as the product name.

Product tagline: **“Unbury what matters.”**

Suggested visual direction:

* Elegant dark navy or charcoal background.
* Restrained emerald, mint, or teal accents.
* Clean typography and strong visual hierarchy.
* Rounded cards, subtle borders, tasteful shadows, and consistent spacing.
* Professional, polished design without unnecessary visual clutter.
* Responsive layouts for desktop, tablet, and mobile.
* Accessible color contrast, keyboard navigation, and visible focus states.

Create a cohesive interface that feels like a real productivity product rather than a generic AI-generated landing page.

### Header

Include:

* UN-BURY logo/wordmark.
* Tagline: “Unbury what matters.”
* A visible “Processed locally” indicator.
* A clear/reset action.

### Main input panel

Include:

* A large text area for pasted chat messages.
* An optional field for the user's name to identify direct mentions.
* A “Try sample conversation” button.
* An “Analyze conversation” button.
* A clear character or message count if useful.
* Input validation and helpful error messages.
* A brief privacy explanation.

### Results dashboard

After analysis, display:

**A. TL;DR**
A concise overview of the conversation based on the actual input.

**B. Priority overview**
A clear visual breakdown of urgent, high, medium, and low-priority findings.

**C. Urgent**
Deadlines, imminent commitments, critical changes, and other high-priority findings.

**D. Your actions**
Tasks, direct requests, unanswered questions, and responsibilities that may concern the user.

**E. Decisions**
Agreements, finalized plans, and decisions made in the conversation.

**F. Schedule changes and announcements**
Meeting changes, location changes, important updates, and announcements.

**G. Evidence**
The original messages supporting every finding.

Include category and priority filters, urgency sorting, useful empty states, and a clear way to return to the input.

Every visible control must work. Do not include decorative buttons that do nothing.

## 5. Local analysis engine

Create a separate, testable module for analyzing conversations.

It must support common chat formats such as:

[10:30 AM] Priya: The submission deadline has moved to tomorrow at 5 PM.
[10:32 AM] Rahul: I will prepare the slides.
[10:35 AM] Priya: Thanushka, please finish the introduction today.
[10:40 AM] Rahul: We agreed to meet in Lab 3 at 2 PM.
[10:45 AM] Priya: The meeting room has changed to Lab 5.

Also support plain-text input without timestamps or sender names, as far as reasonably possible.

Implement these capabilities:

### Message parsing

* Split conversations into individual messages where possible.
* Preserve original message text.
* Extract sender and timestamp when reliably identifiable.
* Handle imperfect formatting gracefully.

### Deadline detection

Detect common phrases such as:

* Today and tomorrow.
* Due dates and submission deadlines.
* Specific dates.
* Times and time changes.
* Phrases indicating a deadline has moved or become more urgent.

Do not invent a specific calendar date if the text does not provide enough information. Distinguish explicit deadlines from inferred urgency.

### Action extraction

Detect phrases such as:

* “Please submit…”
* “Send me…”
* “Finish…”
* “You need to…”
* “Can you…?”
* “I will…”
* “Don't forget to…”

Where possible, distinguish tasks assigned to the user from tasks assigned to other people.

### Direct mentions

* Detect the optional user name in messages.
* Highlight relevant direct requests and mentions.
* Avoid treating every occurrence of a name as an actionable task.

### Decision detection

Identify phrases such as:

* “We agreed…”
* “We decided…”
* “The final plan is…”
* “Let's go with…”
* “Confirmed…”

### Schedule changes and announcements

Identify changes to:

* Meeting times.
* Locations.
* Deadlines.
* Plans.
* Important group announcements.

### Prioritization

Assign transparent priorities based on evidence:

* Urgent: imminent deadlines or time-critical changes.
* High: important tasks, direct requests, and significant changes.
* Medium: relevant decisions, upcoming events, and useful announcements.
* Low: general information and less time-sensitive messages.

Do not classify everything as urgent. Explain briefly why each item received its priority.

### Summary generation

Generate a concise summary using the actual conversation content and extracted findings.

Do not hardcode results for the sample conversation. The sample must pass through the same analysis engine as any user-provided conversation.

### Evidence and uncertainty

Every finding must link to its supporting original message.

Never invent messages, decisions, senders, dates, tasks, or facts. If extraction is uncertain, label it appropriately. Keep the evidence visible so the user can verify the results.

## 6. Demo experience

Include a “Try sample conversation” button that loads realistic example messages involving:

* An assignment deadline moved earlier.
* A task assigned to the user.
* A team decision.
* A meeting time or location change.
* A casual message that should not be prioritized.

The user must be able to click Analyze and see the actual engine produce the results.

Include a “Clear all” function that removes the current input and results.

Provide helpful feedback for empty input, malformed text, and conversations with no significant findings.

## 7. Functional controls

Implement and test:

* Analyze conversation.
* Load sample conversation.
* Clear input and results.
* Filter by priority.
* Filter by category.
* Sort by urgency.
* Copy the summary.
* Export the analysis as a downloadable text or JSON file.
* Switch between results sections.
* Optional user-name mention detection.

Use proper browser APIs and handle failures gracefully.

Do not implement fake authentication, fake integrations, fake AI calls, or nonfunctional buttons.

## 8. Privacy, security, and accessibility

* Process all pasted text locally.
* Make no network requests containing conversation content.
* Avoid analytics and tracking.
* Render pasted text safely; never inject user content as HTML.
* Validate input and impose a sensible maximum size.
* Avoid logging conversation contents.
* Store messages and summaries in memory by default.
* Provide a clear/reset action that removes the current conversation and results.
* If browser persistence is added, make it optional, document it, and provide a way to clear stored data.
* Use semantic HTML, accessible labels, keyboard navigation, and visible focus indicators.
* Handle errors without exposing sensitive information.

Verify that the actual implementation supports every privacy claim displayed in the UI.

## 9. Testing and quality

Write tests for:

* Empty input.
* Plain-text conversations.
* Timestamped conversations.
* Deadline detection.
* Direct mention detection.
* Action extraction.
* Decision detection.
* Schedule changes.
* Priority ranking.
* Evidence preservation.
* Malformed input.
* Conversations with no actionable findings.

Run the tests and production build if the environment permits. Fix errors rather than simply reporting them.

Ensure the website is usable on mobile and desktop and that all controls function.

Do not claim that tests passed unless they were actually run.

## 10. README and hackathon submission

Write a complete README.md containing:

1. Project name: UN-BURY.
2. Tagline: “Unbury what matters.”
3. Problem statement.
4. Solution and core features.
5. Technology stack.
6. How local analysis works.
7. Known limitations.
8. Privacy and security design.
9. Installation and exact commands to run the app.
10. Testing instructions.
11. Static deployment instructions.
12. AI-services disclosure.

Be honest about the AI implementation:

* If the application uses deterministic rules for analysis, say so explicitly.
* Do not describe the rules engine as generative AI.
* Disclose the AI coding assistant used to help create the code separately and accurately.
* Do not invent the use of AI services or claim model integration that does not exist.

## 11. Deployment readiness

Prepare the application for static deployment.

Ensure:

* The production build succeeds.
* Required dependencies and configuration are included.
* No API keys or environment secrets are required.
* The app does not depend on a local development server after deployment.
* Client-side routes, if any, work on the selected static host.

Do not claim that deployment is complete until the website is actually published and verified.

## 12. Work order

Follow this exact order:

1. Inspect the current repository and its existing files.
2. Build the complete working MVP.
3. Implement the local analysis engine.
4. Build the dashboard and responsive UI.
5. Add sample data, evidence, filters, copy/export, and reset.
6. Add tests and README documentation.
7. Run the available tests and production build.
8. Fix any errors you find.
9. Summarize the files created or modified.
10. Give me the exact commands to run the app on Windows.
11. Explain how to test the main features and deploy the finished app.

I am a beginner and have only about three hours. Avoid unnecessary questions and do not stop after giving me instructions. Create the actual files using the tools available in your environment.

If you cannot modify the connected repository directly, clearly explain the limitation and give me the simplest practical alternative.

**Final goal:** Deliver a polished, functional, privacy-first UN-BURY web app that genuinely helps users discover what they missed in unread conversations, with working analysis, verifiable evidence, tests, documentation, and deployment readiness.
