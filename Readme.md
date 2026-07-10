# Agentic YouTube Assistant (LangGraph + Supadata MCP)

This repository contains an end-to-end deployment of an Agentic AI application. It uses a FastAPI/LangGraph backend connected to the Supadata Model Context Protocol (MCP) to extract and analyze YouTube transcripts, tunneled securely via Ngrok to a Vercel-hosted React frontend.

---

## Prerequisites
Before you begin, ensure you have accounts created on the following platforms:
* **AWS:** For hosting the backend on an EC2 instance.
* **Ngrok:** For creating a secure tunnel to the backend (https://ngrok.com/).
* **Vercel:** For hosting the React frontend (https://vercel.com/).
* **Supadata:** For the YouTube MCP API key (https://supadata.ai/).
* **OpenRouter / LangSmith:** For LLM access and tracing.

---

## Step 1: Provisioning the AWS EC2 Instance

1. Go to your AWS Console: https://us-east-1.console.aws.amazon.com/console/home?region=us-east-1#
2. Go to **EC2** > **Instances** > **Launch instances**.
3. **Name:** `backend`
4. **OS Images:** Choose **Debian**.
5. **Instance Type:** Choose `t2.large` (recommended). 
   * *Note: Sometimes, if you’re on the free tier, you may not be able to choose `t2.large`. Check if `m7i-flex.large` is available, then choose that. Else go for the best configuration that’s available for you.*
6. **Key Pair:** * Click **Create new key pair**.
   * **Name:** `backend`
   * **Type:** `RSA`
   * **Format:** `.pem`
   * Click **Create key pair** (Download and save this `.pem` file to your local machine).
7. **Firewall (security groups):** Check all 3 boxes (Allow SSH, HTTP, HTTPS).
8. **Configure storage:** Change disk size to **80 GB**.
9. Click **Launch instance** and click on the Instance ID link it produces.

### Opening Port 8080
Now we will add an additional port for AWS to connect to the UI.
1. Inside your instance details, go to the **Security** tab.
2. Under "Security groups", click the linked security group.
3. Click **Edit Inbound Rules**.
4. Click **Add rule** and configure:
   * **Port range:** `8080`
   * **Protocol:** Custom TCP
   * **Source:** `0.0.0.0/0`
5. Click **Save rules**. We have now added an additional port on top of this instance, which is 8080, and now we can actually communicate from the UI, which uses 8080 to connect to this instance.

---

## Step 2: Connecting to EC2 via GitHub Codespaces

Now we will connect to this instance from the codespaces VM.

1. Go to your running instances in AWS: https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Home: > Instances (running).
2. Open your instance by clicking on the Instance ID and **copy the Public IPv4 DNS** of that instance.
   * *Example: `ec2-98-81-135-44.compute-1.amazonaws.com`*
3. Come back to Codespaces, open a new terminal using `+`.
4. Drag and drop the downloaded `.pem` file into the Codespaces file explorer and rename it to `backend.pem` (if it isn't already).
5. Set the correct permissions for the key so AWS doesn't reject it (Only the file's owner can read the file; no one can write/execute it):
   ```bash
   chmod 400 ./backend.pem
   ```
6. Connect to your EC2 instance via SSH:
   ```bash
   ssh -i backend.pem admin@<YOUR_PUBLIC_DNS>
   ```
   *If successful, your terminal prompt will change to something like: `Linux ip-172-31-39-138 6.12.74+deb13+1-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64`, showing the codespaces VM can connect to the EC2 VM through SSH.*

---

## Step 3: Setting up the Backend on EC2

Once connected to your EC2 instance via SSH, prepare the environment, clone the code, and start the server.

1. **Install dependencies:**
   ```bash
   sudo apt update && sudo apt install git npm python3-venv python3-pip -y
   ```
2. **Clone the repository:**
   ```bash
   git clone [https://github.com/just-joseph/v2-ik-genai-w4.git](https://github.com/just-joseph/v2-ik-genai-w4.git)
   cd v2-ik-genai-w4/backend
   ```
3. **Set up Python Virtual Environment:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install fastapi uvicorn pydantic mcp langchain-mcp-adapters langgraph langchain-openai python-dotenv
   ```
4. **Configure Environment Variables:**
   Navigate back to the root of the repo and create a `.env` file:
   ```bash
   cd ~/v2-ik-genai-w4
   nano .env
   ```
   Paste the following (replace with your actual keys):
   ```text
   OPENAI_API_KEY="your_openrouter_key"
   SUPADATA_API_KEY="your_supadata_key"
   LANGCHAIN_API_KEY="your_langsmith_key"
   LANGCHAIN_TRACING_V2="true"
   LANGCHAIN_PROJECT="sheet_ai_backend"
   ```
   *Save and exit (Ctrl+O, Enter, Ctrl+X).*
5. **Start the FastAPI Server:**
   ```bash
   cd ~/v2-ik-genai-w4/backend
   uvicorn sheet_ai:app --host=0.0.0.0 --port=8080 --env-file=../.env
   ```
   *Leave this terminal running! DO NOT DISTURB THE UVICORN RUN.*

---

## Step 4: Exposing the Backend via Ngrok

Now we want to start a new tunnel on the EC2 instance so that it connects to the secure Ngrok cloud.

1. Go to Ngrok and login with your Google account.
2. Choose platform as Linux: https://dashboard.ngrok.com/get-started/setup/linux
3. In Codespaces, **open a new terminal tab**. 
4. Use the SSH command to login to the EC2 instance again in this new tab (Replace with your public DNS):
   ```bash
   ssh -i backend.pem admin@<YOUR_PUBLIC_DNS>
   ```
5. Install the Ngrok agent by copying the command from Step 1 on the Ngrok dashboard and running it:
   ```bash
   curl -sSL [https://ngrok-agent.s3.amazonaws.com/ngrok.asc](https://ngrok-agent.s3.amazonaws.com/ngrok.asc) \
     | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
     && echo "deb [https://ngrok-agent.s3.amazonaws.com](https://ngrok-agent.s3.amazonaws.com) bookworm main" \
     | sudo tee /etc/apt/sources.list.d/ngrok.list \
     && sudo apt update \
     && sudo apt install ngrok
   ```
6. Add your authtoken (Make sure you replace `$YOUR_AUTHTOKEN` with your token from the dashboard):
   ```bash
   ngrok config add-authtoken $YOUR_AUTHTOKEN
   ```
7. Start the tunnel to get a public URL for your app:
   ```bash
   ngrok http --url=avalanche-batch-cubicle.ngrok-free.dev 8080
   ```
   *It will show: `https://avalanche-batch-cubicle.ngrok-free.dev -> http://localhost:8080`*
   *(This URL `avalanche-batch-cubicle.ngrok-free.dev` is what Ngrok exposes).*

8. **Verify:** Try opening your URL (e.g., `https://avalanche-batch-cubicle.ngrok-free.dev/`) in a new browser tab. Click on “Visit Site”. It should show: 
   `{"message":"Chat Assistant API is running"}`

---

## Step 5: Deploying Frontend to Vercel

Vercel is basically our React frontend which will run the “chat front end” part of our code. Its request and response operate just like Gradio.

1. Sign in (or sign up) to https://vercel.com/ with the same Google account.
2. Click on **Add New** and choose **Project**.
3. Click on **Continue with GitHub** and log in.
4. Click on **Install the GitHub application for the accounts you wish to Import from to continue**.
5. Select **All repositories** and click on **ik-genai-w4** (or `v2-ik-genai-w4`).
6. Under **Root Directory**, click **Edit**, choose `chat-frontend`, and click **Continue**.
7. Under **Environment Variables**:
   * **KEY:** Add `REACT_APP_BACKEND_URL` *(You can see it in code under `chat-frontend > src > app.js`)*
   * **VALUE:** Add the Ngrok URL you copied (e.g., `https://avalanche-batch-cubicle.ngrok-free.dev`)
   * ⚠️ **NOTE: DO NOT ADD A SLASH AT THE END**
8. Press **Deploy**.
9. It will say: *"Congratulations! You just deployed a new project ______ "*
10. In vercel.com, under Deployments, you will see this application. Click on **Visit**.
11. Test your application by asking a question and providing a YouTube link!
