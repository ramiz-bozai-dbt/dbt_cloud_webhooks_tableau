# dbt Cloud Webhooks with Tableau

Welcome!  This repo houses code and instructions on utilizing the delivery of a raw webhook from dbt Cloud to trigger the refresh of a Tableau workbook.  This is useful in refreshing the extracts used in the workbooks upon the completion of a run in dbt Cloud.

## Getting started

In order to set up the integration, you should have familiarity with:
- [dbt Cloud Webhooks](/docs/deploy/webhooks)
- Zapier
- The [Tableau API](https://help.tableau.com/current/api/rest_api/en-us/REST/rest_api.htm)

## Instructions

1. Obtain a [Personal Access Token](https://help.tableau.com/current/server/en-us/security_personal_access_tokens.htm) from your Tableau Server/Cloud instance.  You'll need this to authenticate with the Tableau API.

2. Ensure that your Tableau Workbook uses data sources that allow refresh access.  This is typically set when publishing.

3. Create a new Zap in Zapier with the "Catch Raw Hook" event.  Save the webhook URL presented.

4. Create a new webhook in dbt Cloud.  In this case, we're looking to trigger a refresh upon Run Completed and for a specific job.  Paste in the webhook URL obtained from Zapier in step 3 into the "Endpoint" field and test the endpoint.  Upon completion, copy/save the Key provided.

5. Next, we'll want to store our tokens, keys, and other inputs using the [StoreClient](https://help.zapier.com/hc/en-us/articles/8496293969549-Store-data-from-code-steps-with-StoreClient) utility in Zapier.  This will prevent us from having to leave these as plain-text in our code.

    - a. Create a Storage by Zapier connection and generate a UUID to be used in our final script that will retrieve our secrets.
    - b. Run the following code chunk as a temporary Python step to store the secrets.
        - store.set('DBT_WEBHOOK_KEY', 'abc123') #replace with your dbt Cloud Webhook key
        - store.set('TABLEAU_SITE_URL', 'abc123') #replace with your Tableau Site URL, inclusive of https:// and .com
        - store.set('TABLEAU_SITE_NAME', 'abc123') #replace with your Tableau Site/Server Name
        - store.set('TABLEAU_API_TOKEN_NAME', 'abc123') #replace with your Tableau API Token Name
        - store.set('TABLEAU_API_TOKEN_SECRET', 'abc123') #replace with your Tableau API Secret
    - c. Next, add a code action (Code by Zapier -> "Run Python") to add the script to refresh the workbook.
        - In the "Input data" section, map the following two items:
            - raw_body: 1. Raw Body
            - auth_header: 1. Headers Http Authorization
        - Paste in the following code in the code section.  Be sure to replace the placeholder "YOUR_STORAGE_SECRET_HERE" with the UUID secret generated earlier along with the webhook key, appropriate API version, and Tableau authentication variables.
            - ```python
                import requests
                import hashlib
                import json
                import hmac

                # Access secret credentials
                secret_store = StoreClient('YOUR_STORAGE_SECRET_HERE')
                hook_secret = secret_store.get('DBT_WEBHOOK_KEY')
                server_url = secret_store.get('TABLEAU_SITE_URL')
                server_name = secret_store.get('TABLEAU_SITE_NAME')
                pat_name = secret_store.get('TABLEAU_API_TOKEN_NAME')
                pat_secret = secret_store.get('TABLEAU_API_TOKEN_SECRET')

                #Enter the name of the workbook to refresh
                workbook_name = "YOUR_WORKBOOK_NAME"
                api_version = "ENTER_COMPATIBLE_VERSION"

                #Validate authenticity of webhook coming from dbt Cloud
                auth_header = input_data['auth_header']
                raw_body = input_data['raw_body']

                signature = hmac.new(hook_secret.encode('utf-8'), raw_body.encode('utf-8'), hashlib.sha256).hexdigest()

                if signature != auth_header:
                  raise Exception("Calculated signature doesn't match contents of the Authorization header. This webhook may not have been sent from dbt Cloud.")

                full_body = json.loads(raw_body)
                hook_data = full_body['data'] 

                if hook_data['runStatus'] == "Success":

                    #Authenticate with Tableau Server to get an authentication token
                    auth_url = f"{server_url}/api/{api_version}/auth/signin"
                    auth_data = {
                        "credentials": {
                            "personalAccessTokenName": pat_name,
                            "personalAccessTokenSecret": pat_secret,
                            "site": {
                                "contentUrl": server_name
                            }
                        }
                    }
                    auth_headers = {
                        "Accept": "application/json",
                        "Content-Type": "application/json"
                    }
                    auth_response = requests.post(auth_url, data=json.dumps(auth_data), headers=auth_headers)

                    #Extract token to use for subsequent calls
                    auth_token = auth_response.json()["credentials"]["token"]
                    site_id = auth_response.json()["credentials"]["site"]["id"]

                    #Extract the workbook ID
                    workbooks_url = f"{server_url}/api/{api_version}/sites/{site_id}/workbooks"
                    workbooks_headers = {
                        "Accept": "application/json",
                        "Content-Type": "application/json",
                        "X-Tableau-Auth": auth_token
                    }
                    workbooks_params = {
                        "filter": f"name:eq:{workbook_name}"
                    }
                    workbooks_response = requests.get(workbooks_url, headers=workbooks_headers, params=workbooks_params)

                    #Assign workbook ID
                    workbooks_data = workbooks_response.json()
                    workbook_id = workbooks_data["workbooks"]["workbook"][0]["id"]

                    # Refresh the workbook
                    refresh_url = f"{server_url}/api/{api_version}/sites/{site_id}/workbooks/{workbook_id}/refresh"
                    refresh_data = {}
                    refresh_headers = {
                        "Accept": "application/json",
                        "Content-Type": "application/json",
                        "X-Tableau-Auth": auth_token
                    }

                    refresh_trigger = requests.post(refresh_url, data=json.dumps(refresh_data), headers=refresh_headers)
                    return {"message": "Workbook refresh has been queued"}
     - d. We're now ready to test our action and deploy our Zap.  Upon a successful test, you should see a refresh queued in your Tableau Server/Cloud jobs queue.
