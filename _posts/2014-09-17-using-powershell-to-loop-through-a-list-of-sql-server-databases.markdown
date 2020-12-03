I use Powershell quite a bit to automate routine tasks with SQL Server - mainly because Powershell makes it so simple to perform the same task on a given list of servers. With Powershell, we can execute a .sql script on many servers and save the output for later use. For example, say you need to script out and save database security for all production databases. We can do this with Powershell easily, and I'll show you how.

<a title="Here is the sql script I will be using to script out database level security." href="http://www.sqlservercentral.com/scripts/Security/71562/" target="_blank" rel="noopener">Here is the sql script I will be using to script out database level security.</a>

First of all, there needs to be an inventory or list of the databases that you will be querying. Here is an example of a SQL_DATABASES inventory table, and the query we will be using to get our list of databases. We will need both the server and database names.

```sql
CREATE TABLE [dbo].[SQL_DATABASES](
	[INSTANCE] [varchar](50) NOT NULL,
	[DATABASENAME] [varchar](100) NOT NULL,
	[REGION] [varchar](4) NOT NULL)
--------
SELECT INSTANCE
 ,DATABASENAME
FROM dbo.SQL_DATABASES
WHERE REGION = 'PROD' --or 'DEV' / 'TEST'
```

We will be using <a title="invoke-sqlcmd" href="http://msdn.microsoft.com/en-us/library/cc281720.aspx" target="_blank" rel="noopener">invoke-sqlcmd</a> to make the connection to the inventory table. You can hit up that link to see the differences between sqlcmd and invoke-sqlcmd. Basically, we need to specify our -ServerInstance, -Database and -Query we will use to get the list of our production databases. Then, each Instance and Database pair will be put into the $databases variable.

```powershell
# $databases grabs list of production databases 
# from the SQL_DATABASES table on your Database
# ----------------------------------------------
# NOTE* It is always best to test any process on a single dev or test server
# before trying it on ALL production databases!

$databases = invoke-sqlcmd -ServerInstance Server -Database Database `
-Query "SELECT INSTANCE ,DATABASENAME FROM Database.dbo.SQL_DATABASES WHERE REGION = 'PROD'"


```

Now that we've placed the list of Servers and databases into $databases, we need a way to loop through each database in that list. This is where the foreach loop comes in handy.

```powershell
foreach ($database in $databases) #for each separate server / database pair in $databases
{
# This lets us pick out each instance ($inst) and database ($name) as we iterate
# through each pair of server / database.
$Inst = $database.INSTANCE #instance from the select query
$DBname = $database.DATABASENAME #databasename from the select query
#...
```

Next we will generate the file name for the output of the query for each database. The goal is to have a separate .sql file for each database with a file name format of "Instance_DatabaseName_security.sql". I am using C:\DBA_SCRIPTS\DB_Permissions\ as the filepath for the output files. I've also included the syntax that could be used to do a Replace on named instances to remove the "\" for file names.

```powershell
#...
#generate the output file name for each server/database pair
$filepath = "C:\DBA_SCRIPTS\DB_Permissions\"
$filename = $Inst +"_"+ $DBname +"_security.sql" 

# This line can be used if there are named instances in your environment.
#$filename = $filename.Replace("\","$") # Replaces all "\" with "$" so that instance name can be used in file names.

$outfile = ($filepath + $filename) #create out-file file name
#...
```

Now that we have the list of databases and have generated the output file name for each pair, we can execute our script on each database. For scripting database permissions, I use a variation of <a title="this script" href="http://www.sqlservercentral.com/scripts/Security/71562/" target="_blank" rel="noopener">this script</a> from S. Kusen. This next piece of powershell will connect to each individual server($Inst) and database($DBname) using invoke-sqlcmd, execute our .sql file (SQL_ScriptDBPermissions_V2.sql) to script out database security (using -InputFile), and output the results from each database into a separate .sql file at the $outfile path we created above.

```powershell
#...
#connect to each instance\database and generate security script and output to files
invoke-sqlcmd -ServerInstance ${Inst} -Database ${DBname} -InputFIle "C:\DBA_SCRIPTS\DB_Permissions\SQL_ScriptDBPermissions_V2.sql" | out-file -filepath ($outfile)

} #end foreach loop
```

It's as simple as that, and this is just one example of how powershell can easily be used to accomplish a task on many SQL Servers at once.

Here is the entire powershell script:

```powershell
# NOTE* It is always best to test any process on a single dev or test server before trying it on ALL production databases!
# ----------------------------------------------
Import-Module SqlPs -DisableNameChecking #may only need this line for SQL 2012 +

# $databases grabs list of production databases from the SQL_DATABASES table on your Database
$databases = invoke-sqlcmd -ServerInstance Server -Database Database -Query "SELECT INSTANCE ,DATABASENAME FROM Database.dbo.SQL_DATABASES WHERE REGION = 'PROD'"


foreach ($database in $databases) #for each separate server / database pair in $databases
{
# This lets us pick out each instance ($inst) and database ($name) as we iterate through each pair of server / database.
$Inst = $database.INSTANCE #instance from the select query
$DBname = $database.DATABASENAME #databasename from the select query


#generate the output file name for each server/database pair
$filepath = "C:\DBA_SCRIPTS\DB_Permissions\"
$filename = $Inst +"_"+ $DBname +"_security.sql"
 
# This line can be used if there are named instances in your environment.
#$filename = $filename.Replace("\","$") # Replaces all "\" with "$" so that instance name can be used in file names.
 
$outfile = ($filepath + $filename) #create out-file file name


#connect to each instance\database and generate security script and output to files
invoke-sqlcmd -ServerInstance ${Inst} -Database ${DBname} -InputFIle "C:\DBA_SCRIPTS\DB_Permissions\SQL_ScriptDBPermissions_V2.sql" | out-file -filepath ($outfile)
 
} #end foreach loop
```
