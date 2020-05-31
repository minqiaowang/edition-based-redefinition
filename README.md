# Oracle Database Edition-Based Redefinition

Edition-based redefinition (EBR) enables online application upgrade with uninterrupted availability of the application. When the installation of an upgrade is complete, the pre-upgrade application and the post-upgrade application can be used at the same time. Therefore, an existing session can continue to use the pre-upgrade application until its user decides to end it; and all new sessions can use the post-upgrade application. When there are no longer any sessions using the pre-upgrade application, it can be retired. In this way, EBR allows hot rollover from from the pre-upgrade version to the post-upgrade version, with zero downtime.

EBR enables online application upgrades in the following manner:

- Code changes are installed in the privacy of a new edition.
- Data changes are made safely by writing only to new columns or new tables not seen by the old edition. An editioning view exposes a different projection of a table into each edition to allow each to see just its own columns.
- Crossedition triggers propagate data changes made by the old edition into the new edition’s columns, or (in hot-rollover) vice-versa.

## Workshops
Click on one of our workshops below to access the workshop.

- [I have a Freetier or Oracle Cloud account](https://minqiaowang.github.io/edition-based-redefinition/freetier/index.html)
- [I have an account on LiveLabs](https://minqiaowang.github.io/edition-based-redefinition/livelabs/index.html)


## Get an Oracle Cloud Trial Account for Free!
If you don't have an Oracle Cloud account then you can quickly and easily sign up for a free trial account that provides:
- $300 of free credits good for up to 3500 hours of Oracle Cloud usage
- Credits can be used on all eligible Cloud Platform and Infrastructure services for the next 30 days
- Your credit card will only be used for verification purposes and will not be charged unless you 'Upgrade to Paid' in My Services

Click here to request your trial account: [https://www.oracle.com/cloud/free](https://www.oracle.com/cloud/free)


## Product Pages
- [Oracle Edition-Based Redefinition](https://www.oracle.com/database/technologies/high-availability/ebr.html)
- [Oracle Database 19c](https://www.oracle.com/database/)

## Documentation
- [Oracle EBR Document](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/editions.html)
- [Oracle Database 19c Document](https://docs.oracle.com/en/database/oracle/oracle-database/19/books.html)

### Issues?
Please submit an issue on our [issues](https://github.com/oracle/learning-library/issues) page.  We review it regularly.

-- Oracle Database Product Management
