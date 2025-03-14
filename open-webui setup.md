---
tags: 
ETA: 
Done: false
---

#### System Requirements
- **Operating System**: Linux 
- **Python Version**: Python 3.11+
- **Node.js Version**: 22.10+
#### Development Setup Instruction
###### Clone the Repository
```bash
git clone https://github.com/your-org/open-webui.git  # Replace with actual repository URL
cd open-webui
```

###### Install Node.js
   - Ensure Node.js ≥20.18.1 is installed:
```bash
# Install Node Version Manager (n)
sudo npm install -g n

# Install specific Node.js version
sudo npm install 20.18.1

# if meet any connect error, you can use follow way to install
# install nvm(Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

### install node with verison ID
nvm install 20.18.1

### if need change node version
nvm use 20.18.1
```
###### Install Miniconda
   - Download and install Miniconda:
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh   ### ypu can skip reading install information by enter q

```
   - Configure environment paths:
```bash
# Add Miniconda to PATH (replace /path/to/ with actual installation path)
export PATH="/path/to/miniconda3/bin:$PATH"   ### defaoult path is: /root/miniconda3/bin

# Initialize Conda
conda init
source ~/.bashrc

# Verify installation
conda --version
```

###### Frontend Build and Test

- Enter open-webui
- Create a `.env` file:
  
    ```
    cp -RPp .env.example .env
    ```

+  Update Ollama Serving Address in `.env`

Modify the `.env` file to configure the **Ollama backend URL**. This ensures that requests to `/ollama` are correctly redirected to the specified backend:

```ini
# Ollama URL for the backend to connect
OLLAMA_BASE_URL='http://ip_address:port'

# OpenAI API Configuration (Leave empty if not used)
OPENAI_API_BASE_URL=''
OPENAI_API_KEY=''

# AUTOMATIC1111 API (Uncomment if needed)
# AUTOMATIC1111_BASE_URL="http://localhost:7860"

# Disable Tracking & Telemetry
SCARF_NO_ANALYTICS=true
DO_NOT_TRACK=true
ANONYMIZED_TELEMETRY=false
```

Ensure you replace `ip_address:port` with the actual IP address and port of your **Ollama server** if necessary.

- Install dependencies:
  
    ```
    npm install
    ```
    
- Build frontend server:
  
    ```
    npm run build

    (when building, may meet error likes "Cannot find package 'pyodide'",then you should need to install

    bash
    npm install pyodide
    
    )
    ```
+ After building the frontend, copy the generated `build` directory to the backend and rename it to `frontend`:
  
    ```
   cp -r build ./backend/open-webui/frontend

   ```
###### Backend Build and Setup

- Navigate to the backend:
  
    ```
    cd backend
    ```
    
- Use **Conda** for environment setup:
  
    ```
    conda create --name open-webui python=3.11
    conda activate open-webui
    ```
    

+  Change Pip to Use Aliyun Mirror

Downloading packages from remote sites can be slow. To speed up the process, you can specify a local mirror such as **Aliyun** when installing packages:

```bash
pip install torch -i https://mirrors.aliyun.com/pypi/simple/
```

Alternatively, you can set Aliyun as the default mirror by adding the following lines to `~/.pip/pip.conf`, suggest to this method:

```ini
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
```

This ensures that all future package installations automatically use the Aliyun mirror.

 (It will take a long time to isntall dependencies, open a tmux session to install)
- Install python dependencies:
  
    ```
    yum isntall -y tmux
    tmux new -s test
    pip install -r requirements.txt -U
    ```
- Start the backend:
  
    ```
    sh dev.sh
    ```
