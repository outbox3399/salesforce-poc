# salesforce-poc
This readme file depicts the POC steps of ingestion of data from Salesforce to Azure Data Lake Storage Gen2 using pipelines in Azure Data Factory (ADF).

Follow the below steps to configure your ADF and Salesforce.

We have to create a salesforce developer account using this URL - https://developer.salesforce.com/


===================================================================================
Keep the below data in the CONFIG FILE under "control-plane" container with the name metadata-config.json. We can add more objects names as an when required.

[
  {
    "ObjectName": "Account",
    "Fields": "Id, Name, Type, BillingCity, LastModifiedDate",
    "IncrementalField": "LastModifiedDate",
    "HasAttachments": true
  },
  {
    "ObjectName": "Contact",
    "Fields": "Id, FirstName, LastName, Email, SystemModstamp",
    "IncrementalField": "SystemModstamp",
    "HasAttachments": false
  }
]
===================================================================================
**FOR STRUCTURED DATA:**

**CREATE PERMISSION SET**

Log into Salesforce, click the Gear Icon (top right), and select Setup.
In the "Quick Find" search box on the left, type Permission Sets and click on it.
Click the New button.
Give it a clear label, such as Integration API Access. (Leave the "License" dropdown as "--None--" so it can be applied flexibly). Click Save.


1. Click the Cancel button on the screen you are currently on.
2. This will take you back to the main list of Permission Sets.
3. Look just below the "Integration_API_Access" view name. You will see a small button that simply says New (it has a little icon of a page with a plus/refresh symbol next to it).
4. Click that New button.
5. Now you will be on the correct "Create Permission Set" page. Type Integration_API_Access into the Label field and click Save.
6. Immediately after clicking Save, you will be taken to the overview page for your new Permission Set. Scroll down on that page, and you will finally see the System section with the System Permissions link!
From there, you can click System Permissions -> Edit -> check "API Enabled" -> Save.

===================================================================================
**Create the Integration User**
Now, you will create the non-human user account that Azure Data Factory will use.
1. In the Setup "Quick Find" box, type Users and select the Users menu item.
2. Click New User.
3. Fill out the basics:
    * First Name: Azure Data
    * Last Name: Factory Integration
    * Email: Use an IT distribution list or your own email (e.g., it-integrations@yourcompany.com).
    * Username: Must be formatted like an email and globally unique (e.g., adf-api@yourcompany.com.prod).
4. User License: Look for Salesforce Integration in the dropdown (Salesforce recently gave most companies 5 of these for free specifically for this purpose). If you don't see it, select the standard Salesforce license.
5. Profile: Select a highly restrictive profile. If you chose the Integration license, pick Salesforce API Only System Integrations. If you chose a standard license, pick Minimum Access - Salesforce.
6. Scroll to the bottom and ensure Generate new password and notify user immediately is checked.
7. Click Save.

===================================================================================


**Phase 4: Create an External Client App (The 2026 Method)**
**1.Create the App -**
1. On the screen you are currently on (from your screenshot), from LHS, see for App > App Manager > click the New External Client App button in the top right corner.
2. Fill in the basic information:
    * External Client App Name: ADF Integration
    * Contact Email: Your email address.
    * Distribution State: Select Local (since this is just for your specific Salesforce instance).
**2. Configure OAuth Settings**
1. Check the box for Enable OAuth Settings.
2. For the Callback URL, enter: https://login.salesforce.com/services/oauth2/success
3. Under Selected OAuth Scopes, find the following two scopes in the "Available" list and add them to the "Selected" list using the right arrow:
    * Manage user data via APIs (api)
    * Perform requests at any time (refresh_token, offline_access)
4. Ensure the Enable Client Credentials Flow box is checked (Azure Data Factory specifically uses this authentication flow).
5. Click Save or Create at the bottom.

===================================================================================

**3.Pre-Authorize Your Integration User:** Just like the old method, you need to tell Salesforce that your specific Integration User is allowed to use this app.
1. Open your newly created External Client App.
2. Go to the Policies tab.
3. Under the OAuth Policies section, click Edit.
4. Change the Permitted Users dropdown to Admin approved users are pre-authorized.
5. Now, Save it.
6. On the same page, under App Policies >  scroll to Permission Sets section, click edit, and add the “Integration API Access” permission set you created earlier and click save.

===================================================================================

**4.Get Your Azure Data Factory Keys**
1. Navigate to the Settings tab of your External Client App.
2. Expand the OAuth Settings section.
3. Click on Consumer Key and Secret (Salesforce may email you a verification code to type in before revealing this screen).
4. Copy your Consumer Key (Client ID) and Consumer Secret (Client Secret).

===================================================================================
**Create Connected APP:**
Goto External client app in LHS > settings in LHS under “External Client App” > New connected app

Under the Connected Apps section, there is a large blue button that says New Connected App. Salesforce recently updated this page layout, so instead of a toggle switch, they placed the creation button directly on this settings page.
You can click that blue button right now to jump straight into creating the app.
Here are the exact steps to follow once you click it:

**Step 1: Configure the New App**
1. Click the blue New Connected App button from your screenshot.
2. Connected App Name: ADF_Archival_Integration
3. Contact Email: (Enter your email address).
4. Scroll down to the API (Enable OAuth Settings) section and check the box.
5. Callback URL: Type https://localhost (ADF doesn't actually use this, but Salesforce requires a valid URL format here).
6. Selected OAuth Scopes: Move the following two from the left box to the right box:
    * Manage user data via APIs (api)
    * Perform requests at any time (refresh_token, offline_access)
7. CRITICAL: Check the box right below the scopes that says Enable Client Credentials Flow.
8. Scroll to the bottom and click Save (click Continue if Salesforce gives you a warning about it taking 2-10 minutes to take effect).

The below steps needs to be checked. As this step don’t fit here
**Step 2: Update Azure with the New Keys**
1. After saving, click the Manage Consumer Details button at the top of your new app to reveal your brand new Consumer Key (Client ID) and Consumer Secret (Client Secret).
2. Go to your Azure Key Vault and update your Salesforce-Client-Secret with this new one.
3. Go to your Azure Data Factory Linked Service for Salesforce and update the Client ID box with the new Consumer Key.
   
**Step 3: Assign the Execution User**
1. Back in Salesforce, go to the Quick Find box and search for App Manager.
2. Find your new ADF_Archival_Integration app, click the dropdown arrow on the far right, and select Manage.
3. Click the Edit Policies button at the top.
4. Scroll to the Client Credentials Flow section. In the Run As field, click the magnifying glass and select the Execution User we discussed previously (the user that has the ADF Archival Integration permission set).
5. Click Save.
Once these steps are done, your Azure Data Factory pipeline will be fully authenticated, authorized, and ready to archive your data without that INVALID_TYPE error!

===================================================================================

**Create a sample data in salesforce for the POC using the below steps**

**Mass Upload via Data Import Wizard**
Best for: Ingesting hundreds or thousands of rows from a CSV to test ADF volume, partitioning, and throughput.
If you generate a mock dataset (using a tool like Mockaroo or Excel), you can upload it in bulk.
1. Prepare your CSV: Create a simple CSV file with a header row. For Accounts, use headers like Name, Phone, BillingCity.
2. Open the Wizard: In Salesforce, click the Gear Icon > Setup. In the Quick Find search box (top left), type Data Import Wizard and select it.
3. Launch: Click the Launch Wizard! button.
4. Select Data:
    * Under "What kind of data are you importing?", choose Standard Objects.
    * Select the object you are uploading (e.g., Accounts and Contacts or Leads).
    * Under "What do you want to do?", choose Add new records.
5. Upload & Map:
    * Drag and drop your CSV file into the designated area.
    * Click Next. Salesforce will attempt to auto-map your CSV columns to Salesforce fields. If any are unmapped, click "Map" and select the correct field (e.g., map your CSV's "Account Name" column to the "Name" field).
6. Import: Click Next, then Start Import.
Tip for POCs: Start by importing just Accounts. Once they are in, you can extract them via ADF to ensure the pipeline logic (Bronze -> Raw extract, partitioning) works before spending time linking bulk Contacts or Opportunities.


===================================================================================

**Step 1: Create the Linked Service**
1. Open your Azure Data Factory Studio.
2. On the far-left sidebar, click the Manage icon (the toolbox/wrench icon).
3. Under the Connections section, select Linked services.
4. Click + New.
5. In the search box, type Salesforce and select the Salesforce V2 connector. Click Continue.
   
**Step 2: Map the Salesforce Credentials**
This is where most people get confused because the names in Salesforce don't perfectly match the names in ADF. Follow this mapping carefully:
1. Name: Give it a clear name (e.g., LS_Salesforce_Prod).
2. Authentication type: You must change this to OAuth2ClientCredentials.
3. Environment URL: * For a Developer Edition or Production, use: https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com (This you will get from the URL of salesforce)
4. Client ID: Paste your Consumer Key from the Salesforce “Connected App”.
5. Client Secret: Paste your Consumer Secret. (It is best practice to select "Azure Key Vault" here if you have it set up, but you can use "Factory password" for testing).
6. Salesforce API version = 53.0


===================================================================================
===================================================================================
**Key Vault - Service**
**1. Create the key vault first**

**How to Grant Yourself Access to Create Secrets**
Here is the step-by-step guide to fixing this so you can add your Salesforce credentials:
1. Stay in your Key Vault: Look at the left-hand menu of your Key Vault (pockeyvaultdiageo).
2. Go to IAM: Click on Access control (IAM).
3. Add Role Assignment: Near the top, click the + Add button, then select Add role assignment.
4. Select the Role:
    * You will see a list of roles. Search for and select Key Vault Secrets Officer (this role allows you to read, write, and delete secrets).
    * (Note: Do not select "Key Vault Secrets User" for yourself right now, as that only allows reading, not creating).
    * Click Next.
5. Select Members:
    * Under "Assign access to", keep it as User, group, or service principal.
    * Click + Select members.
    * A panel will slide out on the right. Search for your own name or email address (the account you are currently logged in with).
    * Click on your name so it appears in the "Selected members" section below, then click the Select button.
6. Review and Assign: Click the Review + assign button at the bottom, and then click it one more time to confirm.
7. Wait and Refresh: As the yellow warning box in your screenshot mentions, RBAC changes can take a minute or two to propagate. Wait about 60 seconds, go back to the Secrets blade on the left, and click the Refresh button at the top.
===================================================================================

**Phase 1: Storing Secrets in Azure Key Vault**
You need to store your sensitive connection details as "Secrets" within the Key Vault.
S**tep-by-step to add a secret:**
1. Open your Azure Portal and navigate to your newly created Azure Key Vault.
2. In the left-hand menu, under the Objects section, click on Secrets.
3. Click the + Generate/Import button at the top.
4. Leave the Upload options as Manual.
5. Fill in the Name and Value for each item you need to store. (See the exact list below).
6. Click Create. Repeat this for every secret.
**Exactly what to store (Recommended Naming Convention):**
* Name: Salesforce-Username | Value: Your Salesforce login email.
* Name: Salesforce-Password | Value: Your Salesforce password.
* Name: Salesforce-Security-Token | Value: The alphanumeric security token generated from Salesforce.
* Name: Salesforce-Client-Secret | Value: The consumer secret from your Salesforce Connected App.
* Name: ADLS-Connection-String | Value: The full connection string for your ADLS Gen2 storage account (found under "Access keys" in the Storage Account).

===================================================================================

**Phase 2: Granting ADF Access to the Key Vault**
Before ADF can read these secrets, you must give it permission. Azure uses Managed Identities to do this securely.
Step-by-step to grant access:
1. Still inside your Key Vault in the Azure Portal, go to Access control (IAM) on the left menu.
2. Click + Add and select Add role assignment.
3. In the Role tab, search for and select Key Vault Secrets User (this allows reading secrets, but not changing them). Click Next.
4. In the Members tab, select Managed identity for the "Assign access to" option.
5. Click + Select members.
6. A panel will slide out. Under "Managed identity", select Data Factory (V2), and click on the name of your specific ADF instance.
7. Click Select, then Review + assign.

===================================================================================

**Phase 3: Create the Key Vault Linked Service in ADF**
Now you must tell ADF where the Key Vault is.
Step-by-step to link the Key Vault:
1. Open Azure Data Factory Studio.
2. Click on the Manage tab (the toolbox icon on the bottom left).
3. Under Connections, select Linked services, then click + New.
4. Search for Azure Key Vault and select it.
5. Name: LS_AzureKeyVault.
6. Authentication method: Select System Assigned Managed Identity.
7. Azure subscription: Select your subscription.
8. Key vault name: Select the Key Vault you created from the dropdown.
9. Click Test connection (it should succeed because of Phase 2).
10. Click Create.

===================================================================================

**Phase 4: Using the Key Vault in your Pipeline Linked Services**
Finally, you will configure your Salesforce and ADLS Linked Services to fetch passwords directly from the Key Vault.
A. Configuring the Salesforce Linked Service:
1. Go to Manage > Linked services > + New > Salesforce.
2. Set up your environment URL (e.g., [https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com).
3. Under the fields for Password, Security Token, and Client Secret, you will see a toggle or radio button that says Azure Key Vault. Select it!
4. AKV Linked Service: Select LS_AzureKeyVault from the dropdown.
5. Secret name: Type in the exact name of the secret you created in Phase 1 (e.g., Salesforce-Password).
6. Repeat this process for the Security Token and Client Secret fields, pointing them to their respective secret names in the Key Vault.
7. Click Test connection and Create.

===================================================================================
**ADLS Gen2 Storage Account Linked Service** 

**Step 1: Give ADF permission on the Storage Account**
1. Go to your ADLS Gen2 Storage Account in the Azure Portal.
2. Click on Access Control (IAM) on the left.
3. Click + Add -> Add role assignment.
4. Select the role Storage Blob Data Contributor (this gives ADF permission to read/write data). Click Next.
5. Under Assign access to, choose Managed identity.
6. Click + Select members, find your Data Factory, select it, and click Review + Assign.
   
**Step 2: Create the Linked Service in ADF using Managed Identity**
1. In ADF, go to Manage > Linked services > + New > Azure Data Lake Storage Gen2.
2. Name it LS_ADLSGen2.
3. Under Authentication method, choose System Assigned Managed Identity (Instead of Account Key).
4. Select your Azure Subscription and Storage account name from the dropdowns.
5. Click Test connection and Create.

===================================================================================

The next major step is to create Datasets and build the Main Archival Pipeline that will read your JSON file and loop through the Salesforce objects.
Because you are using a metadata-driven approach, we will use Parameterized Datasets. This means instead of creating 40 different datasets for 40 Salesforce objects, you only create one dataset and pass the object name to it dynamically.
Here is the step-by-step guide for this phase.

**Phase 1: Create the Datasets**
You need to create three datasets. Go to the Author tab (the pencil icon) in Azure Data Factory, click the + button, and select Dataset.
1. Create the Metadata JSON Dataset This tells ADF where your control file is.
1. Select Azure Data Lake Storage Gen2 -> JSON -> Continue.
2. Name: DS_Metadata_JSON
3. Linked service: Select your ADLS Linked Service.
4. File path: Browse to the exact location of your metadata_config.json file in your control-plane container.
5. Click OK.

**2. Create the Parameterized Salesforce Dataset This acts as a reusable template for all 40 Salesforce objects.**
1. Click + -> Dataset -> Salesforce -> Continue.
2. Name: DS_Salesforce_Object_Dynamic
3. Linked service: Select your Salesforce Linked Service. Leave the "Object" dropdown blank for now and click OK.
4. Once the dataset opens, go to the Parameters tab (at the bottom).
5. Click + New. Name the parameter ObjectName and leave the Type as String.
6. Go back to the Connection tab. Next to the "Object" field, click Add dynamic content and select the parameter you just created: @dataset().ObjectName.

**3. Create the Parameterized ADLS Gen2 Parquet Dataset This acts as the reusable template for dropping the extracted data into the Bronze layer.**
1. Click + -> Dataset -> Azure Data Lake Storage Gen2 -> Parquet -> Continue.
2. Name: DS_ADLS_Bronze_Dynamic
3. Linked service: Select your ADLS Linked Service. Click OK.
4. Go to the Parameters tab. Click + New and name it FolderPath (Type: String).
5. Go to the Connection tab. In the "File path" section:
    * File system: Type your target container name (e.g., datalake).
    * Directory: Click Add dynamic content and select @dataset().FolderPath.



In ADLS Gen2, a "Container" is called a "File system".
Here is exactly how to fill out that section based on the screenshot you shared:
1. First Box (currently shows placeholder text "File system"): This is your Container. Type your target container name here (e.g., datalake or whatever you named the container where you want the bronze data to land).
2. Second Box (currently contains /@dataset().FolderPath): You have already added the dynamic content correctly here! However, you should remove the forward slash (/) that you typed before @dataset().FolderPath. The UI already adds a slash between the boxes automatically. Just leave it as exactly @dataset().FolderPath.
3. Third Box (currently shows placeholder "File name"): Leave this entirely blank. Because we are generating a dynamic folder path and writing Parquet files, ADF will automatically generate a unique file name for the output data inside that folder.

===================================================================================

**Phase 2: Build the Main Archival Pipeline (The Orchestrator)**
Now we will build the pipeline that reads the JSON and loops through the objects.
**Step 1: Create the Pipeline and Lookup Activity**
1. Under the Author tab, click + -> Pipeline -> Pipeline. Name it p_sf_to_adls_archival.
2. In the Activities pane on the left, expand General and drag a Lookup activity onto the canvas.
3. Name it: Lookup_Object_List.
4. Go to the Settings tab of the Lookup activity.
5. Source dataset: Select DS_Metadata_JSON.
6. CRITICAL: Uncheck the First row only box. This ensures ADF reads the entire list of 40 objects, not just the first one.
**Step 2: Add the ForEach Activity**
1. Drag a ForEach activity (under Iteration & conditionals) onto the canvas.
2. Connect the green success output from the Lookup_Object_List to the ForEach activity.
3. Select the ForEach activity and go to the Settings tab.
4. Items: Click Add dynamic content and paste the following expression: @activity('Lookup_Object_List').output.value (This tells the loop to iterate over the array of objects returned by the JSON).
**Step 3: Configure the Copy Activity inside the Loop**
1. Select the ForEach activity and click the Edit activities button (the pencil icon directly on the activity). You are now inside the loop.
2. Drag a Copy data activity onto the canvas. Name it Copy_SF_Structured_Data.
3. Go to the Source tab:
    * Source dataset: Select DS_Salesforce_Object_Dynamic.
    * Dataset properties (ObjectName): Click Add dynamic content and enter: @item().ObjectName
    * Use query: Select Query.
    * Query expression: Click Add dynamic content. Based on your architecture (archiving data older than 3 months), use this SOQL expression: @concat('SELECT ', item().Fields, ' FROM ', item().ObjectName, ' WHERE LastModifiedDate < LAST_N_MONTHS:3')
4. Go to the Sink tab:
    * Sink dataset: Select DS_ADLS_Bronze_Dynamic.
    * Dataset properties (FolderPath): Click Add dynamic content. To match the folder structure in your architecture (Bronze / Structured / <Object Name> / Year / Month), use this expression: @concat('bronze/structured/', item().ObjectName, '/', formatDateTime(utcNow(), 'yyyy/MM'))

===================================================================================

**We will encounter an error in salesforce related to permission set. Use the below steps to resolve it.**
**Why this error is happening**
To save money and increase security, Salesforce recently created the "Salesforce Integration" user license specifically for API users (like your Azure Data Factory). However, by default, the base version of this license is "locked down" and is not allowed to access standard CRM objects like Accounts, Contacts, or Opportunities.
Because your Integration_API_Access Permission Set is already assigned to the Factory, Azure Data user, Salesforce is blocking you from adding Account permissions to it, saying, "Wait, the user holding this Permission Set isn't allowed to see Accounts!"
The Fix: Assign the "Permission Set License"
To unlock standard object access for an integration user, you simply have to assign them a specific "Permission Set License" (PSL) first. Think of a PSL as an expansion pack that upgrades the base user license.
Here is the step-by-step fix to make that error go away permanently:
Step 1: Upgrade the User's License
1. In Salesforce Setup, go to the Quick Find box and type Users, then click on Users.
2. Click directly on the name of your integration user: Factory, Azure Data (do not click Edit, click the name to view the profile page).
3. Scroll down the page until you find the related list section called Permission Set License Assignments.
4. Click the Edit Assignments button.
5. Look through the list and check the box next to Salesforce API Integration. (This is the magic key that unlocks standard objects for API users).
6. Click Save.
Step 2: Update the Permission Set Now that the user is officially allowed to see standard objects, Salesforce will let you save your permission set!
1. Go back to your Integration_API_Access Permission Set.
2. Go to Object Settings -> Accounts.
3. Click Edit.
4. Check Read and View All under Object Permissions (and any field permissions you need).
5. Click Save.
You will no longer get that red error screen, and Azure Data Factory will finally be fully authorized to read your Account data!

===================================================================================
===================================================================================
===================================================================================

**FOR UNSTRUCTURED DATA:** pdf, csv, image etc

**Attach Files (csv, image, pdf, excel etc..) under 3 to 4 accounts in Salesforce**
1. Goto salesforce > click 9 dots in the top LHS 
2. Under App launcher > search for word “Accounts”
3. Click the “Recently Viewed” > Select “All Accounts”
4. Click on the first account name link > scroll down to “Notes & Attachments”
5. Click on Upload files > select document e.g. CSV or PDF etc and click upload.
6. Similarly, add more attachments in other accounts as well.

===================================================================================

Handling files is very different from handling rows of data. In Salesforce, attachments aren't stored inside the Account table itself; they are stored in a separate system object (traditionally called Attachment). Furthermore, Azure Data Factory (ADF) has a strict rule: you cannot put a ForEach loop inside another ForEach loop.
Because your main pipeline is already looping through the Objects (Account, Contact), we must build a Child Pipeline specifically for attachments.
Here is the exact step-by-step guide to building this unstructured data architecture.


P**hase 1: Create the REST Linked Service**
The standard Salesforce connector we used earlier is designed for extracting tables (structured data). To extract raw files (binary data), we must talk directly to the Salesforce API using a REST connector.
1. Go to Manage -> Linked services -> + New.
2. Search for REST and select it. Name it LS_Salesforce_REST.
3. Base URL: https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com (from your screenshot).
4. Authentication type: Select OAuth 2.0 Client Credential.
5. Token endpoint: https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com/services/oauth2/token
6. Client ID: Paste your Consumer Key (from your ADF Integration Manoj app).
7. Client secret: Use Azure Key Vault and select your secret.
8. Click Test connection and Create.
===================================================================================

**Phase 2: Create the Binary Datasets**
We need datasets that understand raw files (like PDFs and Images) instead of Parquet files.
1. Create the REST Source Dataset (The Download Link)
1. Go to the Author tab -> + -> Dataset -> REST.
2. Name it DS_Salesforce_Attachment_REST.
3. Select LS_Salesforce_REST as the Linked Service. Click OK.
4. Go to the Parameters tab and create a new parameter named AttachmentId (Type: String).
5. Go to the Connection tab. In the Relative URL field, add this dynamic content: /services/data/v52.0/sobjects/Attachment/@{dataset().AttachmentId}/Body
2. Create the ADLS Sink Dataset (The Destination)
1. Go to + -> Dataset -> Azure Data Lake Storage Gen2 -> Binary.
2. Name it DS_ADLS_Bronze_Unstructured.
3. Select your ADLS Linked Service. Click OK.
4. Go to the Parameters tab. Create three parameters (all Type: String):
    * ObjectName
    * ParentId (The Salesforce Row ID)
    * FileName
5. Go to the Connection tab. Set up the file path dynamically to match your architecture diagram:
    * File system: datalake (or your container name)
    * Directory: @concat('bronze/unstructured/', dataset().ObjectName, '/', dataset().ParentId)
    * File name: @dataset().FileName


===================================================================================

**Phase 3: Build the Attachment Child Pipeline**
This pipeline will take an Object Name (like "Account"), find all its attachments, and download them.
1. Go to + -> Pipeline -> Pipeline. Name it p_extract_sf_attachments.
2. Go to the Parameters tab of this new pipeline and create a parameter named ObjectName (Type: String).
3. Drag a Lookup activity onto the canvas. Name it Lookup_Attachment_List.
    * Source dataset: Select your existing DS_Salesforce_Object_Dynamic.
    * Dataset properties (ObjectName): Type Attachment (hardcoded).
    * Use query: Select Query.
    * Query expression: Paste this SOQL query to find all attachments for the specific object: @concat('SELECT Id, ParentId, Name FROM Attachment WHERE Parent.Type = ''', pipeline().parameters.ObjectName, '''')
    * Ensure "First row only" is unchecked.
4. Drag a ForEach activity onto the canvas and connect the Lookup to it.
    * Items: @activity('Lookup_Attachment_List').output.value
5. Click the pencil icon on the ForEach activity to go inside it.
6. Drag a Copy data activity onto the canvas. Name it Copy_Binary_File.
    * Source tab: Select DS_Salesforce_Attachment_REST. For the AttachmentId parameter, enter: @item().Id
    * Sink tab: Select DS_ADLS_Bronze_Unstructured. Fill in the three parameters:
        * ObjectName: @pipeline().parameters.ObjectName
        * ParentId: @item().ParentId
        * FileName: @item().Name

**Phase 4: Connect it to your Main Pipeline**
Now we just tell your main pipeline to trigger this new child pipeline whenever it sees "HasAttachments": true in your JSON file.
1. Open your main pipeline (p_sf_to_adls_archival).
2. Go inside your existing ForEach loop.
3. Drag an If Condition activity onto the canvas. Name it Check_For_Attachments.
4. Connect the green success arrow from your structured copy activity to this If Condition.
5. In the If Condition Activities (or Settings) tab, set the Expression to: @equals(item().HasAttachments, true)
6. Click the pencil icon inside the True box of the If Condition.
7. Drag an Execute Pipeline activity onto the canvas.
    * Invoked pipeline: Select p_extract_sf_attachments.
    * Expand the Parameters section. For the ObjectName parameter, click Add dynamic content and enter: @item().ObjectName

===================================================================================

**Why this error happened:**
ADF has a hard rule: If your destination (Sink) is a Binary dataset (raw files like PDFs, Images, etc.), your Source must also be a Binary dataset. The REST connector we used earlier is specifically designed to parse structured JSON/XML data. Because it isn't classified as a "Binary" source, ADF blocks the pairing. To download raw files, we must use the HTTP Connector instead of the REST connector.
Because the HTTP connector doesn't have a built-in button for the OAuth2 Client Credentials flow, we will use a very standard ADF workaround: we will fetch the token using a Web Activity, and pass it dynamically to the HTTP connection.
Here is the step-by-step fix to get this publishing and testing:

**Step 1: Create the HTTP Linked Service & Dataset**
First, we build the connection that allows raw binary downloads.
1. Create HTTP Linked Service:
* Go to Manage -> Linked Services -> + New.
* Search for HTTP and select it.
* Name: LS_Salesforce_HTTP
* Base URL: https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com
* Authentication type: Select Anonymous. (We are setting this to anonymous because we will manually inject the secure token in Step 3).
* Click Create.
**2. Create HTTP Binary Dataset:**
* Go to Author -> + -> Dataset -> HTTP -> Binary.
* Name: DS_Salesforce_Attachment_Binary_HTTP
* Linked service: Select LS_Salesforce_HTTP. Click OK.
* Go to the Parameters tab and create a new parameter named AttachmentId (Type: String).
* Go to the Connection tab. In the Relative URL box, click Add dynamic content and enter: @concat('/services/data/v52.0/sobjects/Attachment/', dataset().AttachmentId, '/Body')

**Step 2: Add a Web Activity to Get the Token**
Now, we need to generate the authentication token right before we extract the attachments.
1. Open your child pipeline: p_extract_sf_attachments.
2. Drag a Web activity onto the canvas (under the General section). Name it Get_SF_Token.
3. Connect the green success arrow from Get_SF_Token to your existing Lookup_Attachment_List activity. (This ensures we get the token before looking for files).
4. Click on Get_SF_Token and go to the Settings tab:
    * URL: https://orgfarm-fc8c4b2384-dev-ed.develop.my.salesforce.com/services/oauth2/token
    * Method: POST
    * Headers: Click + New.
        * Name: Content-Type
        * Value: application/x-www-form-urlencoded
    * Body: You need to paste your credentials here in this exact format (replace the placeholder text with the actual Client ID and Secret you used in your Salesforce Linked Service): grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_SECRET
    * Secure Output: Check this box! (This hides the token from the ADF logs for DPDP compliance).

**Step 3: Update the Copy Activity**
Finally, we swap out the source in your Copy Activity and pass it the token.
1. Go inside your ForEach loop in the p_extract_sf_attachments pipeline.
2. Click on your Copy_Binary_File activity.
3. Go to the Source tab:
    * Source dataset: Change this to your new DS_Salesforce_Attachment_Binary_HTTP.
    * AttachmentId: Click Add dynamic content and enter @item().Id.
    * Request method: Select GET.
    * Additional headers: This is the magic step. Click to add dynamic content and paste this exactly: “Authorization: Bearer @{activity('Get_SF_Token').output.access_token}”
**Step 4: Publish and Test!**
Because both the Source and Sink are now classified as "Binary", that validation error will instantly disappear.

===================================================================================

**Step 1: Remove the Hardcoded Secret**
Let's get this saving successfully right now so you don't lose your work.
1. Click on your Get_SF_Token Web Activity.
2. Go to the Settings tab.
3. In the Body field, delete your actual secret and replace it with a dummy word for now. It should look something like this: grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=DUMMY_TEXT
4. Click Save All at the top. (Because the real secret is gone, GitHub will now allow the save!)

**Step 2: Fetch the Secret from Azure Key Vault**
Now we will use a Web Activity to securely grab your secret directly from the Key Vault right before we ask Salesforce for the token.
1. Drag a new Web activity onto the canvas. Place it before your Get_SF_Token activity.
2. Name it Get_AKV_Secret.
3. Connect the green success arrow from Get_AKV_Secret to your Get_SF_Token activity.
4. Click on Get_AKV_Secret and go to the Settings tab:
    * URL: https://[YOUR-KEY-VAULT-NAME].vault.azure.net/secrets/[YOUR-SECRET-NAME]?api-version=7.0 (Replace the bracketed text with your actual Key Vault name and Secret name).
    * Method: GET
    * Authentication: Select System Assigned Managed Identity.
    * Resource: https://vault.azure.net
5. Go to the General tab for this activity and check Secure output (so the secret doesn't print in the logs).

**Step 3: Inject the Secret into the Salesforce Request**
Now we pass the hidden secret from Activity 1 into Activity 2.
1. Click on your Get_SF_Token Web Activity again.
2. Go to the Settings tab.
3. Delete everything in the Body box, click Add dynamic content, and paste this exact formula (make sure to replace YOUR_CLIENT_ID with your actual Client ID): @concat('grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=', activity('Get_AKV_Secret').output.value)
4. Click OK.
5. Go to the General tab for this activity and ensure Secure input and Secure output are both checked.
Click Publish All. Your pipeline is now enterprise-grade and 100% compliant with security policies. You can now safely trigger your debug run to test the unstructured data extraction!

The pipeline will fail after following above steps. 

===================================================================================

**Rewrite and correct the below:**

our pipeline ran perfectly. If you look at your first screenshot, your ForEach activity succeeded but only took 2 seconds to run. This means the Lookup activity successfully queried Salesforce, but Salesforce returned 0 records. The loop ran zero times, which is why your pipeline succeeded but no folder was created.
Here is exactly why this happens and how to fix it.

**The Trap: "Attachments" vs. "Files"**
In older versions of Salesforce (Salesforce Classic), when you uploaded a document, it was stored in a database table called Attachment.
However, in modern Salesforce (Lightning Experience), when you click "Upload Files" in the Notes & Attachments related list (like you did in your screenshot), Salesforce no longer uses the Attachment table. Instead, it converts it into a modern "Salesforce File," which uses a completely different database table called ContentVersion.
Because our pipeline is currently querying the old Attachment table, it can't see the Delhi_csv file you just uploaded!
To make this enterprise-ready, we need to point your pipeline to the modern ContentVersion table.


**How to Fix the Pipeline for Lightning Files**
**Step 1: Update the Lookup Query** We need to tell the Lookup activity to search the modern files table.
1. In your p_extract_sf_attachments pipeline, click on your Lookup_Attachment_List activity.
2. Go to the Settings tab.
3. Replace your old query with this new one: SELECT Id, PathOnClient, FirstPublishLocationId FROM ContentVersion WHERE IsLatest = true (Note: PathOnClient contains the full file name like "Delhi.csv", and FirstPublishLocationId contains the ID of the Account you attached it to).
**Step 2: Update the Download URL** The REST API endpoint for downloading modern files is slightly different than classic attachments.
1. Go to your datasets and open DS_Salesforce_Attachment_Binary_HTTP.
2. Go to the Connection tab.
3. Change the dynamic content in the Relative URL box to this: @concat('/services/data/v52.0/sobjects/ContentVersion/', dataset().AttachmentId, '/VersionData')
**Step 3: Update the Copy Activity Mapping** Finally, we just need to map the new column names from ContentVersion into your variables.
1. Go back into your p_extract_sf_attachments pipeline and click into the ForEach loop.
2. Click on your Copy_Binary_File activity.
3. Go to the Sink tab. Update your Dataset properties:
    * ParentId: Change this to @item().FirstPublishLocationId
    * FileName: Change this to @item().PathOnClient
Test It Again!
Click Publish All, and then hit Debug.

===================================================================================

**Unstructured Folder is not getting created, we have to provide permission by checking the checkbox of “Query All Files”. However, this is not visible under the permission set “Integration_API_Access”. Below is the reason.**

Why is "Query All Files" missing?
When you originally created the Integration_API_Access Permission Set, you likely selected "Salesforce Integration" from the License dropdown.
Salesforce assumes that users with this license shouldn't need certain global permissions, so it completely hides them from the UI. However, because Azure Data Factory is an enterprise ETL tool, it absolutely needs this global access to archive your data properly.

The Fix: The "No License" Workaround
To bypass this UI restriction, we simply need to create a secondary Permission Set that isn't tied to any specific license, and assign it to your integration user.
Step 1: Create the Unrestricted Permission Set
1. In Salesforce Setup, go back to the main Permission Sets list.
2. Click the New button at the top.
3. Label: Type Integration_File_Access.
4. CRITICAL STEP: Look at the License dropdown at the bottom. Make sure it is set to --None-- (Do not select Salesforce Integration).
5. Click Save.
   
Step 2: Enable the Permission Because this new Permission Set has no license restrictions, the entire Salesforce security dictionary is unlocked.
1. In your new Integration_File_Access permission set, scroll down and click System Permissions.
2. Click Edit.
3. Scroll down to the Q's. Query All Files will now be sitting right there between "Queue Manager" and "Quip Metrics"!
4. Check the box for Query All Files and click Save.
   
Step 3: Assign it to your ADF User
1. Go to the Quick Find box and type Users, then click Users.
2. Click on Factory, Azure Data.
3. Scroll down to Permission Set Assignments and click Edit Assignments.
4. Move your new Integration_File_Access from the left column to the right column.
5. Click Save.
The Final Test!
Your integration user now has the admin-level file visibility required.

===================================================================================

**#ISSUE which I encountered:**

**Note: Creating new “permission set” will impact the main pipeline which is now dealing with the structured data.**

Based on the architecture we just built, your Integration_API_Access Permission Set is the master key for your Structured Data.
Here is exactly what it impacts across your pipeline, what you need to remember for the future, and what it doesn't impact:
**1. It Controls the Main Archival Loop (Structured Data)**
This permission set is directly responsible for allowing Azure Data Factory to read your standard CRM records (rows and columns).
* Current Impact: Right now, it is explicitly granting Read and View All access to your Account and Contact objects, allowing the dynamic SOQL query to extract them into Parquet files.
* Future Impact: Because your JSON file contains 40 objects, you will need to update this specific permission set 38 more times. Every time you add a new object (like Opportunity, Lead, Case) to your metadataConfig.json file, you must go into Integration_API_Access -> Object Settings and check the "Read" boxes for that object and its fields, otherwise the pipeline will fail with an INVALID_TYPE error.
**2. It Controls the "Write-Back" Deletion Flag**
In Phase 5 of our build, we added the Update_SF_Deletion_Flag activity to write To_be_deleted__c = true back into Salesforce.
* The Impact: Because this permission set controls object access, it also controls write access.
* Action Item: For that write-back step to succeed, you must ensure that inside Integration_API_Access -> Object Settings -> Accounts (and Contacts, etc.), you have checked the Edit box under Object Permissions, AND the Edit Access box next to your custom To_be_deleted__c field.
**3. It Controls the API Authentication**
Because this Permission Set is assigned directly to the Factory, Azure Data user, it is the fundamental reason your Azure Data Factory Linked Services (both the standard Salesforce connector and the HTTP REST connector) are allowed to log in via the OAuth2 Client Credentials flow in the first place.

**What it does NOT impact: Unstructured Files**
It is very important to remember the split we made at the very end of our session:
* Integration_API_Access handles the tables, rows, columns, and write-backs.
* Integration_File_Access (the second permission set we created with No License) handles the Unstructured Data. It contains the "Query All Files" system permission that allows your child pipeline (p_extract_sf_attachments) to see and download the physical PDFs, CSVs, and images from the ContentVersion table.

===================================================================================

**Questions raised:
can't we use "Integration_File_Access" permission set to handle everything for structured and unstructured?
Which all places I need to update what? Also, is this the appropriate way to handle all the permissions in a single permission set?**

The short answer is: Yes, you technically can consolidate them. Because you set the license on Integration_File_Access to "None," it has access to the entire Salesforce security dictionary. It can hold both System Permissions (Files) and Object Settings (Accounts/Contacts) at the same time.
However, to answer your second question: No, consolidating them into a single Permission Set is generally not the appropriate way to handle enterprise Salesforce security.
If you were to present this architecture in a KPMG technical review, Salesforce Architects would strongly recommend keeping them as two separate Permission Sets. Here is exactly why, and what you would need to do if you still chose to combine them.

**Why Keeping Them Separate is the "Enterprise Best Practice"**
In modern Salesforce architecture, the golden rule is Composable Security (creating small, task-specific permission sets rather than massive, catch-all ones). Here is why keeping your current two-part setup is better:
**1. Reusability (The Lego Block Method)** Right now, you have one block for "Read CRM Data" and one block for "Download All Files." Imagine next month, your team builds a PowerBI dashboard that needs to connect to Salesforce to read Account rows, but it has no business downloading secure PDFs. Because you kept them separate, you can easily assign the PowerBI user the Integration_API_Access set, and simply withhold the File access. If you combine them, you are forced to give PowerBI dangerous file permissions just so it can read rows.
**2. Auditing and "Blast Radius"** Query All Files is considered a high-risk system permission. Security teams heavily audit who has it. If it is isolated in a permission set explicitly named Integration_File_Access, it is perfectly transparent. If you hide it inside a massive permission set that also controls 40 different data tables, it becomes much harder for security teams to audit exactly why a user has that permission.
**3. Future-Proofing against Salesforce Updates** Salesforce is aggressively pushing the new "Salesforce Integration" user licenses to tighten security. Right now, using a "No License" permission set to bypass UI restrictions is a known workaround. However, keeping your core data access (Objects/Fields) strictly tied to the official Integration license ensures your pipeline won't suddenly break if Salesforce patches that file loophole in a future release.


**Finally, Keeping two permission sets separately. One for structured data and another for unstructured data.**

===================================================================================

**Step 1: Create the "No-License" Permission Set**
First, we create the container. The secret here is leaving the license blank so Salesforce doesn't hide any system permissions from us.
1. Go to your Salesforce Setup (the gear icon).
2. In the Quick Find box, type Permission Sets and select it.
3. Click the New button at the top of the list.
4. Label: Type Integration_File_Access (the API Name will auto-fill).
5. Description: Type something helpful like "Grants global read access to ContentVersion files for ADF."
6. CRITICAL STEP: Look at the License dropdown at the bottom. Ensure it is set to --None--. Do not select Salesforce Integration.
7. Click Save.

**Step 2: Grant the "Query All Files" Power**
Now that we have an unrestricted permission set, the missing permission will finally be visible.
1. You should now be on the overview page for your new Integration_File_Access permission set.
2. Scroll down to the bottom section and click on System Permissions.
3. Click the Edit button at the top.
4. Scroll down the alphabetical list to the "Q" section.
5. Find Query All Files and check the box next to it.
6. Click Save at the top. (Salesforce might pop up a warning box saying this will enable other dependent permissions—just click Save again to confirm).

**We will unable to see the “Query All files” even though the LICENSE is set to NONE. Therefore, we found a work around. See below steps.**

If you successfully set the license to --None-- and your specific Developer Edition org still refuses to show "Query All Files" (which can occasionally happen depending on the org's patch version), there is a standard workaround for Integration architectures.
Instead of "Query All Files", you can grant the master "View All Data" permission.
1. Inside your Integration_File_Access permission set, go to System Permissions and click Edit.
2. Scroll all the way down to the "V" section.
3. Check the box for View All Data.
4. Click Save.

===================================================================================

I**n Salesforce, Assign the Key to your ADF User**
Now we need to hand this new key to the Azure Data Factory user account so it can use it alongside your API key.
1. In the Quick Find box on the left, type Users and click on the Users menu.
2. Find your integration user in the list (you mentioned it was named something like Factory, Azure Data) and click on their name.
3. Scroll down the user's profile page until you see the Permission Set Assignments section.
4. Click the Edit Assignments button.
5. In the left column (Available Permission Sets), find your new Integration_File_Access.
6. Select it and click the Add arrow (the right-pointing triangle) to move it to the right column (Enabled Permission Sets).
    * Note: Ensure your original Integration_API_Access is also sitting in that right column. The user needs to hold both!
7. Click Save.
===================================================================================

**Still we are unable to see the UNSTRUCTURED folder. Therefore, we are doing the below steps by giving highest privilege as “Modify All Data” in salesforce.**

**Step 1: Update your existing "Integration_File_Access"**
Let's apply the master key to the permission set you already have.
1. In Salesforce Setup, go to Permission Sets and click on your existing Integration_File_Access.
2. Scroll down and click System Permissions.
3. Click Edit.
4. Scroll down to the "M" section and check the box for Modify All Data. (Click Save to confirm the pop-up warning).
5. Click Save at the top of the screen.


===================================================================================

**Suspect 1: The "Wrong User" Linked Service**
We spent a lot of time forging the ultimate master key for your Factory, Azure Data user. But we need to make absolutely sure Azure Data Factory is actually using that specific user to run the Lookup query.
1. On the far-left menu of Azure Data Factory, click the Manage icon (the little toolbox).
2. Under "Connections", click on Linked services.
3. Find your standard Salesforce connection (the one your Dataset uses) and click on it to edit.
4. Look closely at the Username box.
    * The Trap: If it has your personal admin email in there, ADF is completely ignoring all the permissions we just built for the Integration user!
    * The Fix: Change the username to the exact Integration Username from your Salesforce screen (it looked like sahumanoj3399@gmail.com... in your previous screenshot), update the password/security token, and click Save.

**Note: Here we are not using the username and password, so we are using Client ID and Client Secret only.
Below are the details.**

Using the OAuth 2.0 Client Credential flow is actually the highest standard for enterprise security, so you built the Linked Service perfectly. However, because you are using a Client ID and Secret instead of a Username and Password, it changes how Salesforce applies our security settings.
Here is the secret: When Azure Data Factory knocks on Salesforce's door with just a Client ID, Salesforce looks at the "Connected App" to see who is knocking. If the Connected App is not explicitly told to "Run As" your Integration User, it defaults to a background system user that has zero file permissions!
None of the master keys we built earlier are working because Azure is accidentally knocking on the door wearing the wrong name tag.
Here is how to check your Connected App and force it to use your master key, along with checking the results of your Attachment test:

**Step 1: The Connected App "Run As" Check**
We need to tell your Salesforce Connected App to physically adopt the identity of the Factory, Azure Data user whenever it sees your Client ID.
1. Go back to your Salesforce Setup.
2. In the Quick Find box, type App Manager and click on it.
3. Scroll through the list and find the Connected App you created for this Azure integration (it will be the one where you originally generated that Client ID).
4. Click the dropdown arrow on the far right of that specific app's row, and click Manage (Do not click View or Edit, you must click Manage).
5. On the Manage screen, click the Edit Policies button at the top.
6. Scroll down to the Client Credentials Flow section.
7. Look for the Run As field. Click the magnifying glass and select your Factory, Azure Data user.
8. Click Save.

===================================================================================
This is a massive breakthrough! Your screenshot perfectly confirms that the identity crisis is officially solved.
Before you made this change, Salesforce was basically saying: "I see a valid Client ID, but I don't know which user this is, so I'm giving it zero file permissions."
Now, every time Azure Data Factory knocks on the door, Salesforce will say: "Ah, I see this is the Azure Data Factory user. This user has the 'Modify All Data' master key and the 'CRM Content User' badge. Let them see everything!"
Now that the permissions are perfectly aligned, we just need to do the final switch back to our modern files query and run the pipeline.
**Step 1: Switch back to ContentVersion**
Since we temporarily changed the query to Attachment in the last step just to test things, let's point it back to the modern Lightning files table where your Delhi_csv is actually hiding.
1. Go back to your Azure Data Factory authoring canvas.
2. Open your child pipeline: p_extract_sf_attachments.
3. Click on your Lookup_Attachment_List activity and go to the Settings tab.
4. Change the dataset parameter ObjectName back to ContentVersion.
5. Change your Query back to our modern file query: SELECT Id, PathOnClient, FirstPublishLocationId FROM ContentVersion WHERE IsLatest = true
6. Click Publish All.

===================================================================================

**Followup question: I am seeing this under "Connected Apps OAuth Usage", I don't see "Azure Data Factory connected app."**

This is a fantastic catch, and it reveals a tiny bit of "recycled code" in your architecture!
Because Azure Data Factory just successfully connected to Salesforce a few minutes ago (even though it returned 0 rows), it must have an active OAuth token.
Look closely at your list: the only app with an active token right now is the Databricks Ingestion Connector.
Because you are working on a POC, it is highly likely you created that Connected App earlier for a Databricks pipeline, and simply reused its Client ID and Secret when you set up your Azure Data Factory Linked Service. (This is totally fine and a very common shortcut!)
This means the "Databricks" app is actually the one Azure Data Factory is using in disguise. Let's kill its token!

S**tep 1: Revoke the "Databricks" Token**
1. On that exact screen in your screenshot, look at the User Count column for the Databricks Ingestion Connector.
2. Click directly on the number 1.
3. This will open up the active session details. You will see a button to Revoke the token. Click it!
**Step 2: Force the Refresh in Azure Data Factory**
Now we will make that tiny change to your Linked Service to force ADF to realize the old token is dead and ask for a new one.
1. In Azure Data Factory, go to the far-left menu and click the Manage icon (the toolbox).
2. Click on Linked services and open your LS_Salesforce connection.
3. Scroll down to the Salesforce API version box. It likely says 52.0. Change this to 53.0 (or 54.0).
4. Click the Test connection button at the bottom. (This is the critical moment! ADF will reach out, get rejected because we revoked the old token, and instantly request a brand new one using your updated "Run As" identity).
5. Once it says Connection Successful, click Save.
**Step 3: Publish and Debug!**
1. Click Publish All at the top of your ADF screen to save the API version change.
2. Go back to your parent pipeline (p_sf_to_adls_archival).
3. Click Debug.

===================================================================================

Okay, Salesforce is officially guarding this file like it's the gold in Fort Knox! You have uncovered one of the most stubborn security limitations in the entire Salesforce architecture.
The error in your screenshot confirms that Salesforce flat-out refuses to let ContentDocumentLink exist inside any parenthesis (a semi-join), period. It doesn't matter if we query ContentVersion or ContentDocument—Salesforce blocks it.
But don't worry, we are data engineers, and there is always a backdoor. We are going to completely eliminate the subquery and use a Relationship Query (Dot Notation).
Instead of doing a complex nested query, we will just query the ContentDocumentLink table directly, and use a "dot" to reach up and pull the exact ID we need from its parent!

**The Ultimate Backdoor Query**
1. Go back to the Settings tab of your Lookup_Attachment_List activity.
2. Change the dataset parameter ObjectName to ContentDocumentLink.
3. Paste this exact SOQL query. Notice the dot notation (ContentDocument.LatestPublishedVersionId):SQL
The Final Preview
Click Preview data.

Because this is a completely flat query with no nested SELECT statements, it completely bypasses the MALFORMED_QUERY block. And because we are querying the Link table directly with the Account ID, it completely satisfies the Salesforce security model.

SELECT ContentDocumentId, ContentDocument.LatestPublishedVersionId, ContentDocument.Title FROM ContentDocumentLink WHERE LinkedEntityId = '001NS00002hH7WNYA0'
SELECT ContentDocumentId, ContentDocument.LatestPublishedVersionId, ContentDocument.Title FROM ContentDocumentLink WHERE LinkedEntityId IN (SELECT Id FROM Account)

**As we have to make it dynamic, we will use the below query:**x`

**@concat('SELECT ContentDocumentId, ContentDocument.LatestPublishedVersionId, ContentDocument.Title FROM ContentDocumentLink WHERE LinkedEntityId IN (SELECT Id FROM ', pipeline().parameters.ObjectName, ')')**

===================================================================================
**The below is also an issue which was not fixed.**

It means our "backdoor" query was a 100% success. The Lookup activity successfully bypassed Salesforce security, grabbed the file metadata, and handed it directly to your ForEach loop.
The pipeline only failed because we changed the column names in our SOQL query to bypass the security, but we forgot to tell the Copy_Binary_File activity about the new names. It is still looking for the old item().Id column, which no longer exists.
We just need to update the dynamic expressions inside the Copy Activity to match our new query.

**Step 1: Update the Source (The File ID)**
The Copy Activity uses the ID to know which binary file to download from Salesforce. We need to point it to the new Version ID column.
1. Go to your child pipeline (p_extract_sf_attachments).
2. Double-click on the ForEach1 activity box to open the inner canvas.
3. Click on your Copy_Binary_File activity.
4. Go to the Source tab.
5. Find the dynamic content box where you previously wrote @item().Id (this might be inside your Source Dataset parameters or a Relative URL field).
6. Replace item().Id with the exact property name from the error message: @item().ContentDocument.LatestPublishedVersionId
   
**Step 2: Update the Sink (The File Name)**
Your Copy Activity also uses a dynamic expression to name the file when it lands in Azure. Our old query used PathOnClient, but our new query uses Title.
1. While still on the Copy_Binary_File activity, click the Sink tab.
2. Find the dynamic content box where you set the filename (likely referencing @item().PathOnClient).
3. Replace it with: @item().ContentDocument.Title
(Note: Because the Salesforce "Title" field usually does not include the file extension, you may want to manually append the extension so it opens correctly in Azure. E.g.,
@{item().ContentDocument.Title}.csv)

**Step 3: The Final Run!**
1. Click Publish All to save these expression updates.
2. Go back to your parent pipeline (p_sf_to_adls_archival).
3. Hit Debug one last time.

===================================================================================

**The error message gives us the exact answer: property 'FirstPublishLocationId' doesn't exist.**
Because we completely changed our SOQL query to query the ContentDocumentLink table instead of the buggy ContentVersion table, the column FirstPublishLocationId is no longer being returned by Salesforce.
It looks like your Copy_Binary_File activity is still trying to use that old ID (likely to name a folder in your Data Lake to organize files by Account). In the ContentDocumentLink table, the equivalent field is called LinkedEntityId.
Here is how we do the final cleanup to make this work:

**Step 1: Add the new ID to the Lookup Query**
First, we need to tell Salesforce to actually output that new Account ID column so ADF can use it.
1. Go back to your Lookup_Attachment_List activity in the child pipeline.
2. Go to the Settings tab.
3. Update your dynamic SOQL query to include , LinkedEntityId. Copy and paste this exact expression: This is the final expression

**@concat('SELECT ContentDocumentId, ContentDocument.LatestPublishedVersionId, ContentDocument.Title, LinkedEntityId FROM ContentDocumentLink WHERE LinkedEntityId IN (SELECT Id FROM ', pipeline().parameters.ObjectName, ')')**

Step 2: Update the Copy Activity
Now we need to tell the Copy Activity to look for this new column name instead of the old one.
1. Double-click your ForEach1 loop to open the canvas.
2. Click on your Copy_Binary_File activity.
3. Go to the Sink tab (and also check the Source tab parameters if needed).
4. Look for the dynamic content box where you previously wrote @item().FirstPublishLocationId I.e. “ParentId” field (this is likely in your Dataset Properties for the folder path).
5. Replace FirstPublishLocationId with LinkedEntityId so it looks like this: @item().LinkedEntityId

Step 3: Publish and Debug
1. Click Publish All to save these two changes.
2. Go back to your parent pipeline and hit Debug.

===================================================================================

**The pipeline again failed and below is the error message of Copy_Binary_File**

**The expression 'item().ContentDocument.Title' cannot be evaluated because property 'ContentDocument' doesn't exist, available properties are 'ContentDocumentId, ContentDocument.LatestPublishedVersionId, ContentDocument.Title, LinkedEntityId'.**

This is a classic Azure Data Factory expression language trap!
Look very closely at the error message. It says the property 'ContentDocument' doesn't exist, but it lists 'ContentDocument.Title' as an available property.
How is that possible?

When Salesforce returns a "dot notation" query (like ContentDocument.Title), it doesn't return a nested JSON object. It flattens it into a single literal string key. So the property name is literally the full string "ContentDocument.Title".
When you type @item().ContentDocument.Title, ADF gets confused and thinks you are trying to open a folder named ContentDocument and look for a file named Title inside it.
To tell ADF that the dot is actually part of the name, we have to use Bracket Notation.

**The Fix: Switch to Brackets**
We just need to wrap those specific fields in brackets and single quotes.
1. Double-click your ForEach1 loop and click on the Copy_Binary_File activity.
2. Go to the Source tab (or wherever you placed the Version ID).
    * Replace @item().ContentDocument.LatestPublishedVersionId
    * With: @item()['ContentDocument.LatestPublishedVersionId']
3. Go to the Sink tab (or wherever you placed the File Name).
    * Replace @item().ContentDocument.Title
    * With: @item()['ContentDocument.Title']
(Leave @item().LinkedEntityId and @item().ContentDocumentId exactly as they are, because they don't have dots in their names!)
**The Final Run**
1. Click Publish All.
2. Run the Debug from the parent pipeline.

===================================================================================


