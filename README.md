# Yt To Quiz Automation Workflow

## Overview
This n8n workflow ingests YouTube videos, extracts English-language transcripts, generates multiple-choice quizzes with Google Gemini, and delivers the output as a ready-to-share Google Form quiz. Inputs are collected via a public form, and error-handling paths notify the requester when validation or API steps fail.

## Key Capabilities
- Collects form submissions (name, email, video URL, desired question count).
- Validates limits (≤90 questions) before continuing (`c:\Users\user\Downloads\Yt To Quiz (3).json:150-158`).
- Extracts English transcripts with RapidAPI while disabling automatic translation (`c:\Users\user\Downloads\Yt To Quiz (3).json:410-420`).
- Prompts Gemini Flash 2.5 to produce English MCQs with three options each (`c:\Users\user\Downloads\Yt To Quiz (3).json:210-259`).
- Parses LLM output and converts it into Google Forms quiz items (`c:\Users\user\Downloads\Yt To Quiz (3).json:324-359`).
- Creates and shares a Google Form link for responders (`c:\Users\user\Downloads\Yt To Quiz (3).json:288-400`).

## Prerequisites
1. **n8n** instance (self-hosted or cloud) running v1.30+.
2. **RapidAPI** subscription for `youtube-transcript3` (or equivalent) plus API key.
3. **Google Cloud** project with:
   - OAuth 2.0 client granting Forms API scopes (`https://www.googleapis.com/auth/forms.body`, `forms.responses.readonly`).
   - Service account or OAuth credential stored in n8n as `googleOAuth2Api` (`Google account 2`).
4. **Google Gemini** API access (PaLM/Gemini generative language API key).
5. n8n credentials configured for:
   - `googlePalmApi` (Gemini key).
   - `googleOAuth2Api` (Forms OAuth flow).

## Setup Instructions
1. **Import the Workflow**
   1.1 Download `Yt To Quiz (3).json` to your machine.
   1.2 In n8n, click *Import from File* and select the JSON.
2. **Configure Credentials**
   2.1 Open the *HTTP Request* node named "HTTP Request" and assign your RapidAPI key via n8n credentials or environment variables.
   2.2 Verify the Gemini credential (`googlePalmApi`) references a valid API key.
   2.3 Confirm the Google OAuth credential is authorized for Forms API actions.
3. **Customize Limits (Optional)**
   - Adjust the maximum questions check in the `filter` node (`c:\Users\user\Downloads\Yt To Quiz (3).json:134-165`).
   - Update prompt tone or instructions within the `Set Prompt and Model` node (`c:\Users\user\Downloads\Yt To Quiz (3).json:210-223`).
4. **Activate the Workflow**
   4.1 Deploy n8n webhook URL from "Input YouTube URL" node.
   4.2 Share the generated form link with users.

## Node-by-Node Flow
1. **Input YouTube URL (Form Trigger)** — Presents a user form and initiates execution (`c:\Users\user\Downloads\Yt To Quiz (3).json:167-204`).
2. **filter** — Ensures question count is within supported bounds; routes invalid requests to *Error Output*.
3. **Set Prompt and Model** — Builds the Gemini prompt and model selection.
4. **Code in JavaScript** — Extracts the YouTube `videoId` via regex and throws on invalid inputs.
5. **HTTP Request (RapidAPI)** — Fetches transcript in English without translation; handles API key headers.
6. **Code in JavaScript1** — Collates transcript segments into a single `transcriptText` value.
7. **HTTP Request to Gemini** — Submits transcript with prompt to Gemini Flash for quiz generation.
8. **If: Was an error detected?** + **Set Fields: Define Outcome** — Checks Gemini response, retains usage metadata, and gracefully falls back to *Error Output* if missing content.
9. **Google Gemini Chat Model** + **Structured Output Parser** + **Extract JSON** — Ensures LLM output is strict JSON for downstream consumption.
10. **Create a Google Form** — Generates a new quiz shell titled after the form submission.
11. **Prepare Questions for API call** — Transforms parsed questions into Google Forms batch update payloads.
12. **Create MCQ Quizzes** — Pushes the quiz items, enabling grading, feedback messages, and correct answers.
13. **Loop Over Items / Redirect to Google Form** — Finalizes batch operations and redirects submitter to the responder view.
14. **Error Output** — Returns a friendly completion screen when validation or API calls fail.

## Error Handling
- Requests exceeding the question cap route to `Error Output` immediately.
- Transcript fetch or parsing failures throw informative errors, halting the workflow.
- Gemini or Forms API failures send the user to the fallback completion page while logging the error in n8n execution history.

## Customization Tips
- **Localization**: Modify the prompt and Google Form feedback strings for alternate languages.
- **Quiz Difficulty**: Append additional instructions to the Gemini prompt (e.g., "focus on conceptual questions").
- **Additional Metadata**: Extend `Set Fields: Define Outcome` to store token usage or timestamps in your database.
- **Alternate Outputs**: After `Prepare Questions for API call`, branch into other nodes (Google Sheets, email, LMS integrations).

## Troubleshooting
1. **Transcript arrives in non-English text**: Verify the RapidAPI request includes `lang=en&translate=false` and that the source video has English subtitles.
2. **Google Form creation fails**: Re-authenticate the Forms OAuth credential and ensure the account has Forms API enabled.
3. **Gemini errors**: Check quota limits, prompt size (tokens), and ensure `transcriptText` length does not exceed API limits.
4. **No redirect provided**: Confirm that the workflow reaches the "Redirect to Google Form" node and that `formId` is returned from the creation call.

## Security Considerations
- Store all API keys within n8n credentials or environment variables—never hard-code secrets in workflow JSON.
- Limit form exposure to trusted users if API usage costs are a concern.
- Regularly rotate keys for RapidAPI, Google, and Gemini.

## Maintenance Checklist
- Monitor n8n execution logs for repeated failures and adjust prompts or rate limits accordingly.
- Periodically review Google Forms batch API changes and Gemini model updates.
- Validate that RapidAPI quotas meet anticipated usage for long videos (50-minute limit enforced via form guidance).
