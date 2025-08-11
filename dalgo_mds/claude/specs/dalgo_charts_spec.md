# Superset dashboard listing and embedding

This feature allows dalgo users to create, view, edit, update and save a chart. We want to give our users a smooth experience while they build their visualizations.

## Current state

Currently, Dalgo doesn't support any kind of charting capabilities. As part of Dalgo revamp and this spec we will now allow users to create their visualization (charts). The focus here is more on the UI/Ux so that our users have a good experience while using this functionality

## Scope

1. Listing of charts
    1.1 Ability to search charts by their title or/and description in the listing
2. Creation of charts
    2.1 Ability to preview the chart live as its being built/configured
    2.2 Ability to preview data that the chart is plotting. This will be also be dynamic as we configure the chart
3. Updation of charts
    3.1 Ability to preview the chart live as its being built/configured
    3.2 Ability to preview data that the chart is plotting. This will be also be dynamic as we re-configure the chart
4. Deletion of charts


## Chart types and scope of customizations
Our scope will be limited to these chart types with some customizations
1. Bar. Allow configuration/customization for:
    1.1 User should be able to choose between vertical or horizontal bar
    1.2 User should be able to choose whether they want a stacked bar or not. This might not make sense for a single dimension chart
    1.3 Tooltip on hover of bars showing data. This should happen by default
    1.4 User should be able to change X axis & Y axis titles. But we start with some sensible defaults based on column names
    1.5 User should have the option to select whether they want data labels on the bar or not. These should not overlap
    1.6 Have/show legends by default
    1.7 User should be able to add atmost 1 extra dimension
    1.8 User should be able to set/edit a title and a description
2. Pie. Allow configuration/customization for:
    1.1 Tooltip on hover of bars showing data. This should happen by default
    1.2 User should be able to add atmost 1 extra dimension
    1.3 User should be able choose between a donut or a full circle chart
    1.4 Have/show legends by default
    1.5 User should have the option to select whether they want data labels on the pie or not. These should not overlap
    1.6 User should be able to set/edit a title and a description
3. Line. Allow configuration/customization for:
    1.1 User should be able to add atmost 1 extra dimension
    1.2 User should be able choose between a smooth line or a raw line chart
    1.3 Tooltip on hover of bars showing data. This should happen by default
    1.4 User should be able to add atmost 1 extra dimension
    1.5 User should have the option to select whether they want data labels on the points or not. These should not overlap
    1.6 Have/show legends by default
    1.7 User should be able to change X axis & Y axis titles. But we start with some sensible defaults based on column names

## Some considerations

### UI/frontend

1. We want to use apache echarts as our charting library. https://echarts.apache.org/examples/en/index.html
2. We want to full size page for configuring charts giving users a good experience. By default we should collapse the side menu
3. A chart creation can be divided into 4 components
    3.1 The config form
    3.2 Some sort of a form or another way to give users option to customize the chart
    3.3 The chart being rendered
    3.4 The chart's underlying dataset

### Backend

1. We want to use sqlalchemy to build queries that will fetch data from the chart. 
2. No need to worry about caching the chart data for now. We can worry about that later.
3. We also want to resuse our query builder class `class AggQueryBuilder:` to construct these queries. 

## High level architecture

1. Users will build charts from the data in their warehouse
2. Backend will be responsible for (in that order)
    2.1 Generating the query behind the chart
    2.2 Running it against the warehouse of that org
    2.3 Transforming the data and building the apache echarts config json. 
3. Backend will only fill the bare minimum fields in the echarts config json.
4. As users customize the chart from the UI, this config json will have more fields added to it. Backend will then save this update config json object. 
5. Backend should make sure when they its updating the data fields in the apache echarts config json object, rest of the fields shouldn't get messed up. However, when its a new object the backend should add sensible defaults based on the customizations in the scope
6. Apart from the config json object, backend will also be responsible for powering the data preview of the chart. Basically running that query and handling pagination. 
7. The best experience of charting will come when user will be able to see chart being built alongside the data preview.

## Implementation specific details

1. The chart config will be slightly different for different chart types. 
2. Once a user selects the dataset (schema name & table name). The frontend should then hit backend after each config form change to get the fresh echarts config json & try to render the chart. 
3. Similarly, backend on each chart config form change will try to construct that query and if it gives back some data we send it in the echarts config json otherwise we keep it with defaults.
4. All details about chart type and their forms along with the apache echarts config is present in this sheet 
https://docs.google.com/spreadsheets/d/1LXpij5ztLGjAOvqxES7cr0nYJedfnuW-oA6cdq0hO_c/edit?gid=1297238612#gid=1297238612
5. The focus first should be get a bare minimum chart successfully rendered before getting into customizations

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

