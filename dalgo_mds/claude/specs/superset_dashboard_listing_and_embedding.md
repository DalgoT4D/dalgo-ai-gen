# Superset dashboard listing and embedding

This feature shows a list of superset dashboards and clicking on one, it will show that particular dashboard embedded in the webapp

## Current state

Currently Dalgo users who have subscribed for the superset plan can view their superset instance on the old webapp under analysis page. But users will need to authenticate in the embedded superset app to see their dashbaords. Alternative way it to open their instance outside of Dalgo platform, which most of the users do most of the times. 

We want now users to now come to our platform more often and hence showing them a list of their superset  dashboards has value in it. Once we have the charting & dashboard builder functionality, users will also be able to create native dashboards. And then the list of dashboards will show both native and superset dashboards

## What we want

For this feature we will only focus on the superset dashboards and not native charts/dashboards. 
1. User should be able to view the list under the dashboards side menu item. 
2. Users should be able search dashboards by title
3. Users should be be able to view a single dashboard by clicking on it. And this should embed that superset dashboard on that view

## Implementation specific details

### Backend

1. The main repo you will be working in is the django backend
2. For orgs that have superset will be able to use this functionality for others we just send an empty list of dashboards
3. You will see that `viz_url` in the org table will have the url to their superset instance
4. Superset api docs are here https://superset.apache.org/docs/api/
5. To talk to superset we will need to authenticate against this superset. Generate some kind of token.
6. Using this token we will talk to this superset instance of the org and fetch various resources. 
7. We need to think of some caching strategy for this token (maybe store in db or redis) so that we dont decrease the latency of apis. If it becomes invalid, regenerate or get a new one
8. We want to have a common file like `superset_service` that is the gateway to communicate between the backend service and any superset instance. 
9. We will have to save an admin user creds for that superset instance in our secrets manager and map the key somewhere in our db to retrieve. Assume the admin user is created outside of Dalgo and given to dalgo via an api. Subsequent token generation should happen via this admin user of the particular superset.


### Webapp

1. This feature is part of the dalgo's revamp and so we will be using the new webapp for this.
2. The dashboard listing page should be responsive
3. The dashboard page should allow users to search via dashboard title
4. It should allow users to filter dashboards via relevant status of a dashboard. You can look at superset api docs to figure this out
5. We want to make sure the dashboard cards listed on page have equal size even if the title length is longer
6. This is the package we should be using to embed the dashboards https://www.npmjs.com/package/@superset-ui/embedded-sdk
7. When we click on the card, a single dashboard should be embedded using the above sdk
8. On the listing page, we also want to render the thumbnail of each dashboard to make it a bit more attractive to the user. Superset api docs should be able guide you here.
9. We want to the experience of viewing embedded dashboards to be very smooth and user friendly. 

## Testing & Validation

1. The Dalgo server should be up and running after the changes.
2. Make sure you are able to access all the new pages created without any errors.
3. Write test cases for both frontend & backend and make sure they execute without any errors. 

## Review and refactor

1. Once you are done with everything. Review the code to make sure its production ready
2. Refactor if you think there is redundancy.
3. Add comments on the parts of the code where the logic seems to be complicated and might not be easy to read. 
4. Make sure all the newly created or updated functions have proper docstrings

## Don'ts

1. Avoid importants inside the function. If you run into circular import, feel free to refactor and create a new file
2. Ultrathink on whether the logic you are implementing is already present or not. We dont want redundancy in our code and want to reuse as much as possible
3. Carefully understand whats already been and what you need to more. 

