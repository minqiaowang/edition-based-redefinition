# Workshop Introduction and Overview #

## Introduction to Oracle Edition-Based Redefinition

- Edition-based redefinition (EBR) enables online application upgrade with uninterrupted availability of the application. When the installation of an upgrade is complete, the pre-upgrade application and the post-upgrade application can be used at the same time. Therefore, an existing session can continue to use the pre-upgrade application until its user decides to end it; and all new sessions can use the post-upgrade application. When there are no longer any sessions using the pre-upgrade application, it can be retired. In this way, EBR allows hot rollover from from the pre-upgrade version to the post-upgrade version, with zero downtime.

  EBR enables online application upgrades in the following manner:

  - Code changes are installed in the privacy of a new edition.
  - Data changes are made safely by writing only to new columns or new tables not seen by the old edition. An editioning view exposes a different projection of a table into each edition to allow each to see just its own columns.
  - Crossedition triggers propagate data changes made by the old edition into the new edition’s columns, or (in hot-rollover) vice-versa.

## About the Labs Environment

In this workshop, we will using the Oracle Database version 19c which setup on Oracle Cloud Infrastructure. The database has a default pdb named **orclpdb**. If you have your own on-premise database. You can skip the Lab1, Lab2, and Lab3. 

## More Information on EBR

Please see the EBR document at [https://www.oracle.com/database/technologies/high-availability/ebr.html](https://www.oracle.com/database/technologies/high-availability/ebr.html)

## Acknowledgements

- **Authors/Contributors** - Minqiao Wang, DB Product Management, June 2020
- **Last Updated By/Date** - 
- **Workshop Expiration Date** - 

See an issue?  Please open up a request [here](https://github.com/oracle/learning-library/issues).   Please include the workshop name and lab in your request. 
