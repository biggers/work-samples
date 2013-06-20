=================================
Deploying XYZ code to ABC servers
=================================

:Author: Mark Biggers <biggers@utsl.com>
:Description: step-by-step code & data deployment for XYZ
:Ref: _`runit process manager`: <https://wiki.archlinux.org/index.php/Runit>
:Ref: _`Git tips, tricks & workflows`: <http://nuclearsquid.com/writings/git-tricks-tips-workflows/>
:Ref: _`restview, Restructured Text Viewer`: <https://pypi.python.org/pypi/restview>
:Revision: 1.0
:To View: restview fhn-code-deployment.rst
:Metainfo: `Introductory ReST docs <http://docutils.sf.net/rst.html>`_
:Organization: ABCWidgets, Inc. -- <http://www.abcwidgets.com>
:Date: 16 June 2013

:Copyright: ABCWidgets, Inc. 2013

-------------------------------------

.. contents:: **Table of Contents**

.. section-numbering::

-------------------------------------

Deploy the Qualcomm .war file
=============================

Deploying the Qualcomm "service" is as easy as a rsync-copy, to ``Tomcat7``.
Tomcat7 will auto-deploy the ``.war`` bundle, when the file is received. ::

 rsync -av Qualcomm.war  prod:/var/lib/tomcat7/webapps/

On Production ``fhn.abcwidgets.com``, note the results. ::

 $ ll /var/lib/tomcat7/webapps/

  total 3064
  drwxr-xr-x 5 tomcat7  tomcat7     4096 Jun 12 16:08 Qualcomm/  <<== deployed!
  -rw-r--r-- 1 mbiggers mbiggers 3123554 Jun 11 00:14 Qualcomm.war
  drwxr-xr-x 3 root     root        4096 May 28 17:40 ROOT/


If there is a need to restart the ``Tomcat7`` app-server, do: ::

 $ sudo  service tomcat7 status  
  * Tomcat servlet engine is running with pid 1421

 $ sudo  service tomcat7 restart
  * Stopping Tomcat servlet engine tomcat7  [ OK ] 
  * Starting Tomcat servlet engine tomcat7  [ OK ] 

 $ sudo  service tomcat7 status
 
"Upgrade" the MySQL DB
======================
**NOTE**: parts of this are preliminary, and *not* the best way to handle DB changes in the future!

-------------------------
Dump the MySQL DB for XYZ
-------------------------
On a developer desktop, or on ``dev.abcwidgets.com``, we first *dump* the current MySQL DB.  There will be a request for the DB password! ::

 $ HOST=localhost  DBUSER=root  DB=abcwidgets  DATE=$(date +%F)

 $ mysqldump -h$HOST -u$DBUSER -p$ASK_FOR_PW --hex-blob --routines --triggers ${DB} | gzip > ${DB}-${DATE}.sql.gz


Copy that *SQL dump* to Production. ::

 $ rsync -av ${DB}-${DATE}.sql.gz  prod:~/   ## developer homedir

------------------------------
Stop Apache2 (XYZ application)
------------------------------
Login to Production.

We are using ``runit`` for process management, going forward.  Currently *celeryd* and *apache2* services, are managed by ``runsvdir``, configured within ``/etc/service/service-name``.

Please refer to ``man sv`` for details - the ``runit`` UI for managing processes. ::

 $ alias ru='sudo sv -v'

 $ ru status /etc/service/*   ## All processes that are managed by 'runit'

  run: /etc/service/apache2: (pid 2666) 18581s
  run: /etc/service/celeryd: (pid 750) 23259s

 $ ru stop apache2

 $ ru status /etc/service/*   ## Review services status, again!

-----------------------------------
Load the XYZ DB data, on Production
-----------------------------------
**WARNING:** this will destroy the old XYZ DB, on Production!  It can be backed up first, using ``mysqldump`` as shown above! ::

 $ DB=abcwidgets
 $ echo "DROP DATABASE $DB; CREATE DATABASE $DB;" | mysql --verbose -u root -p

Now, use the ``mysql`` command to load the ``XYZ`` DB dump, from above. Keep a log of the load-attempt, to review for any errors. ::

 $  HOST=localhost  DBUSER=root  DB=abcwidgets  DATE=$(date +%F)
 $  ZDUMP=${DB}-${DATE}.sql.gz

 $ cd
 $  script -c "zcat $ZDUMP | mysql -h$HOST -u$DBUSER -p$ASK_FOR_PW --verbose $DB" db.0.log

Deploy the changes to XYZ code
==============================

-----------------------------
"Git" the updated code & data
-----------------------------
Login to Production, and do the following.  *Log* the git-pull, to see any issues from the Git operation.

If necessary, *first* edit the ``/var/django/xyz_mobile/.git/config`` file, to change to your ``Github`` credentials.  *NOTE: there has to be a better way to do this!*  ::

 $ cd /var/django/xyz_mobile

 $ script -c "git pull" pull.1.log

---------------------------------
Restart Apache2 (XYZ application)
---------------------------------
Now the Django site will have the latest code, after restart of the ``apache2`` service.  ::

 $ alias ru='sudo sv -v'

 $ ru status /etc/service/*   ## All processes that are managed by 'runit'

 $ ru restart apache2

 $ ru status /etc/service/*   ## Review services status, again!


README :: various XYZ Notes
===========================

-----------------------------
The Python env for XYZ-django
-----------------------------
There is a *virtualenv* installed, in ``/var/django/xyz_mobile/abcwidgets``.  All required Python packages including Django 1.4, are specified in ``requirements.txt``, and were installed by ``pip`` into ``lib/python2.7/site-packages``.

To run the Django XYZ-service manually, do: ::

 $ ru stop apache2  ## probably a very good idea!

 $ cd /var/django/xyz_mobile/abcwidgets

 $ . bin/activate

 $ ./manage.py runserver

 $ deactivate  ## the virtualenv, when finished

----------------
XYZ WSGI scripts
----------------
The mod_wsgi script is located in and served by Apache2, from ``/var/django/xyz_mobile/abcwidgets/sbin/wsgi.py``.

The ``celeryd`` and ``apache2`` scripts for ``runit``, are also in that ``sbin/`` folder.

