# LinkedIn Post Automation for Online Blogs
<img width="1536" height="337" alt="image" src="https://github.com/user-attachments/assets/a5eda4e0-24ce-4176-8fc6-39330e309e26" />

## Overview

This repository contains the n8n workflow configuration for automating LinkedIn content publication. The pipeline sources a topic and its article URL either automatically or from manual input, generates a highly structured LinkedIn post using a Large Language Model, and publishes it directly to a company page.

The system features a branched routing mechanism at the very start of the workflow. This branch controls two things at once: how the keyword is chosen and how the article URL is obtained.

- **Random Mode:** picks an unpublished keyword from Notion and extracts the matching URL from the live sitemap automatically.
- **Manual Mode:** skips Notion and the sitemap lookup entirely. Both the keyword and the article URL are typed directly into the trigger form.

## Tech Stack

- **Workflow Execution**
  * [n8n](https://n8n.io/)
  * Serves as the foundational infrastructure. It handles the visual routing, API authentication, and JavaScript execution. This workflow is designed for a self-hosted Docker deployment.
- **Content Database**
  * [Notion API](https://developers.notion.com/)
  * Stores the SEO keywords and tracks publication status. The pipeline queries this database to select unpublished topics and updates the row status upon successful publication to prevent duplicates. Only used in Random Mode.
- **Generation Engine**
  * [Google Gemini API](https://ai.google.dev/)
  * Utilizes the Gemini 2.5 Flash model. It processes the target keyword and strict prompt parameters to output plain text, problem-solution formatted posts free of Markdown or formatting artifacts.
- **Publishing Endpoint**
  * [LinkedIn REST API](https://learn.microsoft.com/en-us/linkedin/)

## Pipeline Architecture

The workflow follows a clear, linear progression with a single conditional split at the start. The path taken depends entirely on the mode selected at trigger time.

### 1. Trigger and Input Routing

The execution begins with an **n8n Form Trigger**. The form has three fields: a **Mode** dropdown (Manual or Random), a **Keyword** field, and a **URL** field. A **Switch** node reads the Mode value and sends the execution down one of two paths.

- **Random Mode:** The Keyword and URL fields are ignored. The pipeline queries the Notion database, retrieves all rows where the "Published" column is empty, and uses a Code node to isolate exactly one random keyword.
- **Manual Mode:** The pipeline bypasses Notion entirely. An Edit Fields node takes whatever was typed into the Keyword and URL fields and passes them straight through as the target keyword and article URL.

### 2. URL Extraction (Random Mode only)

This step only runs when Random Mode is selected. An HTTP GET request fetches the raw XML from `https://mywebsite.com/post-sitemap.xml`. A custom JavaScript node converts the selected keyword into a URL-friendly slug and uses Regular Expressions (Regex) to locate the exact `<loc>` tag containing the matching article link. It outputs a clean JSON object containing `targetKeyword` and `articleUrl`.

In Manual Mode, this step is skipped. The `targetKeyword` and `articleUrl` values come directly from the form instead, so the URL you type in needs to be the exact, correct link.

### 3. Content Generation

Both paths converge here. The data is passed to the Gemini node. The prompt is strictly engineered to act as the Lead Editor. It forces a Problem-Solution structure, limits paragraph length, and appends the article URL. The node includes built-in retry logic (5-second delay, up to 3 tries) to handle Google API rate limits automatically.

### 4. Publication

A custom HTTP Request node formats the generated text into the exact JSON payload required by LinkedIn. It injects the `LinkedIn-Version: 202604` header to ensure compatibility with modern endpoint requirements and publishes the post to the specified Organization ID.

### 5. Status Update

If the workflow ran in Random Mode, a final Notion node takes the original `pageId` and updates the database row status to "Published," closing the loop. Manual Mode has no Notion row to update, so this step does not apply.

## Setup and Installation

### Prerequisites

- A running instance of n8n (Docker recommended).
- API keys for Google Gemini and Notion.
- An active LinkedIn OAuth2 API connection within n8n, fully authorized for the `w_organization_social` scope.
- Your LinkedIn Organization ID (found in your page URL).

### Import Instructions

1. Download the `.json` workflow file from this repository.
2. Open your n8n workspace.
3. Click **Add Workflow** in the top right corner.
4. Click the options menu (three dots) and select **Import from File**.
5. Upload the `.json` file.

### Configuration

1. Open the **Get Keywords** (Notion) node and connect your credentials. Ensure the database ID matches your specific tracking sheet. This only matters for Random Mode.
2. Open the **Message a model** (Gemini) node and connect your credentials.
3. Open the **HTTP Request** node under Publication. Connect your LinkedIn OAuth2 credentials. Update the JSON body parameter, replacing the placeholder organization URN with your actual Organization ID.
4. Save the workflow.

## Usage

To execute the pipeline, do not use the standard "Test workflow" button at the bottom of the canvas.

1. Double-click the **On form submission** node.
2. Click **Test step**.
3. A form panel will open on the right side of your screen. Select your desired Mode from the dropdown.
4. If **Random**, leave the Keyword and URL fields blank. The workflow picks both for you.
5. If **Manual**, fill in both the Keyword and the exact article URL you want the post to link to.
6. Click **Submit**. The workflow will run synchronously and output the final execution log.
