~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
               *Windtastic Productions* PRESENT                                      
                              ~~                                                      
                "YOU WANNA CHANGE DATA IN PROD"                    
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is it.
You have data from five organizations and twenty tables to merge.
You're going to have to map data from various sources into Salesforce.**IT'S THAT BIG MIGRATION TIME.**

Well let's make sure you dont have to do it again in two days because data is missing, or delete production data.

Salesforce does not back up your data.
If you delete your data, and the amount deleted is bigger than what is in the recycle bin, if will be deleted forever. You could try restoring it via Workbench, praying that the automated Salesforce jobs haven't wiped your data yet.
If you update data, the moment the update hits the database (the DML) is done, the old data is lost. Forever.
You could try seing if you turned on field history.

If worst comes to worst you can pay 10 000€ (not joking) to Salesforce to restore your data. https://help.salesforce.com/articleView?id=000003594

But let's try to avoid these situations.
These steps apply to any massive data load, but especially in case of deletions.



**GENERAL DATA OPERATIONS STUFF**

- Tools
	You are using Jitterbit because it is a real data loading tool unlike dataloader.
	Jitterbit is like Dataloader but better. If you don't have it installed, install it now. IT IS FREE. It is great. If you tried doing a full data migration with Dataloader, you will not be helped. https://www.jitterbit.com/solutions/salesforce-integration/salesforce-data-loader/

- Volume
	If you are loading a big amount of data or the org is mature, read this document entirely before doing anything.
	https://resources.docs.salesforce.com/sfdc/pdf/salesforce_large_data_volumes_bp.pdf


- Deletions
	Apart from the "GENERAL DATA OPERATIONS STUFF" section, read this all.
	If you delete data in prod without a backup, this is bad.
	If the data backup was not checked, this is bad.
	If you did not check automations before deleting, this is also bad.
	Read the other sections.

- Data Mapping
	Don't map the data yourself.
	ANY data mapping you do is with someone from the client's who can understand WTF you're saying.
	If no one like this is available, you spend time with a business operative so you can do the mapping and make them sign off on it.
	The client SIGNS OFF on the mapping.
	There is no glory in doing a load, realizing you fucked up a map, and starting again.
	That said, basic operations are as follow:
    - study Source and target data model
    - establish mapping from table to table, field to field, or both if necessary.
    - for each table and field, establish Source of Truth, meaning which data should take priority if conflicts exist
    - establish an ExternalId from all systems to ensure data mapping is correct
    - define which users can see what data. Update permissions if needed.

- Data retrieval
    - Data needs to be extracted from source system. This can be via API, an ETL, a simple CSV extract, etc.
    - Verify Data Format
        - Date format yyyy-mm-dd
        - DateTime format yyyy-mm-ddT00:00:00z
        - Emails not longer than 80 char
        - Text containing carriage returns is qualified by "
        - Other field-specific verifications re. length and separators for text, numbers, etc.
    - Verify Table integrity
        - Check that all tables have basic data for records:
            - LastName, Account for Contact
            - Name for Account
            - Any other system mandatory fields
        - Check that all records have the agreed-upon external Ids
    - Verify Parsing
        - Do a dummy load to ensure that the full data extracted can be mapped and parsed by the selected automation tool

- Data Matching
	You should already have created External Ids on every table, if you are upserting data.
	If not, do so now.
	DO NOT match the data in excel.
	Yes, INDEX(MATCH()) is a beautiful tool. No, no one wants you to spend hours doing that when you could be doing other stuff, like drinking a cold beer.
	Store IDs of the external system in the target tables, in the ExternalId field. Then use that when recreating lookup relationships to find the records.
	This saves time, avoids you doing a wrong matching, and best of all, if the source data changes, you can just run the data load operation again on the new file, without spending hours matching IDs.



**SHIT YOU DO BEFORE YOU DO ANYTHING ELSE**
- Login to Prod. Is there a weekly backup running, encoded as UTF-8, in Setup > Data Export ?
	- Nope
		Select encoding UTF-8 and click "Export Now". This will take hours.
		Turn that weekly stuff on.
		Make sure the client KNOWS it's on.
		Make sure they have a strategy for downloading the ZIP file that is generated by the extract weekly.
	- Yup.
		Is it UTF-8 and has run in the last 48 hours ?
		- Yup
			Confer with the client to see if additional backup files are needed.
		- Nope.
			If the export isn't UTF-8, it's worthless.
			If it's more than 48h old, confer with the client to see if additional backup files are needed. In all cases, you should consider doing a new, manual export.
- SERIOUSLY MAKE SURE YOU CHANGE THE ENCODING. Salesforce has some dumb rule of not defaulting to UTF-8. YOU NEED UTF-8. This is Europe, not Texas. Accents and ḍîáꞓȑîȶîꞓs exist.

- If Data Export is not an option because it has run too recently, or because the encoding was wrong, you can also do your export by using Jitterbit to Query all the relevant tables. 
	- In Jitterbit also, remember to select "Write File as UTF-8" when prompted.

- CHECK THE CODE AND AUTOMATION IN THE ORG.
	- Seriously, look over all triggers that can fire when you upload the data.
	You don't want to be that consultant that sent a notification email to 50000 people.
	Just check the triggers and see what they do.
	If you can't read triggers, ask a dev to help you.
	- Yes, Check the Workflows and Process Builders too. They can send Emails as well
	- Check Process Builders again. Are there a lot that are firing on an object you are loading ? Make note of that for later, you may have to deactivate them.

- Check data volume.
	- Is there enough space in the org to accomodate the extra data ? (this should be pre-project checks, but check it again)
	- Are volumes to load enough to cause problems API-call wise ? If so, you may need to consider using the BULK jobs instead of normal operations
	- In case data volumes are REALLY big, you will need to abide by LDV (large data volume) best practices, including not doing upserts, defering sharing calculations, and grouping records by Parent record and owner before uploading. Full list of these is available here: https://resources.docs.salesforce.com/sfdc/pdf/salesforce_large_data_volumes_bp.pdf



**PREPARING THE JOBS**
	Before creating a job, ask yourself which job type is best.
	Upsert is great but is very resource intensive.
	Maybe think about using the BULK Api.
	In all cases, study what operation you do and make sure it is the right one.
	Once that is done...

	As such, you are able to create insert, upsert, query and deletion jobs, and change select parts of it.
	This is important because this means you can:
	- Create a new Sandbox
	- In Jitterbit, create a new Folder for each operation type you will do. Name it "mydatauploadname 20180805" for example. You will put your jobs in there for easier archival.
	- Prepare each job. Rename them from the default 'Account Insert' to  1 - Account upsert, 2 - Contact upsert, etc.
	In the end, you will have 1 through X of jobs you need to run sequentially, then additional numbers that are added to finalize any rejects from those loads if needed.
	- Do a dummy load in sandbox. Make sure to set the start line to something near the end so you don't clog the sandbox up with all the data.
	- Make sure everything looks fine.

	If something fails, you correct the TRANSFORMATION, not the file, except in cases where it would be prohibitively long to do so. Meaning if you have to redo the load, you can run the same scripts you did before to have a nice CSV to upload.



**GETTING READY TO DO THAT DATA OPERATION**

	You've got backups of every single table in the Production org.
	Even if you KNOW you do, you open the backups and check they are not corrupt or unreadable. Untested backups are no backups.
	You know what all automations are going to do if you leave them on.
	You talked with the client about possible impacts, and the client is ready to check the data after you finish your operations.

	You set up, with the client, a timeframe in which to do the data operation.
	If the data operation impacts tables that users work on normally, you freeze all those users during that timeframe.

	Remember to deactivate any PB, WF, APEX that can impact the migration. You didn't study them just to forget them.
	If this is an LDV job, take into account any considerations listed in "GENERAL DATA OPERATIONS STUFF" and in "SHIT YOU DO BEFORE YOU DO ANYTHING ELSE"


**DATA OPERATION**

	Go to Jitterbit and edit the Sandbox jobs.
	Edit the job Login to point to production.
	Save all the jobs.

	You run, in order, the jobs you prepared.

	When the number of failures is low enough, study the failure files, take any corrective action necessary, then use those files as a new source for a new data load operation.
	Continue this loop until the number of rejects is tolerable.
	This will ensure that if some reason you need to redo the entire operation, you can take the same steps in a much easier fashion.
	Once you are done, take the failure files, study them, and prepare a recap email detailing failures and why they failed. It's their data, they have a right to know.



**POST-MIGRATION**

- Make sure everything looks fine, that you carried everything over.
- Warn their PM that the migration is done and request testing from their side.
- If you deactivated Workflows or PBs or somthing so the migration passes, ACTIVATE THEM BACK AGAIN.
- Unfreeze users if needed.
- Go drink champagne.



**REVERTING CHANGES**

	You have a backup. Don't panic.
	Identify WTF is causing data to be wrong.
	Fix that.
	Get your backup, restore data to where it was before the fuckup. Ideally, only restore affected fields. If needed, restore everything.
	Redo the data load if needed.
