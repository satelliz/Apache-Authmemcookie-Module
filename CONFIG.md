# What is "Auth MemCookie"?

"Auth MemCookie" is an Apache v2 Authentication and authorization modules are based on "cookie" Authentication mechanism.

The module doesn’t make Authentication by it self, but verify if Authentication "the cookie" is valid for each url protected by the module. The module validate also if the "authenticated user" have authorization to access url.

Authentication is made externally by an Authentication html form page and all Authentication information necessary to the module a stored in memcached identified by the cookie value "Authentication session id" by this login page.

# How it Works

## Phase 1 : The login Form

Authentication is made by a login form page.

This login page must authenticate the user with any authenticate source (ldap, /etc/password, file, database....) accessible to language of the page (php, perl, java... an ldap login page sample in php are in samples directory).

Then this page must set cookie that contains only a key the "Authentication unique id" of the "Authentication session".

The login page must store authorization and user information of the authenticated user in memcached identified by the cookie key "Authentication unique id".

The login page can be developed in any language you want, but must be capable to use memcached (they must have memcache client api for us)

## Phase 2 : The Apache v2 Module

After the user is logged, the apache 2 module check on each protected page by apache ACL the presence of the "cookie".

If the "cookie" exists, try to get session in [memcached](http://memcached.org/) with the "cookie" value if not found return "**HTTP_UNAUTHORIZED**" page. 

If session exists in [memcached](http://memcached.org/) verify if ACL match user session information if not match return "**HTTP_FORBIDDEN**" page. 

# Session format stored in memcached

The session store in [memcached](http://memcached.org/) are composed with multiple line in form of "**name**" equal "**value**" ended by "**\r\n**". some are mandatory, other are optional and the rest are information only (all this field are transmitted to the script language protect the module).

Session format:

    UserName=<user name>\r\n
    Groups=<group name1>:<group name2>:...\r\n
    RemoteIP=<remote ip>\r\n
    Password=<password>\r\n
    Expiration=<expiration time>\r\n
    Email=<email>\r\n
    Name=<name>\r\n
    GivenName=<given name>\r\n

- **Username**: are mandatory.
- **Groups**: are mandatory, are used to check group in apache acl. if no group are know for the user, must be blank (Groups=\r\n)
- **RemoteIP**: are mandatory, used by remote ip check function in apache module.
- **Password**: are not mandatory, and is not recommended to store in memcached for security reson, but if stored, is sent to the script language protected by the module.
- The other fields are information only, but they are sent to the langage that are behind the module (via environement variable or http header).

The session field size is for the moment limited to 10 fields by default.

# Build dependency

You must have compiled and installed:

- [libevent](http://libevent.org/) use by [memcached](http://memcached.org/).

- [memcached](http://memcached.org/) the cache daemon it self.

- [libmemcached](http://libmemcached.org/) the C client API needed to compile the Apache Module.

# Compilation

```
# ./configure --with-apxs=/path/to/apache/httpd/bin/apxs --with-libmemcached=/path/to/libmemcached/
# make
# make install
```

After that the "mod_auth_memcookie.so" is generated in apache "modules" directory.

# How to configure Apache Module

## Module configuration option:

This option can be used in "location" or "directory" apache context.

- **Auth_memCookie_Memcached_Configuration**

This configuration directive permit to configure libmemcached initialisation.
The syntax of this directive value are defined her: http://docs.libmemcached.org/libmemcached_configuration.html

With that directive you can specify a liste of ip or host adresse(s) and port ':' separed of memcache(s) daemon to be used.

For exemple: 
```
    Auth_memCookie_Memcached_Configuration "--SERVER=host10.example.com --SERVER=host11.example.com --SERVER=host10.example.com"
```
- **Auth_memCookie_Memcached_SessionObject_ExpireTime**

Session object stored in memcached expiry time, in secondes. 

Used only if "Auth_memCookie_Memcached_SessionObject_ExpiryReset" is set to 'on'.

Set to 3600 seconds by default.

- **Auth_memCookie_Memcached_SessionObject_ExpiryReset**

Set to 'off' to not reset object expiry time in memcache on each url, is set to 'on' by default.

- **Auth_memCookie_SessionTableSize**

Max number of element in session information table, is set to 10 by default.

- **Auth_memCookie_SetSessionHTTPHeader**

Set to 'on' to set session information to http header of the authenticated users, is set to 'off' by default.
Each session field are sended to backend. 
Each session field name are prefixed by Auth_memCookie_SetSessionHTTPHeaderPrefix changed to uppercase.

- **Auth_memCookie_SetSessionHTTPHeaderEncode**

Set to 'on' to mime64 encode session information to http header, is set to 'off' by default.

- **Auth_memCookie_SetSessionHTTPHeaderPrefix** 

Set HTTP header prefix, is set to 'MCAC_' by default.

- **Auth_memCookie_CookieName**

Name of the cookie to used for check authentification, is set to "AuthMemCookie" by default.

- **Auth_memCookie_MatchIP_Mode**

To check cookie ip adresse, Set to '1' to use 'X-Forwarded-For' http header, to '2' to use 'Via' http header, and to '3' to use apache remote_ip, is set to '0' by default to desactivate the ip check.

- **Auth_memCookie_GroupAuthoritative** (only on apache <2.4)

Set to 'off' to allow access control to be passed along to lower modules, for group acl check, is set to 'on' by default.

- **Auth_memCookie_Authoritative**

Set to 'off' to allow access control to be passed along to lower modules, is set to 'on' by default.

- **Auth_memCookie_SilmulateAuthBasic**

Set to 'off' to not fix http header and auth_type for simulating auth basic for scripting language like php auth framework work, is set to 'on' by default.

with this option this $_SERVER variable are normaly set on php: 
  AUTH_TYPE = "basic"
  PHP_AUTH_USER = "user"
  PHP_AUTH_PW = "password"

- **Auth_memCookie_DisableNoStore**

Set to 'on' to stop the sending of a Cache-Control no-store header with the login screen. This allows the browser to cache the credentials, but at the risk of it being possible for the login form to be resubmitted and revealed to the backend server through XSS. Use at own risk.

# On the backend application

The application recieve this information: 

- REMOTE_USER are set to the user logged name
- AUTHMEMCOOKIE_PREFIX are set to Auth_memCookie_SetSessionHTTPHeaderPrefix
- AUTHMEMCOOKIE_AUTH are set to "yes" when protected, "no" when in public zone.

And all session field (prefixed by Auth_memCookie_SetSessionHTTPHeaderPrefix/AUTHMEMCOOKIE_PREFIX) if Auth_memCookie_SetSessionHTTPHeader is on.

And if Auth_memCookie_SilmulateAuthBasic is set, they recieve also this $_SERVER variable : 

- AUTH_TYPE = "basic"
- PHP_AUTH_USER = "user"
- PHP_AUTH_PW = "password"

# Apache 2.3/2.4 [authn/authz model](https://httpd.apache.org/docs/2.4/howto/auth.html)

The module add some ["Require"/"authz"](https://httpd.apache.org/docs/2.4/mod/mod_authz_core.html#require) provider:

- **Require mcac-group**

To limit access to groups specified in session ("groups" session field) by the login script.
Use the same syntax than ["Require group"](https://httpd.apache.org/docs/2.4/mod/mod_authz_groupfile.html#requiredirectives).
But "Require group" on apache 2.3/2.4 work only with [mod_authz_groupfile](https://httpd.apache.org/docs/2.4/mod/mod_authz_groupfile.html).

They also support multiple groups like that:
```
 Require mcac-group group1 group2 group3
```

If one match on group of the "groups" session field they are granted.

- **Require mcac-public**

They make possible to specify public access zone.
In that zone authenticated or not are granted but authenticated can send session information to backend depend on Auth_memCookie_SetSessionHTTPHeader flag.

```
   <Location /publiczone>
      Require mcac-public
   </Location>
```

- **[Require valide-user](https://httpd.apache.org/docs/2.4/mod/mod_authz_user.html#requiredirectives)** and **[Require user](https://httpd.apache.org/docs/2.4/mod/mod_authz_user.html#requiredirectives)**

All the two a provided by [mod_authz_user](https://httpd.apache.org/docs/2.4/mod/mod_authz_user.html) core apache module.

## Sample to configure Apache v2.4 Module:

Configuration sample for using Auth_memcookie apache V2.4 module:

    LoadModule mod_auth_memcookie_module modules/mod_auth_memcookie.so

    <IfModule mod_auth_memcookie.c>
     <Location />
     Auth_memCookie_CookieName myauthcookie
     Auth_memCookie_Memcached_Configuration --SERVER=127.0.0.1:11000

     # to redirect unauthorized user to the login page
     ErrorDocument 401 "/gestionuser/login.php"

     # to specify if the module are autoritative in this directory
     Auth_memCookie_Authoritative on
     # must be set without that the refuse authentification
     AuthType Cookie
     # must be set (apache mandatory) but not used by the module
     AuthName "My Login"
     require mcac-public
     </Location>

    </IfModule>

    # to protect juste user authentification
    <Location "/myprotectedurl">
     require valid-user
    </Location>

    # to protect acces to user in group1
    <Location "/myprotectedurlgroup1">
     require mcac-group group1
    </Location>

# Apache 2.0/2.2 [authn/authz model](http://httpd.apache.org/docs/2.2/howto/auth.html)

- **Require group groupname [groupname]...**

Only users with the specified groups can access the resource.

- **Require valid-user**

Any valid user can access the resource.

- **Require user user_id [user_id]...**

Only specified users can access the resource.

all this directive are from [mod_auth_basic](http://httpd.apache.org/docs/2.2/mod/core.html#require) module.

## Sample to configure Apache v2.0 Module:

Configuration sample for using Auth_memcookie apache V2.0 module:

    LoadModule mod_auth_memcookie_module modules/mod_auth_memcookie.so

    <IfModule mod_auth_memcookie.c>
     <Location />
     Auth_memCookie_CookieName myauthcookie
     Auth_memCookie_Memcached_Configuration --SERVER=127.0.0.1:11000

     # to redirect unauthorized user to the login page
     ErrorDocument 401 "/gestionuser/login.php"

     # to specify if the module are autoritative in this directory
     Auth_memCookie_Authoritative on
     # must be set without that the refuse authentification
     AuthType Cookie
     # must be set (apache mandatory) but not used by the module
     AuthName "My Login"
     </Location>

    </IfModule>

    # to protect juste user authentification
    <Location "/myprotectedurl">
     require valid-user
    </Location>

    # to protect acces to user in group1
    <Location "/myprotectedurlgroup1">
     require group group1
    </Location>
