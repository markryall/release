# Example for pushing a static site to heroku.

heroku apps:create
heroku config:set BUILDPACK_URL=https://github.com/ddollar/heroku-buildpack-multi.git API_BASE=http://deets.herokuapp.com
git push heroku master
