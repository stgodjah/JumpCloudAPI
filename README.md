
**Our API documentation has been moved to [docs.jumpcloud.com](https://docs.jumpcloud.com) please refer to these docs as this file will not be maintained or updated.**

JumpCloud API
============

* [Introduction](#introduction)
* [Authentication](#authentication)
* [Parameters](#parameters)
    * [the `filter` parameter](#the-filter-parameter)
* [Data structures](#data-structures)
* [Triggers](#triggers)
* [Systems](#systems)
* [Tags](#tags)
* [System users](#system-users)
* [Command Scheduler](#command-scheduler)
* [Command Results](#command-results)
* [Examples](#examples)



## Introduction

The JumpCloud API is a REST API for retrieving and manipulating the systems, system users, and tags managed by JumpCloud.
To use the JumpCloud API, you must first [create a JumpCloud account](https://console.jumpcloud.com/register/).


## Authentication

Log into the JumpCloud console. Go to the settings dropdown (top-right) and retrieve the API key. This API key is associated to the currently logged in administrator. Other admins will have different API keys.

**NOTE: Please keep this API key secret, as it grants full access to any data accessible via your JumpCloud console account.**

The API key will be passed in as a header with the header name "x-api-key".

For example,

```
curl -H "x-api-key: [YOUR_API_KEY_HERE]" "https://console.jumpcloud.com/api/tags"
```

### Recycling API Key

In order to revoke access with the current API key, simply regenerate a new key. This will render all calls using the previous API key inaccessible.

## Parameters

Most parameters can be passed to all GET, PUT, and POST methods and controls what data is returned from the API. Parameters can be passed as url query parameters or in the body of the POST or PUT method. When using PUT or POST parameters the format of the request body needs to match the Content-Type specified. The API supports both `application/json` and `application/x-www-form-urlencoded` parameter content types when using the POST or PUT method.


|Parameter|Description|Usage|
|---------|-----------------|-----|
|`limit` `skip`| `limit` will limit the returned results and `skip` will skip results.  | ` /api/systems?limit=5&skip=1` returns records 2 - 6 . |
|`sort`         | `sort` will sort results by the specified field name.                      | `/api/systems?sort=displayName&limit=5` returns tags sorted by displayName in ascending order. `/api/systems?sort=-displayName&limit=5` returns systems sorted by displayName in descending order. |
|`fields`       | `fields` is a space-separated string of field names to include or exclude from the result(s). | `/api/system/:id?fields=-hostname -displayName` will return system records *excluding* the `hostname` and `displayName` fields. `/api/system/:id?fields=hostname displayName` will return system records *including only* the `hostname`, `displayName`, and `_id`. **NOTE: the `_id` field will always be returned.**  |


### The `filter` parameter

To support advanced filtering of there is a **`filter` parameter** that can only be passed in the body of `POST /api/search/*` routes. The `filter` parameter must be passed as Content-Type application/json supports advanced filtering using the [mongodb JSON query syntax](http://docs.mongodb.org/manual/reference/operator/query/). The `filter` parameter is an object with a single property, either `and` or `or` with the value of the property being an array of query expressions. This allows you to filter records using the logic of matching *ALL* or *ANY* records in the array of query expressions. If the `and` or `or` are not included the default behavior is to match *ALL* query expressions.

This endpoint is only valid for `/api/search/systems/` and `/api/search/systemusers/`

#### `filter` parameter examples

Get all systems with a hostname start with "www" or "db".

```
{
"filter" :
    {
        "or" :
            [
                {"hostname" : { "$regex" : "^www" }},
                {"hostname" : {"$regex" : "^db"}}
            ]
    },
"fields" : "os hostname displayName"
}
```

Get only Ubuntu servers with a hostname starting with "www"

```
{
"filter" :
    {
        "and" :
            [
                {"hostname" : { "$regex" : "^www" }},
                {"os" : {"$regex" : "Ubuntu", $options: "i"}}
            ]
    },
"fields" : "os hostname displayName"
}
```

Get system users that have an email from "gmail.com"

```
{
"filter" : [{"email" : { "$regex" : "gmail.com$"}}],
"fields" : "email username sudo"
}
```


## Data structures

### Input data

All PUT and POST methods should use the HTTP Content-Type header with a value of 'application/json'.
PUT methods are used for updating a record and POST methods are used for adding new records.

The following example demonstrates how to update the `displayName` of the system.

```
curl \
  -d '{"displayName" : "updated-system-name-1"}' \
  -X 'PUT' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systems/[YOUR_SYSTEM_ID_HERE]"
```


### Output data

All results will be formatted as [JSON](www.json.org). There are two main kinds of output data single record and multi-record.

#### Single record output:

Example of returning a single system record.

```
{
  "__v": 0,
  "_id": "525ee96f52e144993e000015",
  "agentServer": "lappy386",
  "agentVersion": "0.9.42",
  "arch": "x86_64",
  "connectionKey": "127.0.0.1_51812",
  "displayName": "ubuntu-1204",
  "firstContact": "2013-10-16T19:30:55.611Z",
  "hostname": "ubuntu-1204"
  ...
```


#### Multi record output:

Example of returning multiple system records.

```
{
  "totalCount": 1,
  "results": [
    {
      "_id": "52e96aef8977c9010e0003c0",
      "amazonInstanceID": "",
      "arch": "x86_64",
      "displayName": "01ec40634d50",
      "hostname": "9209cad3cbff",
      "lastContact": "2014-01-29T23:16:31.870Z",
      "os": "Ubuntu",
      "version": "12.04",
      "active": true,
      ...

```

#### System and user binding output:

Example of returning a list of system bindings for specific system user:

```
{
  "systemID1": {
    "displayName": "System 1 Display Name"
  },
  "systemID2": {
    "displayName": "System 2 Display Name"
  },
  ...

```

Example of returning a list of system user bindings for specific system:

```
{
  "systemUserID1": {
    "username": "systemuser1"
  },
  "systemUserID2": {
    "username": "systemuser2"
  },
  ...

```

## Triggers

The Triggers section of the JumpCloud API allows you to use a webhook to launch a command or a workflow.  When JumpCloud receives a webhook containing the trigger name, JumpCloud will automatically execute any command(s) with that trigger name, in parallel, across all the servers defined in the saved commands.

Triggers can be executed solely with a trigger name (the name you provide in the "Trigger Name:" field when you set a Command to "Run on Trigger (webhook)", or you can post a JSON object in the POST body, and all top level string or numeric objects in the JSON object will be sent to the command as environment variables.

For example, posting:

```
{
    "srcip": "192.168.1.32",
    "attack": "Attempted Cross-Site Request Forgery"
}
```

will generate the following environment variables for the started command:

```
srcip="192.168.1.32"
attack="Attempted Cross-Site Request Forgery"
```

Because sub-objects and arrays are not stringified before being passed to the command, the command will see the environment variables set to "[object Object]" in these cases.

More information on the use of triggers is available [here](http://support.jumpcloud.com/knowledgebase/articles/347356-how-to-use-command-triggers).

### Modifiable properties

None: Triggers are designed as webhooks, and as such specify an event, not a change to system state.

### Routes

|Method |Path             |Description|
|-------|-----------------|-----------------------------------------------------------------------------------|
|POST    |`/api/command/trigger/<triggerName>`     | Launch a command with trigger name `triggerName`.|

## Systems

The Systems section of the JumpCloud API allows you to retrieve, delete, and modify systems. The vast majority of system records properties are *read-only* because the JumpCloud agent, running on your systems, is the source of most of the data. There are, however, some properties of a system record that can be modified allowing you to control the configuration of your systems.

### Modifiable properties

|System property                  |Type       |Description|
|---------------------------------|-----------|-----------|
|`tags`                           |*array*    | An array of tag id's, or names, to which the system belongs. Optional field. If not sent, tag membership will remain unchanged. If tags are sent, the system will be updated to match the specified tag list. To remove all tags, pass in an empty array as the tags property. Note: If the tags property is omitted from the initial request, the response will also omit tag information.|
|`displayName`                    |*string*   | A string representing the name to display in the JumpCloud for the system.|
|`allowSshPasswordAuthentication` |*boolean*  |`true` will enable password based authentication and `false` will disable password based authentication for ssh logins.|
|`allowSshRootLogin`              |*boolean*  |`true` will enable root user authentication and `false` will disable root user authentication ofr ssh.|
|`allowMultiFactorAuthentication` |*boolean*  |`true` will enable multi-factor authentication and `false` will disable multi-factor authentication for ssh.|
|`allowPublicKeyAuthentication`   |*boolean*  |`true` will enable public-key authentication and `false` will disable public-key authentication for ssh.|

**Note: Adding of a system is only allowed via the [Kickstart script](http://support.jumpcloud.com/knowledgebase/articles/301849-other-ways-to-install-the-jumpcloud-agent).


### Routes

|Method |Path             |Description|
|-------|-----------------|-----------------------------------------------------------------------------------|
|GET    |`/api/systems`     | Get systems in [multi record format](#multi-record-output)|
|POST   |`/api/search/systems`     | Get systems in [multi record format](#multi-record-output) allowing for the passing of the `filter` parameter. The route WILL NOT allow you to add a new system. |
|GET    |`/api/systems/:id` | Get a system record by `_id` in [single record format](#single-record-output) |
|PUT    |`/api/systems/:id` | Update a system record by its `_id` and return the modified system record in [single record format](#single-record-output). |
|DELETE |`/api/systems/:id` | Delete a system record by its `_id`. **NOTE: This command will cause the system to uninstall the JumpCloud agent from its self which can can take about a minute. If the system is not connected to JumpCloud the system record will simply be removed.** |

## Tags

The Tags section of the JumpCloud API allows you to add, retrieve, delete, and modify Tags. Tags are used to associate system users to systems, and to create groups of systems for easy management in the Commands tab. For example, if you have user "jsmith" and a system both associated to the same Tag, then the system user "jsmith" will be able to login to that system. If either the system or "jsmith" are removed from the Tag, "jsmith" will no longer have access to that system.

For more information about tags see: [How to Use Tags](http://support.jumpcloud.com/knowledgebase/articles/295858-how-to-use-tags)

### Modifiable properties

|Property                  |Type       |Description|Required|
|---------------------------------|-----------|-----------|:--------:|
|`name`                           |*string*   | A unique name for the Tag. | **X** |
|`systems`                        |*array*    | An array of system ids that are associated to the Tag.|
|`systemusers`                    |*array*    | An array of system user ids that are associated to the Tag.|
|`expirationTime`                 |*datetime* | A date timestamp indicating when this Tag will expire itself. When a Tag expires it will revoke any system user to system associations.|

### Routes

|Method |Path             |Description|
|-------|-----------------|-----------|
|GET    |`/api/tags`        | Get tags in [multi record format](#multi-record-output)|
|POST   |`/api/tags`        | Add a new Tag and return the newly created Tag in a [single record format](#single-record-output) |
|GET    |`/api/tags/:name`  | Get a Tag record by `_id` or `name` in [single record format](#single-record-output) |
|PUT    |`/api/tags/:name`  | Update a Tag record by its `_id` or `name` and return the modified Tag record in a [single record format](#single-record-output). |
|DELETE |`/api/tags/:name`  | Delete a Tag record by its `_id` or `name`. |

### Search

Search through tags using query parameters. The example below demonstrates how to search the **name** field of tags using the search term **debian** and limiting the results to **5**

```
curl -g -H "x-api-key: [YOUR_API_KEY_HERE]" "https://console.jumpcloud.com/api/tags?search[fields][]=name&search[searchTerm]=debian&limit=5"
```

## System Users

The System Users section of the JumpCloud API allows you to add, retrieve, delete, and modify System Users. To add a System User to JumpCloud, you must provide an email address and username. The System Users email will be used to contact the new System User, allowing them to be *activated*. During activation, the user will prompted to set their password, an SSH public key, and to set up Google Authenticator for Multi-factor authentication.

Adding a new System User will not grant them access to servers until they're associated to a Tag, and a System User can be assigned to a Tag as soon as they're created. However, they will not gain access to any servers until they have activated.


### Modifiable properties

|System property                  |Type       |Description|Required|
|---------------------------------|-----------|-----------|:--------:|
|`email`                          |*string*   | The email address of the new System User. | **X** |
|`username`                       |*string*   | The username of the new System User.  | **X** |
|`password`                       |*string*   | An administrative override the System Users password. **Do not set the password when creating a new user. JumpCloud will send an activation email.**   |  |
|`allow_public_key`               |*boolean*  | Allow this user to authenticate with public key access. |  |
|`passwordless_sudo`              |*boolean*  | Allow this user to use sudo with no password (Linux-only). |  |
|`sudo`                           |*boolean*  | Allow this user to have sudo access Linux (Administrator on Windows). |  |
|`public_key`                     |*string*   | The ssh public key for this user. |  |
|`unix_uid`                       |*integer*  | The unix user id for this user. **Do not change this unless you really know what you're doing. JumpCloud will auto assign ids above 5000+** |  |
|`unix_guid`                      |*integer*  | The unix group id for this user. **Do not change this unless you really know what you're doing. JumpCloud will auto assign ids above 5000+** |  |
|`tags`                           |*array*    | An array of tag id's, or names, to which the systemuser belongs. Optional field. If not sent, tag membership will remain unchanged. If tags are sent, the systemuser will be updated to match the specified tag list. To remove all tags, pass in an empty array as the tags property. Note: If the tags property is omitted from the initial request, the response will also omit tag information. |  |
|`attributes`                     |*array*    | An array of attributes associated with the systemuser.  Attributes are organized in a name and value pair.  If attributes are sent, the systemuser will be updated to match the values passed; i.e., when attributes are PUT, they will replace any existing attributes, not append to the existing data. |  |

### Routes

|Method |Path                      |Description|
|-------|--------------------------|-----------------------------------------------------------------------------------|
|GET    |`/api/systemusers`        | Get System Users in [multi record format](#multi-record-output) |
|POST   |`/api/search/systemusers` | Get System Users in [multi record format](#multi-record-output) allowing for the passing of the `filter` parameter.|
|POST   |`/api/systemusers`        | Add a new System User and return the newly created System User in a [single record format](#single-record-output) |
|GET    |`/api/systemusers/:id`    | Get a System User record by `_id` in [single record format](#single-record-output) |
|PUT    |`/api/systemusers/:id`    | Update a System User record by its `_id` and return the modified System User record in a [single record format](#single-record-output). |
|DELETE |`/api/systemusers/:id`    | Delete a System User record by its `_id`. |

## System/User Binding
The System/User Binding section of the JumpCloud API allows you to modify one-to-one binding relationships between a system and a system user.

### Modifiable properties
|System property  |Type     |Description                        |Required|
|-----------------|---------|-----------------------------------|:------:|
|`add`            |*array*  | An array of ids that are to be bound |**X**|
|`remove`         |*array*  | An array of ids that are to be unbound |**X**|

### Routes
|Method   |Path             |Description                                 |
|---------|-----------------|--------------------------------------------|
|GET      |`/api/systemusers/:systemUserID/systems`  | [List system bindings for specific system user](#list-system-bindings-for-specific-system-user) in a [system and user binding format](#system-and-user-binding-output). |
|PUT      |`/api/systemusers/:systemUserID/systems`  | [Add (or remove) system to system user](#add-or-remove-system-to-system-user) |
|GET      |`/api/systems/:systemID/systemusers`  | [List system user bindings for specific system](#list-system-user-bindings-for-specific-system) in a [system and user binding format](#system-and-user-binding-output). |
|PUT      |`/api/systems/:systemID/systemusers`  | [Add (or remove) system user to system](#add-or-remove-system-user-to-system) |

## Command Scheduler
The Command Scheduler can be used to run commands across your systems/tags either immediately, at a specific time in the future, or repeatedly. Please note that `systems` OR `tags` is a required field. Setting both is unsupported. You can retrieve the results by using the Command Results API.

### Modifiable Properties
|System property  |Type     |Description                        |Required|
|-----------------|---------|-----------------------------------|:------:|
|`name`           |*string* | The display name for the command. |        |
|`command`        |*string* | The command to execute on the server.|**X**|
|`user`           |*string* | The id of the JC managed user to run the command as.|**X**|
|`systems`        |*array*  | An array of system ids to run the command on.|
|`schedule`       |*string* | A crontab that consists of: `(seconds) (minutes) (hours) (days of month) (months) (weekdays)` or `immediate`. If you send this as an empty string, it will run immediately.|
|`files`          |*array*  | An array of file ids to include with the command.|
|`tags`           |*array*  | An array of tag ids to run the command on.|
|`timeout`        |*string* | The time in seconds to allow the command to run for.|

### Routes
|Method   |Path             |Description                                 |
|---------|-----------------|--------------------------------------------|
|GET      |`/api/commands`  | Get Saved Commands in [multi record format](#multi-record-output).|
|POST     |`/api/commands`  | Add a new Saved Command record and return the newly create Saved Command in a [single record format](#single-record-output).|
|GET      |`/api/commands/:id` | Get a Saved Command record by `_id` in [single record format](#single-record-output). 
|PUT      |`/api/commands/:id`| Update a Saved Command record by it's `_id` and return the modified Saved Command in [single record format](#single-record-output).|
|DELETE   |`/api/commands/:id` | Delete a Saved Command record by its `_id`.|

## Command Results
The Command Results section of the JumpCloud API allows you to retrieve and delete the results of commands that you ran on your system through JumpCloud. None of the properties are modifiable, but you can delete records.

### Properties
|System property  |Type     |Description                        |
|-----------------|---------|-----------------------------------|
|`command`        |*string* | The command that was executed on the system.|
|`system`         |*string* | The id of the system the command was executed on.|
|`organization`   |*string* | The id of the organization.       |
|`user`           |*string* | The user the command ran as.      |
|`files`          |*array*  | An array of file ids that were included in the command.|
|`requestTime`    |*date*   | The time that the command was sent.|
|`responseTime`   |*date*  | The time that the command was completed. |
|`response.id`    |*string*|This is the same id as the parent.  |
|`response.data.output`|*string*|The stdout from the command that ran.|
|`response.data.exitCode`|*integer*|The exit code from the command that ran.|
|`response.error`|*string*|The stderr output from the command that ran.|

### Routes
|Method   |Path             |Description                                 |
|---------|-----------------|--------------------------------------------|
|GET      |`/api/commandresults`  | Get Command Results in [multi record format](#multi-record-output).|
|GET      |`/api/commandresults/:id` | Get a Command Result record by `_id` in [single record format](#single-record-output). 
|DELETE   |`/api/commandresults/:id` | Delete a Saved Result record by its `_id`.|

## Examples

### Launch a Command via a Trigger

```
curl --silent \
     -X 'POST' \
     -H "x-api-key: [YOUR_API_KEY_HERE]" \
     "https://console.jumpcloud.com/api/command/trigger/[TRIGGER_NAME_HERE]"
```

### Launch a Command via a Trigger passing a JSON object to the command

```
curl --silent \
     -X 'POST' \
     -H "x-api-key: [YOUR_API_KEY_HERE]" \
     -H 'Accept: application/json' \
     -d '{ "srcip":"192.168.2.32", "attack":"Cross Site Scripting Attempt" }' \
     "https://console.jumpcloud.com/api/command/trigger/[TRIGGER_NAME_HERE]"
```

### Add a new System User

This examples assumes there is already a Tag named "admins" in your JumpCloud account.

```
curl \
  -d '{"email" : "bob@myco.com", "username" : "bob", "tags" : ["admins"]}' \
  -X 'POST' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systemusers"
```

### Add attributes to a System User

```
curl \
 -X 'PUT' \
 -H 'Content-Type: application/json' \
 -H 'Accept: application/json' \
 -H "x-api-key: [YOUR_API_KEY_HERE]" \
 -d '{ "attributes" : [ { "name" : "myhappyattribute", "value" : "myhappyattributevalue" }] }' \
 "https://console.jumpcloud.com/api/systemusers/:id"
```

### Find a System User by username

```
curl \
  -d '{"filter": [{"username" : "bob"}]}' \
  -X 'POST' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/search/systemusers"
```

### Create a new Tag

```
curl \
  -d '{"name" : "Developers"}' \
  -X 'POST' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/tags"
```

### List system bindings for specific system user

```
curl \
  -H 'Content-Type: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systemusers/[SYSTEM_USER_ID_HERE]/systems"
```

### Add (or remove) system to system user

```
curl \
  -d '{ "add": ["[SYSTEM_ID_TO_ADD_HERE]"], "remove": ["[SYSTEM_ID_TO_REMOVE_HERE]"] }' \
  -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systemusers/[SYSTEM_USER_ID_HERE]/systems"
```

### List system user bindings for specific system

```
curl \
  -H 'Content-Type: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systems/[SYSTEM_ID_HERE]/systemusers"
```

### Add (or remove) system user to system

```
curl \
  -d '{ "add": ["[SYSTEM_USER_ID_TO_ADD_HERE]"], "remove": ["[SYSTEM_USER_ID_TO_REMOVE_HERE]"] }' \
  -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-api-key: [YOUR_API_KEY_HERE]" \
  "https://console.jumpcloud.com/api/systems/[SYSTEM_ID_HERE]/systemusers"
```

### More examples

Please check out the examples folder for more detailed examples.

[Add System User to Tags](/examples/nodejs/add-user-to-tags.js)



