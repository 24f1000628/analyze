# Automated Data Processing and Publishing Pipeline

This project demonstrates a robust and automated pipeline for processing Excel data using Python with Pandas, linting code with Ruff, and publishing the results via GitHub Actions and GitHub Pages.

## Project Overview

The core of this project is an `execute.py` script that reads data from an Excel file (`data.xlsx`), performs data transformation, and outputs the processed data as a JSON file. A comprehensive GitHub Actions workflow automates the execution of this script, ensures code quality, and deploys the generated JSON to GitHub Pages for easy access.

## Features

-   **Data Ingestion**: Reads `.xlsx` files using Pandas.
-   **Robust Data Transformation**: Handles common data quality issues (e.g., non-numeric values in numeric columns) during processing.
-   **Code Quality**: Integrates `ruff` for fast and efficient Python code linting.
-   **Automated Execution**: GitHub Actions automatically runs the data processing script on every push.
-   **GitHub Pages Deployment**: Publishes the `result.json` output to GitHub Pages, making it publicly accessible.
-   **Reproducibility**: `data.xlsx` is converted to `data.csv` and committed, providing a source-controlled CSV version.

## Project Structure

```
.github/
└── workflows/
    └── ci.yml             # GitHub Actions workflow for CI/CD
data.xlsx                # Original Excel data file
data.csv                 # Converted CSV data file (from data.xlsx)
execute.py               # Python script for data processing
index.html               # A simple HTML page providing project overview
LICENSE                  # MIT License details
README.md                # This README file
```

*Note: `result.json` is generated during the CI/CD pipeline and is not committed to the repository.*

## Setup and Local Execution

To set up and run this project locally, ensure you have Python 3.11+ installed.

1.  **Clone the repository (hypothetically):**

    ```bash
    git clone <your-repo-url>
    cd <your-repo-name>
    ```

2.  **Install dependencies:**

    ```bash
    python -m venv .venv
    source .venv/bin/activate  # On Windows: .venv\Scripts\activate
    pip install pandas openpyxl ruff
    ```

3.  **Prepare `data.xlsx` and `data.csv`:**

    The original data is provided in `data.xlsx`. For better version control and compatibility, it has been converted to `data.csv` and both are committed.

    `data.xlsx` (example content assuming `Category` and `Amount` columns):

    | Category    | Amount |
    | :---------- | :----- |
    | Electronics | 100    |
    | Books       | 50     |
    | Electronics | N/A    |
    | Books       | 75     |
    | Food        | 20     |
    | Electronics | 200    |
    | Food        |        |
    | Books       | -      |

    `data.csv` (converted from `data.xlsx`):

    ```csv
    Category,Amount
    Electronics,100
    Books,50
    Electronics,N/A
    Books,75
    Food,20
    Electronics,200
    Food,
    Books,-
    ```

4.  **Run the data processing script locally:**

    ```bash
    python execute.py > result.json
    ```

    This will generate a `result.json` file in your current directory, containing aggregated data (e.g., total amount per category).

## `execute.py` Details

The `execute.py` script is responsible for reading `data.xlsx`, processing it, and printing the results as JSON to standard output. A non-trivial error related to data type handling in the 'Amount' column was identified and fixed to ensure robust processing.

### The Problem

Originally, the script might have failed if the `Amount` column in `data.xlsx` contained non-numeric entries (e.g., "N/A", "-", or empty cells). A direct summation `df['Amount'].sum()` would raise a `TypeError` in such cases, halting the script.

### The Fix

The corrected `execute.py` incorporates two key steps to handle these data quality issues gracefully:

1.  **Coercing to Numeric**: `df['Amount'] = pd.to_numeric(df['Amount'], errors='coerce')`
    -   This line attempts to convert all values in the 'Amount' column to a numeric type. If a value cannot be converted (e.g., "N/A", "-", or an empty string), `errors='coerce'` will replace that value with `NaN` (Not a Number) instead of raising an error.
2.  **Handling Missing Values**: `df['Amount'].fillna(0, inplace=True)`
    -   After coercing, any `NaN` values (resulting from non-numeric entries) are replaced with `0`. This ensures that these entries contribute zero to the subsequent sum and allows aggregation to proceed without errors.

This approach makes the script resilient to common real-world data imperfections in the `Amount` column.

### Fixed `execute.py` Code

```python
import pandas as pd
import json
import sys

def process_excel_to_json(excel_path="data.xlsx"):
    """
    Reads an Excel file, processes it, and prints the result as JSON to stdout.

    Assumes the Excel file has 'Category' and 'Amount' columns.
    Converts 'Amount' to numeric, handling non-numeric entries by coercing to NaN
    and then filling NaN values with 0 before aggregation.
    """
    try:
        df = pd.read_excel(excel_path)

        # Validate required columns exist
        required_columns = ['Category', 'Amount']
        if not all(col in df.columns for col in required_columns):
            missing_cols = [col for col in required_columns if col not in df.columns]
            raise KeyError(f"Missing required columns in Excel file: {', '.join(missing_cols)}")

        # FIX: Non-trivial error handling for 'Amount' column.
        # Convert 'Amount' to numeric, coercing errors to NaN.
        # This handles cases where 'Amount' might contain strings like 'N/A' or '-'.
        df['Amount'] = pd.to_numeric(df['Amount'], errors='coerce')

        # Fill NaN values in 'Amount' with 0 before summing.
        # This ensures aggregation works without errors and treats missing/non-numeric
        # amounts as zero for summation purposes.
        df['Amount'].fillna(0, inplace=True)

        # Group by 'Category' and sum 'Amount'
        grouped_data = df.groupby('Category')['Amount'].sum().reset_index()

        # Rename 'Amount' to 'TotalAmount' for clarity in the output JSON
        grouped_data.rename(columns={'Amount': 'TotalAmount'}, inplace=True)

        # Convert the DataFrame to a list of dictionaries (records)
        result_dict = grouped_data.to_dict(orient='records')

        # Print the JSON output to stdout
        print(json.dumps(result_dict, indent=4))

    except FileNotFoundError:
        print(f"Error: The Excel file '{excel_path}' was not found.", file=sys.stderr)
        sys.exit(1)
    except KeyError as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"An unexpected error occurred: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    process_excel_to_json()
```

## GitHub Actions CI/CD Workflow (`.github/workflows/ci.yml`)

A GitHub Actions workflow is configured to run on every `push` to the `main` branch and on `pull_request` targeting `main`. It performs code linting, executes the data processing script, and publishes the resulting JSON file to GitHub Pages.

### Workflow Steps:

1.  **Checkout Repository**: Fetches the code.
2.  **Set up Python 3.11**: Configures the Python environment.
3.  **Install Dependencies**: Installs `pandas`, `openpyxl`, and `ruff`.
4.  **Run Ruff Linter**: Checks Python code for style and quality issues.
5.  **Execute Data Processing Script**: Runs `execute.py` and redirects its `stdout` to `result.json`.
6.  **Upload result.json for GitHub Pages**: Makes the `result.json` available for the `deploy-pages` job.
7.  **Deploy to GitHub Pages**: Deploys the uploaded artifact to GitHub Pages, making it accessible at `https://<your-username>.github.io/<your-repo-name>/result.json`.

### `.github/workflows/ci.yml` Code

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for actions/checkout (if using with specific permissions)
      pages: write    # To publish to GitHub Pages
      id-token: write # Required for OIDC authentication for pages

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas openpyxl ruff

      - name: Run Ruff Linter
        run: ruff check .

      - name: Execute data processing script
        run: python execute.py > result.json

      - name: Upload result.json for GitHub Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'result.json'

  deploy-pages:
    needs: build-and-deploy
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
