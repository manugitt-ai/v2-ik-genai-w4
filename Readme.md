# Introduction
This is the Interview-Kickstart GenAI Week-4 exercise.

#### Create EC2 box on AWS
- Create a new EC2 instance (debian t3.small 80GB).
- Create new rule and add port 8080 to incoming security group.
- Download the .pem file from security tab, and name it `backend.pem`.
- Keep the 'Public DNS' copied from 'details' of the EC2 instance, somewhere safe for later use.

#### Create a codespace or clone to local
You have two options, either local or github codespace VM.
1. Local
Create a clone of the repository `git clone https://github.com/manupatet/ik-genai-w4.git` and open in VSCode using devcontainer. This will create the VM with all requisite installs.

2. Github Codespaces
Initite a github codespace from Code -> codespace -> new-with-options -> IK-GenAI-W4

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

### Setup backend on EC2

On your VM (local or codespaces) copy the backend.pem file to the root folder.
The run the following commands to connect to the (substitute with your public amazon DNS):
```
chmod 400 ./backend-pem.pem
scp -i backend.pem archive.tar admin@ec2-52-91-249-145.compute-1.amazonaws.com:~/
scp -i backend.pem .env admin@ec2-52-91-249-145.compute-1.amazonaws.com:~/
```

*The artifacts and the token have now been copied to EC2 box.*

- Open a new terminal and run:
```
ssh -i ./backend-pem.pem admin@ec2-52-91-249-145.compute-1.amazonaws.com

sudo apt update && sudo apt install git npm
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
mkdir repo/
cd repo
git clone https://github.com/manupatet/ik-genai-w4.git
cd ik-genai-w4
```

First remove `gdrive-mcp-server` directory and then `untar` the archive.tar :
```
rm -rf gdrive-mcp-server/
tar -xf ~/archive.tar
```

Copy the `.env` file over from the build to project root directory: `cp ~/.env .`

#### Prepare the backend server to run:

```
cd ~/repo/ik-genai-w4/backend
uv venv .venv
source .venv/bin/activate
uv pip install .
uvicorn sheet_ai:app --host=0.0.0.0 --port=8080  --env-file=../.env
```