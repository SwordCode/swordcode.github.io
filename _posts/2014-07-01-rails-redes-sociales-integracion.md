--- 
layout: default
title: Autenticación con credenciales de proveedores (Twitter, Facebook, Github, etc) en aplicaciones Rails usando Devise y Omniauth
comments: true
---

Integrar nuestra aplicación Rails para permitir que los usuarios utilicen sus credenciales de Twitter, Facebook, etc,  es relativamente rápido usando Devise y tomando ventaja de su integración con Omniauth.

<a href="https://github.com/intridea/omniauth">Omniauth</a> es una librería que nos permite estandarizar el proceso de autenticación en aplicaciones web con diferente proveedores.  Y <a href="https://github.com/plataformatec/devise">Devise</a> es el gem que actúa como nuestro administrador de todo lo concerniente a usuarios, incluyendo la administración de sesiones, destrucción de usuarios, etc.


El primer paso es registrar tu app en Twitter usando el Twitter Application Management (https://apps.twitter.com/).

![Interface de apps.twitter.com]({{ site.url }}/assets/swdcd_blog1.png)


Este proceso debe permitirnos recibir nuestras llaves (API Keys) que luego utilizaremos para que Twitter nos permita utilizar su servicio de autorización.

![Interface de apps.twitter.com]({{ site.url }}/assets/swdcd_blog.png)

Es importante no olvidar marcar la opción Allow this application to be used to Sign in with Twitter

![Interface de apps.twitter.com]({{ site.url }}/assets/swdcd_blog2.png)


Agregamos los siguientes gems al Gemfile de nuestro proyecto:

{% highlight ruby %}
gem 'devise'
gem 'omniauth-twitter'
{% endhighlight %}


Instalamos nuestros gems

{% highlight ruby %}
bundle install
{% endhighlight %}


Procedemos a instalar Devise

{% highlight ruby %}
rails g devise:install
{% endhighlight %}


Instalamos el modelo User de Devise,  que es el modelo que contendra los registros de los usuarios que se han registrados en nuestro app usando algún proveedor.

{% highlight ruby %}
rails d devise user
rake db:migrate
{% endhighlight %}

Agregamos el modelo identity


{% highlight ruby %}
rails g migration add_name_to_users name:string
rails g model identity user:references provider:string uid:string
{% endhighlight %}


Editamos el archivo app/models/identity.rb,  debe verse así:

{% highlight ruby %}
class Identity < ActiveRecord::Base
  belongs_to :user
  validates_presence_of :user_id, :uid, :provider
  validates_uniqueness_of :uid, :scope => :provider def self.find_for_oauth(auth)
   identity = find_by(provider: auth.provider, uid: auth.uid)
   if identity.nil?
     identity = create(uid: auth.uid, provider: auth.provider)
   end
   identity
  end
end

{% endhighlight %}

Es aqui donde vamos a utilizar las llaves secretas que generamos registando nuestra app en Twitter.  Abrimos el archivo app/config/initializers/devise.rb y agregamos las siguientes líneas al final del archivo:

{% highlight ruby %}
Devise.setup do |config|
...
 require 'omniauth-twitter'
 config.omniauth :twitter, 'TWITTER_CONSUMER_KEY', 'TWITTER_CONSUMER_SECRET'
...
end

{% endhighlight %}

Así es como debe quedar nuestro archivo config/routes.rb

{% highlight ruby %}
Rails.application.routes.draw do
...
 devise_for :users, :controllers => { omniauth_callbacks: 'omniauth_callbacks'}
 
  devise_scope :user do
   get 'sign_out', :to => 'devise/sessions#destroy', :as => :destroy_user_session
  end
...
{% endhighlight %}

Nuestro controlador app/controllers/omniauth_callbacks_controller.rb esta customizado para funcionar solo con Twitter,   pero puede ser implementado para trabajar con cualquier proveedor:

{% highlight ruby %}
class OmniauthCallbacksController < Devise::OmniauthCallbacksController
 def twitter
   auth = env["omniauth.auth"]
   @user = User.find_for_twitter_oauth(request.env["omniauth.auth"], current_user)
   if @user.persisted?
     flash[:notice] = I18n.t "devise.omniauth_callbacks.success"
     sign_in_and_redirect @user, :event => :authentication
   else
     session["devise.twitter_uid"] = request.env["omniauth.auth"]
     redirect_to new_user_registration_url
   end
 end
end
{% endhighlight %}


Nuestro modelo User en app/models/User.rb


{% highlight ruby %}
class User < ActiveRecord::Base
  devise :rememberable, :omniauthable
  attr_accessible :provider, :uid, :email, :name
  has_many :authentications
  def self.find_for_twitter_oauth(auth, signed_in_resource=nil)
    user = User.where(:provider => auth.provider, :uid => auth.uid).first
    if user
      return user
    else
      registered_user = User.where(:email => auth.uid + "@twitter.com").first
      if registered_user
        return registered_user
      else
        user = User.create(name:auth.extra.raw_info.name, provider:auth.provider, uid:auth.uid, email:auth.uid+"@twitter.com", password:Devise.friendly_token[0,20])
        return user
      end
    end
  end
end
{% endhighlight %}

No como otros proveedores,  Twitter no proporciona el correo de sus usuarios,  por lo tanto en el model User  a la hora de crear un nuevo usuario creamos el correo a partir del uid (id del usuario).

Y listo,  ya hemos implementado la autenticación utilizando Omniauth para Twitter, puedes probar su funcionamiento en <a href="http://localhost:3000/users/auth/twitter/">http://127.0.0.1:3000/users/auth/twitter/</a>

Visita el repositorio de está implementación en <a href="https://github.com/SwordCode/stock-dashboard">Stock Dashboard</a>. 

Un demo funcional puede ser visto en esta dirección <a href="http://gentle-tor-3942.herokuapp.com/">http://gentle-tor-3942.herokuapp.com/</a>

