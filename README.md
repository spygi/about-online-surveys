## Context
So, a friend was participating in a photography competition. You know those where people can vote and "have only one vote".
I wanted to check if this is really the case, if yes how they do it and if not how can I make her win :). This is how I went about it.

### First things I tried:
- disabling the cache and reloading in Chrome (Cmd+Shift+R): didn't work, they must store something locally.
- opening different browsers: worked but it takes some effort to install/uninstall 200 browsers (see was behind by ~200 votes already :P)
- opening incognito browser: worked and it's easier but still manual work.
- hm, I didn't try the "Don't track me" feat (but probably is the same as incognito).

### Enter developer mode
Opening the developer console, I first checked if I could programmatically "click" to vote. ```$('.Option-5').click()``` did the trick [x].
It looks like the survey is rendered in an iframe with React. Well React is my guess from the id and class names, I didn't see in the resources React being loaded, probably they bundled it with their (minified/uglified) JS?

Anyway, clicking the element does a GET to a server with a lot of query params. Some of them look "suspicious" like ```eid```, or ```duid```.

Naive attempt, CURL it as it was -> got some jibberish back and no vote was recorded. Fair enough, I had just voted with the same request, it would be dump to just let it vote again.
Tried to see if I could find in local/session/cookie storage any of those "weird" query params but at first sight I didn't spot them.

I wanted to check which of those parts of the request are unique per request (and potentially if I could generate them) and which stay the same.
With some help from ```vimdiff```, I noticed the parameters that changed across the 2 requests were 6. No cookies were set and the requests got a 200 and no response:
- ```dtm```: timestamp, easy
- ```eid```: that looked like ```ee9e09c1-ac41-4225-ba61-04ebb928d106```, scary.
- ```duid```: similar
- ```nuid```: similar
- ```sid```: similar
- ```cx```: this one started with the same characters ```W3sic2NoZW1hIjoiY29tLmJvb21ib3gucGxhdGZvcm0vdWlkL2pzb25zY2hlbWEvMS0wLTAiLCJkYXRhIjp7InVpZCI6In...``` and at some point had some randomness.

OK, this doesn't look promising I thought, they could compute crypto stuff based on the timestamp and the client and whatever in their obfuscated JS and then send that to the server. What if I de-obfuscate their JS (there are tools I think)?
Still the server can keep state of the request and check against (some of) the parameters sent along if they match the value it has..
Looking a bit more closely in the local/session/cookie storage I was able to spot 3 out of the 5 scary query params but still was missing 2 and had not idea how to compute them.

### Enter Browserstack
I decided to go with the more difficult scenario of the server checking against the stuff it gets. This means you need to actually open the site somehow (programmatically) and click from within.
Enter Browserstack, a tool intended for automated testing of sites.

I had almost no idea how to use it, so [copy-pasta](https://www.browserstack.com/automate/python) time!
```driver = webdriver.Remote(...)``` actually this is cool, the test is running on their servers so if the survey tool records IPs they will see the IPs of browserstack :) Oh crap, it has an identifier to my account :P Anyway..

```driver.find_element_by_class_name``` didn't work, NoSuchElementException.. WTF?
OK, maybe it hasn't loaded yet (I had noticed in the site it does take some time to load the pics). I tried both explicit and implicit waits.. Nada.

If only I could see what the test sees... ```driver.save_screenshot('screenshot.png')``` BAOUM.
![Screenshot from IE7 on XP](./IEscreenshot.png)

Why does it look like shiaaa... Oh, ```desired_cap = {'os': 'Windows', 'os_version': 'xp', 'browser': 'IE', 'browser_version': '7.0' }``` that's why! Middle finger to you too sample code.

Changing to Chrome, the screenshot looks normal now but the element I want to click is not visible, you have to scroll down.. is Browserstack that stupid? Anyway, ```driver.execute_script("window.scrollTo(0, 1110);")``` (yes, you can run JS from the driver - awesome!) Btw, Browserstack is not that stupid, I was :)

Still nothing, fuck! I see it, it's there! Click it god damn it!

Then it dawned on me, the fricking iframe! ```driver.switch_to_frame(driver.find_element_by_class_name("widget-iframe"))``` Still nothing!

I was about to give up when I noticed that in the screenshots other pics (competitors!) were highlighted (-> voted) randomly. Wuuut? My selector was correct. Tried changing the selector randomly but I was always voting other pics and not the same one either, GKRRR!
Hm, I thought it maybe has to do with the country the test runs from (the location of the Browserstack server). Enter Tunnelbear and VOILA, the page renders differently from US, compared to DE or CH. The exact reason why is not clear to me, could be a React thingy as well.

OK, let's switch to local Chromedriver (make sure to install it on your PATH) with ```webdriver.Chrome()```. Yeah, my IP is doing the request but they could trace me back from Browserstack's identifier anyway.
BADOOM, works! But still inconsistently: ```WebDriverWait(driver, 5).until(EC.element_to_be_clickable((By.CLASS_NAME, "Option-5")))``` did the trick!

## Aftermath
I ran the script sometimes more..hey, in the process I had voted some other pics too, okay? But I didn't let it run 200 times so my friend didn't win the contest. Sorry for having values here!

My votes were recorded normally. I am kinda surprised this worked. I would expect the tool to have some basic "anomaly detection" mechanism: my "votes" were coming from the same IP with the same "browser" with a difference of some seconds. I am not sure if Browserstack even sends some User Agent or other header along that they could use to filter out automated votes. Maybe police comes knocking on my door soon..?
More sophisticated solutions could be: setting it up to run distributed with random intervals and random browsers so it would be even harder to filter those votes out.

In any case.. peoples, don't trust online voting without a registration of sorts. I go restart my router now..
