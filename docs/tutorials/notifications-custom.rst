.. _tutorial-notifications-custom-code:

How to run custom code on notifications ?
=========================================

Kinto is able to execute some custom code when a particular event occurs.
For example, when a record is created or updated in a particular collection.

This tutorial presents the basic steps to run some code:

* synchronously in Python;
* asynchronously using a Redis queue, consumed via the language of your choice.


Run synchronous code
--------------------

In this example, we will send an email to some administrator every time
a new bucket is created.

To run this in production, we would rely on a local email server acting as a relay
in order to avoid bottlenecks. Or use the asynchronous approach otherwise.


Implement a listener
''''''''''''''''''''

Create a file :file:`kinto_email.py` with the following scaffold:

.. code-block:: python

    from cliquet.listeners import ListenerBase

    class Listener(ListenerBase):
        def __call__(self, event):
            print(event.payload)

    def load_from_config(config, prefix=''):
        return Listener()

Then, we will read the email server configuration and recipients from
the settings.


.. code-block:: python
    :emphasize-lines: 1,4,7-10,16-20

    from urllib.parse import urlparse

    from cliquet.listeners import ListenerBase
    from pyramid.settings import aslist

    class Listener(ListenerBase):
        def __init__(self, server, from, recipients):
            self.server_url = urlparse(server)
            self.from = from
            self.recipients = recipients

        def __call__(self, event):
            print(event.payload)

    def load_from_config(config, prefix=''):
        settings = config.get_settings()

        server = settings[prefix + 'server']
        from = settings[prefix + 'from']
        recipients = aslist(settings[prefix + 'recipients'])

        return Listener(server, from, recipients)


Now, every time a new event occurs, we send an email:


.. code-block:: python
    :emphasize-lines: 1,14-28

    import smtplib
    from urllib.parse import urlparse

    from cliquet.listeners import ListenerBase
    from pyramid.settings import aslist

    class Listener(ListenerBase):
        def __init__(self, server, from, recipients):
            self.server_url = urlparse(server)
            self.from = from
            self.recipients = recipients

        def __call__(self, event):
            recipients = ", ".join(self.recipients)
            subject = "%s %sd" % (event.payload['resource_name'],
                                  event.payload['action'])
            text = "User id: %s" % event.request.prefixed_userid
            message = ("\n"
                       "From: %s \n"
                       "To: %s \n"
                       "Subject: %s \n"
                       "%s \n") % (self.from, recipients, subject, text)

            server = smtplib.SMTP(self.server_url.netloc)
            server.starttls()
            server.login('username', 'password')
            server.sendmail(self.from, self.recipients, message)
            server.quit()

    def load_from_config(config, prefix=''):
        settings = config.get_settings()

        server = settings[prefix + 'server']
        from = settings[prefix + 'from']
        recipients = aslist(settings[prefix + 'recipients'])

        return Listener(server, from, recipients)


Add it to Python path
'''''''''''''''''''''

TBD

* setup.py ?
* ``export PYTHONPATH="${PYTHONPATH}:/path/to/script"`` ?

Enable in configuration
'''''''''''''''''''''''

:ref:`As explained in the settings section <configuring-notifications>`, just
enable a new listener pointing to your python module:

.. code-block:: ini

    cliquet.event_listeners = send_email

    cliquet.event_listeners.send_email.use = kinto_email
    cliquet.event_listeners.send_email.server = user:password@mail.server.com
    cliquet.event_listeners.send_email.from = postmaster@localhost
    cliquet.event_listeners.send_email.recipients = kinto@yopmail.com



Run asynchronous code
---------------------

TBD

* Setup Redis queue
* Consume events
