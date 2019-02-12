# Shiny demo app with Apache load balancer on Heroku

This is a Shiny app that uses an Apache load balancer to launch multiple R processes on a single Heroku performance dyno.

## Heroku setup

1. Create a new app via the Heroku web dashboard. Ours is called "apache-shiny-demo".
2. Downgrade the [Heroku stack](https://devcenter.heroku.com/articles/stack) to heroku-16:
         
	```bash
   heroku stack:set heroku-16 --app apache-shiny-demo
   ```

3. Add the following two buildpack URLs in the app settings:
	* [https://github.com/harvard-vpal/heroku-buildpack-r](https://github.com/harvard-vpal/heroku-buildpack-r)
	* [https://github.com/harvard-vpal/heroku-buildpack-apache](https://github.com/harvard-vpal/heroku-buildpack-apache)

 To lock the buildpack versions to a specific Git branch or tag (recommended), you can instead use this format:
 
   * `https://github.com/harvard-vpal/heroku-buildpack-r.git#v1.0.0`
   * `https://github.com/harvard-vpal/heroku-buildpack-r.git#some-branch-name`

4. Set the Git remote for Heroku and push the code:

   ```bash
   heroku git:remote --app apache-shiny-demo
   git push heroku master
   ```
5. Add a single web dyno in the Resources section of your Heroku's app dashboard.

6. Open the app in your browser and confirm it works.

Since Shiny uses a persistent Websocket connection to send data between browser and R, a user's browsing session must be backed by a persistent R process on the server. Thus, we must run the app on a single dyno and can't use Heroku's auto-scaling feature. Instead, we can choose a performance dyno, which has multiple cores, and use an Apache load balancer to launch multiple R processes and gain some concurrency.

## Dependencies

The Shiny app must have dependencies tracked with [Packrat](https://rstudio.github.io/packrat). Run `packrat::snapshot()` before committing code to ensure all required package versions are locked down.

There might be additional Ubuntu/Debian dependencies for certain packages. Specify these in `Aptfile` (one per line). For instance, this demo app loads [dplyr](https://github.com/tidyverse/dplyr/), so `libxml2-dev` is required. If a deployment fails while restoring Packrat packages, note which library is missing and add it to `Aptfile`.

## Build cache

The first push to your Heroku remote will compile R, Apache, and install all required dependencies. These results will be cached so that subsequent deploys are faster. However, you must manually invalidate the build cache if you do any of the following:

* change the buildpacks
* add a dependency in `Aptfile`
* make a new Packrat snapshot after removing/changing/adding an R package

To invalidate the cache, install [heroku-repo](https://github.com/heroku/heroku-repo) and run:

```bash
heroku repo:reset --app apache-shiny-demo
```

The next deploy will recompile all of your dependencies.

## Required files

### httpd.conf

This is the Apache configuration file. Note the `BalancerMember` directives. These route requests to 5 concurrent R processes over both HTTP and Websocket protocols. The number of upstream servers should not exceed the number of cores your dyno has and might need to be lower in case memory usage is too high.

### run_workers.sh

This is the command that launches the specified number of R processes to serve the Shiny app. The number of workers and ports should match the `BalancerMember` directives in `httpd.conf`.

### Procfile

This is required by Heroku and specifies how the app's web process should be run (in this case, `run_workers.sh`). Other processes, like for a background worker, [can be added here](https://devcenter.heroku.com/articles/procfile).

### run.R

`run_workers.sh` calls this R script with a specified port as the only argument. It starts your Shiny app on that port via `shiny::runApp()`. Note that we are not using [Shiny Server](https://www.rstudio.com/products/shiny/shiny-server/).

### .r-version

Set the R version to install in this file. 3.4.3 has been tested extensively.

#### .Rprofile

Packrat adds this file automatically, so make sure it's also checked in to Git.

## Authentication

For mod_cas authentication, uncomment the `auth_cas_module` section in `httpd.conf`. You must also set the `CAS_HOSTNAME` variable to your registered domain (e.g. `example.harvard.edu`) and add the domain in your Heroku's app settings. Heroku can automatically manage the SSL certificate for your domain in most cases.

## Tips

### Prevent a page timeout by modifying a hidden page element every 10 seconds:

##### ui.R
```r
textOutput("keep_alive")
```
	
##### server.R
```r
output$keep_alive <- renderText({
req(input$alive_count)
input$alive_count
})
```

##### Javascript
```javascript
var socket_timeout_interval;
var n = 0;
	
$(document).on('shiny:connected', function(event) {
socket_timeout_interval = setInterval(function() {
Shiny.onInputChange('alive_count', n++)
}, 10000);
});

$(document).on('shiny:disconnected', function(event) {
clearInterval(socket_timeout_interval);
});
```
	
##### CSS
```css
#keep_alive {
visibility: hidden;
}
```

### Improve the session disconnect experience

Since making a new deployment restarts the dyno, it will interrupt the browsing sessions of current users and they will need to refresh the page. Avoid doing this while many users are connected. However, if you do deploy or the dyno cycles or restarts for some other reason (you can be assured the dyno will cycle/restart once per day), you can prompt the user to refresh the page by listening for the Shiny disconnect event:

	```javascript
	$(document).on('shiny:disconnected', function(event) {
	  // display a message, open a modal window, etc.
	});
	```

### Character encoding

You might need to include `Sys.setlocale("LC_ALL", "en_US.utf8")` in your R code if you notice problems with character encodings on the site.

### Reduce slug size
* Use a `.slugignore` file to exclude unnecessary files from Heroku's build process (see [this article](https://devcenter.heroku.com/articles/slug-compiler#ignoring-files-with-slugignore))
* Run `packrat::clean()` to find and remove unused packages in your code, which will make your slug size smaller.

### Keep secrets in config vars

Load passwords, secrets, and other environment-specific variables using [config vars](https://devcenter.heroku.com/articles/config-vars).

For instance, set a variable `DEPLOY_ENV` and load it in R via `deploy_env <- Sys.getenv("DEPLOY_ENV")` to make your R code aware of whether it's running in a staging, test, or production environment.
