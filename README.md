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
