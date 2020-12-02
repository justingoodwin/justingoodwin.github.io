{% seo %}
<h2>Intro</h2>
One of the <a href="https://msdn.microsoft.com/en-us/library/bb510411.aspx" target="_blank" rel="noopener">many new features</a> introduced in SQL Server 2016 is <a href="https://msdn.microsoft.com/en-us/library/mt163865.aspx" target="_blank" rel="noopener">Always Encrypted</a>. In the words of Microsoft: "Always Encrypted provides a separation between those who own the data (and can view it) and those who manage the data (but should have no access)."

If there is <a href="https://en.wikipedia.org/wiki/Personally_identifiable_information" target="_blank" rel="noopener">PII</a> in a table or database, all Database Administrators (or anyone tasked with administering that database that have the enough permissions) can view that data. Generally, those who are hired to manage that database should be trusted to not go digging through personal information; however, with Always Encrypted, you can completely eliminate having to even think about who may or may not have access to the sensitive data. Because having access to an encryption certificate is needed to view the data, it becomes very easy to control who has access.

As a DBA, I can say that not even having to worry about what data I may or may not see is a huge relief.

<span style="text-decoration: underline;">In this post, I will cover:</span>
1. <a href="#Encrypting">How to encrypt a column with sensitive data using the GUI in SSMS.</a>
2. <a href="#SSMS">How to view Always Encrypted Data from SSMS on the machine where the Always Encrypted certificate was generated.</a>
3. <a href="#SSMS2">How to export/import the Always Encrypted certificate so that Always Encrypted data can be viewed from multiple machines.</a>
<h2 id="Encrypting">Encrypting a Column</h2>
I will be using the trusty AdventureWorks database for this demonstration (I know, I know. WorldWideImporters is sooo much better), and I will be looking at the Sales.CreditCard table.

Here is what the data in that CreditCard table looks like. Our goal will be to encrypt the CardNumber column.

```sql
Use AdventureWorks
GO
SELECT TOP 5 * FROM [Sales].[CreditCard];
```

<a href="https://sqljgood.files.wordpress.com/2017/02/top5initialresults.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156611 size-full" src="https://sqljgood.files.wordpress.com/2017/02/top5initialresults.png" alt="top5initialresults" width="553" height="143" /></a>

First, let's create a quick copy the CreditCard table. We'll copy it over to a new table: CreditCard_Encrypted.

```sql
--create copy of the CreditCard table
DROP TABLE IF EXISTS Sales.CreditCard_Encrypted

Select * 
INTO Sales.CreditCard_Encrypted
from Sales.CreditCard

GO
```

Since I am a <a href="http://www.adweek.com/core/wp-content/uploads/sites/prnewser/2014/06/Millennials-are.png" target="_blank" rel="noopener">lazy millennial</a>, I'm just going to use the GUI to encrypt the CardNumber column.

First, right-click on the table and choose Encrypt Columns...

<a href="https://sqljgood.files.wordpress.com/2017/02/rightclick_encrypt.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156613 size-full" src="https://sqljgood.files.wordpress.com/2017/02/rightclick_encrypt.png" alt="rightclick_encrypt" width="469" height="264" /></a>

Next, select the column(s) you want to encrypt and choose the Encryption Type (<a href="https://msdn.microsoft.com/en-us/library/mt163865.aspx#Anchor_2" target="_blank" rel="noopener">Deterministic or Randomized</a>) and the Encryption Key you want to use. I'm just going to be generating a new key for this demonstration.

<a href="https://sqljgood.files.wordpress.com/2017/02/columnselection.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156614 size-large" src="https://sqljgood.files.wordpress.com/2017/02/columnselection.png?w=620" alt="columnselection" width="620" height="564" /></a>

Choose the Column Master Key. Once again, I'm generating a new master key and storing it in the Windows Certificate store.

<a href="https://sqljgood.files.wordpress.com/2017/02/mkconfig.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156615 size-large" src="https://sqljgood.files.wordpress.com/2017/02/mkconfig.png?w=620" alt="mkconfig" width="620" height="564" /></a>

Finish navigating through the next few screens and finish the Encryption tasks:

<a href="https://sqljgood.files.wordpress.com/2017/02/results.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156616 size-large" src="https://sqljgood.files.wordpress.com/2017/02/results.png?w=620" alt="results" width="620" height="564" /></a>

Once that is complete, we can try querying the CreditCard_Encrypted table:

```sql
Use AdventureWorks
GO
SELECT TOP 5 * FROM [Sales].[CreditCard_Encrypted];
```

<a href="https://sqljgood.files.wordpress.com/2017/02/top5encryptedresults.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156612 size-large" src="https://sqljgood.files.wordpress.com/2017/02/top5encryptedresults.png?w=620" alt="top5encryptedresults" width="620" height="112" /></a>

Our CardNumber column is now encrypted! That wasn't too difficult.

We can also see the Master Key and Encryption Key that were created:

<a href="https://sqljgood.files.wordpress.com/2017/02/keys.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156610 size-full" src="https://sqljgood.files.wordpress.com/2017/02/keys.png" alt="keys" width="208" height="94" /></a>

We can also script out the DDL for those keys...

```sql
--Master Key
CREATE COLUMN MASTER KEY [CMK_Auto1]
WITH
(
	KEY_STORE_PROVIDER_NAME = N'MSSQL_CERTIFICATE_STORE',
	KEY_PATH = N'CurrentUser/my/F7D16243AF2D3AA32FAF06EC759F435AB44CB919'
)
GO

--Encryption Key
CREATE COLUMN ENCRYPTION KEY [CEK_Auto1]
WITH VALUES
(
	COLUMN_MASTER_KEY = [CMK_Auto1],
	ALGORITHM = 'RSA_OAEP',
	ENCRYPTED_VALUE = 0x016E000001630075007200720065006E00740075007300650072002F006D0079002F0066003700640031003600320034003300610066003200640033006100610033003200660061006600300036006500630037003500390066003400330035006100620034003400630062003900310039004CDE3BDBF014EAD0C066A97E5E93CCFFF36BE6F8A9B5EBB7F790659AD3CA626C48F0514D77C05052C4235A58BDF6908B054DF9DF9F01C2690AD89C5E71F5987A0054F913F161547DF740E7E9A3DF3695A8D0027BC5CAB779D1AB8CDDBED76649F56DC1EE107ACAC5A09BD4CCE8075707F0CCA3F00EF4376D428E56C35B12AEBF924978C8CA460F89F004AC1B9E0F58DBB3CE2B455DDC3F1C39526488A63AB7417CF0D5CD0D75E9F76F08736CF35DFF435BC1363C38C0FF3A0FB0C21D3CB4E0E3153F8C7928895E08F849274E3A16C146A48A4A9A0B574565D89E5727FA56ADFFFA77F50F6468FF21EBB03A3A7C432E18644D8EAFA91471E059729FEB9D6682A33E23A395B7915A425E24C75B1477302232C9657C1AE4F7ADF4439EC62E355944BFC9393458A319DF0191B2531E51965AA191D0EA1B4089DC5B426FB0511DD089ACE81F73A3AC3A45D11CA7F8EB5EC9026266DC596AB1E98A4F30194909BC464CF89978F0ABF6BF6BED36A0B5AC3ADC4B25C7AFCE3DC9AD0F0B20EE42871FA9713474C1D344879B6F0120B4FEDC6208296663C36823CDBA71AFBE1EDDBE87BCE451ED5A2ABD0275D203AA05310FAE4738FB7EEAF99CDB36C73BC3844224F96EDD9BFC5415575A42C54986775B36994C0B1C1F3B5C9EC51DF7D3F50B8CA74385A67E5549EF17DF010AAB0C7C529C5F82BC3C8837F0472C5CFF452CD2029AB46964
)
GO
```

...and we can script out the CREATE statement for the table to see what the Always Encrypted syntax looks like:

```sql
CREATE TABLE [Sales].[CreditCard_Encrypted](
	[CreditCardID] [int] IDENTITY(1,1) NOT NULL,
	[CardType] [nvarchar](50) NOT NULL,
	[CardNumber] [nvarchar](25) COLLATE Latin1_General_BIN2 ENCRYPTED WITH 
		  (COLUMN_ENCRYPTION_KEY = [CEK_Auto1], ENCRYPTION_TYPE = Randomized, 
		  ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256') NOT NULL,
	[ExpMonth] [tinyint] NOT NULL,
	[ExpYear] [smallint] NOT NULL,
	[ModifiedDate] [datetime] NOT NULL
) ON [PRIMARY]

GO
```

<h2 id="SSMS"><strong>So what do we do if we actually want to view the encrypted data in SSMS? </strong></h2>
We'll need to pass in this connection parameter in the SSMS connection options: Column Encryption Setting = Enabled

<a href="https://sqljgood.files.wordpress.com/2017/02/connectoptions.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156623 size-full" src="https://sqljgood.files.wordpress.com/2017/02/connectoptions.png" alt="connectoptions" width="477" height="520" /></a>

Now I'll reconnect and try to select from the Encrypted table once again:

```sql
Use AdventureWorks
GO
SELECT TOP 5 * FROM [Sales].[CreditCard];
```

<a href="https://sqljgood.files.wordpress.com/2017/02/top5initialresults.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156611 size-full" src="https://sqljgood.files.wordpress.com/2017/02/top5initialresults.png" alt="top5initialresults" width="553" height="143" /></a>

I am now able to view the encrypted data. Why is this?

Because I am connecting to the database from the database server, I have access to the Encryption Certificate that was generated. I can verify this by opening <a href="https://msdn.microsoft.com/en-us/library/e78byta0(v=vs.110).aspx" target="_blank" rel="noopener">certmgr.msc</a> and browsing to Personal -&gt; Certificates:
<a href="https://sqljgood.files.wordpress.com/2017/02/manage-user-certificates.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156655 size-full" src="https://sqljgood.files.wordpress.com/2017/02/manage-user-certificates.png" alt="manage-user-certificates" width="216" height="34" /></a>

<a href="https://sqljgood.files.wordpress.com/2017/02/certmgr.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156617 size-large" src="https://sqljgood.files.wordpress.com/2017/02/certmgr.png?w=620" alt="certmgr" width="620" height="270" /></a>

Without access to that certificate, I would not be able to see the encrypted data.

We can test this by attempting to connect to the database with the encrypted table from another machine and passing in the "Column Encryption Setting = Enabled" connection option in SSMS.

When trying to do a select on CreditCard_Encrypted, we receive this error message:
<pre><span style="color: #ff0000;">Msg 0, Level 11, State 0, Line 0
Failed to decrypt column 'CardNumber'.
Msg 0, Level 11, State 0, Line 0
Failed to decrypt a column encryption key using key store provider: 'MSSQL_CERTIFICATE_STORE'. The last 10 bytes of the encrypted column encryption key are: '36-B6-E0-8F-9F-54-AC-CF-9B-CB'.
Msg 0, Level 11, State 0, Line 0
Certificate with thumbprint 'BA85A469AE80E4001B1FD9E71D89165593DE9009' not found in certificate store 'My' in certificate location 'CurrentUser'. Verify the certificate path in the column master key definition in the database is correct, and the certificate has been imported correctly into the certificate location/store.
Parameter name: masterKeyPath</span>
</pre>
<h2 id="SSMS2"><strong>What about viewing the encrypted data from SSMS on another machine? </strong></h2>
So how can we get a copy of the Always Encrypted certificate over to the other machine?

We must go back to the original machine or server where the certificate was generated. Re-open certmgr.msc, browse to the Always Encrypted Certificate, right-click -&gt; All Tasks -&gt; Export...

<a href="https://sqljgood.files.wordpress.com/2017/02/certexport1.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156618 size-full" src="https://sqljgood.files.wordpress.com/2017/02/certexport1.png" alt="certexport1" width="535" height="523" /></a>

I've had success selecting these options:

<a href="https://sqljgood.files.wordpress.com/2017/02/certexport2.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156619 size-full" src="https://sqljgood.files.wordpress.com/2017/02/certexport2.png" alt="certexport2" width="535" height="523" /></a>

Give the certificate a password if you wish:

<a href="https://sqljgood.files.wordpress.com/2017/02/certexport3.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156620 size-full" src="https://sqljgood.files.wordpress.com/2017/02/certexport3.png" alt="certexport3" width="535" height="523" /></a>

Choose a file name for the exported certificate:

<a href="https://sqljgood.files.wordpress.com/2017/02/certexport4.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156621 size-full" src="https://sqljgood.files.wordpress.com/2017/02/certexport4.png" alt="certexport4" width="535" height="523" /></a>

Finish the export wizard and browse to the destination folder. We now have a copy of the Always Encrypted certificate.

<a href="https://sqljgood.files.wordpress.com/2017/02/cert_file.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156622 size-full" src="https://sqljgood.files.wordpress.com/2017/02/cert_file.png" alt="cert_file" width="400" height="62" /></a>

The certificate can now be copied to the secondary machine that needs access to the encrypted data. Open 'Manage User Certificates' (AKA certmgr.msc). Browse to Personal -&gt; Certificates. Right click on the Certificates folder -&gt; All Tasks -&gt; Import...

<a href="https://sqljgood.files.wordpress.com/2017/02/importcert1.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156624 size-full" src="https://sqljgood.files.wordpress.com/2017/02/importcert1.png" alt="importcert1" width="495" height="274" /></a>

Browse to and select the Always Encrypted certificate that was copied from the database server:

<a href="https://sqljgood.files.wordpress.com/2017/02/importcert2.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156625 size-full" src="https://sqljgood.files.wordpress.com/2017/02/importcert2.png" alt="importcert2" width="535" height="523" /></a>

Enter the password if one was used. Also, check Include all extended properties.

<a href="https://sqljgood.files.wordpress.com/2017/02/importcert3.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156626 size-full" src="https://sqljgood.files.wordpress.com/2017/02/importcert3.png" alt="importcert3" width="535" height="523" /></a>

Place the certificate in the Personal store.

<a href="https://sqljgood.files.wordpress.com/2017/02/importcert4.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156627 size-full" src="https://sqljgood.files.wordpress.com/2017/02/importcert4.png" alt="importcert4" width="535" height="523" /></a>

Step through the final screens and finish the import process.

After the import is finished, open certmgr.msc and ensure the Always Encrypted certificate shows up under Personal -&gt; Certificates.

Now that the certificate has been imported, we can try to query the Encrypted table again.

<a href="https://sqljgood.files.wordpress.com/2017/02/finalresults.png" target="_blank" rel="noopener"><img class="aligncenter wp-image-156628 size-full" src="https://sqljgood.files.wordpress.com/2017/02/finalresults.png" alt="finalresults" width="556" height="216" /></a>

BOOM! We can now query the encrypted data on a different machine with SSMS.

<a href="http://img.wonkette.com/wp-content/uploads/2015/08/Jon-Stewart-boom-gif.gif" target="_blank" rel="noopener"><img class="aligncenter" src="http://img.wonkette.com/wp-content/uploads/2015/08/Jon-Stewart-boom-gif.gif" alt="BOOM" width="245" height="179" /></a>
<h2>Conclusion</h2>
In this blog post, I've shown how to easily encrypt a column using Always Encrypted. I've also shown how to enable users using SSMS to view that encrypted data.

I'm pretty excited about Always Encrypted and think that it could potentially be a great solution for encrypting sensitive data.