# LLM Summarization of failed pipelines

This enhancement helps Dalgo to auto generate the summarization of failure logs of pipeline. 

## What we want

Each pipeline's status is notified by prefect to Dalgo by a webhook. In case of failures we should 
fetch the logs and generate summaries to be then saved in Dalgo's database for faster retrieval. 

## Current state

Currently it takes about 30-45 seconds for a user on Dalgo to generate summary of failure, which is not a good user experience. The current feature of AI summarization can summarize logs from airbyte sync as well as a pipeline.

## Implementation specific details

1. The entry will be the webhook api that is hit by prefect to notify failures or success.
2. All jobs in Dalgo whether they are airbyte syncs or pipelines are essentially a prefect delpoyment. 
3. We already have a summarize_logs function in place that we want to reuse in the webhook handler now.
4. A prefect deployment can have multiple tasks running and it can fail at any point.
5. If it fails at a particular point, subsequent tasks are not executed. 
6. Our current state, gives you the ability to summarize the logs of that particular task.
7. If the task failed is an airbyte sync, we ought to fetch the last failed job of it since we wont have that information in the flow run (of deployment) parameters. 
8. Dalog now stores airbyte job details in its db for quick retrieval. 
9. We need to make sure that the summarization logic is ran after the airbyte job is synced in the db. 
10. The webhook handler does a bunch of stuff which we dont want to touch and also make sure our logic is at the end so that it doens't affect other important steps being handled. 


## Testing & Validation

1. The Dalgo server should be up and running after the changes.
2. Write test cases and make sure they execute without any errors. 


## Review and refactor

1. Once you are done with everything. Review the code to make sure its production ready
2. Refactor if you think there is redundancy.
3. Add comments on the parts of the code where the logic seems to be complicated and might not be easy to read. 
4. Make sure all the newly created or updated functions have proper docstrings

## Don'ts

1. Avoid importants inside the function. If you run into circular import, feel free to refactor and create a new file
2. Ultrathink on whether the logic you are implementing is already present or not. We dont want redundancy in our code and want to reuse as much as possible

