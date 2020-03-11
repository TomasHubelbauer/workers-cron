# CloudFlare Workers Cron

I am looking for a quick and dirty way to get a cron-like thing going on CloudFlare
Workers. So far I've attempted to use `waitUntil` to carry out further execution
after the response has been sent and in it, execute self so that the worker keeps
calling itself in a never-ending cycle and in effect behaves like a scheduled job.

This does not work, the worker doesn't seem to ever call itself.

None of this is working:

```javascript
addEventListener('fetch', event => {
  //fetch('https://test.tomashubelbauer.workers.dev/');
  event.respondWith(handleRequest(event.request));
  //event.waitUntil(issueAnotherRequest());
  //event.waitUntil(fetch('https://test.tomashubelbauer.workers.dev/'));
});

async function handleRequest(request) {
  await KV.put('hits', Number(await KV.get('hits')) + 1);
  //fetch('https://test.tomashubelbauer.workers.dev');
  return new Response(await KV.get('hits'));
}

async function issueAnotherRequest() {
  //await new Promise(resolve => setTimeout(resolve), 1000);
  //await KV.put('hits', Number(await KV.get('hits')) +  .1);
  //await fetch('https://test.tomashubelbauer.workers.dev/');
}
```

## To-Do

### Try ping-pong calling two workers
