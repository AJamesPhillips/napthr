
# Napthr

Provides a base web app from which to efficiently build secure and scalable
applications.

* Nginx - high performance web server
* Ansible - straightforward provisioning and deployment tool
* PostgreSQL - backend data store (would be possible to change for another SQL
    or non SQL store)
* Typescript + tslint - for refactoring, code completion and code navigation
    (see Jest for testing)
* [Hapi.js](github.com/hapijs/hapi) - secure, well maintained, high performance web application server.
    Enables isomorphic application if required through set up of API urls, model
    / data validation, model / data schemas, business logic between front end and
    server side.  Django was a close second (admin interface and safer model
    SQL migrations).
* React.js + [Redux](http://redux.js.org/) - liberating UI and state management
    libraries.  Enables complex and compelling UX when used correctly.
* Sequelize - ORM for PostgreSQL.
    Caveat: Sequelize is a fantastic project but there are 2 main down sides of
    which this project has only tackled the first.
    * Firstly the database config has to be extracted from the .env* files as they
        cannot be provided via the sequelize cli.
    * Secondly model types are enforced in one set of files, the database schema
        are written manually in others, and the migrations are again manually written
        in a third.
    It would be preferable to have a proper tool chain such as Django's South
    migration manager.
* Webpack - for HMR (Hot Module Replacment) and asset bundling etc.
* Jest - for unit and integration testing
* Create React App - Webpack, Jest, and React are brought in by Create React App.
    Specifically [the typescript fork](github.com/wmonk/create-react-app-typescript/)
    of this project.

### TODO

- [ ] Move payload_validation from signin/up to RequestPayload namespace
- [ ] Add robots.txt
- [ ] Refactor redux store to follow best practices (using action creators) which
        will enable server side view rendering.
- [ ] CSRF protection
- [ ] [install fail2ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [ ] email & txt client
- [ ] Add email confirmation field to user model, email to user, confirmation page
- [ ] Add password reset mechanism
- [ ] Add admin panel to see users
- [ ] Ensure `clean_up` does not need to be called to close db connections when
        in tests which are purely of front end.
- [ ] Fix / return promises to jest testing framework / hard fail with env var
        when tests expectations fail.
- [ ] automate renewing ssl certification using something like
        https://gist.github.com/renchap/c093702f06df69ba5cac#file-readme-md
        and submit PR back to lets_encrypt repo
- [ ] Get `@import 'some_css'` working in .tsx and .css
- [ ] Get `:root { --colour-white: #ffffff;` and `var(--colour-white)` working.
- [ ] Enable development in vagrant / virtualbox.  Webpack HMR causes box to hang.
- [ ] Localisation
- [ ] Docker containerise
- [ ] Support (long polling etc) / warn when multiple tabs open
- [ ] Experiment with isomorphic and use mapDispatchToProps instead of the "dispatchers" etc to allow this.
- [ ] Post ejection TODO list (see below)

## Updating

As this app uses the [wmonk's Typescript fork](https://github.com/wmonk/create-react-app-typescript)
of the [Facebook create-react-app](https://github.com/facebookincubator/create-react-app)
it can and will be updated.
Follow
[instructions to update](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#updating-to-new-releases)
to new releases.

## Setup

On Mac:

    $ git clone --recursive git@github.com:AJamesPhillips/napthr.git
    $ brew install yarn postgresql
    $ brew services start postgresql  # If it hasn't started by itself
    $ npm install -f npm-run

    app/frontend$ yarn install
    app/frontend$ npm start

    app/server$ yarn install
    app/server$ ./db.sh setup  # you will need to edit your .env files to change DB_DATABASE if you have another version of this project

    app/server$ npm run build
    # Undo any migrations, run all migrations and reseed db
    app/server$ ./db.sh dev DESTROY

## Database

### Migrations

To apply a newly written migration

    app/server$ npm run build
    app/server$ ./db.sh dev migrate

### Postgres shell and Sequelize migrations

    $ ./db.sh t|test|d|dev
    $ ./db.sh t|test|d|dev m|migrate
    $ ./db.sh t|test|d|dev u|undo

#### Common commands

    main_data=> \l             # list databases
    main_data=> \c db_name     # change database
    main_data=> \d             # list tables
    main_data=> \d+ "user";    # describe a table
    main_data=> \?             # list available commands

## Tests

Start the tests for frontend and backend server.  Note the `--runInBand` option
runs the tests sequentially which is necessary for tests touching the db.

    # app$ npm test -- --runInBand  # TODO fix this

### Open test connections causing failure

The tests sometimes fail hard with Jest failing to capture and handle the
failures.  This can result in connections being left open to the database.  If
these exceed the maximum number of connections, all the tests may start failing
with strange errors.  The max connections is about 100 and as more than 7
connections are made every test run, these can be exhausted very quickly.

To see the open connections run:

    select * from pg_stat_activity where datname = 'test_db';

To kill the open connections run:

    SELECT pg_terminate_backend(pg_stat_activity.pid) FROM pg_stat_activity WHERE pg_stat_activity.datname = 'test_db' AND pid <> pg_backend_pid();

## Start front and back end development servers

Start the create-react-app frontend server:

    app/frontend$ npm start

Start the backend server:

    app/server$ npm start

## Debugging

    ./app_shell.sh dev  # start a shell to access the models

For the front end, react-devtools-extension and redux-devtools-extension are
supported.

### Live

#### Tail logs

    remote$ tail -f /var/log/web_server/*.log /var/log/nginx/*.log

#### When all else fails: Restarting web app server

    $ service web_app_server restart

## Provisioning and Deploying a server

Uses ansible roles and config for provisioning new machines.

### Setup local ansible

You will need to have git, virtualenv, python, and pip installed already

    $ source s/setup-ansible.sh

### Create new server

If possible, create the machine with:

 * an ssh key (for ssh access as root user)

#### Inventory file

Add its IP address to `deploy/inventory_file` and to your `~/.ssh/config`.
The former is the default inventory file as specified by `ansible.cfg` which
will allow ansible to find the machine.  (See the `deploy/inventory_file.template`
for a pattern to copy).
The .ssh/config will allow you to ssh onto the machine directly in case you
need to manually perform actions such as check logs, temporary one-off config
etc.  Enter something like:

    Host <ip addresses from inventory_file> <your domain name.com, same as REACT_APP_SERVER_HOST>
        User <your user name>
        IdentityFile ~/.ssh/<your_private_key_here>

Note:  We override `remote_user` as `root` in `playbook_bootstrap.yml`

### Bootstrap server(s)

This is to perform several things, including:

* Ensures that a host is configured for management with Ansible.
* Change from using root user for ssh'ing in to using your user.

Run:

    $ ENV=production ansible-playbook ./deploy/playbook_bootstrap.yml --limit "<production_yourdomain or whatever value you changed it to in your inventory_file>"

You should now be able to ssh into the server as the deploy user with your ssh
key.  Because this is not idempotent, if run without error, it can only be and
only needs to be run once per machine.

#### Check it's working

    provision$  ansible all -m ping

Note this will fail with `/bin/sh: 1: /usr/bin/python: not found` before the
bootstrap is performed.

### Main Provision and Deploy

The `playbook_bootstrap` should only need to be run once and as it's
not idempotent it can only be run once per machine (if it runs without error).
The provision playbook/roles, whilst unlikely to change much
may need to be run more than once and the deploy playbook/roles will need to be
run every time the code changes.

    $ export ENV=production &&
    $ export ENV=staging &&

      export APP_COM=webservers &&
      export APP_COM=dbserver &&
      export APP_COM=load_balancer &&

      ansible-playbook ./deploy/playbook.yml --limit "$APP_COM:&$ENV" --tags "provision_$APP_COM"
      ansible-playbook ./deploy/playbook.yml --limit "$APP_COM:&$ENV" --tags "deploy_$APP_COM"

Or

    $ export ENV=production &&
    $ export ENV=staging &&

    ansible-playbook ./deploy/playbook.yml --limit "$ENV"

    --tags "deploy"  # all necessary deploy roles run
    --tags "update_user_access"  # Update all user permissions on all servers

### Tail logs

    remote$ tail -f /var/log/web_server/**/*.log /var/log/nginx/*.log

## Maintenance

### Renewing SSL certificates

* manually ssh onto server
* sudo su
* cd /usr/local/share/letsencrypt/env
* service nginx stop
* bin/letsencrypt renew
* service nginx start

## Ejecting

This is based upon [`create-react-app`](https://github.com/facebookincubator/create-react-app#converting-to-a-custom-setup)
and can use the one way, irrevocable "ejection" mechanism to give full control
over all the configuration being used.

    npm run eject

### Features todo post eject

- [ ] Optionally split front end app code into 3 parts.  Currently all the code
    is visible to anyone from <REACT_APP_SERVER_HOST>.  It could be split into
    `app/public`, `app/private` (which is only visible to those user who have
    signed in) and `app/private_admin` (content only shown to trusted users).
    `server` is obviously private and resides solely on the server.
- [ ] Remove APP_NAME from environment variables and commit to code.
