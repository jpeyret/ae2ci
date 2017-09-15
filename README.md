# ae2ci - PeopleSoft Application Engine-Component Interface

This repository contains an extensible Application Class that takes Rowset-based data, typically from an Application Engine and writes it to a PeopleSoft database using Component Interface.

## How it works:

#### Declaring your custom subclass

````
import AE2CI:*;</span>

class Wrap_CI_PERSONAL_DATA extends AE2CI:CiWrapper
   method Wrap_CI_PERSONAL_DATA();
   method callci(&rec_comm As Record, &rec_data As Record) Returns boolean;
   method ci_business_logic(&rec_comm As Record, &data As Record) Returns boolean;
end-class;

method Wrap_CI_PERSONAL_DATA
   %Super = create AE2CI:CiWrapper(CompIntfc.CI_PERSONAL_DATA);
end-method;
````

***
**Note:**  method **callci** needs to be in your subclass but can be copied from base's implementation.
***


#### Component Business logic goes into method *ci\_business\_logic*:

Here's a minimal implementation that only updates 1 field:

````
method ci_business_logic
   /+ &rec_comm as Record, +/
   /+ &data as Record +/
   /+ Returns Boolean +/

   Local boolean &needs_saving = False;
   %Super.myCI.KEYPROP_EMPLID = &data.EMPLID.Value;

   &needs_saving = %Super.myCI.Get();
   If Not &need_saving Then
      /*  user message, will be persisted even after a Rollback */
      %Super.fld_message.Value = "no PERSONAL_DATA for EMPLID." | &data.EMPLID.Value;
      Return False;
   End-If;

   If All(&data.BIRTHCOUNTRY.Value) Then
      %Super.myCI.PROP_BIRTHCOUNTRY = &data.BIRTHCOUNTRY.Value;
      &needs_saving = True;
   End-If;

   Return &needs_saving;

end-method;
````

#### Sample Application Engine peoplecode:

***
**note:**  Note the AE's Step attributes below:
***

`On Error: Ignore`  


````
/* assume this AE step is called by a loop and TCI_AET.EMPLID holds the lookup value for the data to load */

import TCI:*;

/* we're wrapping 2 Component Interfaces in a transaction */
Component TCI:Wrap_CI_JOB_DATA &ci_job;
Component TCI:Wrap_CI_PERSONAL_DATA &ci_personal;

Local boolean &saved = False;

/* this is the state record we use to save status/error messages on 
   It needs to be an AE State record and needs a status field and a message field.
*/
Local Record &rec_comm = GetRecord(Record.AE2CIAET);

If (&ci_job = Null) Then
   &ci_job = create TCI:Wrap_CI_JOB_DATA();
End-If;

If (&ci_personal = Null) Then
   &ci_personal = create TCI:Wrap_CI_PERSONAL_DATA();
End-If;

Local Rowset &rs_data = CreateRowset(Record.TCI_SOURCE);
&rs_data.Fill("WHERE EMPLID = :1", TCI_AET.EMPLID);
Local Record &rec_data = &rs_data.GetRow(1).GetRecord(Record.TCI_SOURCE);

/* first CI update */
try
   &saved = &ci_job.callci(&rec_comm, &rec_data);
catch Exception &e_any_job
   /* strictly speaking we don't need a rollback, yet, since is the first CI */
   SQLExec("ROLLBACK");
   Exit (0);
end-try;

/* second CI update, in a transaction */
try
   &saved = &ci_personal.callci(&rec_comm, &rec_data);
catch Exception &e_ci_personal
   SQLExec("ROLLBACK");
   Exit (0);
end-try;

````

#### Cleanup: your AE updates the status/user message fields in a separate step:

`Commit:  After Step`  


````sql
UPDATE PS_TCI_SOURCE 
  
  SET
  /* user visible message */ 
  MESSAGE_TEXT_254 = %Bind(AE2CIAET.MESSAGE_TEXT_254)
  /* status field, used to control the loop logic */
  , SET_NOTIFY_FLAG = %Bind(AE2CIAET.SET_NOTIFY_FLAG) 
 WHERE EMPLID = %Bind(EMPLID)

````



## The AE2CI framework handles:

- calling the Component Interface
- checking the Component Interface session for error conditions.
- Exceptions should not result in Application Engine abends but still allow updating of notification/feedback fields, even in case of a Rollback.  This is the case even in pretty extreme conditions such as divide by zero, referencing missing fields and compilation errors on imported Peoplecode.


Most of the actual work is done by the framework so a minimal implementation may run on the order 40-50 lines of code in 3 Application Engine steps and another 40-50 in the subclass.

See examples for more details...


## Installation

This framework has been tested on PeopleTools 8.51 and 8.54 and with tools 8.5x in general.  In order to avoid PeopleTools version dependencies, the installation process is manual in nature.

### Copy and paste these 5 files:


### Into an Application Package with the following structure:

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/ApplicationPackage.AE2CI.png "Application Package structure")


