# LinkedIn Post Automation for Online Blogs

## Overview
This repository contains the n8n workflow configuration for automating LinkedIn content publication. The pipeline extracts SEO keywords from a content database, matches them against the live website sitemap to extract the relevant URL, generates a highly structured LinkedIn post using a Large Language Model, and publishes it directly to a company page.

The system features a branched routing mechanism, allowing for fully automated random keyword selection or targeted manual input.

## Tech Stack

* **Workflow Execution**
    * [n8n](https://n8n.io/)
    * Serves as the foundational infrastructure. It handles the visual routing, API authentication, and JavaScript execution. This workflow is designed for a self-hosted Docker deployment.
* **Content Database**
    * [Notion API](https://developers.notion.com/)
    * Stores the SEO keywords and tracks publication status. The pipeline queries this database to select unpublished topics and updates the row status upon successful publication to prevent duplicates.
* **Generation Engine**
    * [Google Gemini API](https://ai.google.dev/)
    * Utilizes the Gemini 2.5 Flash model. It processes the target keyword and strict prompt parameters to output plain text, problem-solution formatted posts free of Markdown or formatting artifacts.
* **Publishing Endpoint**
    * [LinkedIn REST API](https://learn.microsoft.com/en-us/linkedin/)

## Pipeline Architecture

The workflow follows a clear, linear progression with a single conditional split at the start.

### 1. Trigger and Input Routing
The execution begins with an **n8n Form Trigger**. The user selects the operation mode via a dropdown menu.
* **Random Mode:** The pipeline queries the Notion database, retrieves all rows where the "Published" column is empty, and uses a Code node to isolate exactly one random keyword.
* **Manual Mode:** The pipeline bypasses Notion entirely and accepts a custom keyword typed directly into the trigger form.

### 2. URL Extraction
An HTTP GET request fetches the raw XML from `https://mywebsite.com/post-sitemap.xml`. A custom JavaScript node converts the selected keyword into a URL-friendly slug and uses Regular Expressions (Regex) to locate the exact `<loc>` tag containing the matching article link. It outputs a clean JSON object containing `targetKeyword` and `articleUrl`.

### 3. Content Generation
The data is passed to the Gemini node. The prompt is strictly engineered to act as the Lead Editor. It forces a Problem-Solution structure, limits paragraph length, and appends the extracted article URL. The node includes built-in retry logic (15-second delay) to handle Google API rate limits automatically.

### 4. Publication
A custom HTTP Request node formats the generated text into the exact JSON payload required by LinkedIn. It injects the `LinkedIn-Version: 202604` header to ensure compatibility with modern endpoint requirements and publishes the post to the specified Organization ID.

### 5. Status Update
If the workflow ran in "Random" mode, a final Notion node takes the original `pageId` and updates the database row status to "Published", closing the loop.

## Setup and Installation

### Prerequisites
* A running instance of n8n (Docker recommended).
* API keys for Google Gemini and Notion.
* An active LinkedIn OAuth2 API connection within n8n, fully authorized for the `w_organization_social` scope.
* Your LinkedIn Organization ID (found in your page URL).

### Import Instructions
1.  Download the `.json` workflow file from this repository.
2.  Open your n8n workspace.
3.  Click **Add Workflow** in the top right corner.
4.  Click the options menu (three dots) and select **Import from File**.
5.  Upload the `.json` file.

### Configuration
1.  Open the **Get Keywords** (Notion) node and connect your credentials. Ensure the database ID matches your specific tracking sheet.
2.  Open the **Message a model** (Gemini) node and connect your credentials.
3.  Open the **Create a post** (HTTP Request) node. Connect your LinkedIn OAuth2 credentials. Update the JSON body parameter, replacing the placeholder `YOUR_PAGE_ID` with your actual Organization ID.
4.  Save the workflow.

## Usage
To execute the pipeline, do not use the standard "Test workflow" button at the bottom of the canvas. 

1.  Double-click the **n8n Form Trigger** node.
2.  Click **Test step**.
3.  A form panel will open on the right side of your screen. Select your desired mode (Random or Manual).
4.  If Manual, input your exact keyword.
5.  Click **Submit**. The workflow will run synchronously and output the final execution log.
