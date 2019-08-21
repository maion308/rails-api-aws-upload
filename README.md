# rails-api-aws-upload

## Setting Up Your Rails App
Let's start by creating a basic Rails API by typing the following in the terminal:
```
rails new picture-app --api --database=postgresql
```

Once the new Rails API is finished building, `cd` into it and create a basic User model with the following:

`rails g model User name`

Then, run `rails db:create` and `rails db:migrate`.

We'll come back to our Rails app in a moment, but before we continue, let's get started on AWS.

## Setting Up AWS S3 Bucket
Amazon Web Services has a lot of available services. The S3 bucket is a popular storage system for files — especially images. 

**NOTE:** In order to set up an account, you will have to put a credit card on file. There will be no charge for a small app like we're building here; however, if you're not careful with your credentials and someone else gets access to your account, they may use an expensive amount of storage space and you will get charged for it by AWS. Their security team is pretty good at handling fraudulent instances like this; however, it's definitely a problem you want to avoid, so **hide your credentials**.

Sign up for an AWS account. When you log in, make sure you use **the email address** you signed up with, not a username. If the login screen is asking for your "IAM" username, you're probably logging in incorrectly. 

Once you're logged in, use the following guide to set up an S3 bucket on AWS: https://medium.com/alturasoluciones/setting-up-rails-5-active-storage-with-amazon-s3-3d158cf021ff

>Note: Bucket names on AWS **need to be unique** across all of Amazon Web Services, so you may have to be pretty creative. 

The important pieces you'll need are:
- your S3 bucket name
- your region (should be in the format of `us-east-1`. You'll should be able to see it in the URL bar)
- your access key id
- your secret access key

The last two are the most important to hide in credentials.

## Hiding Your Credentials
You can use the following guide if you'd like to know more about credentials: https://medium.com/craft-academy/encrypted-credentials-in-ruby-on-rails-9db1f36d8570

Here's the important steps:
- In the terminal, type the following (swap out `atom` for whatever text editor you're using):
```
EDITOR="code --wait" rails credentials:edit
```

>Note: If you're using a different text editor, you'll need a slightly different command. For example, if you're using Atom, the command will be `EDITOR="atom --wait" rails credentials:edit`. 

This command will open a file in your text editor where you can add your AWS credentials, like this:
```
  aws:
    access_key_id: your_access_key_id
    secret_access_key: your_secret_access_key
```

Save the file and close it. You should see a message in your terminal that says "New credentials encrypted and saved."

## Adding AWS to Your Rails Project

Inside your `config/storage.yml` file, uncomment the following AWS code:
```
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: your_region
  bucket: your_bucket_name
```

>Note: Make sure the `region` and `bucket` match up with the bucket you set up on AWS.

Next, head to `config/environments/development.rb` and change the line `config.active_storage.service = :local` to say `config.active_storage.service = :amazon`. 

In the future, when you deploy your app, you'll want to make the same change in your `config/environments/production.rb` file.

## Adjusting Your User Model
Go to the `user.rb` file in your models folder and add the following:
```
class User < ApplicationRecord
  has_one_attached :picture
end
```

You don't have to use the phrase `:picture`. You can call it whatever you want; however, you'll need to make sure that the `user_params` in your user controller and the file input in your user form use the exact same name that you use here.

## Setting Up Your Routes and UsersController
In your `config/routes.rb` file, set up your routes with `resources :users`.

Then, create a new file in your `app/controllers` folder called `users_controller.rb`. You can use the following code in that file:

```
class UsersController < ApplicationController
  def index
    @users = User.all
    render json: @users
  end

  def show
    @user = User.find(params[:id])
    render json: @user
  end

  def create
    @user = User.new(user_params)
    if @user.save
      render json: @user, status: :created
    else
      render json: @user.errors, status: :unprocessable_entity
    end
  end

private

  def user_params
    params.permit(:name, :picture)
  end

end
```

>Note: Notice how, in the `user_params` that we included `:picture`, just like we wrote in the User model.

>Note: Also notice how, in the `user_params`, that we didn't write `require(:user)`. This is **not** an issue with uploading images - it's just how the app in this lesson is set up. Basically, the simplified front end that we'll be using for this specific lesson doesn't package data inside of an object called "user" at this stage of the process, so that line would most likely create a bug. Normally, with a real React front end, you would package the data inside of an object called "user," and this would not be an issue. However, if you ever run into a bug here, this is an okay solution. A better solution is figuring out how to package your data inside an object called "user," but... one thing at a time. :)

## Setting Up Cors and Adding AWS Gem
Go to your gem file and add `gem 'aws-sdk-s3'` and uncomment the following line: `gem 'rack-cors'`. Then run `bundle install` in your terminal.

Next, go to `config/initializers/cors.rb` and uncomment the following:
```
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

>Note: Don't forget to update the `origins` to '*'

## Add ActiveStorage
We almost forgot a key ingredient here - ActiveStorage! As you know ActiveRecord is how Rails talks to your database. ActiveStorage is an optional part of ActiveRecord that helps you store files. 

Go ahead and add ActiveStorage through the terminal with the following command: `rails active_storage:install`.

Although this isn't a model, you'll still have to migrate it using `rails db:migrate`. For learning purposes, go ahead and take a look at the migration file before you do, but don't make any changes.

**Great work! Your API is now ready to store images to AWS S3 when a new user is created.** 

Now we need to hit our `/users` endpoint with a post request from a front end client. 

Let's do that!

## Creating a Front End
Let's build a quick front end with regular JavaScript to test our API. Create a new project folder for your front end with `mkdir picture-app-client` and `cd` into it. Then, create HTML and JS files with `touch index.html script.js`.

### index.html
```
<!DOCTYPE html>
<html>

<head>
  <title>AWS Practice</title>
  <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
  <script defer src="script.js"></script>
</head>

<body>
  <form>
    <input id="name" type="text" name="name" placeholder="Name...">
    <input id="file" type="file" name="picture">
    <input type="submit" id="button" name="" value="Submit">
  </form>
</body>

</html>
```

## JS
```
const button = document.querySelector("#button")

button.addEventListener("click", async (e) => {
  e.preventDefault();
  const form = document.querySelector("form");
  const userData = new FormData(form);
  const resp = await axios.post("http://localhost:3000/users", userData);
  console.log(resp);
});
```

## Test It Out
Launch your Rails server, load your front end in the browser, and try out your new app by entering a user name and uploading an image. If you open up the console in the Chrome inspect tools, you should see the `resp` logging a 200 status code with the new user you created.

>Note: If you did not get a 200 status code, read your error messages and work through whatever bugs you see before continuing.

Unfortunately, even if everything worked correctly, the API is only responding with the new user's name; it's not responding with the user's picture yet. We'll get to that in just a second.

For now, let's just make sure the image actually uploaded to AWS and our database. To check this, first go to your AWS S3 bucket and see if anything has been added. You should see something that looks like this:

![](https://res.cloudinary.com/briandanger/image/upload/v1566282779/Screen_Shot_2019-08-20_at_2.32.38_AM_ewifxf.png)

Next, check your Rails console in your terminal by typing `rails c`. Once in the terminal, check to see if your user got a picture with `User.last.picture`. You should see something like:
```
#<ActiveStorage::Attached::One:0x00007f8d3a3dc560 @name="picture", @record=#<User id: 11, name: "Brian", created_at: "2019-08-19 21:05:28", updated_at: "2019-08-19 21:05:28">, @dependent=:purge_later>
```

If you want to see the image's URL, you can check it with `User.last.picture.service_url`. You should see a crazy URL like: 
```
"https://rails-upload-practice.s3.amazonaws.com/inxTtWGzg6f4sdzuYj67uhpP?response-content-disposition=inline%3B%20filename%3D%221231695_10153219715030352_539034509_n.jpg%22%3B%20filename%2A%3DUTF-8%27%271231695_10153219715030352_539034509_n.jpg&response-content-type=image%2Fjpeg&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIA34WTDVOYV4FH6MW5%2F20190820%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20190820T063721Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=a8f93a875fac380d41d061c68044c3bd27bf1722ee6853d5b1bd3a9933099a24"
```

Try copying and pasting this URL into your browser. You should see the image that you just uploaded.

If you've gotten this far, that means that everything's working; however, as things stand, getting the URL of your uploaded image from your API will be challenging. Let's do a little more work now to make our future lives easier.

## Upgrading our API with Rails Serializer
Serializer is a Rails gem that allows us to format our JSON without having to lift a finger on the front end. We can select only the information we want, as well as get access to our relationships with a single request.

Add the Serializer gem to your Gemfile with `gem 'active_model_serializers'` , and then run `bundle install` in your terminal.

Next, generate your Serializer folder by typing `rails g serializer user` in your terminal. This should create the following folder and file in your Rails API: `/app/serializers/user_serializer.rb`.

Because you installed Serializer, Rails will now look to this folder before rendering a resource. Each model in your Rails API will need a corresponding serializer file in this `/app/serializers` folder. Thats because when you pass an object to a `render :json` line in your controller, it’s now up to Serializer which attributes will actually be sent as a JSON response.

In our UserSerializer, we'll add the following code:
```
attributes :id, :name, :picture

def picture
  object.picture.service_url if object.picture.attached?
end
```

This code tells our UserController that, when it responds to an HTTP request with a user object (e.g. `render json @user`), it should respond with the user's id, name, and picture. Because "picture" is not a built in part of the User model, we need to define exactly what we want our API to respond with for that attribute. The conditional `if` helps prevent errors by not trying to force your json to return a picture for users who's picture may be broken, deleted, or otherwise unavailable.

## Re-Testing Our App
Re-launch your Rails server, go back to your front-end in the browser, and create another user with an image. When you check the response in the terminal now, you should see a response like:

```
data:
id: 2
name: "Brian"
picture: "https://rails-upload-practice.s3.amazonaws.com/wdDAEfvF25NuU8nt5wVAgbT4?response-content-disposition=inline%3B%20filename%3D%221231695_10...
```

Nice! Next, check `localhost:3000/users`. You should see a JSON response containing an id, name, and picture for each user.

# You did it! Wooooooo!

![](https://media0.giphy.com/media/jga0lOVI14kco/giphy.gif?cid=790b7611f29c7b0dbc725e17fa2e0b0b11258ad4b1aa59de&rid=giphy.gif)



