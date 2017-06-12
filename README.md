# Warcraft III connector specification - v1.0.0-DRAFT

## What is this?
This documents provides technical specification for a unified Warcraft III connector with the following goals:
- allows hostbot communities to advertise their game lists independently of battle.net servers
- allows users to login with their favorite community account
- games are joinable via LAN interface
- there is no central server for logon and no central game lists, connector works with a set of community servers with well defined interfaces

## Components

1. Desktop client
- provides UI for user login
- retrieves and locally broadcasts the game list
2. Authentication servers
- authenticates a user for a specific community
3. Game list servers
- provides list of available games for each community

## Flow overview
![basicflow](http://files.eurobattle.net/images/misc/basic_flow.png)

## Authentication server specification
W3-hoster community provides a basic implementation of authentication server. The logic for verification of username and password is implemented by each community depending on their backend infrastructure (usually a forum). Auth server provides a REST interface with a single endpoint:
```
POST http://auth.community.com/v1/authenticate
``` 
Body:
```
{
    "username":"user",
    "password:"pass"
}
```

Response is a signed JWT token (https://jwt.io/introduction/):
..something like eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE0MjU1ODg4MjEsImp0aSI6IjU0ZjhjMjU1NWQyMjMiLCJpc3MiOiJzcC1qd3Qtc2ltcGxlLXRlY25vbTFrMy5jOS5pbyIsIm5iZiI6MTQyNTU4ODgyMSwiZXhwIjoxNDI1NTkyNDIxLCJkYXRhIjp7InVzZXJJZCI6IjEiLCJ1c2VyTmFtZSI6ImFkbWluIn19.HVYBe9xvPD8qt0wh7rXI8bmRJsQavJ8Qs29yfVbY-A0

Token is constructed from 3 parts separated by dot, the parts are
### JWT header
```
{
  "alg": "RS256",
  "typ": "JWT"
}
```
### Payload
```
{
    "iat":"Timestamp when token was issued",
    "nbf":"Timestamp of when the token should start being considered valid.",
    "exp":"Timestamp of when the token should cease to be valid.",
    ...
    custom fields of whatever we want, such as username, email, realm (community name) etc.
}
```
### Signature
Signature in base64, basically payload signed with private RSA key.

W3-hoster community implements the REST endpoint and JWT generation logic, the function that validates username and password against backend remains to be implemented by each community.

Suggested language: PHP

## Game list server specification
Game list server maintains a list of active games, accepts new game creations and sends game lists to clients.

We should take a look at bnet packet data structures to figure out the needed information about games.
- game creation: C>S 0x1C SID_STARTADVEX3 https://bnetdocs.org/packet/265/sid-startadvex3
- game creation response: S>C 0x1C SID_STARTADVEX3 https://bnetdocs.org/packet/178/sid-startadvex3
- get game list: C>S 0x09 SID_GETADVLISTEX https://bnetdocs.org/packet/386/sid-getadvlistex
- get game list response: S>C 0x09 SID_GETADVLISTEX https://bnetdocs.org/packet/266/sid-getadvlistex

Remarks:
- Game creation should probably be protected with a random password which can be added to hostbot and game server configs so random people can't advertise the games.
- When sending game list request, client should include the JWT token which game list server then verifies.
- Game list server stores public key of each community in config in order to be able to verify the JWT token signature.
- After initial game list request, TCP connection can remain open and server pushes game updates to the client.
- Client can connect to several game list servers at once and combine all of them locally in a single LAN broadcast.
- Perhaps we need another packet so client can gracefully terminate sending of fresh game list updates or we can just close TCP connection.

Suggested language: modern C++.  
One negative side discovered so far is the lack of C++ JWT libraries (found 1-2 that look kinda ok).

## Desktop client specification
Desktop client is a GUI application that performs the authentication and does the local LAN broadcast of games.
The core functionality (without UI) should probably be written as a library so it can be embedded in existing community clients or even gproxy implementations.

Authentication servers and game list servers are configurable. Client authenticates against a single authentication server
and can request games from multiple sources.

Suggested language: Qt.

### LAN broadcast
 TODO: the packets needed to broadcast LAN games and the technical aspects of it

## Other concerns

### Name spoofing
How to prevent it?

### Account bans
Bans are on hostbot level for username+realm.

### How does gproxy fit in all this?
Never used gproxy via LAN before, I believe the order would be client->gproxy forward->w3. How do you hook w3 into gproxy if w3 already listens on local interface?
Clarifications needed.

Gproxy could also act as a client in itself, which is why core client functionality should be made into a lib of some sorts.

### Getting W3 patches to end user
WVS?

### Updating the client
Either by releasing a new version and request players to download it or by implementing an updater.