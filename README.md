![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Create SQL Job Step Notification For SQL Agent Jobs
**Post Date: July 1, 2016**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>If you're here… This means you want a solution to sending an email notification about a particular Job and it's history and maybe add the Job step durations along the way. Here's some quick SQL logic to save you the time.
Ok; so I hear you say.. "What does the logic do exactly"

It first checks to see if you have database mail configured. If not; it will automatically configure it for you. What does this mean for you? It means you can create this step on any server you want; and as long as you supply the correct SMTP server name (look for MySMTPServer.MyDomain.com) in the SQL logic then it should work in most cases. It chooses a series of defaults, but if your environment has some of those defaults configured differently then it may not work. All you need to do is edit it accordingly, but it's designed right now to work as-is.

Then it creates a temporary table which looks on par with what you see in if you were looking at the job history from within Management studio, but this time it formats it into HTML Tables, and imbeds them into an email so you can always refer to it without any fuss. It's also a lot easier on the eyes cause it's configured to appear as a spreadsheet view. This is done by providing some CSS in the email formatting which is created at run-time.

I've deployed this across several of my servers SQL 2008, 2012, and 2014 and it works great.
How does this work?

Ok so you take the script and change the SMTP server name, Job Name you want to get the notification on. Then simply add a step and paste the entire script in there and you're done.
Here's the SQL Logic:</p>      



## SQL-Logic
```SQL

-- to use this process; simply replace My_Job_Name_Goes_here with the Job name you want to notify you and simply add this as the final step in the job.  
set nocount on
set ansi_nulls on
set quoted_identifier on
 

-- Configure SQL Database Mail if it's not already configured.
if (select top 1 name from msdb..sysmail_profile) is null
    begin

        -- Enable SQL Database Mail
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 

        -- Add a profile
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';

        -- Add the account names you want to appear in the email message.
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @email_address      = 'sqldatabasemail@MyDomain.com'
        ,   @mailserver_name    = 'MySMTPServer.MyDomain.com'  
        --, @port           = ####          --optional
        --, @enable_ssl     = 1         --optional
        --, @username       ='MySQLDatabaseMailProfile' --optional
        --, @password       ='MyPassword'       --optional
 
        -- Adding the account to the profile
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 

        -- Get Server info for test message
 
        declare @get_basic_server_name          varchar(255)
        declare @get_basic_server_name_and_instance_name    varchar(255)
        declare @basic_test_subject_message     varchar(255)
        declare @basic_test_body_message        varchar(max)
        set @get_basic_server_name          = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name= (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message     = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message        = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 

        -- Send quick email to confirm email is properly working.
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients = 'SQLJobAlerts@MyDomain.com'
        ,   @subject    = @basic_test_subject_message
        ,   @body       = @basic_test_body_message;
 
        -- Confirm message send and run this ( select * from msdb..sysmail_allitems )
    end
 

-- get basic server info.
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
declare @server_time_zone       varchar(255)
set     @server_name_basic      = (select cast(serverproperty('servername') as varchar(255)))
set     @server_name_instance_name  = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
declare @since_midnight         varchar(10) = (select convert(char, getdate(), 110))
declare @since_30_days_back     varchar(10) = (select convert(char, dateadd(day,-30,Getdate()), 110))
declare @tomorrow           varchar(10) = (select convert(char, dateadd(day,1,Getdate()), 110))

-- set message subject.
declare @message_subject    varchar(255)
set @message_subject    = 'Status Report for Job: My_Job_Name_Goes_here on ' + @server_name_instance_name
 

-- get total duration of job run across all run-times for the last 30 days.
 
if object_id('tempdb..#job_duration_total') is not null
    drop table  #job_duration_total
create table    #job_duration_total
(
    [server]    varchar(255)
,   [job_name]  varchar(255)
,   [day]       varchar(255)
,   [start_time]    varchar(255)
,   [step_id]   varchar(255)
,   [step_name] varchar(255)
,   [status]    varchar(255)
,   [hhmmss]    varchar(255)
)
 
insert into #job_duration_total
select
    [server]        = upper(@@servername)
,   [job_name]      = sj.name
,   [day]           = datename(dw, cast(str(sjh.run_date, 8, 0) as datetime))
,   [start_time]    = replace(replace(left(cast(str(sjh.run_date, 8, 0) as datetime) + cast(stuff(stuff(right('000000' + cast (sjh.run_time as varchar(6)), 6), 5, 0, ':'), 3, 0, ':') as datetime), 19), 'AM', 'am'), 'PM', 'pm')
,   [step_id]       = sjh.step_id
,   [step_name]     = sjh.step_name
--, [end_time]      = replace(replace(left(dateadd(second, ( ( sjh.run_duration / 1000000 ) * 86400 ) + ( ( ( sjh.run_duration - ( ( sjh.run_duration / 1000000 ) * 1000000 ) ) / 10000 ) * 3600 ) + ( ( ( sjh.run_duration - ( ( sjh.run_duration / 10000 ) * 10000 ) ) / 100 ) * 60 ) + ( sjh.run_duration - ( sjh.run_duration / 100 ) * 100 ), cast(str(sjh.run_date, 8, 0) as datetime) + cast(stuff(stuff(right('000000' + cast (sjh.run_time as varchar(6)), 6), 5, 0, ':'), 3, 0, ':') as datetime)), 19), 'AM', 'am'), 'PM', 'pm')
,   [status]        =   case sjh.run_status
                            when 0 then 'Failed'
                            when 1 then 'Succeded'
                            when 2 then 'Retry'
                            when 3 then 'Cancelled'
                            when 4 then 'In progress'
                        end
,   [hhmmss]        = stuff(stuff(replace(str(run_duration, 6, 0), ' ', '0'), 3, 0, ':'), 6, 0, ':')
 
from
    sysjobhistory sjh inner join sysjobs sj on sj.job_id = sjh.job_id
where
    sj.name = 'My_Job_Name_Goes_here'
    and sjh.step_id = '0'
    and msdb.dbo.agent_datetime(run_date, run_time) between @since_30_days_back and @tomorrow
order by
    [start_time] desc
 

-- get duration per step for current day.
 
if object_id('tempdb..#job_duration_steps') is not null
    drop table  #job_duration_steps
create table    #job_duration_steps
(
    [server]    varchar(255)
,   [job_name]  varchar(255)
,   [day]       varchar(255)
,   [start_time]    varchar(255)
,   [step_id]   varchar(255)
,   [step_name] varchar(255)
,   [status]    varchar(255)
,   [hhmmss]    varchar(255)
)
 
insert into #job_duration_steps
select
    [server]        = upper(@@servername)
,   [job_name]      = sj.name
,   [day]           = datename(dw, cast(str(sjh.run_date, 8, 0) as datetime))
,   [start_time]    = replace(replace(left(cast(str(sjh.run_date, 8, 0) as datetime) + cast(stuff(stuff(right('000000' + cast (sjh.run_time as varchar(6)), 6), 5, 0, ':'), 3, 0, ':') as datetime), 19), 'AM', 'am'), 'PM', 'pm')
,   [step_id]       = sjh.step_id
,   [step_name]     = sjh.step_name
--, [end_time]      = replace(replace(left(dateadd(second, ( ( sjh.run_duration / 1000000 ) * 86400 ) + ( ( ( sjh.run_duration - ( ( sjh.run_duration / 1000000 ) * 1000000 ) ) / 10000 ) * 3600 ) + ( ( ( sjh.run_duration - ( ( sjh.run_duration / 10000 ) * 10000 ) ) / 100 ) * 60 ) + ( sjh.run_duration - ( sjh.run_duration / 100 ) * 100 ), cast(str(sjh.run_date, 8, 0) as datetime) + cast(stuff(stuff(right('000000' + cast (sjh.run_time as varchar(6)), 6), 5, 0, ':'), 3, 0, ':') as datetime)), 19), 'AM', 'am'), 'PM', 'pm')
,   [status]        =   case sjh.run_status
                            when 0 then 'Failed'
                            when 1 then 'Succeded'
                            when 2 then 'Retry'
                            when 3 then 'Cancelled'
                            when 4 then 'In progress'
                        end
,   [hhmmss]        = stuff(stuff(replace(str(run_duration, 6, 0), ' ', '0'), 3, 0, ':'), 6, 0, ':')
 
from
    sysjobhistory sjh inner join sysjobs sj on sj.job_id = sjh.job_id
where
    sj.name = 'My_Job_Name_Goes_here'
    and sjh.step_name   not in ('send notification')
    and msdb.dbo.agent_datetime(run_date, run_time) between @since_midnight and @tomorrow
order by
    [start_time] desc

-- create conditions for html tables in top and mid sections of email.
 
declare @xml_top        NVARCHAR(MAX)
declare @xml_mid        NVARCHAR(MAX)
declare @body_top       NVARCHAR(MAX)
declare @body_mid       NVARCHAR(MAX)
 

-- set xml top table td's
-- create html table object for: #job_duration_total
set @xml_top = 
    cast(
        (select
            [server]    as 'td'
        ,   ''
        ,   [job_name]  as 'td'
        ,   ''
        ,   [day]       as 'td'
        ,   ''
        ,   [start_time]    as 'td'
        ,   ''
        --, [step_id]   as 'td'
        ,   ''
        ,   case    [step_name]
                when '(Job Outcome)' then 'Total Job Duration'
                else [step_name]
            end     as 'td'
        ,   ''
        ,   [status]    as 'td'
        ,   ''
        ,   [hhmmss]    as 'td'
        ,   ''
        from  #job_duration_total 
        --order by
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
 

-- set xml mid table td's
-- create html table object for: #job_duration_steps 
set @xml_mid = 
    cast(
        (select
            [server]    as 'td'
        ,   ''
        ,   [job_name]  as 'td'
        ,   ''
        ,   [day]       as 'td'
        ,   ''
        ,   [start_time]    as 'td'
        ,   ''
        ,   case    [step_id]
                when '0' then 'Completed'
                else [step_id]
                end as 'td'
        ,   ''
        ,   case    [step_name]
                when '(Job Outcome)' then 'Total Job Duration'
                else [step_name]
            end     as 'td'
        ,   ''
        ,   [status]    as 'td'
        ,   ''
        ,   [hhmmss]    as 'td'
        ,   ''
        from  #job_duration_steps 
        --order by
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
 

-- format HTML email
set @body_top =
        '<html>
        <head>
            <style>
                h1{
                    font-family: sans-serif;
                    font-size: 90%;
                }
 
                h3{
                    font-family: sans-serif;
                    color: black;
                    font-size: 95%
                }
                     
                table, td, tr, th {
                    font-family: sans-serif;
                    border: 1px solid black;
                    border-collapse: collapse;
                    font-size: 88%;
                }
 
                th {
                    text-align: left;
                    background-color: gray;
                    color: white;
                    padding: 5px;
                    font-size: 95%;
                }
 
                td {
                    padding: 5px;
                }
            </style>
        </head>
        <body>
        <H3>' + @message_subject + '</H3>
        <h1>Job Total Durations</h1>
        <table border = 1>
        <tr>
            <th> Server   </th>
            <th> Job Name </th>
            <th> Day  </th>
            <th> Start Time   </th>
            <th> Step Name    </th>
            <th> Status   </th>
            <th> HHMMSS   </th>
        </tr>'
         
set @body_top = @body_top + @xml_top + '</table>
 
<h1>Job Step Duration History</h1>
 
<table border = 1>
        <tr>
            <th> Server   </th>
            <th> Job Name </th>
            <th> Day  </th>
            <th> Start Time   </th>
            <th> Step ID  </th>
            <th> Step Name    </th>
            <th> Status   </th>
            <th> HHMMSS   </th>
        </tr>'    
         
+ @xml_mid + '</table>'
+ '</body></html>'
 

-- send email.
 
EXEC msdb.dbo.sp_send_dbmail
    @profile_name   = 'SQLDatabaseMailProfile'
,   @recipients = 'MyAccount@MyDomain.com'
,   @subject    = @message_subject
,   @body       = @body_top
,   @body_format    = 'HTML';
 
drop table #job_duration_total
drop table #job_duration_steps
```


By the way; if you want to set a more custom subject message you can add this to the logic flow (just above the send email process)



## SQL-Logic
```SQL
declare @message_subject varchar(255)
set @message_subject =  case
        when exists (select 1 from #job_duration_steps where [step_id] > 0 and [status] in ('Failed', 'Retry', 'In progress')) then ' The Load ETLS job has an Error'
        when exists (select 1 from #job_duration_steps where [step_id] > 0 and [status] in ('Cancelled')) then 'The Load ETLS job was Cancelled'
        else 'The Load ETLS job was Successful'
                end
                +  ' on Server ' + @server_name_instance_name
```
Here's what the email looks like:

![Example HTML Email]( https://mikesdatawork.files.wordpress.com/2016/07/image001.png "Example HTML and CSS SQL Generated Email")
 
 


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

