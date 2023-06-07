# Functional Partitioning and API Gateway

The service you implemented in [Project 1][1] is a good start, but there are several limitations:

+ There is only a single instance of the service

+ The service is a monolith that handles two different sets of functionality

+ Most endpoints are not authenticated

In this project you will make some design changes to the code to allow for future scaling, then implement authentication and load balancing through an API gateway.

### Learning Goals

The following are the learning goals for this project:

1. Gaining further experience with implementing back-end APIs in Python and Quart.

2. Working with existing code produced by another team.

3. Implementing functional partitioning of services by splitting a monolithic web service.

4. Running and debugging multiple instances of a web service process.

5. Configuring and testing an HTTP reverse proxy and load balancer.

### Libraries, Tools, and Code

The requirements for this project are the same as for [Project 1][1], with the addition of the [Nginx][2] web server.

In order to complete this project, you will need to have access to a working version of the service from the previous project. Members of the team are welcome to re-use the service they developed with a different team in the last project. If no member of your team was on a team that produced a complete, working version of a service from a previous project, you are encouraged to collaborate and share code across teams.

Note, however, that **all functionality specified below** must be your own original work or the original work of other members of your current team, with the usual exception of sources specified in previous projects such as code provided by the instructor and documentation for the various libraries.

### Splitting the monolith

In Project 1, you implemented a single monolithic service that exposed two different sets of resources, one for users and one for games.

**Creating microservices**

Split the code from Project 1 into two separate microservices:

1. A Users service that handles account creating and authentication

2. A Games service that handles game creation, guessing, and game state

**Removing coupling**

For each microservice, create a separate SQLite database and update the schema and Python code to use only that database to provide the service. The two services should be capable of running independently. In particular, services:

+ Should not rely on other another service to be available

+ May not access the database belonging to another service

+ Should not rely on internal details (e.g. database keys) of other services.

Note that this will likely mean duplicating information (e.g. usernames) across databases. While the schema in a single relational database should usually be [normalized][3], denormalization is a common technique for [performance][4] and [scaling][5].

While users have a natural unique identifier (i.e. usernames), games do not. Use this [uuid][6] module to generate a unique ID for each game.


### Authenticating Endpoints

One particular consequence of separating services requires special handling: each service has one or more endpoints that requires authentication. In Project 1, we largely ignored this issue, choosing to implement HTTP Basic for a single endpoint and leaving the rest unauthenticated. This was an intentional design choice to allow for easier decoupling in this project.

Now that the Games service is independent from the Users service, we are left with some unattractive alternatives if we want game endpoints to be authenticated:

+ Introduce dependencies between the Games service and the Users service

+ Duplicate data and introduce consistency issues by storing usernames and passwords separately in the database for each service.

Instead, we will move the responsibility for authenticating game endpoints to a reverse proxy using the [Nginx][2] web server that you configured in [Exercise 2].

To install an additional module to allow Nginx to use an external authentication service, use the following commands on Tuffix 2020:

```
sudo apt update
sudo apt install --yes nginx-extras
```

**Authenticating based on subrequests**

Configure the tuffix-vm virtual host you configured in Exercise 2 to use the Nginx [auth_request][7] module to authenticate all incoming requests. This module allows authentication to be done by making a "subrequest" to an external service. In this case, the external service will be the authenticated endpoint of the Users microservice.

For a similar configuration, see [Protecting web sites with NGINX subrequest authentication][8] by Andy Gock, but note the following differences:

+ The authenticated endpoint in the Users service returns HTTP 200

+ Authentication is handled by the Users service rather than by a separate auth-server application.

+ Your Users service is written in Python with Quart rather than Node.js with Express

+ Authentication is done via HTTP Basic rather than JWT

+ You do not need to pass Set-Cookie: headers

+ There is no "logout" endpoint

When configuration is done, restart Nginx and start the Users microservice. Attempts to access http://wordle-vm/ from a web browser should now pop up an HTTP Basic Authentication dialog box, and you should not be able to see the "Hello, Nginx!" page until you have supplied a valid username and password for the Users microservice. On each request to Nginx, you should be able to see a corresponding request to the Users microservice.

### Acting as an API gateway

Now that Nginx is able to delegate authentication to the Users service, we can configure it as an [API gateway][9], performing three functions:

1. Directing traffic to the correct microservice

2. Authenticating requests to the Games service

3. Load-balancing requests across multiple instances of the Games service

To begin, modify your Procfile and use Goreman to start both microservices.

**Directing traffic**

Use the Nginx [proxy_pass][10] directive to pass user registration requests unauthenticated to the Users service. Requests to the authentication endpoint should be made only by Nginx, and do not need to be made available to external users.

All requests for game endpoints should be authenticated using auth_request prior to being forwarded to the Games service.

**Testing**

After restarting Nginx, confirm that:

1. You can register new users without authentication

2. That each request for a game endpoint requires authentication

3. That each request for a game endpoint results in two HTTP requests logged in Goreman: a request to the Users service, followed by a request to the Games service

**Load Balancing**

Restart Goreman with the `--formation` switch, starting a single instance of the Users service and three instances of the Games service.Test that each service instance can be accessed separately.

*Note:* the original article Introducing Goreman shows more than one process being run using the command-line switch `-c` (for "concurrency"), but current versions use `-m` or `--formation` instead.

Configure Nginx to [load balance][11] between the three instances of the Games service.

**Testing**

Load balancing is correctly configured consecutive calls to the Games service through Nginx are routed to separate instances.

You can check that load-balancing is working correctly by examining the Goreman logs. Make several requests to Nginx for a Games service endpoint, and verify that they are routed to different service instances (e.g. games.1, games.2, and games.3).



---

[1]: ./backend-apis.md
[2]: https://nginx.org/en/docs/
[3]: https://en.wikipedia.org/wiki/Database_normalization
[4]: https://en.wikipedia.org/wiki/Denormalization
[5]: http://www.globule.org/publi/SODDSWA_www2008.html
[6]: https://github.com/google/uuid
[7]: https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-subrequest-authentication/
[8]: https://gock.net/blog/2020/nginx-subrequest-authentication-server/
[9]: https://traefik.io/glossary/reverse-proxy/
[10]: http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass
[11]: https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-load-balancing
