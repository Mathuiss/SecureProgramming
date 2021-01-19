# Les 2
## Order Status
```python
@app.route('/trackorder', methods=['GET'])
def trackorder():
    if request.method == 'GET':
        r = random.randrange(1,20)
        return render_template('trackorder.html',
            orderid = request.args.get('orderid'),
            orderstatus = orderstatus[random.randrange(0,len(orderstatus))])
    else:
        return render_template('trackorder.html',
            orderid = None,
            orderstatus = orderstatus[random.randrange(0,len(orderstatus))])
```
```html
{% if orderid is not none %}
<br>Status of order with orderid: <b>{{orderid}}</b> is <b>{{orderstatus}}</b><br><br>
<a href=/trackorder?orderid={{orderid}}>Check again</a>
{% endif %}
```
### Probleem
Wanneer de ```<a>``` tag word gerendert, wordt de input niet gecontroleerd. De server moet rekenen op client side verificatie. Eigenlijk zou deze verificatie op de server zelf moeten plaatsvinden voordat er iets in de response word geplakt.

### Bewijs
```
http GET localhost:5000/trackorder orderid=="h onclick=alert(1)"
```
Response:
```html
<a href=/trackorder?orderid=h onclick=alert(1)>Check again</a>
```
### Oplossing
Als de server het orderid controleert voordat de server deze rendert zou dit het probleem oplossen:

```python
if request.method == 'GET':
        r = random.randrange(1,20)
        orderid = request.args.get('orderid')

        if orderid == None:
            # Render exception

        if not bool(re.match(r"order[0-9]{5}", orderid)):
            # Render exception

        return render_template('trackorder.html',
            orderid=orderid,
            orderstatus=orderstatus[random.randrange(0,len(orderstatus))])
```

## Blog
```python
comments = [
    {'user': 'Calvin', 'comment':
        'My lawyers checked it carefully and concluded that Craig is Satoshi.'},
    {'user': 'Peter', 'comment': 'CSW is a fraud! See you in court!'}
]

@app.route('/blog', methods=['GET', 'POST'])
def blog():
    if request.method == 'GET':
        return render_template('blog.html', comments=comments)

    comment = request.form.get('comment')
    user = request.form.get('user')

    # remove all html tags from the comment and the user in order to prevent XSS
    comment = re.sub(r"\<[a-zA-Z][^\<\>]*>", '', comment)
    user = re.sub(r"\<[a-zA-Z][^\<\>]*>", '', user)

    comments.append({'user': user, 'comment': comment})

    return render_template('blog.html', comments=comments)
```
### Probleem
Er is een poging gedaan om de input te filteren door middel van een regular expression:
```python
comment = re.sub(r"\<[a-zA-Z][^\<\>]*>", '', comment)
```
Op deze manier worden tags zoals ```<script>``` vervangen door ```''```. De regex is echter voor de gek te houden door tags twee keer in te voeren: ```<<script>script>alert(1)</script>``` Op deze manier word het ```<script>``` gedeelte eruit gefilterd en blijft er ```<script>alert(1)</script>``` over.

### Bewijs
Bij het sturen van de ```POST``` request hoeft er alleen maar een script ingevoerd te worden met dubbele tags:
```
http -f POST localhost:5000/blog user=test2 comment="<<script>script>alert(1)</script>"
```
Op de pagina werd daarna het volgende gerenderd:
```html
<p><i>test2</i> wrote: <br>
<script>alert(1)</script></p>
```

### Oplossing
In plaats van het verwijderen van tags zou de applicatie een ```Exception``` moeten gooien als er tags in de input zitten. Op deze manier komt de foute input niet verder de applicatie in:
```python
if bool(re.search(r"\<[a-zA-Z][^\<\>]*>", input_variable)):
    raise Exception("[Some error message]")
```

## User Profile
```javascript
function store() {
	if (typeof(Storage) !== "undefined") {
	    document.getElementById("result").innerHTML = localStorage.setItem("name",
            document.getElementById("name").value);
	}
}

if (typeof(Storage) !== "undefined") {
	document.getElementById("result").innerHTML = "Hi " + localStorage.getItem("name");
}
```
### Probleem
Het probleem bevindt zich hier in het feit dat de invoer van het tekstvak direct in het DOM wordt geplakt zonder te valideren of er een geldige invoer is. De applicatie gaat er vanuit dat de gebruiker een naam invoert die vervolgens in het ```<span></span>``` terecht komt. De gebruiker kan echter HTML invoeren om het DOM te panipuleren.

### Bewijs

Web Pagina:
```html
<span id="result">
</span>
```

Invoer:
```html
</span>
    <button type="button" onclick="alert(1)">ClickMe!</button>
<span>
```

Resultaat:
```html
<span id="result">
</span>
    <button type="button" onclick="alert(1)">ClickMe!</button>
<span>
</span>
```

### Oplossing
Een oplossing zou kunnen zijn door alle ```<``` en ```>``` tekens te vervangen door ```&lt;``` en ```&gt;```. Op deze manier ziet de gebruiker nogsteeds de tekens verschijnen op het scherm maar blijft de invoer binnen het ```<span>```.

## Timer
```html
<h1>Timer active</h1>
    <p>
      Wait for the timer to finish.
    </p>
    <img src="/static/loading.gif" onload="startTimer('{{ timer }}');" />
    <br>
    <div id="message">The timer will execute in {{ timer }} seconds.</div>
</body>
```
### Probleem
Ik herken deze opdracht van de Google XSS course.

Op het eerste gezicht is hier niks mis mee. De web server filtert namelijk de tags uit de ```{{ timer }}```. Dat betekent dat het geen zin heeft om tags in te voeren in ```timer```
```html
<div id="message">The timer will execute in {{ timer }} seconds.</div>
```
Het probleem ontstaat in het ```onload``` event van de ```<img>```. De web server rendert namelijk het volgende:
```html
<img src="/static/loading.gif" onload="startTimer('{{ timer }}');" />
```
In het ```onload``` event kan code worden geïnjecteerd die buiten de functie ```startTimer()``` zal worden uitgevoerd.

### Bewijs
```
http GET localhost:5000/timer timer=="}}'); alert(1); //"
```

Resultaat:
```html
<img src="/static/loading.gif" onload="startTimer('}}&#39;); alert(1); //');" />
```

De browser voert dit uit en laat ook daadwerkelijk de alert zien.
### Oplossing
Voordat de web server de pagina rendert zou de server moeten controleren of de invoer wel een getal is. Dit gebeurt momenteer op de client maar zoals aangetoond heeft dit geen zin. Als de invoer is geparsed zou er ook een minimum en maximum op kunnen zitten zodat alle mogelijkheden zijn afgedekt.

```python
@app.route('/timer', methods=['GET'])
def timer():
    timer = request.args.get('timer')
    
    if not timer.isnumeric(timer):
        # Render exception 

    timer = int(timer)

    if timer < 0:
        # render exception

    if timer:
        return render_template('timer.html', timer = timer )
    else:
        return render_template('settimer.html')
```