# Local Setup Instructions

## Environment Setup

This project uses a conda environment for dependency management.

### 1. Create and Activate Environment

```bash
# Create the environment from the environment.yml file
conda env create -f environment.yml

# Activate the environment
conda activate google-agents
```

### 2. Configure API Key

1. **Get your Google AI Studio API Key**
   - Visit [Google AI Studio](https://aistudio.google.com/app/api-keys)
   - Create a new API key or use an existing one

2. **Create your `.env` file**
   ```bash
   # Copy the template
   cp env.template .env
   ```

3. **Edit the `.env` file**
   - Open `.env` in your editor
   - Replace `your_google_api_key_here` with your actual API key
   
   Example:
   ```
   GOOGLE_API_KEY=AIzaSyAaBbCcDdEeFfGgHhIiJjKkLlMmNnOoPpQq
   GOOGLE_GENAI_USE_VERTEXAI=FALSE
   ```

4. **Verify setup**
   - The `.env` file is already in `.gitignore` to prevent accidental commits
   - Never commit your actual API key to version control

### 3. Run Jupyter Notebooks

```bash
# Make sure you're in the google-agents environment
conda activate google-agents

# Start Jupyter
jupyter notebook
```

Navigate to `notebooks/day-1a-from-prompt-to-action/` and open the notebook to begin.

## Troubleshooting

### "GOOGLE_API_KEY not found" Error
- Ensure you created the `.env` file in the project root
- Check that your API key is correctly formatted (no quotes, no spaces)
- Make sure the `.env` file is in `/home/chrisfkh/google_5day_agent_intensive/`

### Import Errors
- Ensure you've activated the conda environment: `conda activate google-agents`
- Try reinstalling: `conda env remove -n google-agents && conda env create -f environment.yml`

### API Key Issues
- Verify your API key is valid at [Google AI Studio](https://aistudio.google.com/app/api-keys)
- Check if there are any usage limits or billing issues

## Project Structure

```
google_5day_agent_intensive/
├── environment.yml          # Conda environment specification
├── env.template            # Template for .env file
├── .env                    # Your API keys (create this, not in git)
├── .gitignore              # Prevents committing sensitive files
├── SETUP.md               # This file
└── notebooks/
    ├── day-1a-from-prompt-to-action/
    ├── day-1b-agent-architectures/
    ├── day-2a-agent-tools/
    └── day-2b-agent-tools-best-practices/
```

## Next Steps

Once setup is complete, start with Day 1a:
- `notebooks/day-1a-from-prompt-to-action/day-1a-from-prompt-to-action.ipynb`

Follow the course in order through the remaining notebooks.

