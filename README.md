# Photo Timeline

A single-page web app that reads a Google Drive folder and renders your photos as a scrollable timeline grouped by month.

## Setup

### 1. Get a Google Cloud OAuth Client ID

1. Go to [Google Cloud Console](https://console.cloud.google.com/) and create a new project (or select an existing one).
2. Enable the **Google Drive API**: APIs & Services → Library → search "Google Drive API" → Enable.
3. Go to **APIs & Services → OAuth consent screen**:
   - Choose **External** (or Internal if G Workspace).
   - Fill in app name, support email.
   - Add scope: `https://www.googleapis.com/auth/drive.readonly`
   - Add your Google account as a test user (while in testing mode).
4. Go to **APIs & Services → Credentials → Create Credentials → OAuth Client ID**:
   - Application type: **Web application**
   - Authorized JavaScript origins: add `http://localhost:PORT` for local dev, and `https://YOUR_USERNAME.github.io` for production.
5. Copy the **Client ID** (looks like `123456789-abc.apps.googleusercontent.com`).

### 2. Configure the app

Open `index.html` and find the `CONFIG` object near the top of the `<script>` tag:

```js
const CONFIG = {
  CLIENT_ID: 'YOUR_GOOGLE_OAUTH_CLIENT_ID',  // ← paste here
  FOLDER_ID: 'YOUR_DRIVE_FOLDER_ID',         // ← paste here
  SCOPES: 'https://www.googleapis.com/auth/drive.readonly'
};
```

To find your **Folder ID**: open the Google Drive folder in your browser — the ID is the long string in the URL after `/folders/`.

### 3. Deploy to GitHub Pages

1. Create a new GitHub repository.
2. Push `index.html` and `README.md` to the `main` branch.
3. Go to **Settings → Pages → Source** → select `main` branch, `/ (root)`.
4. GitHub will provide a URL like `https://YOUR_USERNAME.github.io/REPO_NAME`.
5. Add that URL to your OAuth Client ID's **Authorized JavaScript origins** in Google Cloud Console.

The app is live at your GitHub Pages URL. No server required.

## Local development

Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Make sure `http://localhost:8080` is in your OAuth client's Authorized JavaScript origins.
