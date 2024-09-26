# Writing Plugins

## Plugins Overview

Plugins are the core component UnMTA. Without them, the server can’t do anything besides reject mail.

Plugins bind to one or more server events, called [Hooks](#hooks). A plugin can read and modify [Session](#sessions) data to guide its behavior. After performing its designated function, a plugin can either trigger a [Response](#responses) to the client or allow the server to continue processing other plugins and hooks.

---

Writing plugins for UnMTA is designed to be quick and easy. Let’s take a look at a basic example. This plugin binds to the `onRcptTo` hook and will accept `peter.gibbons@initech.com`, defer `milton.waddams@initech.com`, and reject all others.

Let’s create a `plugins` directory, if we haven’t already, and add a file called `examplePlugin.ts`.

```bash
mkdir plugins
touch plugins/examplePlugin.ts
```

```typescript title="plugins/examplePlugin.ts"
import { SmtpPlugin, SmtpResponse } from '@unmta/smtp-server';

export const examplePlugin: SmtpPlugin = {
  pluginName: 'examplePlugin',
  onRcptTo: (session, recipient, command) => {
    if (recipient.address === 'peter.gibbons@initech.com') {
      return SmtpResponse.RcptTo.accept();
    } else if (recipient.address === 'milton.waddams@initech.com') {
      return SmtpResponse.RcptTo.defer(421, "Yeah, we can't actually find a record of him being a current employee here");
    }
    return SmtpResponse.RcptTo.reject();
  },
};
```

!!! note

    A unique `pluginName` property is required as it allows the plugin to store [session data](#plugin-session-data) within its own namespace and allows other plugins to access that session data.

We can now add this plugin to our SMTP Server like so:

```typescript title="index.ts" hl_lines="1 2 4"
import { SmtpServer, smtpPluginManager } from '@unmta/smtp-server';
import { examplePlugin } from './plugins/examplePlugin';

smtpPluginManager.loadPlugins([examplePlugin]);
const server = new SmtpServer();
server.start();
```

That’s it. The server will now apply the `examplePlugin`’s logic to the `onRcptTo` hook.

!!! note

    - Plugins will be called in the order they’re loaded via the `loadPlugins` method.
    - A plugin can contain one or many event hooks

A plugin can be a mere observer, or it can take near complete control of the server. Within a plugin, you can access data from a database, store session data, pass messages to other plugins, determine the server’s responses, and more. To learn how to make the most of plugins, let’s dig into the remaining fundamentals.

## Configuration

Plugins have read access to all server and plugin configuration parameters as defined in the default [Configuration File](/configuration). Additionally, you can define plugin-specific parameters in one of two ways:

1. Add to the `[plugins]` section in the `unfig.toml` file:

   ```toml title="config/unfig.toml"
   [plugins]

   [plugins.examplePlugin]
   doStuff=true
   ```

2. Create a new TOML configuration file in the `config/` directory using the `pluginName` of your plugin:

```toml title="config/examplePlugin.toml"
doStuff=true
```

!!! note

    Parameters set in an external config file (Ex: `config/yourPluginName.toml`) take precedence over parameters set in the `config/unfig.toml` file (Ex: `[plugins.yourPluginName]`).

### Reading Configuration Data

```typescript hl_lines="1 5 6"
import { unfig } from '@unmta/smtp-server';
export const examplePlugin: SmtpPlugin = {
  // ...
  onConnect: async (session) => {
    const authEnabled = unfig.auth.enable;
    const doStuff = unfig.plugins.examplePlugin.doStuff;
    // ...
  },
  // ...
};
```

!!! warning

    All configuration parameters are _readonly_

## Sessions

Session data is a per-connection store of information set by the server, other plugins, and your plugins. Plugins have read access to session data set by the server and by other plugins, and read/write access to its own session data. Let's take a look at the server-level session data.

### Server Session Data

| Property              | Type                        | Description                                                                              |
| --------------------- | --------------------------- | ---------------------------------------------------------------------------------------- |
| **id**                | `number`                    | A unique identifier for the session                                                      |
| **activeConnections** | `number`                    | Total number of active connections                                                       |
| **startTime**         | `number`                    | The time the session started ([`Date.now()`](https://mzl.la/3XHjKv1){:target="\_blank"}) |
| **remoteAddress**     | `string`                    | The remote IP address of the client                                                      |
| **phase**             | `string`                    | The current [phase](#session-phases) of the SMTP session                                 |
| **greetingType**      | `string` or `null`          | The greeting type used by the client (`HELO` or `EHLO`)                                  |
| **isSecure**          | `boolean`                   | Whether the connection is secured via TLS or STARTTLS                                    |
| **isAuthenticated**   | `boolean`                   | Whether the client has authenticated successfully                                        |
| **isDataMode**        | `boolean`                   | Whether the session is currently receiving data in `DATA` mode                           |
| **dataStream**        | `PassThrough` or `null`     | Incoming message data stream                                                             |
| **sender**            | `EnvelopeAddress` or `null` | The sender specified via `MAIL FROM`                                                     |
| **recipients**        | `EnvelopeAddress[]`         | The recipient(s) specified during `RCPT TO`                                              |

#### Session Phases

The session `phase` is primarily used by the server to ensure things are happening in their proper order. However, it can be useful for some plugins to know where they are in a transaction when, say, an `RSET` command is issued.

| Phase          | Description                                    |
| -------------- | ---------------------------------------------- |
| **connection** | Initial connection before any commands         |
| **auth**       | Authentication phase                           |
| **helo**       | `HELO`/`EHLO` command                          |
| **sender**     | `MAIL FROM` command                            |
| **recipient**  | `RCPT TO` command                              |
| **data**       | After `DATA` command and during data reception |
| **postdata**   | After data reception has completed             |

#### Reading Server Session Data

Accessing the session data from within a plugin is as simple as referencing the property directly from the `session` object:

```typescript hl_lines="4"
const examplePlugin: SmtpPlugin = {
  pluginName: 'examplePlugin',
  onConnect: (session) => {
    console.log(session.id); // Ex: 123
  },
};
```

!!! warning

    Server session properties are _readonly_

### Plugin Session Data

Plugins can read/write to their own namespace (as denoted by their `pluginName` property). Additionally, plugins may read from other plugins session data using their respective `pluginName`.

```typescript hl_lines="4 5 6"
const examplePlugin: SmtpPlugin = {
  pluginName: 'examplePlugin',
  onConnect: (session) => {
    session.setOwnPluginData('key', 'value'); // Sets session data in the examplePlugin namespace
    session.getOwnPluginData('key'); // Returns: 'value'
    session.getPluginData('anotherPluginName', 'key'); // Returns value for 'key' parameter from an external plugin, if set
  },
};
```

## Global Context

Many plugins will also have the need to reference data that transcends the life of a given connection. For example, your plugin may want establish a database connection. That's where the global SMTP Context fits in.

```typescript hl_lines="6 10"
export const examplePlugin: SmtpPlugin = {
  // ...
  onServerStart: async () => {
    const db = await initDatabase();
    // Store the db connection when the server starts
    SmtpContext.getInstance().set('exampleDBConnection', db);
  },
  onConnect: async (session) => {
    // Access the db connection anywhere it's needed
    const db = SmtpContext.getInstance().get('exampleDBConnection');
    // ...
  },
  // ...
};
```

Once set, the context value remains until the server is stopped or the value is overwritten.

!!! warning

    Global values are, well, global. Use descriptive names to ensure you don't overwrite values set by 3rd party plugins.

## Hooks

UnMTA exposes two types of hooks: SMTP Hooks for SMTP transactions, and Server Hooks for server events. Let’s start with SMTP Hooks:

### SMTP Hooks

SMTP Hooks are events triggered during an SMTP session. UnMTA allows you to attach to virtually every event in an SMTP transaction to shape the outcome of that event.

!!! note

    Hooks may be called multiple times per session if the client issues a `RSET` or another `HELO`/`EHLO` command. In some cases, your code may need to account for this fact.

| Hook            | Parameters                                     | Description                                                                                    |
| --------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **onConnect**   | [`session`](#sessions)                         | Called when a connection is first established with a client.                                   |
| **onHelo**      | [`session`](#sessions), `hostname`, `verb`     | Called after `HELO` / `EHLO` command. `verb` specifies which command was used.                 |
| **onAuth**      | [`session`](#sessions), `username`, `password` | Called after `AUTH` command once `username` and `password` have been provided from the client. |
| **onMailFrom**  | [`session`](#sessions), `address`, `command`   | Called after `MAIL FROM`. Includes `EmailAddress` object and full `SmtpCommand`.               |
| **onRcptTo**    | [`session`](#sessions), `address`, `command`   | Called after `RCPT TO`. Includes `EmailAddress` object and full `SmtpCommand`.                 |
| **onDataStart** | [`session`](#sessions)                         | Called after the DATA command.                                                                 |
| **onDataEnd**   | [`session`](#sessions)                         | Called after the entire message has been received from the client.                             |
| **onQuit**      | [`session`](#sessions)                         | Called after the `QUIT` command just before the connection is closed.                          |
| **onClose**     | [`session`](#sessions)                         | Called after the connection has been closed. Cannot return a response.                         |
| **onRset**      | [`session`](#sessions)                         | Called after the `RSET` command, before the server resets the session data.                    |
| **onHelp**      | [`session`](#sessions)                         | Called after the `HELP` command.                                                               |
| **onNoop**      | [`session`](#sessions)                         | Called after the `NOOP` command.                                                               |
| **onVrfy**      | [`session`](#sessions), `command`              | Called after the `VRFY` command. Includes full `SmtpCommand`.                                  |
| **onUnknown**   | [`session`](#sessions), `command`              | Called after an unknown command. Includes full `SmtpCommand`.                                  |

### Server Hooks

There are two additional hooks that relate to the server itself: `onServerStart`, and `onServerStop`. Unlike SMTP Hooks, they accept no parameters and can return no values. The `onServerStart` hook can be used to [establish database connections](#global-context), etc, while the `onServerStop` hook may be used to close database connections and perform cleanup tasks when the server is shutdown.

## Responses

All SMTP Hooks, except `onClose`, can _optionally_ return an `accept`, `defer`, or `reject` response. If a plugin returns a response, the server will skip any other plugins for that hook during the given connection. By not returning any value, the plugin allows the server to continue calling other plugins for the current hook.

!!! tip

    Choosing the correct response code and message can be challenging. Certain responses are not permissible depending on the phase of the SMTP session. UnMTA has created "guardrail" response classes that make it easy to say the right thing at the right time.

```typescript hl_lines="2 3 7 8 11 12"
onConnect: async (session) => {
    doStuff();
    // No response returned. Other onConnect plugins will be called
  },
export const examplePlugin: SmtpPlugin = {
  onHelo: (session) => {
    // Don't call any other plugins, just send an accept response
    return SmtpResponse.Helo.accept();
  },
  onAuth: (session, username, password) => {
    // Don't call any other plugins, send a custom reject response
    return SmtpResponse.Auth.reject(535, '5.7.8 YOU SHALL NOT PASS!');
  },
};
```

Note how each hook has a corresponding SMTP response type (`SmtpResponse.Helo` for `HELO`/`EHLO`, `SmtpResponse.Auth` for `AUTH`, etc). These response types provide a default way to either `accept` (send a `2XX` or `3XX` response), defer (send a `4XX` response), or reject (send a `5XX` response).

By simply calling a relevant `accept()`, `defer()`, or `reject()`, the server will issue a standard RFC-compliant response. If you want to send a non-default (but still RFC-compliant) status code, you can specify that as the first parameter in an accept/defer/reject method. And, if you'd like to add a custom message, you can specify that as the second parameter.

!!! tip

    The easiest way to see all of the available response codes and messages is to look at the [SmtpResponse](https://github.com/unmta/smtp-server/blob/main/lib/SmtpResponse.ts) Class

While it's recommended to rely on the guardrail response classes to ensure you stay RFC compliant, if you wish to respond with a status code and/or message that the guardrail classes don't permit, there's `SmtpResponseAny`:

```typescript hl_lines="4"
export const examplePlugin: SmtpPlugin = {
  onHelo: (session) => {
    // Return a non-standard response. Because reasons.
    return SmtpResponseAny(469, '4.2.0 I believe you have my stapler');
  },
};
```

!!! note

    - Certain responses (example, a defer response that returns a 421) will also close the connection.

## Logging

Plugins can piggy-back on UnMTA's logger:

```typescript hl_lines="1 4"
import { logger } from '@unmta/smtp-server';
export const examplePlugin: SmtpPlugin = {
  onHelo: (session) => {
    logger.debug('oh hai, mark');
  },
};
```

The logger supports the following log levels:

| Level | Method | Details                                                                                                 |
| ----- | ------ | ------------------------------------------------------------------------------------------------------- |
| 0     | error  | Uncaught exceptions and other unrecoverable issues                                                      |
| 1     | warn   | Potential problems that may not currently affect the system’s operation but could lead to future issues |
| 2     | info   | General operational messages that describe normal functioning                                           |
| 3     | debug  | Typically used during development or troubleshooting                                                    |
| 4     | smtp   | Displays full SMTP client<->server chatter                                                              |

To setup log rotation and other advanced logging features, see [Deploying to Production](/deploying-to-production)

## Writing Public Plugins

!!! tip

    UnMTA welcomes 3rd party contributions!

First, let’s discuss the philosophy behind writing public plugins for use with UnMTA. Your plugin should be designed to “do the thing” it’s supposed to do, and nothing more. If you’re writing a plugin to scan for viruses, your plugin should, for example, scan for viruses and store the results in the session, and that’s all. This enables your end-users to write simple downstream plugins to handle those results however they prefer.

### A good plugin example :thumbsup:

```typescript
onDataEnd: async (session) => {
  const result = await doVirusScan();
  session.setOwnPluginData('result', result);
  // This will allow downstream plugins to decide how to handle the message.
},
```

### A bad plugin example :thumbsdown:

Many MTAs have plugins that try and do everything. Taking the virus scan example, these plugins often offer up configuration parameters on whether or not to reject the message, add a prefix to the subject line, add an X-Header, etc. The result is a far more complicated plugin than what is really needed and there will be an endless supply of use-cases that your plugin doesn’t _quite_ fit.

```typescript
onDataEnd: (session) => {
  const result = await doVirusScan();
  if (result.hasVirus) {
    if (unfig.plugins.virusPlugin.rejectInfectedEmails) {
      return SmtpResponse.Data.reject(550, 'Message contains a virus');
    } else if (unfig.plugins.virusPlugin.tagInfectedEmails) {
      return SmtpResponse.Data.accept(250, 'Message accepted for delivery (virus detected)');
    } else if (unfig.plugins.virusPlugin.quarantineInfectedEmails) {
      return SmtpResponse.Data.accept(250, 'Message quarantined (virus detected)');
    } else if (unfig.plugins.virusPlugin.deleteInfectedEmails) {
      return SmtpResponse.Data.accept(250, 'Message deleted (virus detected)');
    } else if (unfig.plugins.virusPlugin.deferInfectedEmails) {
      return SmtpResponse.Data.defer(450, 'Message deferred (virus detected)');
    }
    // ... and on
    // ... and on
    // ... and on it goes
  }
};
```

See the difference? Let’s keep the Simple in SMTP.
