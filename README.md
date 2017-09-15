# ae2ci - PeopleSoft Application Engine-Component Interface

This repository contains an extensible Application Class that takes Rowset-based data, typically from an Application Engine and writes it to a PeopleSoft database using Component Interface.

##How it works:

####Declaring your custom subclass

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

####Component Business logic goes into method *ci\_business\_logic*:

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

####Sample Application Engine peoplecode:






##The AE2CI framework handles:

- calls the Component Interface
- tests the Component Interface session for error conditions.
- manages exceptions in such a way that they should not result in Application Engine abends but still allow updating of notification/feedback fields, even in case of a rollback

##You are responsible for:

- subclassing the base class and wring your business logic and data-to-CI mapping.  A method is provided for that, `ci_business_logic`.
- client Application Engine peoplecode to interact with the class.

Most of the actual work is done by the framework so a minimal implementation may run on the order 40-50 lines of code in 3 Application Engine steps and another 40-50 in the subclass.

Sample code is provided.


##Installation

This framework has been tested on PeopleTools 8.51 and 8.54 and with tools 8.5x in general.  In order to avoid PeopleTools version dependencies, the installation process is manual in nature.

###Copy and paste these 5 files:


###Into an Application Package with the following structure:

![alt text](file:///Users/jluc/kds2/mygithub/ae2ci/ae2ci/media/ApplicationPackage.AE2CI.png "Application Package structure")


