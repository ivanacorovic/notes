heroku create --stack cedar

git remote set-url heroku git@heroku.com:ancient-springs-6301.git

heroku run rake db:migrate

git push heroku master