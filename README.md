# Nexmo Coding Challenge - Data Engineer
#### Andrew Austin (ajausti1@gmail.com) - 09/22/17

## Solution Description
This solution, coded in Java and targeting MySQL, leverages the [Spring Framework's](https://projects.spring.io/spring-framework/) [Spring Boot](https://projects.spring.io/spring-boot/) module for quick project scaffolding and the [Spring Batch](http://projects.spring.io/spring-batch/) module for robust out-of-the-box ETL functionality including resumable jobs, comprehensive job control behavior and performant batch prepared statement inserts.  The solution could be considered a 'minimum viable product' suitible for code-review and stakeholder demonstration for the proposed scenario:

>Your task is to build a data pipeline, which will process data from logs generated by an API platform and load it into a data warehouse, where the data can be accessed by business analysts using SQL and reporting tools.
  
## Code Description

<i>Solution code can be found in the src/main/java folder and all paths below are rooted there</i>

* Job control flow is orchestrated by com.nexmo.jobs.BatchConfiguration
* Lines which fail to be parsed are processed by com.nexmo.jobs.FileVerificationSkipper
* Lines are parsed, mapped and validated by com.nexmo.mappers.LogDataExtractorUtil and com.nexmo.mappers.LogDataLineMapper, with limited computation occurring in com.nexmo.entities.LogData
* The solution has 95% line-test coverage for functional packages (com.nexmo.entities.* and com.nexmo.mappers.*).
* The solution processes the test data set (10721732 lines of data across 4 files) in ~240 seconds on development hardware (example run output can be found in the logs folder).

## System Requirements

This solution was built and tested on MacOSX Sierra 10.2.6 using MySQL 5.7, Java 1.8 JDK and Git 2.7.1.
 
## Installation / Build Steps

1. Clone this repository to your local system and CD to that directory. 
2. Install and start MySQL if not already done so (installation is out of scope for this document).  Run the src/main/resources/setup.sql script from this repository within MySQL to create the target database, application user and table 'log_data'.  Configure the connection string in src/main/java/resources/application.properties with your database's instance information 
3. Set permissions to execute the mvnw script (e.g. chmod 755 mvnw), then run **./mvnw package**; this will build and package the source code for execution as an executable JAR.

## Run Steps
*Ensure Installation / Build Steps are completed first and completed with BUILD SUCCESS*

To run the application from the cloned directory:

>java -jar target/nexmo-0.1.0.jar --file=FILE_PATH | --dir=DIR_PATH

Where only --file OR --dir is supplied.  If --dir is supplied, all files ending with .csv in the target directory will be passed to the parser.  This solution can be scheduled via Cron, Taskmanager, etc, using a similar execution statement.

## Data Assumptions / Expectations
* We assume single-level nested CSV objects that don't contain the char '{' in their value
* We assume nested CSV objects are only one-level deep; more work would need to be done in order to accommodate multi-level nested CSV objects
* We assume all logs are being generated in the [Europe/London] timezone
* We assume messageId and date length and positions are fixed
* We assume currency calculations are trivial (small amounts w/ limited precision requiriments and uniform currency type); thus we leverage BigDecimal and are only trivially concerned with rounding
* We consider all fields and sub-fields mandatory; any missing component forces the record to be logged to the console and discarded

###### A Note on Requirements
I asked Oscar if SQL to find revenue, gross margin, etc, was necessary.  His answer was SQL was only necessary if the data was stored in some format other than listed in the scenario document.  As our MySQL schema matches the listed fields, no additional SQL is provided.

## Additional Questions
>How would you change your solution if you had an additional week to work on it?  Additional month?
    
The solution is currently 'MVP' and ready for review and feedback from stakeholders and engineering team members (for technical validation / suggestions / guidance).  Once feedback / assumption validation (or disproval) and review has completed, I would focus on prioritizing and implementing changes driven by that feedback followed by testing a larger variety / volume of data sources (and another review round).  Once base-functionality is working as intended, I would consider the following changes:

* Refactor to store full currency information in the DB for currency / cost calculations to be performed there OR consider implementing a dedicated currency library if this currency computation was to be done by this application
* Redirect failed lines to a flat-file for resolution / problem-pattern detection
* Abstract out configuration points to config file
* Potential batch size tweaking (rough-tweaking was done (10, 100, 1000, 5000, 10000); additional adjustment may provide limited performance benefits) and a performance review
* Move Spring Batch control tables from local DB into specialized control DB (automatically stored in the base database, which may appear as 'clutter' to analysts)
* Dev-ops related tasks - CI/CD / build management systems integration, containerization (if that is the deployment scheme) and multi-server deployment (if data volume necessitates)
* Rudimentary notification system / integrating into an existing notification system for success/failure alert consolidation (email, etc)
* Discussions w/ stakeholders re: long-term plan for operational workflows (change management, rejected record resolution pipelines, etc)
* Discussions w/ stakeholders re: long-term plan for operational self-service UI for analysts (requires expertise gained from maintaining production system before design / development) 

>Are there any aspects of ETL your solution does not cover that would need to be covered in a production environment?  How would you approach them?
    
* Security configuration for DBMS and DBMS connection (currently ignoring SSL certificate validation issues)
* Application logs are currently being output to the console, configuration to output logs to a file / directory for troubleshooting
* No notification system for failures; see improvements section
* Configuration point abstraction for operational maintainability
    
>If your solution needed to process 10 times the data volume provided, how would it change? 100 times?

From an execution stability / 'will it work' perspective, the application has not been tested with files larger than those included in the test set which are not exceptionally large (max size .75GB).  Testing would be required to validate handling of larger files to eliminate risk of memory pressure issues; file-splitting may be necessary as a last-resort.
From a performance perspective, the solution, on development hardware co-hosting the target database, processed 10721732 lines of data across 4 files in ~240 seconds, yielding an average of roughly 2.65m records / minute.  If those values were to hold in production, the anticipated load window at 10x the data would be 40 minutes and ~6 hours and 45 minutes at 100x.  A 10x solution may not need performance alterations / changes (e.g. load is completed within maintenance window; time required to performance tune may be better spent on other projects), a 100x solution may require us to examine performance bottlenecks (e.g. processor bound, network bound, DB bound) and explore the following solutions accordingly:
  
  * Spring batch parallelism and scaled processing - simultaneous processing of multiple files
  * Distributed processing of multiple files - distributing the solution to multiple hosts and distributing the files to be processed accordingly
  * DBMS scaling issues - index manipulation, DBMS tuning, DBMS hardware enhancements
  