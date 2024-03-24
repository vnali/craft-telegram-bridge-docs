# Telegram Bridge plugin for Craft CMS
This Craft CMS plugin helps you to integrate Craft CMS with Telegram.

This plugin is in the development phase.

## License & Pricing
This is going to be a commercial plugin available through the [Craft plugin store](https://plugins.craftcms.com/developer/vnali).

## Requirement
Craft 4.5 and higher.

## Main features:
- Notify users about events by sending requests through webhooks to Telegram users.
- Providing some tools to get data from Craft CMS sites based on specified steps via telegram chat.
- Providing an interface for users via telegram chat to execute predefined GraphQL queries.

## Setup
The Initial setup steps for using this plugin are as follows:
- Acquiring a telegram bot API key. please follow [Obtain Your Bot Token](https://core.telegram.org/bots/tutorial#obtain-your-bot-token) to create a new bot.
- Set the `TELEGRAM_BOT_TOKEN` environment setting to that bot token.
- Install and enable this plugin via the plugin store

## Notify users about events by sending requests through webhooks to Telegram users

By using this plugin, you can send notifications or messages about various events to Telegram users (for example your site admin, and authors).
- Install webhook plugin https://github.com/craftcms/webhooks.
- Create a new webhook.
- Select an event that you want to send notifications to telegram users about it. 
- Create a POST request with https://{sit url}/telegram-bridge/craft-webhook url.
- You can select some filters for your event. this plugin also provides some filters. for example, a useful filter is `The user has a chat id`. you can use this filter to only fire requests if the current user who triggers the events has a chatId.
- You must have a header with this name:
`X-Craft-Telegram-Bridge-Secret-Token`. value of this custom header should be set to `$X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN` environment variable.
- Select 'send a custom payload'
- Create a custom payload in this format:

```
{{
  {
    time: now|datetime('short'),
    user: currentUser.username ?? null,
    message: 'custom message', // required
    chatIds: 'chatId1,chatId2' // required. this is the list of chatIds separated by ',' or ', '
  }|json_encode|raw
}}
```
- Save the webook. now when the selected event triggers, a message will be sent to the specified telegram chat Ids.

### Webhook examples (needs review before using in production):

#### Create a Backup
```
Name:
Create a backup

Sender Class:
craft\db\Connection

Event Name:
afterCreateBackup

Event filter: The user has a chat id

Request Method & URL:
POST: {sit url}/telegram-bridge/craft-webhook

Custom Headers:
Name: X-Craft-Telegram-Bridge-Secret-Token Value: $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

Send a custom payload:
{% set file = event.file %}
{% set username = currentUser.username %}
{% if username %}
  {% set chatId=craft.telegrambridge.chatId.getChatIdByUser(username) %}
    {% if chatId %}
      {{
        {
          time: now|datetime('full'),
          user: username,
          message: 'backup created: ' ~ file,
          chatIds: chatId
        }|json_encode|raw
      }}
    {% endif %}
{% endif %}
```

#### Login Failure
```
Name:
Login Failure

Sender Class:
craft\controllers\UsersController

Event Name:
loginFailure

Request Method & URL:
POST: http://example.test/telegram-bridge/craft-webhook

Custom Headers:
X-Craft-Telegram-Bridge-Secret-Token $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

Send a custom payload:
{% if event.user and event.user.username %}
  {% set username = event.user.username %}
  {% set chatId=craft.telegrambridge.chatId.getChatIdByUser(username) %}
  {% if chatId %}
    {{
      {
        time: now|datetime('full'),
        message: 'login failure: ' ~ username,
        chatIds: chatId
      }|json_encode|raw
    }}
  {% endif %}
{% endif %}
```

#### Entry Save
```
Name:
Entry Save

Sender Class:
craft\elements\Entry

Event Name:
afterSave

Request Method & URL:
POST: {site url}/telegram-bridge/craft-webhook

Custom Headers:
X-Craft-Telegram-Bridge-Secret-Token $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

Send a custom payload:
{% set title = event.sender.title %}
{% set username = currentUser.username %}
{% if username %}
  {% set chatId=craft.telegrambridge.chatId.getChatIdByUser(username) %}
  {% if chatId %}
    {{
      {
        time: now|datetime('short'),
        user: username,
        message: 'Entry Saved: ' ~ title,
        chatIds: chatId
      }|json_encode|raw
    }}
  {% endif %}
{% endif %}
```

#### ENTRY DELETE
```
Name:
Entry Delete

Sender Class:
craft\elements\Entry

Event Name:
afterDelete

Event filter:
Element is a provisional draft set to false // to prevent webhook execution when provisional drafts are deleted

Request Method & URL:
POST: http://{site url}/telegram-bridge/craft-webhook

Custom Headers:
X-Craft-Telegram-Bridge-Secret-Token $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

Send a custom payload:
{% set title = event.sender.title %}
{% set username = currentUser.username %}
{% if username %}
  {% set chatId=craft.telegrambridge.chatId.getChatIdByUser(username) %}
  {% if chatId %}
    {{
      {
        time: now|datetime('short'),
        user: username,
        message: 'Entry Deleted: ' ~ title,
        chatIds: chatId
      }|json_encode|raw
    }}
  {% endif %}
{% endif %}
```

#### Order paid // Craft commerce
```
Name:
Order paid

Sender Class:
craft\commerce\elements\Order

Event Name:
afterOrderPaid

Request Method & URL:
POST: http://{siteUrl}/telegram-bridge/craft-webhook

Custom Headers:
X-Craft-Telegram-Bridge-Secret-Token $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

{% set customer = event.sender.customer.email %}
{% set totalPrice = event.sender.totalPrice %}
{% set totalQty = event.sender.totalQty %}
{% set username = currentUser.username %}
{% if username %}
    {% set items = '' %}
    {% for lineItem in event.sender.lineItems %}
        {% set items = items ~ lineItem.qty ~ ' ' ~ lineItem.description %}
    {% endfor %}
    {{
      {
        time: now|datetime('short'),
        user: username,
        message: 'Paid order: by ' ~ username ~ ' for ' ~ customer ~ ' Qty: ' ~ totalQty ~ ' Price: ' ~ totalPrice ~ items,
        chatIds: 'groupId,channelId'
      }|json_encode|raw
    }}
{% endif %}
```

#### Form submit // Freeform 4
```
Name:
Form Submit

Sender Class:
Solspace\Freeform\Services\SubmissionsService

Event Name:
afterSubmit

Request Method & URL:
POST: http://{siteUrl}/telegram-bridge/craft-webhook

Custom Headers:
X-Craft-Telegram-Bridge-Secret-Token $X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN

{% set chatId= 'xyz' %}
{% set username = currentUser.username %}
{{
  {
    time: now|datetime('short'),
    user: username,
    message: 'New submission for ' ~ event.form.name ~ ' form',
    chatIds: chatId
  }|json_encode|raw
}}
```

## Setup for interacting with a Craft site via a bot (Required for executing GQL queries and accessing extra tools)
To be able to interact with a bot and access provided tools or execute GraphQL queries, in addition to basic setup steps, you should also follow these steps:
- In the dev environment, keep the get updates page open.
  - In the production environment instead, go to the setting page and set webhook for your site to `https://{example.test}/telegram-bridge/telegram-webhook`.
- Open the bot through the Telegram application and press `/start`.
- The plugin only responds to the specified chat ids in `ALLOWED_TELEGRAM_CHAT_IDS` environment setting so add allowed tegeram chat ids to this environment setting.
  -  If you want to allow all telegram chat Ids to interact with the plugin  - currently only executing GQLs- you should set `ALLOW_OTHER_TELEGRAM_CHAT_IDS_GQL` to true
  -  If you don't know your telegram chat id, set `ALLOW_CHAT_ID_COMMAND` environment setting to true and send `/chatid` command. the bot returns your chat id.
  -  The format of `ALLOWED_TELEGRAM_CHAT_IDS` is `'chatId1||chatId2||...'`
    - Make sure you set `ALLOWED_TELEGRAM_CHAT_IDS` correctly; otherwise, you may inadvertently give another person unintended access.
    - We may automate the process of setting `ALLOWED_TELEGRAM_CHAT_IDS` in next versions by sending a command from the Telegram user itself.
- Now the bot and plugin respond to your messages so you can access the tools provided by the plugin or execute your GraphQL queries based on your permission.
- Please follow the instructions below


## Providing some tools to get data from Craft CMS sites
Currently, this plugin provides tools based on Craft and Craft Commerce plugin widgets to fetch reports from Craft site in two categories:
- Craft // These tools are like Craft's widgets in dashboard
  - Recent Entries
  - My Drafts
- Craft Commerce // These tools are like Craft Commerce's widgets in the Craft dashboard
  - Average Order Total
  - New Customers
  - Repeat Customers
  - Recent Orders
  - Top Customers
  - Top Product Types
  - Top Products
  - Top Purchasables
  - Total Orders // output also can be represented in chart format
  - Total Orders by Country // output also can be represented in chart format
  - Total Revenue // output also can be represented in chart format
 
- You can manage the permissions for users to only access Craft/Craft Commerce tools.
- By setting SHOW_RESULT_CHART to true, tools that support charts, show a chart to the user.
- Please watch an example video [here](https://www.youtube.com/watch?v=Ad9I4pSfKq0)

#### Example 1:  
We have two users with these specifications
- username: userA
- user Id: 10
- user chat id: 123
and:
- username: userB
- user Id: 11
- user chat id: 124

We want userA to access craft and craft commerce tools.  
We want userB to access only craft tools.  

The .env file in this scenario should be:
```
ALLOWED_TELEGRAM_CHAT_IDS='123||124'
ALLOWED_TELEGRAM_CHAT_IDS_USER='10-userA||11-userB'  // in this way, you can match the first item of CHAT_IDS_USER to CHAT_IDS and so on.
```

Now in the permission tab of each user, you can choose the tools that you want to give to the users.

## Providing an interface via telegram bot to execute GraphQL queries

By using this plugin, you can let telegram users execute predefined GQL queries via their specified tokens.  
This plugin helps telegram users to send required arguments step by step for gathering query variables for executing GQL queries.  
We also suggest options for these steps based on their token's permission: site, siteId, section, sectionId, type, typeId, volume, volumeId and folder.

- First of all, you must specify GraphQL API endpoint via `GRAPHQL_API` in .env file
- You should specify structure sections that keep entries that include GraphQL queries via `GRAPHQL_QUERY_SECTION` in .env file
   - you can specify different sections. Each user only sees entries that its GQL token has access to read.
- You should specify a text field handle that keeps GraphQL queries via `GRAPHQL_QUERY_FIELD` in .env file
- Now for each GraphQL query, you should create an entry.
  - If you would like to categorize GraphQL queries, you should create a parent entry and leave the body of the query field empty, so we know that is a parent item and we show them with 📂 -. then create child entries for that keep GQL queries - we show them with ❓.
  - Currently, these entries which represent GQL queries only must be in primary site.
- Only telegram chat ids specified in `ALLOWED_TELEGRAM_CHAT_IDS` environment setting can access GQL.
  - the format is `{chatId1}`\|\|`{chatId2}`
- You should specify which GQL token each telegram chat id uses in `ALLOWED_TELEGRAM_CHAT_IDS_GQL_TOKEN` 
  - the format is `tokenId-TokenName||tokenId2-TokenName2` or `public` // both tokenId and tokenName are used in the format to prevent mistakes.
- Plese watch an example video [here](https://www.youtube.com/watch?v=y2Ij74CILc4)

#### Example 2:  
We have two users with these specifications
- username: userA
- email: userA@example.test
- user Id: 10
- user chat id: 123
and:
- username: userB
- email: userB@example.test
- user Id: 11
- user chat id: 124

We want userA to run all GraphQL queries via the Telegram interface and also access craft and craft commerce tools.  
We want userB to run some read GraphQL queries via the Telegram interface and also access only craft tools.  
We want all telegram users who can access the bot to run some basic GraphQL queries.  

In this case, to separate the access to queries, we should put queries into two different sections. in our example we use these handles: 'gqlMutation' and 'gqlRead' and we
use a text field with the handle 'textQuery' to keep the query.  
In the next step, we create two different tokens, full and basic. the full token with id 3 can access to view entries on the 'gqlMutation' and 'gqlRead' sections. the basic token with id 2 can access to view entries on read sections.  
For other telegram users, we can edit Public Schema and give them some basic access to other schemas.

For gqlMutation section, the structure of entries is like this:
```
- Assets creation // an entry with empty 'textQuery'. we do this to categorize entries
   - query1 // The query should be in the 'textQuery' field
   - query2 // The query should be in the 'textQuery' field
- Entry creation  // an entry with empty 'textQuery'. we do this to categorize entries
   - query3
......
```

A sample for saving assets via GraphQL query is:
```
mutation save($fileInput_file: FileInput!, $title: String) {
  save_testImage_Asset(_file: $fileInput_file, title: $title){
    title
  }
}
```

The .env file in this scenario should be:
```
ALLOWED_TELEGRAM_CHAT_IDS_USER='10-userA||11-userB'
ALLOWED_TELEGRAM_CHAT_IDS='123||124'
ALLOWED_TELEGRAM_CHAT_IDS_GQL_TOKEN='3-full||2-basic'
ALLOW_OTHER_TELEGRAM_CHAT_IDS_GQL=true
OTHER_TELEGRAM_CHAT_IDS_GQL_TOKEN='public'
GRAPHQL_QUERY_SECTIONS='gqlRead||gqlMutation' // These sections only should be used for entries that include GQL queries. 
GRAPHQL_QUERY_FIELD='textQuery'
```

Finally we let UserA and UserB use Craft/Craft Commerce tools by adjusting their permissions in each user's permission tab.

## Plugin Pages

### Get updates
  - In the dev environment, you can get the latest updates from chats by keeping the get updates page open. to be able to get updates from Telegram bot via the get updates page, you must delete the webhook on settings page. -because both methods won't work together-
    - In the production environment, you should use webhook method instead.
  - You can set auto-refresh rate of get updates page in milliseconds via `AUTO_REFRESH_GET_UPDATES_PER_MILLI_SEC` environment setting.

### Settings
At the top of this page, you can see the information about your telegram bot and webhook.  
If there are any warnings or errors about your environment settings, you can see them on this page.
  - For example, you can see errors about mismatched token ids and token names, mismatched user id and username, token not found, user not found, and ...
If there is no error and `$SHOW_CHAT_ID_TABLE` is set to true, you can see list of allowed chat Ids and their users and tokens.
At the bottom of the page, you can see set webhook form.
  - On the production environment, you should set webhook and the telegram will send updates to webhook url and this plugin takes care of proccessing and responding to them

## List of all supported environment settings  

You can get a copy of sample environment settings [here](https://github.com/vnali/craft-telegram-bridge-docs/blob/main/.env.example).  


Environment setting | Example value | Description
--- | --- | ---
X_CRAFT_TELEGRAM_BRIDGE_SECRET_TOKEN | a-complex-plugin-secret | Use a complex secret for verifying the authenticity of requests sent via the craft webhook.
X_TELEGRAM_BOT_API_SECRET_TOKEN | a-complex-telegram-secret | Use a complex secret for verifying the authenticity of requests sent via the Telegram webhook.
TELEGRAM_BOT_TOKEN | telegram-bot-token | 
TELEGRAM_WEBHOOK_ADDRESS | https://{example.test}/telegram-bridge/telegram-webhook | The destination address where Telegram requests are delivered. you should only replace the {site}.
ALLOWED_TELEGRAM_CHAT_IDS | 1001\|\|1002 | Define which Telegram chat IDs have permission to interact with your website through the Telegram bot. separate the chat ids with `\|\|`.
ALLOWED_TELEGRAM_CHAT_IDS_GQL_TOKEN | 1-full\|\|public or  \|\|public or 1-full\|\| | The tokens that each chat ID utilizes to execute GraphQL commands through the Telegram bot. the format is `{token ID-token name}` or `public` and you should separate the them by `\|\|`.
ALLOWED_TELEGRAM_CHAT_IDS_USER | 1-user1\|\| or  \|\|1-user1 or  1-user1\|\|2-user2 | The user element related to each telegram chat id. the format is `{user Id-username}\|\|{user Id-username}` and separate the users by `\|\|`.
ALLOW_OTHER_TELEGRAM_CHAT_IDS_GQL | true | If other chat ids are allowed to execute GraphQL queries.
OTHER_TELEGRAM_CHAT_IDS_GQL_TOKEN | 2-basic | Which token, other chat Ids use when they are allowed to execute GraphQL queries. it should be `{token Id- token name}` or `public`.
SHOW_CHAT_ID_TABLE | true | Show list of allowed chat Ids and their users and tokens at the setting page
GRAPHQL_API | https://{example.test}/api |
GRAPHQL_QUERY_SECTIONS | mutation\|\|reads\|\|basic | Specify the section handles you use for keeping Graph QL queries. telegram bot only shows the sections which the token of that chatId can read. separate section handles by `\|\|`.
GRAPHQL_QUERY_FIELD | queryField | Represents the text field handle that includes the GraphQL queries.
SHOW_GRAPHQL_QUERY | true | Show executed query for a GraphQL query. it can be helpful in dev environment to see executed query and passed variables.
SHOW_RESULT_CHART | true | Show a chart for a tool result if the chart is available for the result.
AUTO_REFRESH_GET_UPDATES_PER_MILLISEC | 2000 | Represents the duration, in milliseconds, defining the time interval between automated refresh for the get update page.
ALLOW_CHAT_ID_COMMAND | true | Allow to respond to /chatId.
CURL_IP_PROXY | ip |
CURL_PORT_PROXY | port number |

## Contact
Feel free to contact me by email at vnali.dev@gmail.com or direct message me via 'vnali' on [Craft CMS Discord](https://craftcms.com/discord) channel.
