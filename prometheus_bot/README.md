- [prometheus_bot](#prometheus-bot)
  * [Docker Image](#docker-image)
  * [Compile](#compile)
  * [Usage](#usage)
    + [Configuring alert manager](#configuring-alert-manager)
  * [Test](#test)
    + [Create your own test](#create-your-own-test)
  * [Customising messages with template](#customising-messages-with-template)
    + [Template extra functions](#template-extra-functions)
      - [Support this functions list](#support-this-functions-list)
  * [Production example](#production-example)
  * [Telegram is blocked by ISP](#telegram-is-blocked-by-isp)

# prometheus_bot

This bot is designed to alert messages from [alertmanager](https://github.com/prometheus/alertmanager).

## Docker Image

You can run it using deployed image in ghcr.io/incaller/prometheus_bot.

## Compile

[GOPATH related doc](https://golang.org/doc/code.html#GOPATH).
```bash
export GOPATH="your go path"
make clean
make
```

## Usage

1. Create Telegram bot with [BotFather](https://t.me/BotFather), it will return your bot token

2. Specify telegram token in ```config.yaml```:

    ```yml
    telegram_token: "token goes here"
    # ONLY IF YOU USING DATA FORMATTING FUNCTION, NOTE for developer: important or test fail
    time_outdata: "02/01/2006 15:04:05" 
    template_path: "template.tmpl" # ONLY IF YOU USING TEMPLATE
    time_zone: "Europe/Rome" # ONLY IF YOU USING TEMPLATE
    split_msg_byte: 4000
    send_only: true # use bot only to send messages.
    ```

3. Run ```telegram_bot```. See ```prometheus_bot --help``` for command line options
3. Get chat ID with one of two ways
    1. Start conversation, send message to bot mentioning it
    2. Add your bot to a group. It should report group id now. To get ID of a group if bot is already a member [send a message that starts with `/`](https://core.telegram.org/bots#privacy-mode)

### Configuring alert manager

Alert manager configuration file:

```yml
- name: 'admins'
  webhook_configs:
  - send_resolved: True
    url: http://127.0.0.1:9087/alert/chat_id
```

Replace ```chat_id``` with the value you got from your bot, ***with everything inside the quotes***.
(Some chat_id's start with a ```-```, in this case, you must also include the ```-``` in the url)
To use multiple chats just add more receivers.

If you want send messages to topic chat, append ```topic_id``` after ```chat_id```.
```yml
- name: 'admins'
  webhook_configs:
  - send_resolved: True
    url: http://127.0.0.1:9087/alert/chat_id/topic_id
```

## Test

To run tests with `make test` you have to:

- Create `config.yml` with a valid telegram API key and timezone in the project directory
- Create `prometheus_bot` executable binary in the project directory
- Define chat ID with `TELEGRAM_CHATID` environment variable
- Ensure port `9087` on localhost is available to bind to

```bash
export TELEGRAM_CHATID="YOUR TELEGRAM CHAT ID"
make test
```
### Create your own test
When alert manager send alert to telegram bot, *only debug flag ```-d```* Telegram bot will dump json in that generate alert, in stdout.
You can copy paste this from json for your test, by creating new .json.
Test will send ```*.json``` file into ```testdata``` folder

or

```sh
TELEGRAM_CHATID="YOUR TELEGRAM CHAT ID" make test
```

## Customising messages with template

This bot support [go templating language](https://golang.org/pkg/text/template/).
Use it for customising your message.

To enable template set these settings in your ```config.yaml``` or template will be skipped.

```yml
telegram_token: "token here"
template_path: "template.tmpl" # your template file name
time_zone: "Europe/Rome" # your time zone check it out from WIKI
split_token: "|" # token used for split measure label.
disable_notification: true  # disable notification for messages.
```

You can also pass template path with `-t` command line argument, it has higher priority than the config option.

[WIKI List of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

Best way for build your custom template is:
-    Enable bot with ```-d``` flag
-    Catch some of your alerts in json, then copy it from bot STDOUT
-    Save json in testdata/yourname.json
-    Launch ```make test```

```-d``` options will enable ```debug``` mode and template file will reload every message, else template is load once on startup.

Is provided as [default template file](testdata/default.tmpl) with all possibile variable.
Remember that telegram bot support HTML tag. Check [telegram doc here](https://core.telegram.org/bots/api#html-style) for list of aviable tags.

### Template extra functions
Template language support many different functions for text, number and data formatting.

#### Support this functions list

-   ```str_UpperCase```: Convert string to uppercase
-   ```str_LowerCase```: Convert string to lowercase
-   ```str_Title```: Convert string in Title, "title" --> "Title" fist letter become Uppercase
-   DEPRECATED  ```str_Format_Byte```: Convert number expressed in ```Byte``` to number in related measure unit. It use ```strconv.ParseFloat(..., 64)``` take look at go related doc for possible input format, usually every think '35.95e+06' is correct converted.
Example:
    -    35'000'000 [Kb] will converter to '35 Gb'
    -    89'000 [Kb] will converter to '89 Mb'
-   ```str_Format_MeasureUnit```: Convert string to scaled number and add append measure unit label. For add measure unit label you could add it in prometheus alerting rule. Example of working: 8*e10 become 80G. You cuold also start from a different scale, example kilo:"s|g|3". Check production example for complete implementation. Require ```split_token: "|"``` in conf.yaml
-   ```HasKey```: Param:dict map, key_search string Search in map if there requeted key

-    ```str_FormatDate```: Convert prometheus string date in your preferred date time format, config file param ```time_outdata``` could be used for setup your favourite format
Require more setting in your cofig.yaml
```yaml
time_zone: "Europe/Rome"
time_outdata: "02/01/2006 15:04:05"
```
[WIKI List of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

## Production example

Production example contains a example of how could be a real template.

```testdata/production_example.json```
```testdata/production_example.tmpl```

It could be a base, for build a real template, or simply copy some part, check-out how to use functions.
Sysadmin usually love copy.

## Telegram is blocked by ISP

Your ISP is blocked Telegram cannot use Default Telegram API

### Add Telegram custom API to ```config.yaml```.

```yml
telegram_url_api: "https://url/bot%s/%s" # Url custom must has %s
telegram_token: "token here"
template_path: "template.tmpl" # your template file name
time_zone: "Europe/Rome" # your time zone check it out from WIKI
split_token: "|" # token used for split measure label.
disable_notification: true  # disable notification for messages.
```

### Your can custom code in ```main.go```.

Add new line to ```type Config struct {```

```bash
// add custom URL API, use pointer to allow nil or undefined
TelegramURLAPI      *string `yaml:"telegram_url_api"`
```

Comment command NewBotAPI with default Telegram API

```bash
//bot_tmp, err := tgbotapi.NewBotAPI(cfg.TelegramToken)
```

Add new command NewBotAPIWithAPIEndpoint

```
if cfg.TelegramURLAPI == nil {
  // use endpoint temp to get default TelegramAPI
  endpoint := tgbotapi.APIEndpoint

  // change endpoint, use & with variable because cfg.TelegramURLAPI is a pointer
  cfg.TelegramURLAPI = &endpoint
}
// Print TelegramURLAPI to log
//log.Printf("cfg.TelegramURLAPI: %s", *cfg.TelegramURLAPI)
bot_tmp, err := tgbotapi.NewBotAPIWithAPIEndpoint(cfg.TelegramToken, *cfg.TelegramURLAPI)
```

### Use Cloudflare Worker like middleware API
1. Login Dashboard Cloudflare
2. Workers
3. Create Worker
4. Pass Javascript code to [Worker File](#worker-file)
5. Deploy
6. Open worker URL

#### Worker File
  ```js
  // Telegram Bot API base URL
  const TELEGRAM_API_BASE = 'https://api.telegram.org';

  // HTML template for documentation
  const DOC_HTML = `<!DOCTYPE html>
  <html>
  <head>
      <title>Telegram Bot API Proxy Documentation</title>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <style>
          body {
              font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
              line-height: 1.6;
              max-width: 800px;
              margin: 0 auto;
              padding: 20px;
              color: #333;
          }
          h1 { color: #0088cc; }
          .code {
              background: #f5f5f5;
              padding: 15px;
              border-radius: 5px;
              font-family: monospace;
              overflow-x: auto;
          }
          .note {
              background: #fff3cd;
              border-left: 4px solid #ffc107;
              padding: 15px;
              margin: 20px 0;
          }
          .example {
              background: #e7f5ff;
              border-left: 4px solid #0088cc;
              padding: 15px;
              margin: 20px 0;
          }
      </style>
  </head>
  <body>
      <h1>Telegram Bot API Proxy</h1>
      <p>This service acts as a transparent proxy for the Telegram Bot API. It allows you to bypass network restrictions and create middleware for your Telegram bot applications.</p>
      
      <h2>How to Use</h2>
      <p>Replace <code>api.telegram.org</code> with this worker's URL in your API calls.</p>
      
      <div class="example">
          <h3>Example Usage:</h3>
          <p>Original Telegram API URL:</p>
          <div class="code">https://api.telegram.org/bot{YOUR_BOT_TOKEN}/sendMessage</div>
          <p>Using this proxy:</p>
          <div class="code">https://{YOUR_WORKER_URL}/bot{YOUR_BOT_TOKEN}/sendMessage</div>
      </div>

      <h2>Features</h2>
      <ul>
          <li>Supports all Telegram Bot API methods</li>
          <li>Handles both GET and POST requests</li>
          <li>Full CORS support for browser-based applications</li>
          <li>Transparent proxying of responses</li>
          <li>Maintains original status codes and headers</li>
      </ul>

      <div class="note">
          <strong>Note:</strong> This proxy does not store or modify your bot tokens. All requests are forwarded directly to Telegram's API servers.
      </div>

      <h2>Example Code</h2>
      <div class="code">
  // JavaScript Example
  fetch('https://{YOUR_WORKER_URL}/bot{YOUR_BOT_TOKEN}/sendMessage', {
      method: 'POST',
      headers: {
          'Content-Type': 'application/json',
      },
      body: JSON.stringify({
          chat_id: "123456789",
          text: "Hello from Telegram Bot API Proxy!"
      })
  })
  .then(response => response.json())
  .then(data => console.log(data));
      </div>
  </body>
  </html>`;

  async function handleRequest(request) {
    const url = new URL(request.url);

    if (url.pathname === '/' || url.pathname === '') {
      return new Response(DOC_HTML, {
        headers: {
          'Content-Type': 'text/html;charset=UTF-8',
          'Cache-Control': 'public, max-age=3600',
        },
      });
    }

    // Extract the bot token and method from the URL path
    // Expected format: /bot{token}/{method}
    const pathParts = url.pathname.split('/').filter(Boolean);
    if (pathParts.length < 2 || !pathParts[0].startsWith('bot')) {
      return new Response('Invalid bot request format', { status: 400 });
    }

    // Reconstruct the Telegram API URL
    const telegramUrl = `${TELEGRAM_API_BASE}${url.pathname}${url.search}`;

    // Create headers for the new request
    const headers = new Headers(request.headers);
    
    // Forward the request to Telegram API
    let body = undefined;
    if (request.method !== 'GET' && request.method !== 'HEAD') {
      try {
        body = await request.arrayBuffer();
      } catch (err) {
        return new Response(`Failed to read request body: ${err.message}`, { status: 400 });
      }
    }

    const proxyReq = new Request(telegramUrl, {
      method: request.method,
      headers: request.headers,
      body,
      redirect: 'follow',
    });

    try {
      const tgRes = await fetch(proxyReq);
      const res = new Response(tgRes.body, tgRes); // Copy response as-is
      res.headers.set('Access-Control-Allow-Origin', '*');
      res.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
      res.headers.set('Access-Control-Allow-Headers', 'Content-Type');
      return res;
    } catch (err) {
      return new Response(`Error proxying request: ${err.message}`, { status: 500 });
    }
  }
  // Handle OPTIONS requests for CORS
  function handleOptions(request) {
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
      'Access-Control-Max-Age': '86400',
    };

    return new Response(null, {
      status: 204,
      headers: corsHeaders,
    });
  }

  // Main event listener for the worker
  addEventListener('fetch', event => {
    const request = event.request;
    
    // Handle CORS preflight requests
    if (request.method === 'OPTIONS') {
      event.respondWith(handleOptions(request));
    } else {
      event.respondWith(handleRequest(request));
    }
  ```