11/30/19


Deployment not working - saying that it can’t open the database connection
It seems that the way that I’ve specified the database file path is incorrect on the heroku backend
Going to test whether I can change it on github so that I remove the ‘short-message-survey’ piece of the database URI.
The next step is to figure out how to specify different values for whether I’m in my dev environment or deploying on heroku

Changing the URI worked, I get the placeholder app page that shows the questions now when I deploy.
However, I text our number and it doesn’t respond with the survey, so I think I probably have to login to twilio and/or run the source.env on the account because I don’t think I do that anywhere

First gonna try to open up the heroku bash and setting the environmental variables for our twilio account

That didn’t work. Texted the number and still nothing.

Works after I pointed the webhook url to the short-message-survey.herokuapp.com/message

Didn’t have to do anything else with the credentials, it appears to just work with the webhook set



Now need to try to set up the database config. Need to use the postgres when on Heroku or it doesn’t keep the stuff in storage. I also need to set it up to use sqlite when in dev environment, but postgres when in production. This will likely require setting environment vars and possibly doing this via a (travis) config file to make sure it always happens right.

First see if I can get the app to be using the postgres

I know how to switch the database URI, but then how do I make sure it’s working right? Also, do I need to seed the database and run the database server and if so, wil the commands that are preprogrammed in the app work, or will I need to define these for the postgres environment. And lastly, can I have heroku do periodic database dumps/backups?


Seems like I was able to move over to postgres but I still don’t know exactly what I did or if I’ll have to do it again. I ran some of the commands here by logging into heroku bash after the app didn’t deploy correctly: https://notes.andreagalloni.eu/flask-db-migrate-target-database-is-not-up-to-date/
When I got the “target database not up to date” error, and that seemed to work, especialy flask db stamp heads seemed to help. It made the app not throw an error on deploy, so that’s promising. Still need to figure out how to set different environments for development and production though.

I texted the number and I got the response “sorry, but there are no surveys to be answered”
I recognize this from somewhere in the code and I believe there is a command that will load the survey from the json file into the database and then it should work.

I did the dbseed command and that worked, and now the thing is up and running using the postgres database



12/6/19 

App is fully working, but need to now work on building out some of the functionality

What I’m focusing on is the storing of the numbers in the database

First and foremost, I need a way to store and retrieve the list of phone numbers from the postgres database

So I need to figure out a way to hold the numbers in the database, whether that means creating a new table or finding a way to incorporate that in the existing tables

So the first action item would appear to be looking at the existing structure of the database to see where the number list thing would fit in.

Then I need to figure out how I can add to the models script to add or modify the tables as necessary. 

Then I need to figure out what sequence of restarting/reseeding the database that I need to do to have the new tables take effect.

Then I need to figure out how to make the query to the database within the app in the send sms module to get the numbers to feed into the message the list function

I think that another aspect of this will be to modify the existing Q and A tables so that they also contain the student’s phone number as a unique ID, because it doesn’t seem that that’s in there anywhere right now. Will need to cross reference this with what twilio saves to make sure we have more than one way of getting at that info

There may be issues if I just modify the models.py script to add another db model where the preprogrammed tables are referenced and I’ll need to add something there, but this seems like the place to start, but try to figure out if the models are imported or referenced in any of the other db setup code


It seems relatively straightforward to set up the new table in models (remember that when it becomes a big PITA) but something to take a note of is the foreign key attribute, which seems to be for connecting columns for multiple tables 




Heroku cli command sequence:

heroku login (asks to press one key to open up browser and verify)

heroku run bash -a short-message-survey





SO HERES HOW I GOT THE DATABASE TO INCLUDE MY NEW TABLE

It seems like I did this whole command sequence before, with the exception of the first one, and it didn’t work, so even though I’m not sure how this relates, this is what I did

I accessed the postgres database from the terminal and the Heroku cli using this command

heroku pg:psql -a short-message-survey

I used 

\dt 

To show the tables 

Which showed this

             	List of relations
 Schema |  	Name   	| Type  | 	Owner 	 
--------+-----------------+-------+----------------
 public | alembic_version | table | eylbtbwiiflsbs
 public | answers     	| table | eylbtbwiiflsbs
 public | questions   	| table | eylbtbwiiflsbs
 public | surveys     	| table | eylbtbwiiflsbs

Then I used 

DROP TABLE alembic_version;

To get rid of that table, based on the advice I found here: https://github.com/prerit2010/Result-aggregation-server/issues/37

Then in another terminal I opened bash using this command:

heroku run bash -a short-message-survey

And I deleted the migrations folder (because running these commands in a previous iteration said that the problem was the migrations folder already existed) by doing:

rm -rf migrations

Then I ran these commands in sequence at the heroku bash (based on advice from the same github thread above)

python manage.py db init
python manage.py db migrate
python manage.py db upgrade

And once I had done that my new table shows up (along with the alembic_version table, lol)




The next order of business is to figure out how to make the calls to the new table to get the numbers into the send_sms script.  Maybe I need to modify the database schema to either A) have a flag for which sample that we’re using the app for in the numbers table, or B) to set up different numbers tables for different subsamples


Is there a way to redirect the app behavior from the sent message stage? Like if I send the survey prompt message, send responses to the autosurvey loop. But it I send the phone number prompt message, send it to another routing that puts the phone numbers into the database. But there is no phone number prompt message. These are unprompted messages. So maybe since it’s at a different time I can manually switch the default behavior at the different times and based on what we want to do.


To add stuff to the database manually, I first login to Heroku bash

heroku run bash -a short-message-survey

Then I open python

python

Then I import the app and db

from sms_app import app, db

Then import the database model of interest

from sms_app.models import Number

Then add the row

db.save(Number(number=’+15172400923’, name=’alex’))


So I’ve run into a slight snag when it comes to querying the database from python and getting the results to drop to a dictionary. The sqlalchemy query functionality is more complicated than I have the bandwidth to absorb right now, but I do have a slightly hacky way to do it that will work, so I’m going to go with that right now.

12/12 

The next thing to do is to test the send_sms function, see if I can message the list. The main thing I need to figure out is where exactly I need to put this in the internal server logic/structure. Do I need to trigger this from the manage.py script or what do I need to do exactly?

Then I think the thing I need to do is look at different routing behavior. See what I can do about different responses to incoming texts

The send sms and message the list functions are working correctly, and I fixed the code that pulls the numbers from the database into the list object. Then I added the import line to pull in this code into the scheduling script. The schedule script will then be used to create the message job queue, and then I think that it’ll just be a matter of importing the scheduler object into the manage.py script, and running the run scheduled jobs function.  I don’t know where this has to go, whether in the main body of the script, or before or after the main run function call (probably after? But maybe that makes it not run until I pull the server down)



So when I added the template code that I had copied as and example and put it in the manager script, it makes the app time out for some reason

I uncommented the import statement for the scheduler ojbect, we’ll see what happens. That made the app mad, so apparently there’s some bad code in that scheduling file?

I deleted the open-ended while loop in the scheduling file and that fixed it of course


Running the run_schedule() function in the interactive session seems to do it, even though it returns a weird error message, and actually it only sent the message once.

When i run the scheduler on bash it sends me the message every minute, so I uncomment that code int the manager script, and the app runs fine, but it doesn’t send the messages


I’m looking into the possibility of using the python AP scheduler package instead of schedule, just because it seems like I’ve got a bit of troubleshooitng to do here anyway, and AP scheduler has a flask extension that goes with it, and it seems like it has a little bit more functionality, like you can store the jobs in a persistent way using the database, and the support for running jobs just once seems a bit more robust, but mainly I’m hoping that the support of the flask extension will make things a little bit simpler for managing the process of integrating it with the app, with regard to the multi threading and what not. T

THere are multiple types of schedulers, and the background scheduler seems to be the right option which should hopefully work the right way without interfering with the main thread


TO DO:
Include the ap scheduler and flask extension in the requirements file
Put the import statements in the main file




This from APScheduler works interactively:

From apschedler.schedulers.background import BackgroundScheduler
scheduler = BackgroundScheduler()
scheduler.add_job(outgoing_sms, 'interval', minutes=1, args = ['+15172400923', 'Lets see if this works'])
scheduler.start()

scheduler.shutdown()


But again doesn’t work when I put it in the app and push it to live. It’s definitely related to the response in the apscheduler FAQ thing about not running my jobs, due to some combination of threading issues and/or WGSI issues. I need to start by looking at that response and trying to investigate that further. See what they say the answer is. It may be doing --enable-threads, but I don’t know if what I’m using is ‘uWGSI’ (https://apscheduler.readthedocs.io/en/latest/faq.html). I’ll need to be thinking about whether I’ll need to use the flask extension or if the straight APscheduler will work for me. Presumably they have the solution if this is a Flask endemic issue, but their code seems slightly sketchy compared to APScheduler, so if I don’t have to use it, that’s probably better.

Could also be related to the manager thing, which is discussed here
https://stackoverflow.com/questions/46201734/integrating-flask-apscheduler-with-flask-migrate-and-flask-script

