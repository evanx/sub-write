# sub-write

A microservice to subscribe to a Redis pubsub channel, and print messages to the console.

The essence of the implementation is as follows:
```javascript
async function startProduction() {
    sub.on('message', (channel, message) => {
        if (process.env.formatter === 'jsome') {
            jsome(JSON.parse(message));
        } else if (process.env.formatter === 'prettyjson') {
            console.log(prettyjson.render(JSON.parse(message)));
        } else if (process.env.jsonIndent > 0) {
            console.log(JSON.stringify(JSON.parse(message), null, parseInt(process.env.jsonIndent)));
        } else {
            console.log(message);
        }
    });
    sub.subscribe(config.subscribeChannel);
}
```
where `config` is populated from environment variables as follows:
```javascript
const config = ['subscribeChannel'].reduce((config, key) => {
    assert(process.env[key], key);
    config[key] = process.env[key];    
    return config;
}, {});
```

For example the following command line runs this service to subscribe to channel `logger:mylogger` and log messages.
```shell
formatter=jsome subscribeChannel=logger:mylogger npm run development
```
where the `jsome` JSON colorizer/formatter is specified.


## Sample use case

This service is intended for a personal requirement to subscribe to logging messages published via Redis.
These are arrays published via pubsub.
```
redis-cli publish 'logger:mylogger' '["info", {"name": "evanx"}]'
```
where we might subscribe in the terminal as follows:
```
redis-cli psubscribe 'logger:*'
```
where we see the messages in the console as follows:
```
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "logger:*"
3) (integer) 1
1) "pmessage"
2) "logger:*"
3) "logger:mylogger"
4) "[\"info\", {\"name\": \"evanx\"}]"
```
However we want to pipe to a command-line JSON formatter to enjoy a more readable rendering:
```json
[
  "info",
  {
    "name": "evanx"
  }
]
```

We found that `redis-cli psubscribe` didn't suit that use case. So we wish to use this service:
```shell
 npm start | jq '.'
```

## Related code

Incidently, some sample Node code for a client logger that publishes via Redis:
```javascript
const createRedisLogger = (client, loggerName) =>
['debug', 'info', 'warn', 'error'].reduce((logger, level) => {
    logger[level] = function() {
        if (!client || client.ended === true) { // Redis client ended
        } else if (level === 'debug' && process.env.NODE_ENV === 'production') {
        } else {
            const array = [].slice.call(arguments);
            const messageJson = JSON.stringify([
                level,
                ...array.map(item => {
                    if (lodash.isError(item)) {
                        return item.stack.split('\n').slice(0, 5);
                    } else {
                        return item;
                    }
                })
            ]);
            client.publish(['logger', loggerName].join(':'), messageJson);
        }
    };
    return logger;
}, {});
```
where logged errors are specially handled i.e. a slice of the `stack` is logged e.g.:
```json
[
  "error",
  [
    "ReferenceError: queue is not defined",
    "    at /home/evans/phantomjs-redis/build/index.js:57:59",
    "    at Generator.next (<anonymous>)",
    "    at step (/home/evans/phantomjs-redis/build/index.js:119:191)",
    "    at /home/evans/phantomjs-redis/build/index.js:119:437"
  ]
]
```
where the first item `"error"` is the logger `level` which indicates this was logged via `logger.error()`


## Related projects

We plan to publish microservices that similarly subscribe, but with purpose-built rendering for logging messages e.g. error messages coloured red.

Watch
- https://github.com/evanx/sublog-console
- https://github.com/evanx/sublog-web
