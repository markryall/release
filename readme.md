# Technology stack series part N of N

We've been developing a new application from scratch recently and have taken the opportunity to
adopt a new highfalutin tech stack.

In this series of posts, we're going to walk backwards through the stack starting with the
production deployment because that should be the first step for any kind of project.

The front end is a single page web app developed with browserify and react.  The back end is a rails
application containing only JSON REST services (no html rendering at all).  Each component is
deployed and configured independently.

This post will just focus on the release and deployment process for the front end. Subsequent posts
will walk backwards through the remainder of the application.

Our initial idea was to generate the static assets (media, css and js) for the front end then deploy
and host it as a site on S3 then push to the cloudfront CDN.  This was fine except that our front
end routing uses HTML5 history.pushState and we wanted automatic gzip compression of static assets.
Neither of these problems are solved cleanly with S3 hosting.

divshot.io seemed like a promising option until we started testing the application with Internet
Explorer 9.  The CORS implementation in IE9 is so limited that it almost seems pointless to have it
at all. We decided instead to avoid CORS entirely by proxying back end services.

This solution we have chosen is to host the static content in our own deis environment using a
custom nginx build pack.  This post walks through deployment to heroku because that works exactly
the same way.

Another requirement we realised would be necessary is a mechanism to have environment specific js
configuration on each deployment environment. divshot.io have their own mechanism to deal with this
so we rolled our own equivalent.

The prerequisites of this walkthough are that you have created a heroku account and installing the
heroku toolbelt.  You will also need to generate and upload an ssh key in order to be able to `git
push heroku master` for deployment.

Step one is to clone the repository containing our static site.  This repository contains successive
releases of the generated static site (rather than the actual source code).  Commits to this
repository are performed as part of our automated build process.

Other than index.html, each asset will have revisioning using a hash to ensure these resources can
be cached indefinitely. A subsequent post will demonstrate how this asset fingerprinting is done as
part of a release build step.

    git clone http://github.com/markryall/release

Step two is creating a heroku app and specifying the url for a custom build pack.  This is
accomplished by simply setting an environment variable.

    heroku apps:create
    heroku config:set BUILDPACK_URL=https://github.com/ddollar/heroku-buildpack-multi.git NAME=production API_BASE=http://deets.herokuapp.com

This build pack allows multiple build packs to be loaded at the same time. The specific build pack we're using is specified in the `.buildpacks` file in the release repository.  Although we won't need
them, the ngnix build pack allows additional build packs to also be loaded so that it is possible to delegate requests to a node or
rails application.

The final deployment step is simply a git push.

    git push heroku master

Most of the magic for all of this is in the nginx.conf file.  The nginx build pack generates this from an erb template because nginx doesn't make using environment variables particularly convenient.

Here's the configuration in its entirety:

    daemon off;
    worker_processes 4;

    events {
      use epoll;
      accept_mutex on;
      worker_connections 1024;
    }

    http {
      include <%= ENV["HOME"] %>/mime.types;
      root <%= ENV["HOME"] %>/public;

      log_format   main '$remote_addr - $remote_user [$time_local]  $status "$request" $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"';
      access_log logs/nginx/access.log main;
      error_log logs/nginx/error.log;
      rewrite_log on;

      gzip on;
      gzip_disable "msie6";
      gzip_vary on;
      gzip_proxied any;
      gzip_comp_level 6;
      gzip_buffers 16 8k;
      gzip_http_version 1.1;
      gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

      sendfile     on;
      tcp_nopush   on;
      underscores_in_headers on;

      server {
        listen <%= ENV["PORT"] %> default_server;
        server_name _;
        keepalive_timeout 5;

        location /scripts/env.js {
          expires off;
        }

        location / {
          expires max;
          try_files $uri @index;
        }

        location @index {
          expires off;
          rewrite .* /index.html break;
        }

        location /api/ {
          proxy_pass <%= ENV["API_BASE"] %>;
        }
      }
    }

Most of this configuration is nginx boiler plate but there are are few sections that are worth mentioning.  Firstly, the `location /scripts/env.js` rule ensures that the generated js environment file is never cached by a browser.  Likewise. the combination of the `location /` and `location @index` sections ensure that any static assets that are requested will be returned with maximum cache expiry while any other request will return `index.html` with no caching.  This ensures that whenever the site is redeployed, the next request will return a new copy of `index.html` referencing the newly fingerprinted changes static assets.  It also manages push state as any request that does not match a file will return `index.html` which is the entire application.

The `location /api/` rule allows the front end application to proxy the back end services.  This avoids all CORS related issues on legacy browsers (i.e. IE).

The next post will explain the build process that creates the releases we are creating from the source repository and then committing to the release git repository.
