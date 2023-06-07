# Backend APIs

[Wordle][1] is a popular online word game built by a single developer as a side project and recently sold to the New York Times for an undisclosed sum. If you have not played Wordle, first take a break and try it, then come back and finish reading this document.

In this project you will work with a team to build a back-end API for a game similar to Wordle.

The original game is entirely a [client-side application][2], with both current and future answers stored in the JavaScript code, but the New York Times has begun to add server-side features, including syncing progress across devices.

Additional features that might be added with access to a back-end include:

+ Making it [harder to cheat][3]

+ Offering [more than one game per day][4]

+ Offering [different games to different users][5]

### Learning Goals

The following are the learning goals for this project:

1. Determining the back-end functionality required for a web application based on its front-end interface.

2. Designing a back-end API for a web application given a description of its functionality.

3. Implementing back-end APIs in Golang without any framework.

4. Designing and implementing relational database schemas for back-end APIs.

5. [Extracting, transforming, and loading][6] database tables from data sources in other formats.


### Libraries, Tools, and Code

This project must be implemented in Golang along with ancillary tools like [Goreman][7] and the [sqlite3 command line tool][8]. Your queries should use raw SQL; you should not use any ORM like [GORM][9].

**Note:** While your browser may be able to display JSON objects returned in response to an HTTP GET request, you do not need to implement a front-end or other user interface for testing API methods. Use an HTTP client program such as [HTTPie][10] or [curl][11], or the automatic documentation created by Quart-Schema.


### API

Create a RESTful service that exposes the following resources and operations:

**Users**

Each user should have a username and a password. This is only a game, and passwords are only used to keep track of the current game and store a user’s statistics, so they may be stored in cleartext.

The API should allow:

+ Registering a new user

+ Checking a user’s password

**Authentication**

The password-checking endpoint is the only authenticated endpoint, and it should use [HTTP Basic authentication][12] by checking the request.auth object’s type, username, and password properties. If the supplied username and password are valid, return HTTP 200 and the following JSON object:

  ```
  { "authenticated": true }
  ```

If the username and password are not valid, return HTTP `401` and a `WWW-Authenticate` response header. See the `https://httpbin.org/basic-auth/{user}/{passwd}` endpoint for a working example..

*Note:* The httpbin.org endpoint specifies the required username and password as path variables in the URL for the resource. Your service should require the username and password of a registered user instead.

No other endpoints require authentication. If an endpoint requires the name of the current user, you will need to explicitly pass it in the request (see Statelessness below).

**Games**

As described above, we want to offer users the ability to play more than one game per day, and for different users to have different words in each game. Therefore a game must keep track of the user, the secret word for the game, and the number of guesses the user has made.

The API should allow:

+ Starting a new game for a user with a randomly-chosen word

+ Guessing a five-letter word

+ Listing the games in progress for a user

+ Retrieving the state of a game in progress

The secret word should not be sent to the client, only an identifier for the game. A user may start more than one game without finishing.

Games are finished either when the user has guessed the secret word, or when the user has made 6 incorrect guesses.

An incorrect guess should return the following:

+ Whether the guess was a valid word.

+ If the guess was valid, whether the guess was correct (i.e. whether the guess was the secret word).

+ The number of guesses remaining. (Only valid guesses should decrement this number.)

+ Valid but incorrect guesses should also return:

  + The letters that are in the secret word and in the correct spot

  + The letters that are in the secret word but in the wrong spot


When supplied with an identifier for a game that is in progress, the user should receive the same values listed above for each previous valid but incorrect guess (though the number of guesses should not be incremented). If the identifier corresponds to a game that is finished, return only the number of guesses and whether the game was won or lost.

### Service Implementation

Use the Go standard library to define endpoints and representations for the resources above. Each endpoint should make appropriate use of HTTP methods, status codes, and headers.

Your APIs should follow the [principles of RESTful design][13] as described in class and in the assigned reading, with the exception that all input and output representations should be in JSON format with the Content-Type: header field set to application/json.

**Statelessness**

Your service should be stateless. In particular, each request to a game resource will need to identify the user; and that guesses and requests for game state will also require a game identifier.

**Databases**

Informally, your database schema should be in approximately third normal form. If you are not familiar with database normalization, see Thomas H. Grayson’s lecture note [Review: Database Design Rules of Thumb][14] from [Course 11.521 at MIT.][15]

**Database Population**

Populate the possible answers in the database directly from the original Wordle JavaScript code. You can download a copy of the JavaScript code and save only the correct answers and valid guesses with the following commands:

  ```
  wget https://www.nytimes.com/games-assets/v2/wordle.9137f06eca6ff33e8d5a306bda84e37b69a8f227.js
  ```

  ```
  sed -e 's/^.*,ft=//' -e 's/,bt=.*$//' -e 1q wordle.9137f06eca6ff33e8d5a306bda84e37b69a8f227.js > correct.json
  ```

  ```
  sed -e 's/^.*,bt=//' -e 's/;.*$//' -e 1q wordle.9137f06eca6ff33e8d5a306bda84e37b69a8f227.js > valid.json
  ```

You can load the resulting files into Go using the [json module][16] from the standard library, then connect to the database using [CGo-free port of sqlite3.][17]


---


[1]: https://www.nytimes.com/games/wordle/index.html
[2]: https://www.theverge.com/2022/2/1/22912711/wordle-web-save-download-webpage-local-personal
[3]: https://www.pcworld.com/article/606109/how-to-cheat-at-wordle.html
[4]: https://lifehacker.com/finally-there-s-a-way-to-play-wordle-more-than-once-a-1848426643
[5]: https://www.cnet.com/culture/internet/why-wordle-had-two-different-answers-in-one-day/
[6]: https://en.wikipedia.org/wiki/Extract,_transform,_load
[7]: https://github.com/mattn/goreman
[8]: https://zetcode.com/db/sqlite/tool/
[9]: https://gorm.io
[10]: https://httpie.io
[11]: https://curl.se
[12]: https://en.wikipedia.org/wiki/Basic_access_authentication#Protocol
[13]: https://www.restapitutorial.com/lessons/restquicktips.html
[14]: http://web.mit.edu/11.521/www/lectures/lecture10/lec_data_design.html#Database_Design_Rules_of_Thumb
[15]: http://web.mit.edu/11.521/www/index.html
[16]: https://pkg.go.dev/encoding/json
[17]: https://pkg.go.dev/modernc.org/sqlite#section-readme
