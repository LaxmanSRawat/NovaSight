# NYC 911 Data Downloader

Downloads and processes 911 call data from NYC Open Data API.

## Setup

1. Create a Python virtual environment:
   ```bash
   python -m venv .venv
   ```

2. Activate the virtual environment:
   - Windows:
     ```bash
     .venv\Scripts\activate
     ```
   - Linux/Mac:
     ```bash
     source .venv/bin/activate
     ```

3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

4. Now you can run the Jupyter notebooks and Python scripts in the project.

5. Create a `.env` file with your NYC Open Data API credentials:
```
NYC_DATA_AUTH=your_auth_token_here
```

6. Initialize Git repository:
```bash
git init
git add .
git commit -m "Initial commit"
```

7. Add your remote repository:
```bash
git remote add origin <your-repository-url>
git branch -M main
git push -u origin main
```
