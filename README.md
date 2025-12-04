# Snowflake Custom Tools for AI Agents

A collection of powerful custom tools for Snowflake Cortex AI Agents, enabling web scraping, email notifications, currency exchange rates, and web search capabilities.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Tools](#tools)
  - [Web Scraper](#web-scraper)
  - [Email Sender](#email-sender)
  - [Exchange Rate API](#exchange-rate-api)
  - [Web Search](#web-search)
- [Installation](#installation)
- [Usage Examples](#usage-examples)
- [Tool Chaining](#tool-chaining)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This repository contains SQL and Python code for creating custom tools that extend Snowflake Cortex AI Agent capabilities. These tools enable your AI agents to:

- Scrape and extract content from web pages
- Send formatted HTML emails
- Retrieve real-time currency exchange rates
- Perform web searches and return structured results

## ⚙️ Prerequisites

- Snowflake account with appropriate privileges
- `ACCOUNTADMIN` or role with `CREATE INTEGRATION` privileges
- Access to create network rules and external access integrations
- For Exchange Rate tool: API key from [exchangerate-api.com](https://www.exchangerate-api.com/)

## 🛠️ Tools

### Web Scraper

Fetches and extracts text content from web pages using HTTP requests and BeautifulSoup.

**Features:**
- HTTP/HTTPS web page retrieval
- HTML parsing and text extraction
- Clean text output suitable for analysis

**Use Cases:**
- Content monitoring and competitive analysis
- Data enrichment from public sources
- Development and testing datasets

**Setup:**

```sql
-- Create network rule for web access
CREATE OR REPLACE NETWORK RULE Snowflake_intelligence_WebAccessRule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('0.0.0.0:80', '0.0.0.0:443');

-- Create external access integration
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION Snowflake_intelligence_ExternalAccess_Integration
  ALLOWED_NETWORK_RULES = (Snowflake_intelligence_WebAccessRule)
  ENABLED = true;

-- Create the web scraping function
CREATE OR REPLACE FUNCTION Web_scrape(weburl STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = 3.10
HANDLER = 'get_page'
EXTERNAL_ACCESS_INTEGRATIONS = (Snowflake_intelligence_ExternalAccess_Integration)
PACKAGES = ('requests', 'beautifulsoup4')
AS
$$
import _snowflake
import requests
from bs4 import BeautifulSoup

def get_page(weburl):
  url = f"{weburl}"
  response = requests.get(url)
  soup = BeautifulSoup(response.text)
  return soup.get_text()
$$;
```

**Test:**
```sql
SELECT Web_scrape('https://www.snowflake.com/en/blog/ISO-IEC-42001-AI-certification/');
```

**Tool Configuration:**
- **Name:** `Web_scraper`
- **Resource Type:** Function
- **Parameter:** URL to scrape (STRING)
- **Returns:** Extracted text content (STRING)

---

### Email Sender

Sends HTML-formatted emails through Snowflake's notification system.

**Features:**
- HTML email formatting support
- Custom subjects and content
- Built on Snowflake's native email integration

**Use Cases:**
- Automated notifications and alerts
- Report distribution
- Workflow completion notifications

**Setup:**

```sql
-- Grant privileges
USE ROLE accountadmin;
GRANT CREATE INTEGRATION ON ACCOUNT TO ROLE snowflake_intelligence_admin_rl;
USE ROLE snowflake_intelligence_admin_rl;

-- Create notification integration
CREATE OR REPLACE NOTIFICATION INTEGRATION ai_email_int
  TYPE=EMAIL
  ENABLED=TRUE;

-- Create email sending procedure
CREATE OR REPLACE PROCEDURE send_mail(recipient TEXT, subject TEXT, text TEXT)
RETURNS TEXT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'send_mail'
AS
$$
def send_mail(session, recipient, subject, text):
    session.call(
        'SYSTEM$SEND_EMAIL',
        'ai_email_int',
        recipient,
        subject,
        text,
        'text/html'
    )
    return f'Email was sent to {recipient} with subject: "{subject}".'
$$;
```

**Test:**
```sql
CALL send_mail('your.email@company.com', 'Test Email', 'This is a test email sent from Snowflake.');
```

**Tool Configuration:**
- **Name:** `email_sender`
- **Resource Type:** Procedure
- **Parameters:**
  - `recipient`: Email address (TEXT)
  - `subject`: Email subject line (TEXT)
  - `content`: Email body in HTML format (TEXT)

**Best Practice:** Format emails using HTML with colors and icons for better presentation.

---

### Exchange Rate API

Retrieves real-time currency exchange rates from exchangerate-api.com.

**Features:**
- Real-time exchange rate data
- Support for all major currencies
- Comprehensive error handling
- Returns all currency pairs for base currency

**Use Cases:**
- Financial reporting and analytics
- E-commerce dynamic pricing
- Multi-currency data pipeline integration

**Setup:**

1. Sign up for a free API key at [exchangerate-api.com](https://www.exchangerate-api.com/)

2. Create the integration and procedure:

```sql
CREATE OR REPLACE NETWORK RULE EXCHANGERATE_API
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('v6.exchangerate-api.com');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION EXCHANGERATE_API_INTEGRATION
  ALLOWED_NETWORK_RULES = (EXCHANGERATE_API)
  ENABLED = true;

GRANT USAGE ON INTEGRATION EXCHANGERATE_API_INTEGRATION TO ROLE public;

CREATE OR REPLACE PROCEDURE get_exchange_rates(base_currency STRING)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'get_rates'
PACKAGES = ('requests', 'snowflake-snowpark-python')
EXTERNAL_ACCESS_INTEGRATIONS = (EXCHANGERATE_API_INTEGRATION)
AS
$$
import requests
import json

def get_rates(base_currency):
    try:
        if not base_currency or len(base_currency.strip()) != 3:
            return {"error": "Currency code must be exactly 3 characters (e.g., USD, EUR, GBP)"}
        
        base_currency = base_currency.strip().upper()
        
        # Replace <APIKEY> with your actual API key
        url = f"https://v6.exchangerate-api.com/v6/<APIKEY>/latest/{base_currency}"
        response = requests.get(url, timeout=10)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 404:
            return {"error": f"Currency code '{base_currency}' not supported by the API"}
        else:
            return {"error": f"API returned status code: {response.status_code}"}
            
    except Exception as e:
        return {"error": f"Failed to fetch exchange rates: {str(e)}"}
$$;

GRANT USAGE ON PROCEDURE SNOWFLAKE_INTELLIGENCE.TOOLS.get_exchange_rates(string) TO ROLE public;
GRANT USAGE ON PROCEDURE SNOWFLAKE_INTELLIGENCE.TOOLS.get_exchange_rates(string) TO ROLE accountadmin;
```

**Important:** Replace `<APIKEY>` with your actual API key from exchangerate-api.com

**Tool Configuration:**
- **Name:** `get_exchange_rate`
- **Resource Type:** Stored Procedure
- **Parameter:** 3-letter currency code (e.g., USD, EUR, GBP)
- **Returns:** VARIANT (JSON with exchange rates)

---

### Web Search

Performs web searches using DuckDuckGo and returns structured JSON results.

**Features:**
- Returns top 3 organic search results
- Filters out advertisements automatically
- Structured JSON output with title, link, and snippet
- Ideal for AI agent orchestration

**Use Cases:**
- AI agent research workflows
- Automated content discovery
- Link analysis and authority checking
- First step in multi-tool workflows (search → scrape)

**Setup:**

```sql
CREATE OR REPLACE NETWORK RULE Snowflake_intelligence_WebAccessRule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('0.0.0.0:80', '0.0.0.0:443');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION Snowflake_intelligence_ExternalAccess_Integration
  ALLOWED_NETWORK_RULES = (Snowflake_intelligence_WebAccessRule)
  ENABLED = true;

CREATE OR REPLACE FUNCTION Web_search(query STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = 3.10
HANDLER = 'search_web'
EXTERNAL_ACCESS_INTEGRATIONS = (Snowflake_intelligence_ExternalAccess_Integration)
PACKAGES = ('requests', 'beautifulsoup4')
AS
$$
import _snowflake
import requests
from bs4 import BeautifulSoup
import urllib.parse
import json

def search_web(query):
    encoded_query = urllib.parse.quote_plus(query)
    search_url = f"https://html.duckduckgo.com/html/?q={encoded_query}"
    
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }

    try:
        response = requests.get(search_url, headers=headers, timeout=10)
        response.raise_for_status() 
        
        soup = BeautifulSoup(response.text, 'html.parser')
        
        search_results_list = []
        
        results_container = soup.find(id='links')

        if results_container:
            for result in results_container.find_all('div', class_='result'):
                if 'result--ad' in result.get('class', []):
                    continue

                title_tag = result.find('a', class_='result__a')
                link_tag = result.find('a', class_='result__url')
                snippet_tag = result.find('a', class_='result__snippet')
                
                if title_tag and link_tag and snippet_tag:
                    title = title_tag.get_text(strip=True)
                    link = link_tag['href']
                    snippet = snippet_tag.get_text(strip=True)
                    
                    search_results_list.append({
                        "title": title,
                        "link": link,
                        "snippet": snippet
                    })

                if len(search_results_list) >= 3:
                    break

        if search_results_list:
            return json.dumps(search_results_list, indent=2)
        else:
            return json.dumps({"status": "No search results found."})

    except requests.exceptions.RequestException as e:
        return json.dumps({"error": f"An error occurred while making the request: {e}"})
    except Exception as e:
        return json.dumps({"error": f"An unexpected error occurred during parsing: {e}"})
$$;
```

**Tool Configuration:**
- **Name:** `Web_search`
- **Resource Type:** Function
- **Parameter:** Search query (STRING)
- **Returns:** JSON string with top 3 results

**Output Format:**
```json
[
  {
    "title": "Result Title",
    "link": "https://example.com/page",
    "snippet": "Description of the result..."
  }
]
```

---

## 📦 Installation

1. **Clone or download this repository**

2. **Set up network rules and integrations** (run as `ACCOUNTADMIN` or appropriate role)

3. **Create each tool** by executing the SQL code in order

4. **Configure tools in Snowflake Cortex AI**:
   - Navigate to Snowflake Intelligence
   - Add each tool with the specified name and parameters
   - **Important:** Click SAVE and then REFRESH THE PAGE for changes to take effect

## 💡 Usage Examples

### Basic Web Scraping
```sql
SELECT Web_scrape('https://www.example.com/article');
```

### Send a Notification Email
```sql
CALL send_mail(
  'recipient@company.com',
  'Data Pipeline Complete',
  '<h2>Pipeline Status</h2><p>Your ETL job completed successfully!</p>'
);
```

### Get Exchange Rates
```sql
CALL get_exchange_rates('USD');
```

### Search the Web
```sql
SELECT Web_search('Snowflake AI features');
```

## 🔗 Tool Chaining

Combine tools for powerful workflows:

**Example: Web Search → Web Scrape → Email**

Ask your AI agent: *"Get the current number of customers and marketplace listings from Snowflake.com and email it to me"*

The agent will:
1. Use `Web_search` to find relevant Snowflake pages
2. Use `Web_scrape` to extract the content
3. Use `email_sender` to send you the results

## 🔒 Security Considerations

- **Network Access**: Tools require external access integrations - ensure proper network policies
- **API Keys**: Store Exchange Rate API key securely; consider using Snowflake secrets
- **Email Permissions**: Limit email sender access to authorized users
- **Web Scraping**: Respect robots.txt and website terms of service
- **Rate Limiting**: Implement appropriate throttling for external API calls
- **Data Privacy**: Ensure compliance with GDPR and other regulations when scraping
- **Owner Privileges**: Functions execute with OWNER privileges - audit usage regularly

## 🐛 Troubleshooting

### Tool Not Appearing in Agent
- Ensure you clicked SAVE **and** refreshed the page
- Verify the tool name matches exactly
- Check that the resource type (function vs procedure) is correct

### Network Access Errors
- Verify network rules are created and enabled
- Check external access integration is associated correctly
- Ensure your Snowflake account allows external network access

### Email Not Sending
- Verify notification integration is enabled
- Check recipient email address format
- For formatting issues, adjust HTML in agent orchestration tab

### Exchange Rate Errors
- Confirm API key is valid and not expired
- Check currency code is exactly 3 characters
- Verify network access to v6.exchangerate-api.com

### Web Search/Scrape Timeout
- Increase timeout values if needed
- Check if target website is accessible
- Verify User-Agent headers are accepted by target site

## 📚 Additional Resources

- [Snowflake Cortex AI Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex)
- [Snowflake External Access Integrations](https://docs.snowflake.com/en/sql-reference/sql/create-external-access-integration)
- [Python UDF Documentation](https://docs.snowflake.com/en/developer-guide/udf/python/udf-python)

## 📄 License

This code is provided as-is for use with Snowflake. Ensure compliance with all applicable terms of service for external APIs and websites.

## 🤝 Contributing

Contributions welcome! Please ensure:
- Code follows Snowflake best practices
- Tools include comprehensive error handling
- Documentation is clear and complete
- Security considerations are addressed

---

**Note:** Always test tools in a development environment before deploying to production. Monitor usage and implement appropriate governance policies.
