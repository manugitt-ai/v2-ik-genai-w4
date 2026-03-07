# Introduction
This is the Interview-Kickstart GenAI Week-4 exercise.

### Building .zip artifact

Run these commands to setup the box for the build.

First, copy '.env.example' to '.env' 
```
cp .env.example .env
```

Copy your openrouter key to 'OPENAI_API_KEY'

#### Build the server
```
cd gdrive-mcp-server
npm install && npm run build
mkdir .credentials
```

Copy (Drag and drop) the `gcp_oauth_keys.json` file, that was obtained from GCP (rename if necessary) into the newly created `.credentials` directory.


Based on these credentials, we will now obtain a fresh auth token from Google:
```
node --env-file=../.env dist/index.js auth
```

Authenticate on the window that opens (hit 'continue' if warned). The URL will automatically redirect back to localhost:3000 (since local google auth is being used). 

#### Authentication on codespaces VM

In case you're using codespaces, in the URL that redirects and fails, simply replace the 'localhost:3000' with your github URL:
For e.g. if the redirect URL from google is: https://localhost:3000/jkasjdkajslkdjkasjdlkajskdjalsd , 
- your workspaces URL is : `https://urban-couscous-v99v666w96fx9vg.github.dev/` 
- and the port (shown in ports tab) while the command 'node --env-file=../.env dist/index.js auth' is still running is `33445`
Then, replace 'http://localhost:3000/' with 'https://urban-couscous-v99v666w96fx9vg-33445.app.github.dev/', and hit enter.

It should now authenticate showing `Authentication successful! Please return to the console.` on the webpage. It will write a new file '.gdrive-server-credentials.json' to your credentials folder.

As the last step, create the .zip artifact

```
cd /workspace/app
tar -cf archive.tar ./gdrive-mcp-server
```

### Backend : EC2 instance

Over on the EC2 machine, first clone the repo:

```
cd ~/
mkdir repo
cd repo
git clone https://github.com/manupatet/ik-genai-w4.git
cd ik-genai-w4
```

Now `untar` the archive.tar in the root directory of the github repository, so that the build files are correctly laid out.
- Remove the `gdrive-mcp-server` folder from the downloaded github repo and unzip the file in its place:
```
rm -rf ~/repo/ik-genai-w4/gdrive-mcp-server/
tar -xf archive.tar
```

Copy the `.env` file over from the build to directory `~/repo/ik-genai-w4/`

#### Prepare the backend server to run:

```
cd ~/repo/ik-genai-w4/backend
uv venv .venv
source .venv/bin/activate
uv pip install .
uvicorn sheet_ai:app --host=0.0.0.0 --port=8080  --env-file=../.env
```