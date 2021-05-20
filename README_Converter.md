# Web - README Converter

## Challenge description:

![image](https://user-images.githubusercontent.com/70543460/118820037-8fc0b400-b8be-11eb-87ad-6b394497cf0e.png)

## Solution:

The first thing I usually do in CTF web challenges is viewing the page source, because I might find interesting things, such as the developer comments, but in this challenge, nothing interesting was in the source of the index page.

![image](https://user-images.githubusercontent.com/70543460/118822715-0f4f8280-b8c1-11eb-8d72-ba8898b398ae.png)

<br>

According to the challenge description, this web application converts GitHub README files to HTML (from markdown to html).

<br>

As you can see, you need to provide a GitHub repository and the web application will convert the README from the main branch to HTML, and the input format is: **username/repository**, and you also have the ability to save the result.

![image](https://user-images.githubusercontent.com/70543460/118822108-89333c00-b8c0-11eb-9520-76abf962e7c7.png)

So, I created a temporary GitHub repository with a **README.md** file in the **main branch** to test the web application.

![image](https://user-images.githubusercontent.com/70543460/118826479-33609300-b8c4-11eb-9202-9f624e434a75.png)

<br>

![test](https://user-images.githubusercontent.com/70543460/118828641-1d53d200-b8c6-11eb-869d-d838c655cba6.gif)

**Note: The result is saved because I've checked the "Save result" option.**
![image](https://user-images.githubusercontent.com/70543460/118829733-095ca000-b8c7-11eb-8e51-881998848b35.png)

After that, I opened the saved result:

![image](https://user-images.githubusercontent.com/70543460/118830496-a9b2c480-b8c7-11eb-9f6b-910f2b3280bb.png)

Then I viewed the source and there was an interesting comment:

![image](https://user-images.githubusercontent.com/70543460/118831508-83415900-b8c8-11eb-86bb-dc9717b472d7.png)

**Note: I kept that comment as a note until I find more interesting things to connect them together**

<br>

... The first thing I thought of when I focused on the saved result url is **Local File Inclusion (LFI)**

![LFI](https://user-images.githubusercontent.com/70543460/118840595-382b4400-b8d0-11eb-804e-30c4dcbef9af.jpg)

I tried a lot of path traversal techinques (because I wanted to read any file outside the current directory), but none of them worked because the forward slash is filtered:

![image](https://user-images.githubusercontent.com/70543460/118874740-e8f70a80-b8f3-11eb-8cc6-f568d4a7d122.png)

Then I realized that I should have read **view_render.php** before trying moving to files outside the current directory:

```view-source:http://web.ctf.ae:9806/renders/view_render.php?file=view_render.php```

![view_render](https://user-images.githubusercontent.com/70543460/118843562-d9b39500-b8d2-11eb-86f3-22061a3c8bde.png)

<br>

According to the source code of **view_render.php**:
1. As I noticed while testing path traversal payloads, the forward slash is replaced with nothing: ```str_replace('/', '', urldecode($_GET['file']));```
2. The variable **$WEBROOT** is replaced with its value: **/var/www/html/**, and that can be used to read files in that directory (/var/www/html/). (Note: This is related to the comment that says ```<!-- We should really look into removing the ability to replace environment variables with their value...do we need it in this code? -->```

<br>

After that, I was able to read **index.php**, and that was a benefit of replacing **$WEBROOT** with its value **/var/www/html/** as I saw in the source code of **view_render.php**:

```view-source:http://web.ctf.ae:9806/renders/view_render.php?file=$WEBROOTindex.php```

![index](https://user-images.githubusercontent.com/70543460/118878937-cddac980-b8f8-11eb-9333-7aa2f36a5fab.png)

<br>

The comment in **index.php** seems very important because it leads to a new file called **check_webpage.php** that runs **eval()** on the content of the pages!

```Note: The eval() function evaluates a string as PHP code.```

<br>

So, I was able to read **check_webpage.php** using the LFI bug as I did when I read **view_render.php** and **index.php**:

```view-source:http://web.ctf.ae:9806/renders/view_render.php?file=$WEBROOTcheck_webpage.php```

![check_webpage](https://user-images.githubusercontent.com/70543460/118988294-be579100-b989-11eb-9aea-44e2b3425720.png)

The idea of the exploit became clear after I read the source code of check_webpage.php, which is writing a PHP exploit in a README file, and that PHP exploit gives me the content of the binary file that gives me the flag...

<br>

But before writing the exploit, I realizied that I forgot something was mentioned in the challenge description, which is:
``` Note: Directory bruteforcing is allowed in this challenge, please only use "common.txt"```

So, I used **gobuster** to brute-force directories/files with the wordlist **common.txt**:

![gobuster](https://user-images.githubusercontent.com/70543460/118989368-a7656e80-b98a-11eb-9a9b-569069491a8f.png)

<br>

The most interesting results were: **/env** and **/php.ini**, so I checked them:

<br>

```http://web.ctf.ae:9806/env```

![env](https://user-images.githubusercontent.com/70543460/118990310-6de13300-b98b-11eb-8f30-0399f45e37b5.png)

**env** contains the value of the environment variables, and I already know the value of the environment variable **WEBROOT**.

<br>

```http://web.ctf.ae:9806/php.ini```

![ini](https://user-images.githubusercontent.com/70543460/118991419-666e5980-b98c-11eb-89ec-4b22a569fc9e.png)

The PHP configuration file **php.ini** contains the disabled functions.

<br>

So, after checking the disabled functions, I can write the exploit using functions that are not disabled...

Basically, I need the content of the binary file **/get_flag**, and it contains non-printable characters as it is a binary file, so I need to encode the content too, so the final exploit was:

![exploit](https://user-images.githubusercontent.com/70543460/118993678-63746880-b98e-11eb-9ba2-d6000a54905a.png)

<br>

![image](https://user-images.githubusercontent.com/70543460/118994224-d54cb200-b98e-11eb-86d5-fe1475e0fa5d.png)

<br>

![image](https://user-images.githubusercontent.com/70543460/118994450-04632380-b98f-11eb-81fa-ee48a30bd869.png)

<br>

![image](https://user-images.githubusercontent.com/70543460/118994504-11801280-b98f-11eb-8e0f-4caccfe02ff9.png)

<br>

```
//
├─ var/
│  ├─ www/
│  │  ├─ html/
│  │  │  ├─ index.php
│  │  │  ├─ check_webpage.php
│  │  │  ├─ renders/
│  │  │  │  ├─ view_render.php
│  │  │  │  ├─ 4f504ccbb8c22e33e92692080c0d2cbf
├─ get_flag
```

After saving the exploit, I need to pass it as a paramater to **check_webpage.php** because it runs eval() on the content:

```
view-source:http://web.ctf.ae:9806/check_webpage.php?page=renders/4f504ccbb8c22e33e92692080c0d2cbf
```

![final](https://user-images.githubusercontent.com/70543460/118997154-1ba31080-b991-11eb-8f6b-fa42bb40f4c9.png)

![base64](https://user-images.githubusercontent.com/70543460/118997455-586f0780-b991-11eb-97a1-1e5bbb82f280.png)

<br>

The final step and the easiest is decoding the encoded binary file then running it:

![flag](https://user-images.githubusercontent.com/70543460/118998435-1f836280-b992-11eb-8e2b-4f396b098d23.png)

<br>

**CTFAE{ThatWasQuiteTheMaze}**
