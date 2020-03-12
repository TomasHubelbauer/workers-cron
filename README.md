# CloudFlare Workers Cron

I am prototyping a way to do periodical scheduling in CloudFlare Workers.

The idea is to use `waitUntil` and postpone calling self again.
This will result in an infinite loop with a delay time step.
Each invocation from the `waitUntil` can be marked, compared to the last
and scheduling can be implemented on top of that.

The limits concerns and pricing concerns are cleared as per the following:

- The delay limits observed so far:
  - 30- seconds works reliably - never failed to call
  - 40+ seconds does not work - never succeeded in calling
  - This may be subject to the script run time (budget per script run not limit on `waitUntil` run)
- CloudFlare workers is paid:
  - Per 1 M hits - .50 USD
  - By 5 USD a month minimum - 10 M hits
- A month of hits spaced out by:
  - 1 second - 2592000 hits - 25.92 % of 10 M
  - 10 seconds - 259200 hits - 2.52 % of 10 M
  - 20 seconds - 129600 hits - 1.29 % of 10 M
  - 30 seconds - 86400 hits - 0.87 % of 10 M
- Ergo:
  - The cost of running a worker as a cron is negligible
  - The strain on the Workers infrastructure by doing this is negligible

This worker is used to test which delays work and which don't:

```javascript
addEventListener('fetch', event => event.respondWith(handleRequest(event)));

const delay = 40 * 1000;
const expirationDelta = 1000;

async function handleRequest(event) {
  try {
    const url = new URL(event.request.url);

    // Ensure a valid array in case of an empty KV
    let hitsValue = await KV.get('hits');
    if (!hitsValue) {
      hitsValue = '[]';
      await KV.put('hits', hitsValue);
    }

    let hits = JSON.parse(hitsValue);

    // Mark unmarked expired hits
    for (const hit of hits) {
      const age = new Date().getTime() - new  Date(hit.stamp).getTime();

      if (hit.mark === 'pending' && (age > hit.delay + expirationDelta)) {
        hit.mark = 'expired';
      }
    }

    // Insert a new hit
    if (url.pathname === '/') {
      const hit = { hit: hits.length, delay, stamp: new Date().toISOString(), mark: 'pending' };
      hits.unshift(hit);

      // Dispatch the fire and forget procedure to mark the hit
      event.waitUntil(issueMarkHitRequest(hit));

      // Test this works by uncommenting the following lines:
      //const text = await issueMarkHitRequest(hit);
      //hits = JSON.parse(await KV.get('hits'));

      await KV.put('hits', JSON.stringify(hits));
      return new Response(JSON.stringify(hits, null, 2));
    }

    // Mark an existing hit
    let hit = Number(url.pathname.slice('/'.length));
    if (hit < 0 || hit > hits.length) {
      throw new Error(`Invalid hit number ${hit}.`);
    }

    hit = hits.find(h => h.hit === hit);
    hit.mark = {};
    hit.mark.stamp = new Date().toISOString();
    hit.mark.delay = new Date().getTime() - new Date(hit.stamp).getTime();
    hit.mark.diff = hit.mark.delay - hit.delay;
    await KV.put('hits', JSON.stringify(hits));
    return new Response(JSON.stringify(hit, null, 2));
  }
  catch (error) {
    return new Response(error.message, { status: 500 });
  }
}

async function issueMarkHitRequest({ hit, delay }) {
  // Delay the execution
  await new Promise(resolve => setTimeout(resolve, delay));
  const response = await fetch('https://test.tomashubelbauer.workers.dev/' + hit);
  return response.text();
}
```

This worker _should_ be implementing the event loop, but doesn't work:

```javascript
addEventListener('fetch', event => event.respondWith(handleRequest(event)));

const delay = 5000;

async function handleRequest(event) {
  try {
    const isScheduled = event.request.headers.get('X-Scheduled-Call') === 'true';

    // Ensure a valid array in case of an empty KV
    let hitsValue = await KV.get('hits');
    if (!hitsValue) {
      hitsValue = '[]';
      await KV.put('hits', hitsValue);
    }

    const hits = JSON.parse(hitsValue);
    const hit = {
      stamp: new Date().toISOString(),
      mode: isScheduled ? 'scheduled' : 'direct'
    };

    hits.unshift(hit);
    await KV.put('hits', JSON.stringify(hits));

    const scheduledHit = new Date(await KV.get('scheduled-hit'));
    const brokenLoop = (new Date().getTime() - scheduledHit.getTime()) > delay + 1000;

    // Schedule a follow-up call if this call is scheduled itself
    // Schedule a follow-up call if the loop broke (last call older than delay + epsilon)
    if (isScheduled || brokenLoop) {
      // Dispatch the fire and forget scheduled call
      event.waitUntil(reschedule());
    }

    if (isScheduled) {
      // Stop execution of the rest of the script (which handles non-scheduled hits)
      return new Response(undefined);
    }

    return new Response(JSON.stringify({ count: hits.length, hits }, null, 2));
  }
  catch (error) {
    return new Response(error.message, { status: 500 });
  }
}

async function reschedule() {
  // Delay the execution
  await new Promise(resolve => setTimeout(resolve, delay));
  await fetch('https://test.tomashubelbauer.workers.dev', {
    headers: {
      'X-Scheduled-Call': 'true'
    }
  });
}
```

