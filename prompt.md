Hey, I’m building a project called **UN-BURY** for my hackathon. The problem statement is “The Unread Problem — What Did I Miss?”

The idea is to help people catch up on long chat conversations without having to read hundreds of messages. Important things like deadlines, tasks, decisions, meeting changes and messages directed at them often get buried in group chats.

I want you to build a proper working website for this, not just a landing page or a UI mockup. I'm a beginner and have around 3 hours, so please keep the code simple, make sensible decisions yourself and focus on getting the main features working first.

### What I want the website to do

* Let users paste a long conversation into a text box.
* Give them a short summary of what happened.
* Highlight urgent messages, deadlines and important changes.
* Show tasks or requests that need the user's attention.
* Identify decisions made in the conversation.
* Detect direct mentions using an optional name field.
* Show the original message behind every result so users can check whether the analysis is correct.
* Let users filter results by priority and category.
* Include a sample conversation so judges can try the app immediately.
* Add buttons to copy the summary, export results and clear everything.

### Design

Make it look like a polished, modern productivity app. I prefer a dark navy/black theme with subtle green or teal accents, clean typography, rounded cards and a well-organized dashboard. It should look good on both laptops and phones.

The name **UN-BURY** should be used throughout the website, with the tagline “Unbury what matters.”

I don't want it to look like a basic college project with a generic AI template. Make the interface feel cohesive and professional, but don't waste time on unnecessary animations or features.

### One important thing about privacy

The problem statement specifically requires conversations, data and summaries to stay on the user's device. So please make sure all analysis happens in the browser. Don't send messages to a cloud AI API or any external server.

For the first version, use local JavaScript/TypeScript rules to identify deadlines, tasks, mentions, decisions and schedule changes. Generate the summary from the actual messages rather than showing a fixed, hardcoded result. Don't pretend this is generative AI if it isn't. Keep the analysis code separate so we can improve it later if there's time.

Also, show a clear privacy indicator explaining that the conversation is processed locally.

### A few more things

* The app should handle empty or badly formatted input without crashing.
* Priorities should make sense instead of marking everything as urgent.
* Each result should show why it was flagged and which message it came from.
* All buttons and filters should actually work.
* Add basic tests for the main analysis features.
* Don't store conversations permanently by default.
* Make sure pasted messages are displayed safely.
* Keep dependencies minimal and avoid login systems, databases or anything that isn't needed.

### Please build it directly in my connected GitHub repository

First check what files are already there. Then create the complete project, including the code, configuration and README.

Use React with Vite and TypeScript if that's the easiest reliable option. Make it easy to deploy as a static website.

Once you're done, run the tests and production build if possible, fix any errors, and tell me what you created. I need the exact steps to run it on my Windows laptop and deploy it.

The README should explain the features, how to run the project, how privacy works, its limitations and the AI services used. Be honest about the fact that the current analyzer uses rules, and disclose any AI coding assistance separately.

Please prioritize getting a complete, working version done within my time limit. Don't just give me instructions or stop at the design — actually build it using the tools available to you.
