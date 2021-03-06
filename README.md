# BioCatalyst Link - e.g. Aggregate Report API
This external module creates a collection of endpoints that can be used by an outside system to access reports 
for one or more REDCap projects on a REDCap server.  It was designed for an external system called [BioCatalyst](https://github.com/susom/biocatalyst).

In its designed use, a REDCap user can link (or enroll) their REDCap project to BioCatalyst via project-level external module
configuration.  Once linked, the external system can pull all user reports using a system-level API endpoint.

This module uses a single, trusted secret which grants access from the calling server to the REDCap external module.
The API supports a number of queries on behalf of the enrolled projects' users.  This module assumes some level of user
authentication on the external system and trusts that name of the provided user when calls are made.

This is different from the standard REDCap API call to 'Export Reports' which would require the external system to
have an array of API tokens for each user/project combination that was to be linked.

There are 2 security measures which protect the report endpoints:

 1. A complex, shared secret shared between the two endpoints
 2. An IP range (CIDR) to limit requests from trusted networks.
 

## Setup
To enable this capability, this external module must be enabled by a Super User from the **REDCap Control Center**. 

### System Parameters
![ExternalModule](images/ExternalModule.png)

Once the module is enabled for your REDCap instance, open the system configuration file and add
the system parameters neccesary to use this module. 

Note: Project level settings to allow access are also required.

![ConfigFile](images/ConfigFile.png)

There are only a few setup parameters that are required.  The two required setup parameters are:
1) **Shared Token** - token that is configured between the outside user and this External Module so this module will
know the requestor is a trusted user.  It should be long and complex.

2) **Alert email address** - An email will be sent when a request outside the designated IP range is received. If an IP
range is not entered, no alerts will be sent.  For security reasons, we highly suggest IP ranges be used so
that tighter controls can be maintained over data stored in REDCap.

Optional data:

1) **Valid IP Ranges** - enter IP ranges where requests can originate from. This data is **not** required since
there may be instances where the IP ranges are not known or cannot be determined in advance.

2) **Enable module on all projects by default** - This allows all project owners to have the ability to enroll their projects
in the aggregate group for reporting.  It does **NOT** mean that they are auto-enrolled, but rather gives them the ability
to do so in the future.  Since access to reports from projects needs an additional project-level checkbox, there is no
greater security risk to enable this module on each project. If this checkbox is **NOT** selected, a REDCap Administrator
will be required to enable this module on each project that chooses to allow this functionality.

3) **Make module discoverable by users** - If the module is not automatically enabled on each project, you can
select option so users can find the module but they will still need to request the module to be enabled for
their project from an Administrator.

### Project Parameters
Once the system parameters are configured, go to the REDCap projects that you want to access reports from and select
the External Module link. 

![ProjectExternalModule](images/ProjectExternalModule.png)

Open the Project Config File and select the **Enable Stanford Biocatalyst to access reports in this project**.

![ProjectConfigFile](images/ProjectConfigFile.png)

To restrict Biocatalyst to be able to access on administrator-selected reports in the project, select "yes" in response to **Should Biocatalyst be limited to access only specific reports?** and enter `report_id` numbers in the **Allowed report** fields that appear.  Select `+` next to the **Allowed report** field to add more than one permitted report.  (*IMPORTANT: If either the "no" option or no response is selected in response to **Should Biocatalyst be limited to access only specific reports?**, all reports in this project will be available to Biocatalyst.*)

Save the configuration.  This project's reports are now accessible from the system-level API endpoints.

## API Calls and Example Syntax
There are four POST calls that are supported with this module. Below are examples of each API endpoint.

### User Rights Request
This API call retrieves the user rights values for 'Data Export Tool' and 'Reports and Report Builder' for each 
of the users in the list for each project enabled with this module. Since both user rights are required
in order to retrieve data, a check to ensure proper user rights access should be performed before requesting data.
    
    Example request:
    {"token":"<shared token>", "request": "users", "user": "<username1>,<username2>"}
    
    Example return:
    [
      {
        "user":"<username1>",
        "projects": [
          {
            "project_id": "1",
            "project_title": "Project 1",
            "rights": {
              "data_export_tool": "1",
               "reports": "1"
            }
          },
          {
            "project_id": "2",
            "project_title": "Project 2",
            "rights": {
              "data_export_tool": "1",
              "reports": "1"
            }
        ]
      },
      {
        "user2": "<username2>",
        "projects": [
          ...
        ]
      }
    ]
        
### List of Reports Request
For a given project, the list of reports that a user has access to can be retrieved.
    
    Example request:
    {"user":"<username>", "token":"<token>", "request":"reports", "project_id":"28"}

    Example return:
    {
        "project_id": 28,
        "reports": [
            {
                "report_id": "8",
                "title": "Report 1"
            },
            {
                "report_id": "9",
                "title": "Report 2"
            }
        ]
    }
     
### Retrieve Report Data Request
To retrieve report data, users must have 'Data Export Tool' and 'Report and Report Builder' rights to the project.
Data can be retrieved in raw form or labelled form.  Raw form returns coded values for multi-choice fields and checkboxes
and labelled form retrieves the labels on the choices and checkboxes.

    Example request:
        For raw data:
        {"user":"<username>", "token":"<token>","request":"reports","project_id":"28","report_id":"8","raw_data":"1"}
    
        Example return: 
        [
            {
                "study_id": "1",
                "first_name": "Test_First",
                "last_name": "Test_Last",
                "address": "asdfasdf",
                "telephone_1": "(415) 555-1212",
                "email": "xxxx@yyyy.com",
                "dob": "2018-08-02",
                "age": "1",
                "ethnicity": "2",
                "race": "5",
                "sex": ""
            }
        ]
        
        
        For labeled data:
        {"user":"<sunetid>", "token":"<token>","request":"reports","project_id":"28","report_id":"8"}
        
        Example return: 
        [
            {
                "study_id": "1",
                "first_name": "Test_First",
                "last_name": "Test_Last",
                "address": "asdfasdf",
                "telephone_1": "(415) 555-1212",
                "email": "xxxx@yyyy.com",
                "dob": "2018-08-02",
                "age": "1",
                "ethnicity": "Unknown / Not Reported",
                "race": "More Than One Race",
                "sex": ""
            }
        ]
    
    
### Retrieve Column Metadata Request
The columns API can be used to retrieve metadata about each column in the specified report. The metadata fields
include: form_name, field_name, field_order, field_label, field_type, field_options and field_validation.

    Example request:
    {"user":"<username>", "token":"<token>","request":"columns","project_id":"28","report_id":"8"}

    Example return: 
    {
        "report_id": 8,
        "project_id": 28,
        "user": "<username>",
        "columns": [
            {
                "form_name": "demographics",
                "field_name": "study_id",
                "field_order": "1",
                "field_label": "Study ID",
                "field_type": "text",
                "field_options": null,
                "field_validation": null
            },
            {
                "form_name": "demographics",
                "field_name": "first_name",
                "field_order": "2",
                "field_label": "First Name",
                "field_type": "text",
                "field_options": null,
                "field_validation": null
            },
            {
                "form_name": "demographics",
                "field_name": "last_name",
                "field_order": "3",
                "field_label": "Last Name",
                "field_type": "text",
                "field_options": null,
                "field_validation": null
            },
            ...
        ]
    }

