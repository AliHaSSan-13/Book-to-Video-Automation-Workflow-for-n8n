# 📚 Book-to-Video Automation Workflow for n8n

This n8n workflow automates the entire process of turning a book (by title or file) into a fully edited video summary, ready to upload to YouTube. It fetches the book text, generates a cinematic narration script, creates scenes with AI-generated images and audio, stitches them into a video, and uploads the final result.

## 🚀 Features

- Accepts book requests via webhook (title **or** PDF file)
- Searches Project Gutenberg for public domain books and downloads the text
- Handles large books by chunking text to fit LLM context limits
- Generates a compelling 5–10 minute narration script using Google Gemini
- Breaks the script into visual scenes with detailed image prompts
- Creates voiceovers with ElevenLabs TTS
- Generates images using Pollinations.ai (or replaceable with any image API)
- Combines audio and images into video via Cloudinary
- Stitches all scene videos into a single MP4 and uploads to YouTube

## 📋 Prerequisites

- [n8n](https://n8n.io/) (self-hosted or cloud) – tested with version 1.0+
- The **@n8n/n8n-nodes-langchain** package installed (for Gemini nodes)
- Accounts for:
  - **PostgreSQL** (database for book metadata)
  - **AWS S3** (storage for uploaded books)
  - **Cloudinary** (video/audio processing)
  - **Google Gemini** (AI for script & scene generation)
  - **ElevenLabs** (text-to-speech)
  - **YouTube** (for uploading the final video)
- Basic understanding of n8n and webhook calls

## 🛠️ Setup Instructions

### 1. Install Required Nodes

If you haven't already, install the **n8n-nodes-langchain** package in your n8n instance:
```bash
npm install @n8n/n8n-nodes-langchain
```

### 2. Prepare the Database

Create a PostgreSQL database and run the following SQL to create the required tables:
```sql
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    original_title TEXT NOT NULL,
    normalized_title TEXT UNIQUE NOT NULL,
    source TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE media_assets (
    id SERIAL PRIMARY KEY,
    book_id INTEGER REFERENCES books(id) ON DELETE CASCADE,
    file_path TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 3. Configure External Services

#### 📦 AWS S3

  Create a bucket (e.g., your-book-bucket)

  Create an IAM user with s3:PutObject  and  s3:GetObject permissions

  Note the access key and secret key

#### ☁️ Cloudinary

  Sign up at cloudinary.com

  Note your cloud name, API key, and API secret

  Create two upload presets:

  youtube-n8n for images

  youtube-n8n-audio for audio (or adjust names in the workflow)

#### 🤖 Google Gemini

  Enable the Gemini API via Google Cloud Console

  Create an API key with access to models/gemini-2.5-flash and models/gemma-3-27b-it

#### 🎙️ ElevenLabs

  Sign up at elevenlabs.io

  Get your API key from the account settings

#### 🎬 YouTube

  Create a Google Cloud project and enable YouTube Data API v3

  Set up OAuth 2.0 credentials (Desktop app type)

  Authorize the integration in n8n

### 4. Import the Workflow

  Download the workflow JSON file.

  In n8n, go to Workflows → Import from File and upload the JSON.

  The workflow will appear with placeholder nodes. You'll need to configure credentials.

### 5. Set Up Credentials in n8n

In n8n, create the following credentials (use the exact names as in the workflow, or update the nodes accordingly):

| Credential Name   | Type               | Required Fields                                                                 |
|-------------------|--------------------|---------------------------------------------------------------------------------|
| `Postgres account` | PostgreSQL         | Host, Port, Database, User, Password                                            |
| `S3 account`      | S3                 | Access Key, Secret Key, Region, Bucket (e.g., `your-book-bucket`)               |
| `YouTube account` | YouTube OAuth2     | OAuth2 credentials from Google Cloud                                            |

For **Cloudinary** and **ElevenLabs**, you will use **HTTP Request nodes** with API keys. The workflow uses placeholders that you must replace:

- In `Upload Audio to Cloudinary` and `Upload Image to Cloudinary`, change the URL from `https://api.cloudinary.com/v1_1/<Cloudinary cloud name>/...` to your actual cloud name.
- In the same nodes, update the `upload_preset` field to the presets you created.
- In `ElevenLabs TTS1`, replace `<YOUR-API-KEY>` with your ElevenLabs API key.

### 6. Adjust Node Configuration

- **S3 bucket name**: In `Get book from S3` and `Upload a file`, replace `<YOUR S3 BUCKET NAME>` with your actual bucket name.
- **Cloudinary cloud name**: In all Cloudinary nodes, replace `Cloudianry cloud name` with your cloud name (e.g., in `Build Manifest` and `Generate Video URL`).
- **YouTube title**: In `Upload a video`, change the title field to a dynamic expression like `={{ "Book Summary: " + $('Clean Text').item.json.source }}` (adjust based on where the book title is available).
- **Localhost webhook**: The `Submit to Add-Book Webhook` node calls `http://localhost:5678/webhook-test/add-book`. If you are not running n8n locally on that port, update the URL to your actual n8n webhook URL. Alternatively, replace this HTTP call with an **Execute Workflow** node that runs the `Add Book` workflow internally. (See note in Troubleshooting.)

## 🔄 Workflow Overview

The workflow consists of two main webhooks:

### `/request-book` – Main pipeline

1. **Input validation**: Checks if a book name or file is provided (not both).
2. **Check existing book**: Queries the database to see if the book already exists.
   - If found, retrieves the file from S3 and proceeds to text extraction.
   - If not, searches Project Gutenberg via Gutendex, downloads the book, and adds it to the DB and S3 via the `/add-book` webhook.
3. **Text extraction**: Based on file type (PDF, TXT), extracts clean text.
4. **Token limit check**: Estimates tokens; if >240k, splits into chunks.
5. **Chunk summarization (if needed)**: Each chunk is summarized by Gemini, then merged into a complete narration script.
6. **Script generation**: Gemini creates a full narration script with intro, hook, story, and outro.
7. **Scene breakdown**: Gemini divides the script into scenes with image prompts.
8. **Loop over scenes**: For each scene:
   - Generate image via Pollinations.ai
   - Generate audio via ElevenLabs
   - Upload both to Cloudinary
   - Merge audio and image into a video using Cloudinary transformations
   - Save the video scene to Cloudinary
9. **Stitching**: Builds a final video URL by splicing all scene videos.
10. **Upload to YouTube**: Uploads the final video with a title and metadata.

### `/add-book` – Book storage

- Accepts a PDF file and title
- Saves the file to S3
- Inserts metadata into PostgreSQL

## 📡 Usage

### Trigger the main workflow

Send a POST request to `https://your-n8n-instance.com/webhook/request-book` with either:

- **JSON body** with a book title:
  ```json
  { "book_name": "pride and prejudice" }
- **Form-data** with a file:
    ```text

    file: (binary PDF file)
  ```

The workflow will process the request and eventually upload the video to YouTube. The response will be a simple success message (unless an error occurs).
Adding a book manually

You can also add a book via the /add-book webhook by sending a POST with multipart/form-data containing title and file.
### 🧩 Customization

  Image generation: Replace Pollinations.ai with your preferred image API (e.g., DALL-E, Stability AI). Modify the Image Genration node accordingly.

  TTS voice: Change the voice ID in ElevenLabs TTS1 to a different voice (see ElevenLabs voices).

  Video resolution: Adjust the width and height variables in the Generate Video URL node.

  Scene count: Tweak the prompt in Scene Breaker to produce more or fewer scenes.

  Language: Adapt prompts and TTS to other languages.

### 🐞 Troubleshooting
#### The localhost webhook call fails

The workflow contains an internal call to http://localhost:5678/webhook-test/add-book. This works only if n8n runs on localhost with that exact port and the webhook path matches. For production, consider:

    Replace the HTTP Request node with an Execute Workflow node that directly runs the Add Book workflow (requires both workflows to be in the same n8n instance).

    Or set up a proper public URL for your n8n and update the URL in the node.

#### Database connection issues

    Verify that your PostgreSQL credentials are correct.

    Ensure the books and media_assets tables exist with the exact schema.

#### Cloudinary upload errors

    Check that the upload presets exist and have the correct permissions.

    Ensure the cloud name is correctly set in all nodes.

    For audio uploads, the resource type must be video.

#### Gemini API rate limits

    If you hit rate limits, consider adding a delay between chunk summaries or implementing retries.

    The workflow includes a Wait 1 minute node inside the chunk loop – you can remove it if not needed.

#### YouTube upload fails

    Ensure your OAuth2 credentials are valid and the YouTube channel is linked.

    The video must be less than 15 minutes for the default OAuth scope; adjust if needed.

#### 📄 License

This workflow is open source under the MIT License. Feel free to use, modify, and distribute with attribution.

Happy automating! If you have questions or improvements, feel free to contribute or reach out.
