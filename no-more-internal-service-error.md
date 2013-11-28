Date: 2013-11-27
Title: How we solved the immense pain of error code 500:internal service error
Tags: 500, error, internal service error
Slug: no-more-internal-service-error
category: tech
Gplusid: +SalilPanikkaveettil
Summary: How did pnr.me faced a number of internal service errors and how we set out to fix them one by one

We explained earlier [how our servers went down crashing](|filename|/how-we-got-our-first-1000-visitors.md "how our servers crashed after pnr.me was shared by Alok Rodinhood Kejriwal"), after being shared by [mr rodinhood](https://www.facebook.com/rodinhood "Alok Rodinhood Kejriwal"). We needed a fix and we needed it fast

What the problem was not
------------------------

One of our first assumption was that the error had something to do with apache. Reading a lot of hackernews makes you believe that apache is the culprit behind any server side error and nginx is the solution. But as we deepdived into the problem some things became very clear

 * Our apache server was never throttled
 * What went down was the application itself, to be more clear the [prediction engine of pnr.me](http://pnr.me/predictor "pnr prediction engine")

skeleton of pnr.me
------------------

pnr.me has two main components, a flask powered web app and a python predictor deamon communicating with each other using zmq. There is good reason why we arrived at this design. Predictor deamon is a number crunching application which required good amount of memory. To make matters worse, it was using [scipy](http://www.scipy.org/ "scientific python") which is a memory heavy python library. Hence it was not advisable to have the predictor application embedded in the web app as mod_wsgi will have a tough time serving this memory heavy app. 

Now call it bad code, our predictor deamon was throwing some exception and dying at regular intervals. To make the matter worse, we had not set up any kind of monitoring mechanism which would inform us when the predictor deamon goes down. As a ugly hack, we stumbled upon this beatiful code which would respawn a deamon if it gets killed or if it is stopped by some random reason

	:::bash
	until (python predictor_deamon.py); do
	    echo "Server 'myserver' crashed with exit code $?.  Respawning.." >&2
	    sleep 1
	done

Internal service errors would still show for users but atleast it wouldn`t affect others from accessing the website

The web app
-----------

Once internal server error of predictor deamon got almost resolved, we were quite sure that we would not get any more internal service errors. As the great [Thorin Oakenshield](http://www.imdb.com/character/ch0000164/quotes "Thorin Oakenshield") once said "I've never been so wrong in all my life".

We deploy [pnr.me](http://pnr.me "pnr.me") after doing basic testing. But these basic tests cannot handle blackswan errors like the one i recently faced. I was getting pymongo.errors.OperationFailure: quota exceeded error and because of this, the webapp was throwing an internal service error. How did pnr.me solve pymongo.errors.OperationFailure: quota exceeded when using MongoLab is a story for another blog post.

We cannot solve all Blackswan events however hard we try. We needed an alert mechanism by which we could react fast to these errors. By following the beautiful articles on [how to set up a SMTPHandler for flask](http://flask.pocoo.org/docs/errorhandling/ "how to send automated mails when flask server throws an error") and how to use [GMail SMTP server with logging module](http://mynthon.net/howto/-/python/python%20-%20logging.SMTPHandler-how-to-use-gmail-smtp-server.txt "How to use Gmail SMTP server with logging module"), we created a alert mechanism which would mail us the second any error is thrown by the flask web server

Final setup and improvement
---------------------------
* We will be alerted by email when some error is thrown by flask.
* Our predictor deamon will respawn within a second of dying. We can fix the error later as all the errors are anyways logged
* A big TODO is to create such alerts for python jobs which run independant of the flask web service
