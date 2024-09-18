# ado-terraform-orchestrator
this repo is for Azure devops terraform pipeline that works in a similar way to Atlantis.  
the reason behind this project is to have the nice functionality that Atlantis has (mainly the repository lock mechanism) while staying in azure devops pipelines.  

## Terraform job templates
these pipelines mimic atlanits behavior by using repository lock and multiple project planning/applying.  
this is done by using a 'terraform-locks' (configured by the calling pipeline) variable group as the lock mechanism.  
* only terraform plan should be automatically triggered, terraform apply should be required but manually started.  
* unlock pipeline can be started only by using the pipelies tab.
* all of these are configured in the calling repository by creating two pipeline files, one with 'plan' default command and one with 'apply'.
* add build validation to these pipelines to make them work out of pull requests

### requiremnts

1. Service account used (System.AccessToken) allowed to contribute to pull request.
2. Service account used (System.AccessToken) needs access to the  'terraform-locks' variable group as admin
3. Create unique federated credentials in the targeted Service principals using p://{Your_Organization_Name}/{Your_Project_Nmae}/{Your_Pipeline_Name}
4. Variable group for the terraform lock and plan logic.
5. Variable group for the {environment}_CLIENT_ID / {environment}_TENANT_ID 

### Limitations and known issues

1. to dynamically use differrent credentials we use indirect referencing.  
currently we have included in the environment variables of both the PLAN and the APPLY pipeline jobs these environment variables:
```
NONPROD_CLIENT_ID: $(NONPROD_CLIENT_ID)
NONPROD_TENANT_ID: $(NONPROD_TENANT_ID)
PROD_CLIENT_ID: $(PROD_CLIENT_ID)
PROD_TENANT_ID: $(PROD_TENANT_ID)
```
this how the indirect referencing works.  
if we would like to add additional environment it will need to be updated in the pipeline files and in the variable group.  

2. long plan messages will be cut in PR comment. (150,000 max chars)

3. I couldnt get [Create OIDC token REST API](https://learn.microsoft.com/en-us/rest/api/azure/devops/distributedtask/oidctoken/create?view=azure-devops-rest-7.1#taskhuboidctoken) to work with a service connection like it should so currently it works on the entire pipeline - the federated credentials is approving 'the pipeline' and not the Service connection.  

### Workflow of the plan pipeline

1. we get the pullrequest id that trigger the pipeline and verify if there is a lock variable in place in the variable group (this variable will be: "terraform-$(Build.Repository.ID)-lock"). 
if its not there - start a new plan, if its there then we verify if the value of this variable is the same as pull request id,  
if it is regenerate plan, if not fail and write who blocks the repository.  
2. that we continue to the git diff logic to get the folders or 'projects' that have changes in them and save them to the variable group (this variable will be: "terraform-$(Build.Repository.ID)-projects").  
3. we start the project loop on each of the found projects where we perform all the required actions (fmt check, init, validate, tflint, plan).
4. each project plan file is uploaded as an artifact and the artifact name is saved unto the varialbes (this variable will be: "terraform-plan-$(Build.Repository.ID)-$terraformProject-$(Build.BuildId)").
5. the post-pr job is executed last and will check if any message needs to be commented in the pull request and if there is - it will comment that.

### Workflow of the apply pipeline

1. we get the pullrequest id that trigger the pipeline and verify if there is a lock variable in place in the variable group 
if its not there - error the pipeline because no active plan is found, if its there then we verify if the value of this variable is the same as pull request id,  
if it is we continue with the apply process, if not fail and write who blocks the repository.  
3. we start the project loop on each of the projects already found on the plan pipeline and saved to the variable group, in this loop we perform all the required actions (init, apply).
4. each successful apply project is removed from the projects variable.
5. at the end of the apply step if all projects were successful we update the pull request to have auto completion settings with the 'squash' merge type.
6. the post-pr job is executed last and will check if any message needs to be commented in the pull request and if there is - it will comment that.

### Workflow of the unlock pipeline

1. we get the pullrequest id that trigger the pipeline and verify if there is a lock variable in place in the variable group (this variable will be: "terraform-$(Build.Repository.ID)-lock"). 
if its not there - error the pipeline because its not locked by a pull request, if its there then we verify if the value of this variable is the same as pull request id,  
if it is continue unlocking the repository, if not fail and write who blocks the repository.  
2. start to remove all found variables, first project plan variables, then projects variable and lastly the lock variable.
5. the post-pr job is executed last and will check if any message needs to be commented in the pull request and if there is - it will comment that.


### Summary table

| pipeline template | file name                   | command         | what it does                                                                                                                                                                        |
|-------------------|-----------------------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| plan              | terraform-job-plan.yml      | terraform plan  | perform terraform plan on all found terraform folders with changes against the target branch. save all plans as artifacts and create/update locks and plan names in variable group. |
| apply             | terraform-job-apply.yml     | terraform apply | perform terraform apply on all saved plans from the terraform plan job.                                                                                                             |
| unlock            | terraform-job-unlock.yml    | n/a             | perform removal of all saved plans and locks from the variable group - to allow for other pull requests to perform terraform plan.                                                  |
| post-pr           | terraform-job-post-pr.yml   | n/a             | perform posting of terraform commands outputs and other useful information in the relevant pull requests as comment.                                                                |
| variables         | terraform-job-variables.yml | n/a             | not a job template but a variable template for easier management of pipeline variables used in the other templates.                                                                 |
  
# Todo

- [ ] implement a loop logic for comment messages larger then 150,000 chars.
- [ ] make the OIDC token create REST API work with service connection, allowing for granular approach.
- [ ] maybe rethink the indirect referencing logic and implement something better.
- [ ] add logic that after a terraform plan if there are no changes for that terraform project, dont put it has a project for the apply step.

## Special thanks

Thanks for [Atlantis](https://github.com/runatlantis/atlantis) for their awesome tool.