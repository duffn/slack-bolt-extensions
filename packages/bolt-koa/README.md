## Bolt for JavaScript: Koa Receiver

This module provides a `Receiver` implementation for [Koa](https://koajs.com/) users.

### Getting Started

You can create a simple Node app project using the following `package.json` and `tsconfig.json`. Of course, if you would like to use some build tool such as [webpack](https://webpack.js.org/), you can go with your own way and add the necessary dependencies.

##### package.json

```json
{
  "name": "bolt-koa-app",
  "version": "0.1.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "rm -rf dist/ && tsc && npx ts-node src/index.ts"
  },
  "author": "Kazuhiro Sera",
  "license": "MIT",
  "dependencies": {
    "@slack/bolt": "^3.12.2",
    "@koa/router": "^10.1.1",
    "@seratch_/bolt-koa": "^1.0.0",
    "koa": "^2.13.4"
  },
  "devDependencies": {
    "ts-node": "^10.5.0",
    "typescript": "^4.5.5"
  }
}
```

##### tsconfig.json

```json
{
  "compilerOptions": {
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowJs": true,
    "esModuleInterop": true,
    "outDir": "dist",
  },
  "include": ["src/**/*"]
}
```

##### Create a new Slack app at api.slack.com/apps

The next step is to create a new Slack app configuration. You can use the following App Manifest configuration data for it.

```yaml
display_information:
  name: koa-oauth-test-app
features:
  bot_user:
    display_name: koa-oauth-test-app
oauth_config:
  redirect_urls:
    - https://xxx.ngrok.io/slack/oauth_redirect
  scopes:
    bot:
      - commands
      - chat:write
      - app_mentions:read
settings:
  event_subscriptions:
    bot_events:
      - app_mention
  interactivity:
    is_enabled: true
  socket_mode_enabled: true
```

### Place your source code in the project

The last step is to add your code in the project and spin up your app. You can use the following code as-is.

##### src/index.ts

```typescript
import Router from '@koa/router';
import Koa from 'koa';
import { App, FileInstallationStore, LogLevel } from '@slack/bolt';
import { FileStateStore } from '@slack/oauth';
import { KoaReceiver } from '@seratch_/bolt-koa';

const koa = new Koa();
const router = new Router();

const receiver = new KoaReceiver({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  clientId: process.env.SLACK_CLIENT_ID,
  clientSecret: process.env.SLACK_CLIENT_SECRET,
  scopes: ['commands', 'chat:write', 'app_mentions:read'],
  installationStore: new FileInstallationStore(),
  installerOptions: {
    directInstall: true,
    stateStore: new FileStateStore({}),
  },
  koa,
  router,
});

const app = new App({
  logLevel: LogLevel.DEBUG,
  receiver,
});

app.event('app_mention', async ({ event, say }) => {
  await say({
    text: `<@${event.user}> Hi there :wave:`,
    blocks: [
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `<@${event.user}> Hi there :wave:`,
        },
      },
    ],
  });
});

app.command('/my-command', async ({ ack }) => {
  await ack('Hi there!');
});

app.shortcut('my-global-shortcut', async ({ ack, body, client }) => {
  await ack();
  await client.views.open({
    trigger_id: body.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'my-modal',
      title: {
        type: 'plain_text',
        text: 'My App',
      },
      submit: {
        type: 'plain_text',
        text: 'Submit',
      },
      blocks: [
        {
          type: 'input',
          block_id: 'b',
          element: {
            type: 'plain_text_input',
            action_id: 'a',
          },
          label: {
            type: 'plain_text',
            text: 'Comment',
          },
        },
      ],
    },
  });
});

app.view('my-modal', async ({ view, ack, logger }) => {
  logger.info(view.state.values);
  await ack();
});

(async () => {
  await app.start();
  // eslint-disable-next-line no-console
  console.log('⚡️ Bolt app is running!');
})();
```

Finally, your app is now available for running! Set all the required env variables, hit `npm start`, and then enable your public URL endpoint (you may want to use some proxy tool such as [ngrok](https://ngrok.com/)).

```bash
export SLACK_CLIENT_ID=
export SLACK_CLIENT_SECRET=
export SLACK_SIGNING_SECRET=
export SLACK_STATE_SECRET=secret
export SLACK_APP_TOKEN=
npm start
# Visit https://{your public domain}/slack/install
```

Now you can install the app into your Slack workspace from `https://{your public domain}/slack/install`. Enjoy!
