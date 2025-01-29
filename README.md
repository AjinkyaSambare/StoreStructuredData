# **Delivery Email Extractor**

## **Overview**
This Python script extracts delivery-related details from email bodies using Azure OpenAI and stores the extracted data in an Azure SQL Database. It provides a Streamlit-based user interface where users can paste an email body, extract structured delivery details, and store the extracted information in a database.

---

## **How the Script Works**
### **1. User Inputs an Email**
- The user pastes the email body into a text area in the Streamlit UI.
- The script processes this input and sends it to Azure OpenAI for extraction.

### **2. Data Extraction Using Azure OpenAI**
- The script sends the email content to Azure OpenAI with a structured prompt.
- The AI model extracts relevant details, such as delivery status, price, order ID, delivery date, store name, tracking number, and carrier.
- The response is expected in JSON format.

### **3. Processing the Extracted JSON**
- The extracted data is validated and cleaned.
- The script ensures that the AI's response is correctly formatted as JSON.

### **4. Storing the Data in an Azure SQL Database**
- The script connects to an Azure SQL Database using `pymssql`.
- If the `delivery_details` table does not exist, it is created.
- The extracted JSON data is inserted into the database.

### **5. Displaying Data in Streamlit**
- The extracted JSON data is displayed in the UI.
- If the data is successfully inserted into the database, a success message is shown.

---

## **Installation and Setup**
### **1. Prerequisites**
- GitHub Codespaces or a local Python environment

### **2. Install Required Libraries**
In GitHub Codespaces or your local machine, install the required dependencies:
Go into the terminal and type the following commands

```sh
pip install streamlit requests pymssql
```

---

## **Setup your "secrets.toml"**
---

## **Step 1: Add Your Azure OpenAI API Credentials**
In `secrets.toml`, add the **Azure OpenAI API credentials**:

```toml
AZURE_OPENAI_API_ENDPOINT = "https://your-openai-instance.openai.azure.com/v1/chat/completions"
AZURE_OPENAI_API_KEY = "your-azure-openai-api-key"
```

- Replace `"your-openai-instance"` with your actual **Azure OpenAI instance name**.
- Replace `"your-azure-openai-api-key"` with the **API key** provided by your organization.

---

## **Step 2: Add Azure SQL Database Credentials**
Now, add the **Azure SQL database connection details**:

```toml
AZURE_SQL_SERVER = "your-server-name.database.windows.net"
AZURE_SQL_DATABASE = "your-database-name"
AZURE_SQL_USERNAME = "your-database-username"
AZURE_SQL_PASSWORD = "your-database-password"
```

- Replace `"your-server-name.database.windows.net"` with your **Azure SQL Server name**.
- Replace `"your-database-name"` with your **database name**.
- Replace `"your-database-username"` and `"your-database-password"` with your **login credentials**.

⚠️ **Your organization will provide these credentials. Handle them securely.**  

---

## **Step 3: Ensure `secrets.toml` is Ignored in Git**
To **prevent credentials from being uploaded to GitHub**, create a `.gitignore` file (if not already present) and add:

```sh
echo ".streamlit/secrets.toml" >> .gitignore
```

This ensures **Git does not track `secrets.toml`**, keeping your credentials safe.

---
## **Code Breakdown**
### **1. Importing Required Libraries**
The script imports necessary Python libraries:

```python
import streamlit as st
import requests
import json
import re
import pymssql
from typing import Dict, Any
```
- `streamlit`: Provides the web-based UI.
- `requests`: Sends API requests to Azure OpenAI.
- `json`: Handles JSON data formatting.
- `re`: Uses regular expressions to extract JSON from raw text.
- `pymssql`: Connects to Azure SQL Database.
- `typing`: Defines structured data types.

---

### **2. Azure OpenAI Interaction**
This class interacts with Azure OpenAI to extract structured delivery details from email content.

```python
class AzureOpenAIChat:
    def __init__(self):
        """Initialize API credentials from Streamlit secrets."""
        self.API_ENDPOINT = st.secrets.get("AZURE_OPENAI_API_ENDPOINT", "")
        self.API_KEY = st.secrets.get("AZURE_OPENAI_API_KEY", "")
```
- Reads Azure OpenAI credentials stored in Streamlit’s `secrets` file.
- These credentials should be configured in **Streamlit Cloud** or as an environment variable.

```python
def extract_delivery_details(self, email_body: str, max_tokens: int = 300) -> Dict[str, Any]:
```
- This function sends an email body to Azure OpenAI for extraction.
- The `max_tokens` parameter defines the maximum response length.

The **prompt** instructs OpenAI on how to structure the response:

```python
prompt = f"""
Extract delivery-related details from the following email body and return a JSON output with these keys:
- delivery: "yes" if delivery is confirmed, otherwise "no".
- price_num: Extracted price amount, default to 0.00 if not found.
- description: Short description of the product if available.
- order_id: Extracted order ID if available.
- delivery_date: Extracted delivery date in YYYY-MM-DD format if available.
- store: Store or sender name.
- tracking_number: Extracted tracking number if available.
- carrier: Extracted carrier name (FedEx, UPS, USPS, etc.) if available.

Email Body:
{email_body}

Output JSON:
"""
```
This **ensures the AI model returns a structured response**.

The API request is sent to Azure OpenAI:

```python
response = requests.post(self.API_ENDPOINT, headers=headers, json=data)
response.raise_for_status()
return response.json()
```
- `requests.post()` sends the request.
- `raise_for_status()` ensures the request was successful.
- The function returns the extracted JSON response.

---

### **3. Extracting Valid JSON from AI Response**
Azure OpenAI sometimes returns text-wrapped JSON. This function extracts valid JSON.

```python
def extract_valid_json(text: str) -> str:
    text = text.strip()
    text = text.replace("```json", "").replace("```", "")

    json_match = re.search(r"\{.*\}", text, re.DOTALL)
    if json_match:
        return json_match.group(0)

    return text
```
- Removes unnecessary text formatting.
- Uses **regex** to extract JSON content.

---

### **4. Inserting Data into Azure SQL Database**
This function inserts the extracted JSON data into the database.

```python
def insert_into_db(data: Dict[str, Any]):
    conn = pymssql.connect(
        server=st.secrets["AZURE_SQL_SERVER"],
        user=st.secrets["AZURE_SQL_USERNAME"],
        password=st.secrets["AZURE_SQL_PASSWORD"],
        database=st.secrets["AZURE_SQL_DATABASE"]
    )
```
- Connects to the **Azure SQL Database**.
- Credentials are stored in **Streamlit secrets**.

The script **creates the table if it does not exist**:

```python
cursor.execute("""
    IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='delivery_details' AND xtype='U')
    CREATE TABLE delivery_details (
        id INT IDENTITY(1,1) PRIMARY KEY,
        delivery NVARCHAR(10),
        price_num FLOAT,
        description NVARCHAR(255),
        order_id NVARCHAR(50),
        delivery_date DATE,
        store NVARCHAR(255),
        tracking_number NVARCHAR(100),
        carrier NVARCHAR(50)
    )
""")
conn.commit()
```
- Defines the **table structure**.
- Uses `NVARCHAR` for text fields and `FLOAT` for price numbers.

Data is inserted into the table:

```python
cursor.execute("""
    INSERT INTO delivery_details (delivery, price_num, description, order_id, delivery_date, store, tracking_number, carrier)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
""", (
    data["delivery"], 
    data["price_num"], 
    data["description"], 
    data["order_id"], 
    data["delivery_date"], 
    data["store"], 
    data["tracking_number"], 
    data["carrier"]
))

conn.commit()
conn.close()
```
- Ensures extracted data is **saved**.

---

### **5. Streamlit User Interface**
The **main function** provides a web UI for input and extraction.

```python
def main():
    st.set_page_config(page_title="Delivery Email Extractor", page_icon="📩")
    st.title("Delivery Email Extractor")

    email_body = st.text_area("Paste the email body below:")

    if st.button("Extract Details") and email_body:
        with st.spinner("Extracting details..."):
            chat_client = AzureOpenAIChat()
            response = chat_client.extract_delivery_details(email_body)

            if response and "choices" in response:
                extracted_json = extract_valid_json(response["choices"][0]["message"]["content"])
                
                try:
                    parsed_json = json.loads(extracted_json)
                    st.json(parsed_json)
                    insert_into_db(parsed_json)
                    st.success("Data successfully inserted into Azure SQL Database!")

                except json.JSONDecodeError:
                    st.error("Failed to parse JSON response.")
                    st.text(extracted_json)
```
- The UI allows users to **paste email content**.
- Displays extracted **JSON output**.
- Stores results in the database.

---

## **Running the Script**
Run the application in GitHub Codespaces:

```sh
streamlit run app.py
```

This will launch a web interface for extracting and storing delivery details.

---

## **Conclusion**
This script automates **delivery email processing**, integrates with **Azure OpenAI**, and stores extracted data in an **Azure SQL Database** using **Streamlit UI**. It is designed for easy deployment in **GitHub Codespaces**.