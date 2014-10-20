We've developing a new application from scratch recently and have taken the
opportunity to adopt a new highfalutin tech stack.

The front end is a single page web app developed with browserify and react.  The
back end is a rails application containing only JSON REST services (no html
rendering at all).  Each component is deployed independently.

This post will focus on the release and deployment process for the front end.
Subsequent posts will walk backwards through the rest of the application.

The initial plan was to generate the static assets (media, css and js) for the
front end then deploy and host it on S3.  This was fine except that our front
end routing uses push state and we wanted automatic gzip compression of static
assets.

divshot.io seemed like a promising option until we started testing the
application with internet explorer 9.  The CORS implementation in IE9 is so
limited that it seems pointless to have it at all.

The eventual solution we have chosen is to host the static content in our own
deis environment using an nginx custom build pack.  I'll walk through deployment
to heroku because that works exactly the same way.

That is what I will walk through here.  One other requirement we realised would
be necessary is a mechanism to have environment specific js configuration.
divshot.io have their own mechanism to deal with this so we rolled our own
equivalent.

The prerequisites of this walkthough are that you have created a heroku account
and installing the heroku toolbelt.  You will also need to generate and upload
an ssh key in order to be able to `git push heroku master` for deployment.  I'll
just wait here while you do that.  No rush.

Step one is to clone this repository containing our static site.  This
repository contains successive releases of the generated static site.  Other
than index.html, each asset will have revisioning using a hash to ensure these
resources can be cached indefinitely.  A subsequent post will demonstrate how
this is done as part of an automated build step.

    git clone http://github.com/markryall/release

Step two is creating a heroku app and specifying the url for a custom build
pack.  This is accomplished by simply setting an environment variable.

    heroku apps:create
    heroku config:set BUILDPACK_URL=https://github.com/ddollar/heroku-buildpack-multi.git NAME=my_app API_BASE=http://deets.herokuapp.com

This build pack allows multiple build packs to be loaded at the same time.
Although we won't need them, the ngnix build pack allows additional build packs
to also be loaded (such as for a node or rails application).
