# Facebook-connect on top of devise

## Gemfile

```ruby
# Debug gems
group :development do
  gem "quiet_assets"
  gem "better_errors"
  gem "binding_of_caller"
end

# Devise with fb-omniauth extension
gem "devise"
gem 'omniauth-facebook'

# For extended use of fb API (getting user's friends infos, posting on his wall..)
gem 'koala'

# For handling API keys
gem 'figaro', '~> 1.0.0.rc1'
```

Then run `bundle install`.

## Stand-alone devise

Run the following devise generators. Skip these steps if you have already integrated devise.

- `rails generate devise:install` then follow instructions from terminal
- `rails generate devise User`
- `rails generate devise:views` if you want to override devise signin/signup views
- `rake db:migrate` to create the `users` table with devise fields (email, psswd)

Add the following filter to the controller(s) for which you want to impose signin. If you put this filter in `application_controller.rb`, sign-in will be imposed for all views.

```ruby
before_action :authenticate_user!
```

## Integrate omniauth-facebook

What follows is a summary of the [devise omniauth wiki](https://github.com/plataformatec/devise/wiki/OmniAuth:-Overview).

### Create the FB app

- Sign-in on [FB developper platform](https://developers.facebook.com) and create a new app.
- On app settings, **don't forget to add a web-platform with http://localhost:3000/ as your site URL**

Remember your App ID and App Secret for what follows.

### Configure devise with API keys

To protect our keys use the [figaro gem](https://github.com/laserlemon/figaro).

- run `rails generate figaro:install` if you haven't already. It will create a *config/application.yml* to put all your API keys and add this file in *.gitignore*

- copy / paste your FB keys in *config/application.yml* as follows

```
development:
  FB_ID: 4*********0
  FB_SECRET: 5********************2
```

- Then modify `config/initializers/devise.rb` to tell devise to use these keys

```ruby
config.omniauth :facebook, ENV["FB_ID"], ENV["FB_SECRET"]
```

- On OS/X you may need to add these two lines instead to disable certificate verifications in development mode.

```ruby
OpenSSL::SSL::VERIFY_PEER = OpenSSL::SSL::VERIFY_NONE if Rails.env.development?
config.omniauth :facebook, ENV["FB_ID"], ENV["FB_SECRET"]
```

Now you are set up to integrate FB connect in your core-app (routes/controller/model)


### Add omniauth callbacks controller and routing

- Create a new controller **app/controllers/users/omniauth_callbacks_controller.rb**. This will handle all omniauth callbacks from other services. In this controller, add a `facebook` action that handle fb callback as follows

```ruby
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def facebook
    # You need to implement the method below in your model (e.g. app/models/user.rb)
    @user = User.find_for_facebook_oauth(request.env["omniauth.auth"])

    if @user.persisted?
      sign_in_and_redirect @user, :event => :authentication #this will throw if @user is not activated
      set_flash_message(:notice, :success, :kind => "Facebook") if is_navigational_format?
    else
      session["devise.facebook_data"] = request.env["omniauth.auth"]
      redirect_to new_user_registration_url
    end
  end
end
```

- Note that all the magic will come from `User#find_for_facebook_oauth` class method (see next section)

- Change the normal `devise_for :users` route to link omniauth callbacks to this new controller.

```ruby
devise_for :users, :controllers => { :omniauth_callbacks => "users/omniauth_callbacks" }
```

### Now pimp the model

Make your user model omniauthable.

```ruby
class User < ActiveRecord::Base
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable


  # Following line makes your model fb-omniauthable
  devise :omniauthable, :omniauth_providers => [:facebook]

end
```

For the `User` model, we want to add new attributes we will retrieve from FB (like the profile picture, the username, or the FB token in order to use FB API). Generate a new migration for that


```
rails g migration AddColumnsToUsers provider uid picture name token token_expiry:datetime
```

And run `rake db:migrate` to run this migration


Then add the `find_for_facebook_oauth` in your user model. This is the one we call in the facebook action of our callbacks controller. This method will retrieve all user's infos from fb callbacks.

```ruby
def self.find_for_facebook_oauth(auth)
  where(auth.slice(:provider, :uid)).first_or_create do |user|
      user.provider = auth.provider
      user.uid = auth.uid
      user.email = auth.info.email
      user.password = Devise.friendly_token[0,20]
      user.name = auth.info.name   # assuming the user model has a name
      user.picture = auth.info.image # assuming the user model has an image
      user.token = auth.credentials.token
      user.token_expiry = Time.at(auth.credentials.expires_at)
  end
end
```


### Get a cool navbar with FB profile pic

We are nice buddies, we give you the code for a wunderbar-navbar integrating fb profile pic.

```html
<nav class="navbar navbar-default" role="navigation">
  <div class="container-fluid">
    <!-- Brand and toggle get grouped for better mobile display -->
    <div class="navbar-header">
      <a class="navbar-brand" href="#">FACEBOOK-CONNECT</a>
    </div>

    <!-- Collect the nav links, forms, and other content for toggling -->
    <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
      <ul class="nav navbar-nav navbar-right">

        <% if user_signed_in? %>
        <li class="dropdown">
          <% if current_user.provider %>
            <a href="#" class="dropdown-toggle" data-toggle="dropdown"><%= image_tag current_user.picture, class: "img img-circle" %><b class="caret"></b></a>
          <% else %>
            <a href="#" class="dropdown-toggle" data-toggle="dropdown"><%= current_user.email %><b class="caret"></b></a>
          <% end %>
          <ul class="dropdown-menu">
            <li><%= link_to "Sign Out", destroy_user_session_path, method: :delete %></li>
          </ul>
        </li>
        <% else %>

        <% end %>
      </ul>
    </div><!-- /.navbar-collapse -->
  </div><!-- /.container-fluid -->
</nav>
```

Even some css rules to make your navbar look nice

```css
.navbar {
  height: 80px;
}

.navbar >.container-fluid .navbar-brand {
  line-height: 50px;
}

.navbar-default .navbar-nav>li>a {
  line-height: 50px;
}

```


















