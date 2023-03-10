# Terra workspace setup

- Go to the Swagger UI for Terra - this is a web-based form that exposes the API endpoints for Terra. Currently, this is the only way to create an Australian-based workspace in Terra.
  - [https://rawls.dsde-prod.broadinstitute.org/#/workspaces/createWorkspace](https://rawls.dsde-prod.broadinstitute.org/#/workspaces/createWorkspace)
- Click the 'Authorize' button and follow the prompts to authorise with your Garvan Google account.
    <img src="resources/images/terra_workspace_create_1.png" alt="Authorise Google account through Swagger UI, step 1" style="width:1000px" />
    <img src="resources/images/terra_workspace_create_2.png" alt="Authorise Google account through Swagger UI, step 2" style="width:800px" />
- Click on 'Try it out', enter the relevant information into the JSON object in the text field, then click 'Execute'
    <img src="resources/images/terra_workspace_create_3.png" alt="Create Terra workspace through Swagger UI, step 1" style="width:1000px" />
    <img src="resources/images/terra_workspace_create_4.png" alt="Create Terra workspace through Swagger UI, step 2" style="width:1000px" />
    - The JSON data should be structured like this:
        ``` json
        {
        "namespace": "<terra billing project>",
        "name": "<workspace name>",
        "authorizationDomain": [],
        "attributes": {},
        "noWorkspaceOwner": false,
        "bucketLocation": "australia-southeast1"
        }
        ```
    - It's important that the 'bucketLocation' is set to 'australia-southeast1'.
    - Currently the billing project ('namespace') we're using is 'terra-kccg-production'
- The Terra project should now be created and available in the workspace list here: [https://app.terra.bio/#workspaces](https://app.terra.bio/#workspaces).

## Adding workflows to a Terra workspace

To add a workflow to a Terra workspace:
- Go to the "Workflows" tab on Terra and click on "Find a Workflow".
- Under "Find Additional Workflows", click on "Dockstore". This will take you to the Dockstore website.
- Search for the workflow of interest. For example, to add the `WholeGenomeReprocessing` workflow stored within the repository `shyamrav/warp`, search for "shyamrav/warp/WholeGenomeReprocessing".
- Click on the matching workflow to go to the workflow's webpage.
- Under the "Launch with" section, click on "Terra". This takes you to Terra's workflow import page.
- Name the workflow and specify the desination workspace, then click "Import".
