# DEF CAMP CTF Final — Cutitari

**Category:** Web  
**Vulnerability:** Race Condition  
**Technique:** Asynchronous execution / Concurrent requests

## 1. Initial reconnaissance

After downloading the provided files, I decided to interact with the web application first, without inspecting the source code.

![Web application](images/01.png)

We are presented with six buttons:

- `Reserve knife`
- `Safety inspection`
- `Sharpen`
- `Arena inspection`
- `Temper`
- `Request certificate`

I clicked through all the available actions to understand how the application works. However, requesting the certificate did not give me the flag.

![Certificate request failed](images/02.png)

Alright, time to inspect the source code.

## 2. Understanding the requirements

First, I searched for the `/certificate` endpoint to understand what conditions must be satisfied to receive the flag.

```javascript
if (req.method === 'GET' && path === '/certificate') {
    if (blade?.revision === 2 && blade.edge === 'blunt' &&
        blade.marks.safety?.revision === 2 &&
        blade.marks.arena?.revision === 2) {
      return reply(res, 200, { flag });
    }
    return reply(res, 403, {
      error: 'The guild marks do not match the finished knife'
    });
}
```

The conditions are pretty straightforward:

- `blade.revision === 2`
- `blade.edge === 'blunt'`
- `blade.marks.safety.revision === 2`
- `blade.marks.arena.revision === 2`

So, we need a knife with revision 2, a blunt edge, and both inspection marks stamped with revision 2.

But there is a problem.

After using the buttons in the default order, I noticed that the Safety mark remained at revision 0, while Arena ended up at revision 1.

Something was clearly not working as expected.

## 3. Investigating the inspection logic

I started looking for the code responsible for generating the Safety and Arena marks.

I found this piece of code:

```javascript
let seal = await engrave(blade.serial, mark);

if (mark === 'safety') {
    blade.proof = seal;

    res.write(
        `event: proof\ndata: ${JSON.stringify({ proof: seal })}\n\n`
    );

    seal = await engrave(
        blade.serial,
        'safety:finish',
        6000000
    );
}

blade.marks[mark] = {
    revision: blade.revision,
    seal
};

res.write(
    `event: stamped\ndata: ${JSON.stringify({
        mark,
        revision: blade.revision,
        seal
    })}\n\n`
);
```

Both inspections use the same logic. The `mark` variable determines which inspection is currently being performed.

But where does `mark` come from?

A little higher in the source code:

```javascript
if (
    req.method === 'POST' &&
    (path === '/inspect/safety' || path === '/inspect/arena')
) {
    const mark = path.slice('/inspect/'.length);
```

It's simply extracted from the URL.

For example:

- `/inspect/safety` → `safety`
- `/inspect/arena` → `arena`

At this point, I didn't see any obvious bypass. The interesting part was the `seal` variable and the function responsible for generating it.

## 4. Finding the asynchronous bottleneck

Searching further through the source code, I found the `engrave()` function:

```javascript
function engrave(serial, mark, rounds = 3000000) {
    return new Promise((resolve, reject) => {
        pbkdf2(
            `${serial}:${mark}`,
            secret,
            rounds,
            32,
            'sha256',
            (err, seal) =>
                err
                    ? reject(err)
                    : resolve(seal.toString('hex'))
        );
    });
}
```

This function uses `pbkdf2()` to perform a computationally expensive cryptographic operation.

By default, it performs 3,000,000 iterations.

The important part here is that Node.js provides an asynchronous version of `pbkdf2()`.

Initially, I was confused about this. Node.js is single-threaded, so how could another request be processed while such an expensive computation was running?

After spending around 20 minutes reading about Node.js asynchronous execution, I finally understood what was happening.

**The asynchronous `pbkdf2()` operation does not block the event loop.**

The expensive computation is handled by the underlying worker pool, while Node.js can continue processing other incoming requests.

And what about `await`?

```javascript
let seal = await engrave(blade.serial, mark);
```

`await` suspends the execution of the current asynchronous handler until the Promise resolves. It does not freeze the entire Node.js event loop.

This means that while one inspection is waiting for its cryptographic computation to finish, other HTTP requests can still be processed.

That's where things get interesting.

## 5. The actual race condition

Remember the special condition inside the Safety inspection?

```javascript
if (mark === 'safety') {
    blade.proof = seal;

    res.write(
        `event: proof\ndata: ${JSON.stringify({ proof: seal })}\n\n`
    );

    seal = await engrave(
        blade.serial,
        'safety:finish',
        6000000
    );
}
```

Unlike Arena, Safety performs two separate cryptographic operations.

The first one uses the default 3,000,000 iterations.

The second one uses 6,000,000 iterations.

This gives us an especially interesting execution window.

While Safety is waiting for the second cryptographic operation, we can manually send other requests:

- Sharpen
- Arena inspection
- Temper

The key detail is that the inspection mark receives its revision from the current blade revision:

```javascript
blade.marks[mark] = {
    revision: blade.revision,
    seal
};
```

In other words, the revision is read when the asynchronous operation finishes, not when the inspection initially starts.

Meanwhile, Sharpen and Temper can modify the blade revision.

This creates a race condition: the final state depends on the order in which the asynchronous operations complete and the other requests modify the shared blade state.

## 6. Exploitation

Now we know what we need:

1. The knife must have revision 2.
2. Safety must be stamped while the blade is at revision 2.
3. Arena must also be stamped while the blade is at revision 2.
4. The edge must remain blunt.

Sharpen and Temper are responsible for changing the blade revision.

So instead of following the normal button sequence, we can take advantage of the time during which the inspections are performing their cryptographic computations.

My approach was completely manual.

I reserved a knife and then quickly pressed the following buttons:

1. `Safety inspection`
2. `Sharpen`
3. `Arena inspection`
4. `Temper`

The idea was to perform the revision-changing operations while the inspections were still waiting for their cryptographic computations.

I could also use the additional execution time provided by Safety's second PBKDF2 operation.

After that, I requested the certificate.

It took me approximately three attempts to get the timing right.

## 7. Getting the flag

![Flag obtained](images/03.png)

And there it is.

**Flag obtained!**

A simple-looking web application turned out to be vulnerable to a race condition caused by asynchronous operations and shared mutable state.

The most interesting part of this challenge wasn't even the cryptographic function itself, but understanding how Node.js handles asynchronous execution and how that behavior can introduce race conditions.

---

**DEF CAMP CTF Final — Cutitari**  
*Race conditions are sometimes just a matter of timing.*
