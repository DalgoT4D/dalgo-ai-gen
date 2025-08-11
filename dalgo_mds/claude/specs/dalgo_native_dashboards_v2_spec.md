# Dalgo Dashboards v2

This feature allows dalgo users to create a dashboards out of the existing charts. We want to give our users a smooth experience while they build their dashboards.

## Current state

### Current state of charts
Currently, the new webapp v2 for Dalgo allows users to create charts. Through this feature we want to allow users to group charts under a new entity "dashboard". 

Charts are built using apache echarts. The data rendered in the charts is fetched from client's warehouse. All the chart configuration are stored in backend's database. Chart types currently supported are line, bar, pie & number. 

All the queries are built on the fly using sqlalchemy and the aggregate query builder class. The advantage of this is that the chart feature becomes warehouse agnostic, it should/will work on a postgres warehouse and also a bigquery warehouse. 

### Current state of dashboards
Some of the dalgo users use superset as part of our extended offering & so in the full release we plan to show native dashboards as well as superset dashboards.

Fetching list of superset dashboards & embedding is already implemented. This spec is focusing on the native dashboards. On the listing page, however, we do want a filter of some sort to see native vs external (superset) dashboards. By default, everything will be shown.

For this feature we will develop, native dashboards as a separate functionality & then worry about how to merge it with the implemented superset dashboard embedding functionality. 

For native dashboards, we do have a dashboard builder of some sort but we want to re-think and re-do this into something with better user experience.

## Scope

1. Listing of dashboards
    1.1 Give a filter to switch between native & external (superset) dashboards. 
    1.2 User should be able to search dashboard by title
2. Dashboard components
    2.1 Native charts. Configuration can be fetched from backend
    2.2 Text blocks with ability to adjust font size and font type
3. Creation of dashboard (Dashboard builder)
    3.1 The layout of the dashboard builder should be a fluid grid with configurable no of columns like 12, 14, 16, etc 
    3.2 User should be able to drag and drop components on the dashboard.
    3.3 Each component can only be sized to a sub-grid.
    3.4 User should be able to remove components from the layout.
    3.5 User should be able to undo (go back an action) or redo (go forward in action). Any action related to dashboard components will fall under this. We can restrict this to maybe max 20 undos.
    3.6 User should be able to add/configure filters. A dashboard can have multiple filters. Filters will have the following inputs
        - Schema
        - Table
        - Filter type - Value (categorical), Numerical
        - Filter setting
            - For filter type Value, we will have the following settings
                - Has a default value (checkbox)
                - Can select multiple values (checkbox)
            - For filter type Numerical, we will have the following settings
                - Single value vs range. By default range will be selected
    The Value filter will be shown as a dropdown on the dashboard layout. The Numerical filter will be shown as a line widget with a marker(s) that can be moved around. 
    3.7 Filters will be applied only to chart components and not text blocks. 
    3.8 User should be able to work on the dashboard in full screen mode so that maximum screen space is given. 
    3.9 We want the dashboard creation work to be saved automatically every 5 seconds or so. Something like google docs. We can also have a manual button to save the work
4. Dashboard view mode
    4.1 Once the dashboard is configured there should be a view mode that lets users view their visualizations along with the filters configured. User should be able to then filter data in the dashboard. 
    4.2 There should be a full screen mode, to see give the dashboard max screen space
5. Locking dashboard editing- only one org user should be able to edit a dashboard at any given time. If someone else is trying to do it, they should be locked out and be notified of who is editing.


## Implementation specific details

### UI/frontend

1. We want to figure out a dashboarding library or package on the frontend that gives us most of the above features.
2. Dashboard components should be responsive. For example, a chart when stretched or compressed along the grid layout should adjust to its surrounding box
3. All the dashboard configuration should be saved in the backend


### Backend

1. Backend will need to save the dashboard entity, the filters attached to it and also the components mapped to it. 
2. Filtering a dashboard will need to refresh all charts on the fly and update the view. On frontend, it should be as simple as passing some filter config the chart component of dashboard
3. We also want to save the positioning of various components on the layout grid. So users can resume from where they left off. 

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
3. Carefully understand whats already been done and what you need to more. 

