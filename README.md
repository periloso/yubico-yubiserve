# Author & Version 
YubiServe has been written by Alessio Periloso <mail *at* periloso.it>
Version 1.0: 21/05/2010
Version 2.0: 19/11/2010
Version 2.9: 13/12/2010
Version 3.0: 14/12/2010
Version 3.1: 24/03/2011
+ Fixed issue #3, #4, #5, #6

# Description
This simple service allows to authenticate Yubikeys and OATH Tokens using
only a small sqlite database (the mysql support is optional!)
The code has been released under GNU license (license into LICENSE file)

The project is divided into two parts:
 - The database management tool (dbconf.py)
 - The validation server (yubiserve.py)


# Installation
Installation is pretty simple, you just have to install few python packages:

Under Debian, you can run:
`apt-get install python python-crypto python-openssl`

Under Mageia, you will need to use:
  `urpmi python-pycrypto python-OpenSSL`

If you want to add the sqlite support, you should run:
  `apt-get install python-sqlite`
  sqlite3 is also supported, which is included with python in RHEL6.

Or, if you want to add the mysql support, you should run:
  `apt-get install python-mysqldb`
If you chosen the mysql support, you must create a database and create the
tables. The mysql dump is at src/dump.mysql.


Then, you have to generate the certificate for ssl validation, so if you don't
already have a certificate you have to issue the following command to self-sign
one:
`openssl req -new -x509 -keyout yubiserve.pem -out yubiserve.pem -days 365 -nodes`

A good idea would be taking a look at yubiserve.cfg, to configure the validation server settings.

After installing the needed packages, you just need to extract the files
to a directory, add the keys and launch the server (or, if you prefer
you can launch the server before adding the keys, it doesn't matter).


## The database management tool
The database management tool helps you to manage keys in the database.
For detailed help, run the database management tool with ./dbconf.py

The tool allows you to add, delete, disable and enable keys/tokens.
You can also add and remove API keys, to check the server signature in
server responses.
Everything is managed through nicknames, to make keys easy to remember
who belong to.

For example, to add a new yubikey, write:
`./dbconf.py -ya alessio vvkdtkjureru 980a8608b307 f1dc9c6585d600d06f9aae1abea2969e`

In this example, 'alessio' is the key nickname, 'vvkdtkjureru' is the
key public identity (the one you can see at the beginning of your OTPs),
'980a8608b307' is the private identity of the OTP (you can read it when
you program your key), and the last parameter is the AES Key.


To add a new OATH/HOTP:
`./dbconf.py -ha alessio 4rvn24642402 f03ddacdfebb6396f60d7045f41de68f5c5e1c3f`

In this other example, 'alessio' is still the nickname, '4rvn24642402' is
the public identity of the token (it could be also 1, 2, 'alessio' or
whatever you want; the Yubico implementation is 12 characters long)


To add a new API key:
`./dbconf.py -aa alessio`

When you add a new API key, the configuration tool will return both
the api key (ex. 'UkxFMnNFNTV4clRYUExSOWlONzQ=') and the API key id
meant to be used later in your queries to the Yubiserve validation server.


##The Yubiserve Validation Server
Understanding how to use the Yubiserve web application is pretty simple.
You just have to run it (./yubiserve.py) and send your queries through
HTTP GET connections.

The default listening port is 8000, the default listening ip is 0.0.0.0
(so you can connect to it from other machines). If you need it to answer
only from local machine, you can change the ip to 127.0.0.1.
The ssl port is by default the next one, so if the http validation server
answers on port 8000, the ssl will answer on port 8001.
Anyway, everything is easily customizable modifying the yubiserve.py file
and changing the variables "yubiservePORT" for the HTTP port, "yubiserveSSLPORT"
for the SSL port, "yubiserveHOST" for the listening ip.
When you connect to the server (ex. http://192.168.0.1:8000/), it will
answer with a simple page, asking you Yubico Yubikeys OTPs or OATH/HOTP
tokens.

The Yubico Yubikey needs only one parameter: the OTP.
The OATH/HOTP tokens needs two parameters: the OTP itself (6 or 8 digits)
and the Token Identifier. The token identifier can be any character string
you prefer, or, according to the standard OATH implementation, the preceding
string to the OTP. The Yubico implementation follows this standard.
The Yubiserve Validation Server, according to the standard, will try to
find the Token Identifier preceding the OTP. If the string is found, the
OTP will be verified according to that string; in case of LCD tokens,
the string is not automatically added, so you will need to insert your ID
in the second box to allow the Validation Server to find your own identity.


## Querying the Yubiserve Validation Server
Querying the Yubiserve Validation Server is pretty simple.
For Yubico Yubikeys, you will need to send a HTTP GET connection to:
http://<server ip>:<server port>/wsapi/2.0/verify?otp=<your otp>
ex.: http://192.168.0.1:8000/wsapi/2.0/verify?otp=vvnjbbkvjbcnhiretjvjfebbrdgrjjchdhtbderrdbhj
This way you will try to authenticate to it, the simplest way possible.
The response will be something like:

```otp=vvnjbbkvjbcnhiretjvjfebbrdgrjjchdhtbderrdbhj
status=OK
t=2010-11-20T23:54:35
h=```

As you can see, the 'h' parameter is not set, and this is because we didn't use
the signature through API Key. To use it, just add the 'id=<api key id>'
parameter we had when we added the API Key.
ex.: http://192.168.0.1:8000/wsapi/2.0/verify?otp=vvnjbbkvjbcnhiretjvjfebbrdgrjjchdhtbderrdbhj&id=1
This time the response will be like:

```otp=vvnjbbkvjbcnhiretjvjfebbrdgrjjchdhtbderrdbhj
status=OK
t=2010-11-21T00:00:03
h=6lrhQPKo1I/RQA1KPnjpuiOvVMc=```

To check the server signature, check the source code (you will have to do the
exact same procedure to generate it and then just check if they are equal), or
rely on the Yubico documentation on Validation Servers.

For OATH/HOTP keys, the query can be simplified or not.
If your token supports the 'Token Identifier', like Yubico Yubikeys, you can just
send one parameter, the generated string, and the Yubiserve Validation Server will
take care of looking for your key informations in the database.
If your token instead only generates the 6-8 digits, you will have to explicit
your publicID through another parameter.
So, you will have to query, via HTTP GET, the following address:
http://<server ip>:<server port>/wsapi/2.0/oathverify?otp=<your otp>&publicid=<token id>
ex.: http://192.168.0.1:8000/wsapi/2.0/oathverify?otp=80l944311056173483
ex.: ex.: http://192.168.0.1:8000/wsapi/2.0/oathverify?otp=173483&publicid=80l944311056
Both the examples works the same way: in the first case, the Token Identifier was
inside the generated OTP (like in Yubico Yubikey implementation), in the second case
an authentication through a LCD Token was made, so the Yubiserve needed to know who
the token belonged to, and the publicid parameter was added.
The response, like Yubico Yubikey queries, is the following:

```otp=80l944311056173483
status=OK
t=2010-11-21T00:04:59
h=```

The 'h' parameter is not set, because we didn't specified the API Key id. To use the
server signature, we will need to add the 'id' parameter, like in the following query:
ex.: http://192.168.1.2:8000/wsapi/2.0/oathverify?otp=80l944311056173483&id=1
ex.: http://192.168.0.1:8000/wsapi/2.0/oathverify?otp=173483&publicid=80l944311056&id=1

And this would be the the response:

```otp=80l944311056173483
status=OK
t=2010-11-21T00:10:56
h=vYoG9Av8uG6OqVkmMFuANi4fyWw=```

## Basic monitoring of the Yubiserve Validation Server

The yubiserve tool allows you to set up basic monitoring by making an http request using
the URI /healthcheck.

This check will confirm if the yubserve tool is capable of connecting to it's database and
capable of performing queries. It also counts the number of enabled tokens/keys, and will produce
a warning if no keys are enabled.

Options are:
/healthcheck?service=all
/healthcheck?service=oathtoken
/healthcheck?service=yubikeys

Example: 
`curl -s http://localhost:8000/healthcheck?service=yubikeys`

Output format is:
OK: (no message)
WARN: multi-line warning message
CRITICAL: multi-line warning message

HTTP result codes:
200 - OK / WARN
404 - Something bad, the health check did not run
500 - The healthcheck got an un-handled exception during operation. An error will be printed.
503 - The healthcheck found no enabled keys.

The recommended way to perform this is to use Nagios or other monitoring tool to poll the yubiserve health check
and check the returned string starts with "OK:"


                                == Final thoughts ==
That's all. Pretty simple, huh?
Of course you can add new keys while the server is already running, without needing it
to restart, and of course multiple queries a time are allowed, that's why the server
is multithreaded.
