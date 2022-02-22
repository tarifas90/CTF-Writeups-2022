## DOJO 15 Writeup

YesWeHack is a European BugBounty platf
that regularly releases CTF style challenges known as DOJOs. The point of each challenge is to exploit a specific vulnerability such as XSS, SQLi etc.

During DOJO 15, the goal was to achieve XSS by bypassing a list of filters that aimed to prevent injection of such content.

The filtering implemented can be viewed below.

![XSS-Filters](https://user-images.githubusercontent.com/55701068/155088577-245c9d57-8e4c-4e4b-bebe-249f3d9c42ba.png)

Additionally, the HTML of the page we inject into is the following

```html
<html>
<body>
<h1><span style='color:red;'>/</span> The Amazing Blog</h1>
<pre>
  Just sharing a cool photo at my new desktop at the YesWeHack office!
  <img style="width:340px;height:160px" src="https://i.ibb.co/qmMYrjZ/Office-YWH.png" alt="$inject">
</pre>

<script>
   // Also did you knew that HTML execute <script> tags first and then execute all other onload statments after? ;)
  
  var settings = {
    user: ""
  }
  settings.user = "Guest";
  Firewall = "SuperDuperSecureMode";   
</script>
```

The injection point is `$inject` in the `alt` atribute within an image tag (line 6).
The steps below describe the challenges we had to overcome

1. **Break out from the attribute**

To be able to execute JavaScript we had to breakout from the attribute we were injecting in. The WAF ruleset however would escape the `"` character with a backslash. A way to overcome this is to add an additional backslash in an attempt to comment out the backslash added by the WAF. However in this case that was escaped also.
What appeared to work was to URL encode the injected backslash, which once commented out would not be escaped by the WAF. Therefore the first part of our payload was
**Part 1:** `%5c"`

2. **Event Handler**

In order to trigger XSS it would be useful to inject an event handler. However once again the WAF would attempt to prevent that. The `on.*=rule` exists meaning that any word starting with the word `on` and followed by a `=` sign would be removed. In this case we took advantage of another ruleset he had created. The WAF would remove the `%` symbol wherever detected. Removing content can be tricky and can allow to bypass certain protections. What was done was to inject the `%` between the letters `o` and `n` and create the desired event handler, since once the `%` was removed we would end up having the `on` word created and if no recursive checks were made we could bypass the WAF check.
```
Injection: o%n 
After filtering: on
```
Additionally spaces were also blacklisted, this is something that can be easily bypassed by adding comments or slaces `/`.
Using the above 2 pieces of informtion we can craft the following

**Part 2:** `%2fo%25nload=`

3. **Settings user**

Since we now have a way to potentially execute JavaScript, we look back to the first challenge goal. This required from us to change the `settings user` attribute to a value of our choosing. However we had a few filters to bypass once again
```
. (Dot is blacklisted)
' (Single quote is escaped)
" (Double quote is escaped)
```
However, objects in JavaScript (e.g. settings.user) can be represented also as `settings["user"]` which overcomes the issue of using the dot.
Furthermore instead of quotes we can use backticks. Which leads to the 3rd part of the payload

**Part 3:** settings[\`user\`]=\`Admin\`

4. **Poping an Alert**

Last part of the challenge was to pop up an alert with the new user's settings value. However alert, prompt and confirm were blacklisted. Similarly parenthesis were also blocked. This makes us think of a method that can deal with both of those problems (base64 encoding).
We therefore construct the following payload which will try to redirect the user via `location`, decode the base64 value (atob) with uses a javacript: protocol and execute our payload. We should keep in mind that `location` also will trigger the `on.*` check, therefore we need to use once again the trick from Step 2

**Part 4:** locatio%25n=atob\`amF2YXNjcmlwdDphbGVydChzZXR0aW5ncy51c2VyKQ==\`


5. **Comment out the end and get XSS**

The only last part we needed was to comment out the remaining `"`.
This can done by adding two slashes `//` URL encoded as mentioned before

**Part 5:** `%2f%2f`

Combining all the parts from above we can craft the final payload

```
%5c"%2fo%25nload=settings[`user`]=`Admin`;locatio%25n=atob`amF2YXNjcmlwdDphbGVydChzZXR0aW5ncy51c2VyKQ==`%2f%2f
```

![5414d29e-7269-4914-a9d4-acfbead28461-payload-xss](https://user-images.githubusercontent.com/55701068/155089989-bc80065f-033e-444c-8a9b-26d5cab59118.png)
