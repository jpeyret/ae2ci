#### Sample Communication Record.

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/Record.AE2CIAET.png "Record.AE2CIAET")

***
**Note:** from the base class, default values:

Subject| Wrapper Class Attribute | Default value
------------ | -----| -------------
**Name of Message Field**  | `fld_message` | MESSAGE\_TEXT\_254
**Name of Notification Field**  | `fld_notify` | SET\_NOTIFY\_FLAG
**Notification code: Initiated**  | `notify_initiated ` | I (initiated)
**Notification code: Exception**  | `notify_exception ` | F (fail)
**Notification code: Data Error**  | `notify_data_error ` | N (not loaded)
**Notification code: Loaded**  | `notify_data_loaded ` | Y (data loaded)

These can be modified in your subclass's constructor, after calling the **super** constructor, but you need to make sure your Application Engine and communication record match.

***


#### Sample Data Staging Record

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/Record.TCI_SOURCE.png "Record.TCI_SOURCE")

***
**Note:**  If you are loading data from an external system, you probably don't want to add any validation/prompts to the staging record.  Let the Component Interface handle validation logic instead.
***


#### Reset the staging record.

A step to reset the staging record may be useful if you want to run the Application Engine and give users the option to correct and re-try data loads.

````sql
 UPDATE PS_TCI_SOURCE 
  SET SET_NOTIFY_FLAG = 'S' 
  ,MESSAGE_TEXT_254 = ' ' 
 WHERE SET_NOTIFY_FLAG NOT IN ('Y', 'B', 'D') ;
````

In this case, **S** is the "Staged for loaded" code, **Y** means that the data was loaded already.  **B** means user-initiated Block and **D** means Delete mode.

The wrapper class does not have to know about all these codes, it only needs to know what codes your code expect from it.

#### Fetch by Do-while loop:

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/loop.png "loop")

````sql
 %Select(EMPLID, AE2CIAET.SET_NOTIFY_FLAG) 
 SELECT EMPLID 
 , CASE SET_NOTIFY_FLAG WHEN 'D' THEN 'D' ELSE 'I' END 
  FROM PS_TCI_SOURCE /* S stands for staged here */ 
 WHERE SET_NOTIFY_FLAG IN ('S','D') 
  ORDER BY EMPLID ;
````

#### The AE Section doing the actual load:

***
**Warning:** watch the On Error and make sure the **updMsg** step always runs, otherwise a Do-While might run infinitely.
***

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/load.png "load")

###### sql to update the staging record

````sql
UPDATE PS_TCI_SOURCE 
  SET MESSAGE_TEXT_254 = %Bind(AE2CIAET.MESSAGE_TEXT_254), SET_NOTIFY_FLAG = %Bind(AE2CIAET.SET_NOTIFY_FLAG) 
 WHERE EMPLID = %Bind(EMPLID)


````

***
**Remember:** up to now, the status code and notification are only in the AET communication record.  This step writes them to the database, **after** any commit/rollbacks.
***

### Sample unit testing screen for this batch.

No, it's not going to win any cosmetic prizes.  But take a a look at the **Status** and **Results** columns - these are error straight from the CI or even peoplecode errors.

![alt text](https://github.com/jpeyret/ae2ci/blob/master/media/user_screen.png "user_screen")

