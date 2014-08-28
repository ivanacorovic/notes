#Third party javascript widget

###Test web page that will contain your app's widget:

1) $ sudo nano /etc/hosts
  Add lines: 

    127.0.0.1     publisher.dev

2) sudo nano /etc/apache2/apache2.conf
  Add lines: 


    NameVirtualHost *:80
    <VirtualHost *:80>
      ServerName publisher.dev
      DocumentRoot "/Users/username/project/publisher"
    </VirtualHost>

3) sudo service apache2 restart


4) publisher/publish.html:
  Make sure you have 

    <script type="text/javascript" src="http://code.jquery.com/jquery-1.10.1.min.js"></script>
  
5) Add this where you want the widget to show:

     <div data-ideus-product= "5" data-ideus-token="yXuV7qJ8xqMqweMKJpyN" id = "test1"></div>

  ``data-ideus-product`` is id of the product you want to see.
  ``data-ideus-token `` is your private_token for out app. How to get it? Go to our app, create an account if you don't have one. Go to your profile, and the on Account (top of the page). You'll see ``Private token`` text box. Copy its content and that is your provate_token.

6) Add this code as last in the ``body`` of your page:

    <script>
      var script = document.createElement('script');
      script.async = true;
      script.src = "http://localhost:3000/ideus-widget/nutrition_widget.js";
      $("body").append(script);
    </script>

7) Minimum example:
  
    <!DOCTYPE html>
    <meta charset="UTF-8"> 
    <html>
      <head>
        <title>Publisher Test Page</title>
      </head>
      <body>

        <h1>Publisher Test Page</h1>
        <div data-ideus-product= "5" data-ideus-token="yXuV7qJ8xqMqweMKJpyN" id = "test1"></div>
        <script type="text/javascript" src="http://code.jquery.com/jquery-1.10.1.min.js"></script>
        <script>
          var script = document.createElement('script');
          script.async = true;
          script.src = "http://localhost:3000/ideus-widget/nutrition_widget.js";
          $("body").append(script);
        </script>


      </body>
    </html>

For this label, you need UTF-8 encoding. 

8) To run this on localhost (port 80), stop the service (if any) that is currently using port 80. You need admin rightis for these lower ports. To check, run:

    $ sudo  netstat -tulpn | grep :80

  To stop it, run:

    $ sudo service WHICHEVER_SERVICE_IS_ON_PORT_80 stop

  To run your app (in this case publisher), go to the root folder of it and run:

    $ sudo python -m SimpleHTTPServer 80

  You can run it on any free port. Default is 8000.

#####You're good to go!

###On the application side (Rails 4 application)

1) add the foloowing gems to Gemfie:

    gem 'rack-jsonp-middleware', :require => 'rack/jsonp'
    gem 'rack-cors', require: 'rack/cors'

2) $ bundle install

3) application.rb 
  Add lines:
    
    ...
      # Allow access to GitLab API from other domains
      config.middleware.use Rack::Cors do
        allow do
          origins '*'
          resource '/api/*', headers: :any, methods: [:get, :post, :options, :put, :delete]
        end
      end

      config.middleware.use Rack::JSONP
    end
  end

4) application_controller.rb
    
    ...
    protect_from_forgery

    before_filter :cors_set_access_control_headers
    ...

    def cors_set_access_control_headers
      headers['Access-Control-Allow-Origin'] = 'http://publisher.dev'
      headers['Access-Control-Allow-Methods'] = 'GET'
      headers['Access-Control-Request-Method'] = 'GET'
      headers['Access-Control-Allow-Headers'] = 'Origin, X-Requested-With, Content-Type, Accept, Authorization'
      headers['Access-Control-Max-Age'] = "1728000"
      headers['X-Frame-Options'] = 'ALLOW-FROM http://publisher.dev'
    end

5) products_controller.rb

    class ProductsController < ApplicationController

      ...
      respond_to :html, :json
      ...

      def show
          @product = Product.find(params[:id])
          @vitamins = @product.product_vitamins
          @ingredients = @product.product_ingredients
          @nutrients = @product.product_nutrients
          @minerals = @product.product_minerals

          if session["has_counted_view_#{params[:id]}"] == nil
            impressionist(@product)
            session["has_counted_view_#{params[:id]}"] = true
          end

          respond_with(@product.as_json(product_data))
        end
      
      ...

      def product_data
        {include: 
            [{distributors: {only: [:id, :name, :address, :phone, :fax, :email, :website, :pib]}}, 
             {manufacturers: {only: [:id, :name, :address, :phone, :fax, :email, :website, :pib]}},
             {photos: {}},
             {ingredients: {only: [:name, :alternative_name, :description]}},
             {product_nutrients: {methods: %w(name description reference_unit reference_value), only: [:id, :name, :amount]}},
             {product_minerals: {methods: %w(name alternative_name english_name description reference_unit reference_value), only: [:id, :amount]}},
             {product_vitamins: {methods: %w(name alternative_name english_name description reference_unit reference_value), only: [:id, :amount]}},
             {product_rsymbols: {only: [:recycling_code, :abbreviation, :name, :english_name, :description, :uses]}},
             {product_csymbols: {only: [:recycling_code, :abbreviation, :name, :english_name, :description, :uses]}},
             ],
              only: [:id, :name, :barcode, :description, :prepare_instructions, :storage_instructions, :net_weight, :reference_unit, :origin]
        }
      end
      ...
    end


6) product_nutrients.rb

    class ProductNutrient < ActiveRecord::Base
      ...
      delegate :name, :description, :reference_unit, :reference_value, to: :nutrient
    end

7) Create new folder in ~/gitlab/public called ideus-widget

8) Add these tree files to public/ideus-widget: [nutritionLabel.css](https://bitbucket.org/ivanacorovic/gitlabhq2/raw/57689636a6520c608f3a23096cc022e2762cae2f/public/ideus-widget/nutritionLabel.css), [nutritionLabel,js](https://bitbucket.org/ivanacorovic/gitlabhq2/src/57689636a6520c608f3a23096cc022e2762cae2f/public/ideus-widget/nutritionLabel.js?at=cors), [nutrition_widget.js](https://bitbucket.org/ivanacorovic/gitlabhq2/raw/57689636a6520c608f3a23096cc022e2762cae2f/public/ideus-widget/nutrition_widget.js)

For ``nutritionLabel.js`` DON NOT use ``raw`` while copying, it's not encoded right! Copy it from the link provided.

#####Have fun! :)
  
