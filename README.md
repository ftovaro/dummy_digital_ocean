# Description of this project
This project is a simple Rails app that is going to be deployed in DigitalOcean's App Platform. The app is a simple blog with posts and comments.

`main` is going to be always the default branch to test this project, we're going to have another branch called `production` to deploy the app.

# Steps to create this project
1. Fork this project.
2. Create a new branch called `production`.
3. We need the env files for the RAILS_MASTER_KEY, SECRET_KEY_BASE.
    1. To get the RAILS_MASTER_KEY, you can extract it from config/master.key file.
    2. To get the SECRET_KEY_BASE, we need to run the following command in the terminal:
        ```bash
        EDITOR="code --wait" bin/rails credentials:edit
        ```
        This will open a new file in the code editor. We need to copy the SECRET_KEY_BASE and paste it in the env file.
4. Create a .env file with this data, we're going to use it when setting the app.
5. This app starts without a database. We will attach it later to use the development database provided by DigitalOcean's App Platform. Otherwise, the deployment will fail because the development database is deployed after the Rails app, causing a deadlock since the Rails app requires a database to deploy.

# Steps to deploy in App Platform (DigitalOcean)

1. Create a new app in DigitalOcean's App Platform with GitHub as source provider.
![Step 1](doc/images/img1.png)

2. Connect your GitHub account and select the repository, and the `production` branch. Click on the Autodeploy checkbox and then next.
![Step 2](doc/images/img2.png)

3. We're going to keep things simple and cheap as possible.
   1. First of all, we're going to use the default Dockerfile in our app as our Build Strategy. So, if you see and extra instance with a Rails build pack, we're going to remove that.
      ![Step 3](doc/images/img3.png)
   2. We're going to use the smallest instance size, which is Basic - $5/month.
      ![Step 4](doc/images/img4.png)
   3. Now, time to set the env variables, click on `Edit` and then `Add from .env`, copy and paster your variables from your .env file.
      ![Step 5](doc/images/img5.png)
   4. Select the closest region to you.
   5. Choose a name for your app, select the DigitalOcean project and click on `Create app`.
      
4. Once the deployment is finished, you can access your app by clicking on the link provided by DigitalOcean.
![Step 6](doc/images/img6.png)
![Step 7](doc/images/img7.png)

5. Let's add the database now. Go to the `Create` tab and click on `Create/Attach Database`.
![Step 8](doc/images/img8.png)

6. We're going to use the smallest instance size, which is Basic - $7/month. The name of the database is going to be `dummy-digital-ocean-production`. Click on `Create Database`.
![Step 9](doc/images/img9.png)

7. Once the database is created, you can get the credentials by clicking on setting and then on the database, scroll down.
![Step 10](doc/images/img10.png)

8. Now, we're going to use env variable called `DATABASE_URL`.
   1. We're going to use the same DB of the app for cache, queue, cable to keep things simple.
   2. The env variable `DATABASE_URL` should already be defined by DigitalOcean, go to setting, your rails app and then `Environment Variables`.
      ![Step 11](doc/images/img11.png)
   3. Delete the current Dockerfile and remove the `.example` extension from the other Dockerfile.
   4. ⚠**️Important**! ⚠️, push these changes to the `production` branch before continue, I was having a problem with the gems and `bundle install` but this fixed it.
   5. Next, lets set the database in our project following these steps:
      1. Un comment the following gems in the Gemfile
          ```ruby
            gem "activerecord"
            gem "pg", "~> 1.1"
          ```
      2. Run `bundle install`.
      3. Modify your config/application.rb to load Active Record by uncommenting:
          ```ruby
            require "active_record/railtie"
          ```
      4. Remove the `.example` extension for the file in `app/models/application_record.rb.example`
      5. There is a `config/database.yml.example` file with your database settings. Remove the `.example`.
      6. Finally, we need to edit our `bin/docker-entrypoint` to run migrations. Add:
         ```bash
            # If running the rails server then create or migrate existing database
            if [ "${@: -2:1}" == "./bin/rails" ] && [ "${@: -1:1}" == "server" ]; then
            ./bin/rails db:prepare
            fi
         ```
         Just before
          ```bash
            exec "${@}"
          ```
9. Push these changes to the `production` branch and wait for the deployment to finish.
10. Now we can test our app, lets create our post model and run the migrations.
    1. Run the following command to create the post model:
        ```bash
          bin/rails g scaffold post title:string body:text
        ```
    2. Run the following command to create the database:
        ```bash
          bin/rails db:migrate
        ```
    3. Push these changes to the `production` branch and wait for the deployment to finish.
11. Now, access to `/posts` in your app and test creating your first post.
12. You should have a working Rails app deployed in DigitalOcean's App Platform.
![Step 11](doc/images/img11.png)

# How to set up your domain
I'm using GoDaddy for this test.

1. Go to `Settings` and `Domains`.
2. Click on `Edit` then `Add Domain`.
3. Write your domain and click on `Add Domain`.
   ![Step 12](doc/images/img12.png)
4. Copy the CNAME then click on `Add Domain`.
5. Go to Daddy and go to the `Manage DNS` section, then click on `Add new register` (My account is in Spanish).
![Step 13](doc/images/img13.png)
6. Select `CNAME` as the type, write the host and the value, then click on `Save`.
![Step 14](doc/images/img14.png)
7. Now go to `Redirect`, add new domain redirect, select `https`, write the subdomain we just created (with www), select 301 and save.
![Step 15](doc/images/img15.png)
8. Wait a few minutes, and you should be able to access your app with your domain.

# How to set up Active Job with Solid Queue and DigitalOcean Worker

1. Go to the `Create` tab and click on `Create/Attach` like we are attaching a new Web Service.
2. Select Worker instead of Web Service.
3. Same as before, select Docker, basic size, and the closest region.
4. The worker must have access to your database, so put in the env variables `DATABASE_URL` same as your Web Service, typically is something like `${name-of-dabatase.DATABASE_URL}`.
5. Click on `Create`.

### Installing `solid_queue`.

Since this is a test, we're going to use the same database for the queue, cache, and cable.
The `database.yml` already have this configuration.

Credits about how to use a single database for everything goes to [briancasel](https://briancasel.gitbook.io/cheatsheet/rails-1/setup-solid-queue-cable-cache-in-rails-8-to-share-a-single-database)

1. Uncomment the following gems to your Gemfile:
    ```ruby
      gem "solid_queue", "~> 1.1"

      gem "mission_control-jobs"
    ```
    Run `bundle install`. `mission_control` is our dashboard where we can see the current jobs.
2. Run the following command to install it:
    ```bash
      bin/rails solid_queue:install
    ```
3. Now, this is the tricky part, using a single database:
   1. When we ran the `solid_queue:install` command, it created the schema file for `queue`. 
   2. We are going create a new migration:
        ```bash
          bin/rails g migration addSolidQueueTables
        ```
   3. Next, copy what's **INSIDE** `ActiveRecord::Schema[8.0].define(version: 2025_03_16_001049) do`
   4. Paste it in the migration file. Inside the `change`.
   5. Run the migration.
   6. Now, our databases are in sync.
4. My suggestion is to push and wait to deploy these changes before continue. If you have any problem, try to do it one step at the time and pushing the changes.
5. Then, we need to modify `production.rb` to add the following configuration:
    (`SolidQueue` runs by default in our RAM)
    ```ruby
        config.active_job.queue_adapter = :solid_queue
        config.solid_queue.connects_to = { database: { writing: :queue } }
    ```
6. Next, we need to modify `Puma`: 
    1. Add the following line in the `config/puma.rb`:
        ```ruby
            # Run the Solid Queue supervisor inside of Puma for single-server deployments
            plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"] || Rails.env.development?
        ```
7. This still is not going to work in production, we have some missing pieces but we can test it in development.
8. Let's create a new job:
    ```bash
      bin/rails generate job guests_cleanup
    ```
9. Open the new job and write:
    ```ruby
      class GuestsCleanupJob < ApplicationJob
           queue_as :default
    
           def perform(*guests)
            Rails.logger.info("Inside the job")

            sleep 20
            Post.create(title: "Job", body: "Job content: #{Time.now}")
           end
         end
    ```
10. In our PostController, add the following line in the `index` method:
    ```ruby
      GuestsCleanupJob.perform_later
    ```
11. Run the server, for me, at the beginning it didn't work with `rails s`, so I had to run it with `bin/dev`, but after that it started worked with `rails s`.
12. Test your app, you should see the job running in the logs.

### Deploying the Worker
1. ⚠️**️Important**! ⚠️To make this work with Digital Ocean, we need to add the following env variable to the `Worker` in the DigitalOcean dashboard of our worker:
    ```bash
      $PROCESS_TYPE=worker
    ```
    This needs to be deployed before pushing the changes of our entry point.
2. Now, lets modify our `bin/docker-entrypoint`. Add the following lines:
    ```bash
      #!/bin/bash -e

      # Enable jemalloc for reduced memory usage and latency.
      if [ -z "${LD_PRELOAD+x}" ]; then
        LD_PRELOAD=$(find /usr/lib -name libjemalloc.so.2 -print -quit)
        export LD_PRELOAD
      fi

      # Check process type: default is 'web'
      PROCESS_TYPE="${PROCESS_TYPE:-web}"
    
      if [ "$PROCESS_TYPE" = "web" ]; then
        # If running the rails server, then create or migrate existing database
        if [ "${@: -2:1}" == "./bin/rails" ] && [ "${@: -1:1}" == "server" ]; then
        ./bin/rails db:prepare
        fi
    
        exec "${@}"
    
      elif [ "$PROCESS_TYPE" = "worker" ]; then
        echo "Running Solid Queue Worker..."
        exec bundle exec rails solid_queue:start
    
      else
        echo "Unknown PROCESS_TYPE: ${PROCESS_TYPE}"
        exit 1
      fi
    ```

## Notes

At the point of writing this, I don't remember if I had to set up the env variable `SOLID_QUEUE_IN_PUMA`.
I had to do some try and fail to make it work, and I don't remember that step.
But we have two options to define it if we need it:
1. In the `Environment Variables` in the DigitalOcean dashboard. (I think we'll need to set it for the entire app and not just the web service)
    ```bash
      SOLID_QUEUE_IN_PUMA=true
    ```
   I suggest testing this option first.
2. In the `Dockerfile`:
    ```bash
      # Set production environment
      ENV RAILS_ENV="production" \
          BUNDLE_DEPLOYMENT="1" \
          SOLID_QUEUE_IN_PUMA="true" \
          BUNDLE_PATH="/usr/local/bundle" \
          BUNDLE_WITHOUT="development"
    ```

# How to set up Mission Control

Mission Control only works if we have some credential set, for me, the easiest way to do it was:

1. Run the following command (There is a way to specify the environment, but for testing purposes, I'm going to use the default one):
    ```bash
      EDITOR="code --wait" bin/rails credentials:edit
    ```
2. Write the following:
    ```yaml
      mission_control:
        http_basic_auth_user: dev
        http_basic_auth_password: secret
    ```
   Save and close the file.
3. Add this route:
    ```ruby
      mount MissionControl::Jobs::Engine, at: "/jobs"
    ```
4. Now, when you access `/jobs`, you should be asked to enter the credentials, then you can see the dashboard.