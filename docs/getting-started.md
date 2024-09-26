# Getting Started

## Installation

#### Prerequisites:

- Make sure you have Node.js >= v20 installed
- To run the examples, you may want to have tsx installed: `npm install -g tsx`

Let’s start by creating a workspace for the UnMTA Smtp Server:

```bash
mkdir unmta-smtp-server-project
cd unmta-smtp-server-project
npm init
# Follow the `npm init` prompts
```

Now we can install UnMTA Smtp Server:

```bash
npm i @unmta/smtp-server
```

Now let’s create an index.ts or index.js file to bootstrap the server. Feel free to use either TypeScript or JavaScript. We’ll be using TypeScript in our examples.

```typescript title="index.ts"
import { SmtpServer } from "@unmta/smtp-server";

const server = new SmtpServer();
server.start();
```

Now we’re ready to start the server. By default, the server will bind to your localhost interface on port 2525:

```bash
tsx index.ts
```

```log
[info]: Unfig (config) loaded
[info]: Logger initialized. Level: 'smtp'
[info]: UnMTA SMTP server is running on localhost:2525
```

Congratulations! You’re running the UnMTA SMTP Server. What can it do at this point? Absolutely nothing. Let’s fix that by learning about the [Configuration Options](/configuration) and [Writing Plugins](/writing-plugins)
