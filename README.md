# SFMC Email Preference Center
How to store and track a user's preferences within Salesforce Marketing Cloud.

## Overview
This setup is designed to allow users to both define the types of emails they'd like to receive and to unsubscribe from all commercial emails sent by your company. This setup uses a single CloudPage to collect and process the user's preferences and pushes that information to a single, master data extension.

* For preferences, once the user submits their updates, the CloudPage will send their preferences as a single, comma-delimited field. From there, the information can be parsed and used to in an automation to populate a corresponding "Preferences" field on your customer master. From there, users can be filtered out using a data filter or SQL query.

* For unsubscribes, the user's status will be updated within the All Subscribers of the parent business unit and will also be denoted by a "Mailable" field on the master preference center data extension. A user can re-subscribe themselves if they re-submit the preferences form.

* For setting the preferences available on the CloudPage, we will be creating a reference data extension that lists all available preferences and using a loop in AMPscript to display all of the preferences within the reference data extension on the CloudPage. 

**Basic flow:**

![SFMC Preferences Manager Flowchart](https://i.ibb.co/8K7gFrm/sfmc-preferencesmanger-flowchart.png)

## Assumptions

This process assumes: 

- Your business rules allow for users to be opted-in to all preferences by default.
- You are not allowing users to update their email address in the preference center. 
- Your business rules allow for users to re-activate themselves within the preference center after the unsubscribe themselves.

## Instructions

### Before you start
- Ensure you have personalized URLs setup within CloudPages (if desired) and SSL certs.
- If setting up a business unit specific preference center, ensure business unit rules are setup properly.
- Before you set this live, add two fields to your customer master data extension.
  - A "Preferences" field. Set the default to be a comma-delimited value of the preferences you are offering (for example: "Pref1,Pref2,Pref3").
  - A "Mailable" field. Set the default to "Y".
  - Then, run a SQL query that updates all existing customers with these values.
  

### Setting up the data extensions
For this setup, you will be creating two data extensions. One to house the user's preferences and one to house all of the preferences we are listing out on the page.

1. First, you'll want to build the preference center data extension. For this setup, we are creating the data extension within the Shared Data Extensions folder via the parent business unit. Below is the data extension we are using for this setup but make any neccesarry adjustments for your use case.

- **Name:** Master_PreferenceCenter_CustomerPreferences
- **Description:** Master data extension for user's preferences and unsubscribes
- **External Key:** Master_PreferenceCenter_CustomerPreferences
- **Data setup:**

  | Field Name | Data Type | Primary Key | Required | Default Value | Definition |
  |------------|-----------|-------------|----------|---------------|------------|
  | Email_Address | Email Address | 256 | N | Y | N/A | Email address of the record |
  | Created_Date | Date | N | N | Current Timestamp | Date they first took an action on the page |
  | Mailable | Text | 1 | N | N | Y | Denotes whether or not the user has unsubscribed. Y = Active, N = Unsubscribed |
  | Preferences | 4000 | N | N | N/A | Comma-delimited field populated with all preferences user has signed up for |
  | Last_Modified_Date | Date | N | N | N/A | Timestamp of user's most recent action within the preference center |

2. Next, you'll want to setup a data extension that will be populated by the preferences you want listed out in your preference center page. This will allow you to easily add additional preferences as your marketing efforts grow more complex. The "priority" field will be used to denote the order in which the preferences should be displayed. The "key" field will be the same for all preferences and will be used in the AMPscript within the CloudPage.
- **Name:** Master_PreferenceCenter_PreferencesList
- **Description:** List of all email preferences
- **External Key:** Master_PreferenceCenter_PreferencesList
- **Data setup:**

  | Field Name | Data Type | Primary Key | Required | Default Value |
  |------------|-----------|-------------|----------|---------------|
  | Preference_Name | Text | 4000 | Y | Y | N/A |
  | Priority | Text | 50 | N | Y | N/A |
  | Key | Text | 50 | N | N | X |

3. If neccessary, add any additional fields to your customer master you'd like to use in conjunction with the preference center.

### Setting up the CloudPages
For the pages involved, you'll need one CloudPage that will house the form and process the user's data and one page to be used as an error page. 

1. Create a new collection, named "Email Preference Center Collection"

2. Create a new landing page under the new collection named "Error". This will be used as a redirect page in the Email Preference Center CloudPage. This page doesn't need much for this setup, simply add the text:

> An error has occurred. Try your request again or contact your system administrator.

3. Create a new landing page named "Email Preference Center". You can design the form as you wish, but you will need the elements listed below on your page for this setup. For an example page - see the attached document sfmc_prefcenter.html

The below AMPscript code should be added to the top of your page and is used to update the user's on the master preference center data extension and, if the user unsubscribes, within the All Subscribers. It also sets a confirmation message which is display on the page after a user takes an action. If the page is accessed and there is no valid email address provided, the page will redirect to the error page we created in the previous step.
   
   ```
   
   %%[

      /* Set the variables used in this page */

      var @email, @jobid, @submitted, @unsubscribe, @preference, @mod_date

      set @email = emailaddr
      set @jobid = jobid
      set @submitted = requestparameter("submitted")
      set @unsubscribe = requestparameter("unsubscribe")
      set @preference = requestparameter("preference")
      set @mod_date = Now()

      /* If the @email variable is empty but the @submitted or @unsubscribe value are populated with the correct values, populate the @email variable with the email parameter. */

      if empty(@email) and (@submitted == "Yes" or @unsubscribe == "Yes") then 
        set @email = requestparameter("email") 
      else endif 
      
      /* If there is no valid email address provided, then redirect the user to the error page */
      
      if isemailaddress(@email) == "false" then
        set @url = "errorpagelinkhere.com"
        redirect(@url)
      else endif


      /* If the user submitted their preferences, then upsert the record on the master preference center data extension and update their status to active */

      if @submitted == "Yes" then 

        UpsertDE("Master_PreferenceCenter_CustomerPreferences",1,"Email_Address",@email,"Mailable","Y","Preferences",@preference,"Last_Modified_Date",@mod_date)

       SET @sub = CreateObject("Subscriber")
        SetObjectProperty(@sub,"EmailAddress", @email)
        SetObjectProperty(@sub,"SubscriberKey", @email)

        SetObjectProperty(@sub,"Status","Active")
        Set @options = CreateObject("UpdateOptions")
        Set @save = CreateObject("SaveOption")
        SetObjectProperty(@save,"SaveAction","UpdateAdd")
        SetObjectProperty(@save,"PropertyName","*")
        AddObjectArrayItem(@options,"SaveOptions", @save)
        /* Here is where we actually update the Subscriber object */
        Set @update_sub = InvokeUpdate(@sub, @update_sub_status, @update_sub_errorcode, @options)

        /* Set the confirmation message */

        set @message = "Preferences have been updated<br><br>"


      /* Else, if the user unsubscribed, upsert their record accordingly (Clear the Preferences field and flag them as Mailable = "N") and log the unsubscribe */

        elseif @unsubscribe == "Yes" then

        UpsertDE("Master_PreferenceCenter_CustomerPreferences",@email,"Mailable","Y","Preferences",@preference,"Preferences","","Last_Modified_Date",@mod_date)

        /* set the reason for the unsubscribe */
        set @reason = "Custom Unsubscribe"

        /* initiate the LogUnsubEvent request */
        set @lue = CreateObject("ExecuteRequest")
        SetObjectProperty(@lue, "Name", "LogUnsubEvent")

        /* configure the properties of the API object */
        set @lue_prop = CreateObject("APIProperty")
        SetObjectProperty(@lue_prop, "Name", "SubscriberKey")
        SetObjectProperty(@lue_prop, "Value", @email)
        AddObjectArrayItem(@lue, "Parameters", @lue_prop)

        set @lue_prop = CreateObject("APIProperty")
        SetObjectProperty(@lue_prop, "Name", "EmailAddress")
        SetObjectProperty(@lue_prop, "Value", @email)
        AddObjectArrayItem(@lue, "Parameters", @lue_prop)

        set @lue_prop = CreateObject("APIProperty")
        SetObjectProperty(@lue_prop, "Name", "JobID")
        SetObjectProperty(@lue_prop, "Value", @jobid)
        AddObjectArrayItem(@lue, "Parameters", @lue_prop)

        set @lue_prop = CreateObject("APIProperty")
        SetObjectProperty(@lue_prop, "Name", "Reason")
        SetObjectProperty(@lue_prop, "Value", @reason)
        AddObjectArrayItem(@lue, "Parameters", @lue_prop)

        /* this is where the unsubscribe is performed */
        set @lue_statusCode = InvokeExecute(@lue, @overallStatus, @requestId)

        /* check status/errors */
        set @Response = Row(@lue_statusCode, 1)
        set @UStatus = Field(@Response, "StatusMessage")
        set @Error = Field(@Response, "ErrorCode")

        /* Set the confirmation message */

       set @message = "You have been unsubscribed.<br>To resubscribe, click the submit button below.<br><br>"

      else endif


      ]%%
  ```
      
For the form on the page, you can style it as desired, but within the `<form>` tags, the below code will need to be included. This code uses an AMPscript loop to display all of the preferences defined in the "Master_PreferenceCenter_PreferencesList" reference data extension. Within the loop, we also utilize a lookup to define whether or not the checkbox should be checked based on the user's previously defined preferences (if available).
  
   ```
       %%[
    var @preferences, @preference_name, @header, @i, @rows, @row, @checked

    /* The @preferences variable pulls in the value of the user's "Preferences" field in the master preference center data extension, if available. The @userlookupcount looks on the master preference center data extension to see if the record exists and sets the @existinguser variable accordingly. */

    set @preferences = Lookup("Master_PreferenceCenter_CustomerPreferences","Preferences","Email_Address",@email)
    set @userlookupcount = LookupRows("Master_PreferenceCenter_CustomerPreferences","Email_Address",@email)

    if @userlookupcount > 0 then 
      set @existinguser = "Y"
    else 
      set @existinguser = "N"
    else endif

    /* This sets up the loop to bring in all preferences listed in the preferences reference data extension. The @header field defines what the text above the preferences checkbox will be. Set this to whatever you'd like. */

    set @rows = LookupOrderedRows("Master_PreferenceCenter_PreferencesList",0, "Priority asc, Preference_Name","Key","X")
    set @header = "<b>Send me emails about:</b><br><br>"

    for @i = 1 TO RowCount(@rows) DO

      /* If this is the first row pulled, place the @header variable above the checkboxes */

      if @i == 1 then 
        output(v(@header))
      else endif

      /* Pulls in the preference_name field and applies it to the @preference_name variable */

      set @row = Row(@rows,@i)
      set @preference_name = Field(@row, "Preference_Name")


      /* Sets the checkbox as "checked" on the page if the user has previously opted-in to the preference or if the user has not yet defined their preferences */

      if IndexOf(@preferences,@preference_name) > 0 OR @existinguser == "N" then
        set @checked = "checked"
      else 
        set @checked = ""
      endif

      /* Sets the checkbox input field that will be displayed. All of the checkboxes will have the same name and will be submitted as one comma-delimited value */

      output(concat("<input type='checkbox' name='preference' ",@checked," value='",@preference_name,"'>",@preference_name,"<br>"))

    next @i ]%%
   
   ```
    
### Setting up the automation to update the customer master

1. This is a simple step. Setup an automation to run hourly. Within the automation have a SQL query that updates your customer master with the values from the master preference center data extension.


### Final step: watch the preferences roll in.
