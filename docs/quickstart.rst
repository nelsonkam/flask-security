Quick Start
===========

There are some complete (but simple) examples available in the *examples* directory of the
`Flask-Security repo`_

* :ref:`basic-sqlalchemy-application`
* :ref:`basic-sqlalchemy-application-with-session`
* :ref:`basic-mongoengine-application`
* :ref:`basic-peewee-application`
* :ref:`mail-configuration`
* :ref:`proxy-configuration`

.. _basic-sqlalchemy-application:

Basic SQLAlchemy Application
----------------------------

SQLAlchemy Install requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

     $ mkvirtualenv <your-app-name>
     $ pip install flask-security-too flask-sqlalchemy


SQLAlchemy Application
~~~~~~~~~~~~~~~~~~~~~~

The following code sample illustrates how to get started as quickly as
possible using Flask-SQLAlchemy and the built-in model mixins:

::

    from flask import Flask, render_template_string
    from flask_sqlalchemy import SQLAlchemy
    from flask_security import Security, SQLAlchemyUserDatastore, login_required
    from flask_security.models import fsqla

    # Create app
    app = Flask(__name__)
    app.config['DEBUG'] = True
    app.config['SECRET_KEY'] = 'super-secret'
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite://'
    # Bcrypt is set as default SECURITY_PASSWORD_HASH, which requires a salt
    app.config['SECURITY_PASSWORD_SALT'] = 'super-secret-random-salt'

    # As of Flask-SQLAlchemy 2.4.0 it is easy to pass in options directly to the
    # underlying engine. This option makes sure that DB connections from the
    # pool are still valid. Important for entire application since
    # many DBaaS options automatically close idle connections.
    app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
        "pool_pre_ping": True,
    }

    # Create database connection object
    db = SQLAlchemy(app)

    # Define models
    fsqla.FsModels.set_db_info(db)

    class Role(db.Model, fsqla.FsRoleMixin):
        pass

    class User(db.Model, fsqla.FsUserMixin):
        pass

    # Setup Flask-Security
    user_datastore = SQLAlchemyUserDatastore(db, User, Role)
    security = Security(app, user_datastore)

    # Create a user to test with
    @app.before_first_request
    def create_user():
        db.create_all()
        user_datastore.create_user(email='test@me.com', password='password')
        db.session.commit()

    # Views
    @app.route('/')
    @login_required
    def home():
        return render_template_string("Hello {{ current_user.email }}")

    if __name__ == '__main__':
        app.run()

.. _basic-sqlalchemy-application-with-session:

Basic SQLAlchemy Application with session
-----------------------------------------

SQLAlchemy Install requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

     $ mkvirtualenv <your-app-name>
     $ pip install flask-security-too sqlalchemy

Also, you can use the extension `Flask-SQLAlchemy-Session documentation
<http://flask-sqlalchemy-session.readthedocs.io/en/latest/>`_.

SQLAlchemy Application
~~~~~~~~~~~~~~~~~~~~~~

The following code sample illustrates how to get started as quickly as
possible using `SQLAlchemy in a declarative way
<http://flask.pocoo.org/docs/1.0/patterns/sqlalchemy/#declarative>`_:

We are gonna split the application at least in three files: app.py, database.py
and models.py. You can also do the models a folder and spread your tables there.

- app.py ::

    from flask import Flask, render_template_string
    from flask_security import Security, current_user, login_required, \
         SQLAlchemySessionUserDatastore
    from database import db_session, init_db
    from models import User, Role

    # Create app
    app = Flask(__name__)
    app.config['DEBUG'] = True
    app.config['SECRET_KEY'] = 'super-secret'
    # Bcrypt is set as default SECURITY_PASSWORD_HASH, which requires a salt
    app.config['SECURITY_PASSWORD_SALT'] = 'super-secret-random-salt'

    # Setup Flask-Security
    user_datastore = SQLAlchemySessionUserDatastore(db_session,
                                                    User, Role)
    security = Security(app, user_datastore)

    # Create a user to test with
    @app.before_first_request
    def create_user():
        init_db()
        user_datastore.create_user(email='matt@nobien.net', password='password')
        db_session.commit()

    # Views
    @app.route('/')
    @login_required
    def home():
        return render_template_string('Hello {{email}} !', email=current_user.email)

    if __name__ == '__main__':
        app.run()

- database.py ::

    from sqlalchemy import create_engine
    from sqlalchemy.orm import scoped_session, sessionmaker
    from sqlalchemy.ext.declarative import declarative_base

    engine = create_engine('sqlite:////tmp/test.db', \
                           convert_unicode=True)
    db_session = scoped_session(sessionmaker(autocommit=False,
                                             autoflush=False,
                                             bind=engine))
    Base = declarative_base()
    Base.query = db_session.query_property()

    def init_db():
        # import all modules here that might define models so that
        # they will be registered properly on the metadata.  Otherwise
        # you will have to import them first before calling init_db()
        import models
        Base.metadata.create_all(bind=engine)

- models.py ::

    from database import Base
    from flask_security import UserMixin, RoleMixin
    from sqlalchemy import create_engine
    from sqlalchemy.orm import relationship, backref
    from sqlalchemy import Boolean, DateTime, Column, Integer, \
                           String, ForeignKey

    class RolesUsers(Base):
        __tablename__ = 'roles_users'
        id = Column(Integer(), primary_key=True)
        user_id = Column('user_id', Integer(), ForeignKey('user.id'))
        role_id = Column('role_id', Integer(), ForeignKey('role.id'))

    class Role(Base, RoleMixin):
        __tablename__ = 'role'
        id = Column(Integer(), primary_key=True)
        name = Column(String(80), unique=True)
        description = Column(String(255))

    class User(Base, UserMixin):
        __tablename__ = 'user'
        id = Column(Integer, primary_key=True)
        email = Column(String(255), unique=True)
        username = Column(String(255))
        password = Column(String(255))
        last_login_at = Column(DateTime())
        current_login_at = Column(DateTime())
        last_login_ip = Column(String(100))
        current_login_ip = Column(String(100))
        login_count = Column(Integer)
        active = Column(Boolean())
        fs_uniquifier = Column(String(255))
        confirmed_at = Column(DateTime())
        roles = relationship('Role', secondary='roles_users',
                             backref=backref('users', lazy='dynamic'))

.. _basic-mongoengine-application:

Basic MongoEngine Application
-----------------------------

MongoEngine Install requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    $ mkvirtualenv <your-app-name>
    $ pip install flask-security-too flask-mongoengine

MongoEngine Application
~~~~~~~~~~~~~~~~~~~~~~~

The following code sample illustrates how to get started as quickly as
possible using MongoEngine:

::

    from flask import Flask, render_template
    from flask_mongoengine import MongoEngine
    from flask_security import Security, MongoEngineUserDatastore, \
        UserMixin, RoleMixin, login_required

    # Create app
    app = Flask(__name__)
    app.config['DEBUG'] = True
    app.config['SECRET_KEY'] = 'super-secret'
    # Bcrypt is set as default SECURITY_PASSWORD_HASH, which requires a salt
    app.config['SECURITY_PASSWORD_SALT'] = 'super-secret-random-salt'

    # MongoDB Config
    app.config['MONGODB_DB'] = 'mydatabase'
    app.config['MONGODB_HOST'] = 'localhost'
    app.config['MONGODB_PORT'] = 27017

    # Create database connection object
    db = MongoEngine(app)

    class Role(db.Document, RoleMixin):
        name = db.StringField(max_length=80, unique=True)
        description = db.StringField(max_length=255)

    class User(db.Document, UserMixin):
        email = db.StringField(max_length=255)
        password = db.StringField(max_length=255)
        active = db.BooleanField(default=True)
        fs_uniquifier = db.StringField(max_length=255)
        confirmed_at = db.DateTimeField()
        roles = db.ListField(db.ReferenceField(Role), default=[])

    # Setup Flask-Security
    user_datastore = MongoEngineUserDatastore(db, User, Role)
    security = Security(app, user_datastore)

    # Create a user to test with
    @app.before_first_request
    def create_user():
        user_datastore.create_user(email='matt@nobien.net', password='password')

    # Views
    @app.route('/')
    @login_required
    def home():
        return render_template('index.html')

    if __name__ == '__main__':
        app.run()


.. _basic-peewee-application:

Basic Peewee Application
------------------------

Peewee Install requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    $ mkvirtualenv <your-app-name>
    $ pip install flask-security-too flask-peewee

Peewee Application
~~~~~~~~~~~~~~~~~~

The following code sample illustrates how to get started as quickly as
possible using Peewee:

::

    from flask import Flask, render_template
    from flask_peewee.db import Database
    from peewee import *
    from flask_security import Security, PeeweeUserDatastore, \
        UserMixin, RoleMixin, login_required

    # Create app
    app = Flask(__name__)
    app.config['DEBUG'] = True
    app.config['SECRET_KEY'] = 'super-secret'
    app.config['DATABASE'] = {
        'name': 'example.db',
        'engine': 'peewee.SqliteDatabase',
    }
    # Bcrypt is set as default SECURITY_PASSWORD_HASH, which requires a salt
    app.config['SECURITY_PASSWORD_SALT'] = 'super-secret-random-salt'

    # Create database connection object
    db = Database(app)

    class Role(db.Model, RoleMixin):
        name = CharField(unique=True)
        description = TextField(null=True)

    class User(db.Model, UserMixin):
        email = TextField()
        password = TextField()
        active = BooleanField(default=True)
        fs_uniquifier = TextField()
        confirmed_at = DateTimeField(null=True)

    class UserRoles(db.Model):
        # Because peewee does not come with built-in many-to-many
        # relationships, we need this intermediary class to link
        # user to roles.
        user = ForeignKeyField(User, related_name='roles')
        role = ForeignKeyField(Role, related_name='users')
        name = property(lambda self: self.role.name)
        description = property(lambda self: self.role.description)

    # Setup Flask-Security
    user_datastore = PeeweeUserDatastore(db, User, Role, UserRoles)
    security = Security(app, user_datastore)

    # Create a user to test with
    @app.before_first_request
    def create_user():
        for Model in (Role, User, UserRoles):
            Model.drop_table(fail_silently=True)
            Model.create_table(fail_silently=True)
        user_datastore.create_user(email='matt@nobien.net', password='password')

    # Views
    @app.route('/')
    @login_required
    def home():
        return render_template('index.html')

    if __name__ == '__main__':
        app.run()


.. _mail-configuration:

Mail Configuration
------------------

Flask-Security integrates with Flask-Mail to handle all email
communications between user and site, so it's important to configure
Flask-Mail with your email server details so Flask-Security can talk
with Flask-Mail correctly.

The following code illustrates a basic setup, which could be added to
the basic application code in the previous section::

    # At top of file
    from flask_mail import Mail

    # After 'Create app'
    app.config['MAIL_SERVER'] = 'smtp.example.com'
    app.config['MAIL_PORT'] = 465
    app.config['MAIL_USE_SSL'] = True
    app.config['MAIL_USERNAME'] = 'username'
    app.config['MAIL_PASSWORD'] = 'password'
    mail = Mail(app)

To learn more about the various Flask-Mail settings to configure it to
work with your particular email server configuration, please see the
`Flask-Mail documentation <http://packages.python.org/Flask-Mail/>`_.

.. _proxy-configuration:

Proxy Configuration
-------------------

The user tracking features need an additional configuration
in HTTP proxy environment. The following code illustrates a setup
with a single HTTP proxy in front of the web application::

    # At top of file
    from werkzeug.contrib.fixers import ProxyFix

    # After 'Create app'
    app.wsgi_app = ProxyFix(app.wsgi_app, num_proxies=1)

To learn more about the ``ProxyFix`` middleware, please see the
`Werkzeug documentation <https://werkzeug.palletsprojects.com/en/0.15.x/middleware/proxy_fix/#module-werkzeug.middleware.proxy_fix>`_.

.. _Flask-Security repo: https://github.com/jwag956/flask-security
