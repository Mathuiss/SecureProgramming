# Lession 1
## Guess
```python
@app.route('/guess', methods=['GET','POST'])
def guess():
    if request.method == 'GET':
        return render_template('guess.html', numtries = '0')

    time.sleep(2)
    
    
    r = random.randrange(1,20)
    number = request.form.get('number')
    if number == None:
        return render_template('guess.html',result = "Your input is ignored")
    else:
        number = int(number)
    if number in range(1,r):
        return render_template('guess.html',result = "the number is too low!")
    if number in range (r+1,20):
        return render_template('guess.html',result = "the number is too high!")
    else:
        return render_template('guess.html', result = "%d is correct!" % (number))
```
### Problem
De bovenstaande source code is van een endpoint waar een nummer naartoe word gepost. Het nummer zou tussen ```1``` en ```20``` moeten zijn. Allereerd wordt er gekeken naar de http methode. Als de methode ```GET``` is zal de guess.html pagina direct worden gerenderd. Als de methode ```POST``` is zal de applicatie eerst 2 milliseconden wachten om vervolgens een willekeurig getal tussen 1 en 20 aan te maken. Hierna word er een nieuw variabel aangemaakt: ```number = request.form.get('number')```. Als er geen number in de form is opgegeven zal de applicatie direct het resultaat ```result = "Your input is ignored"``` renderen. Als de applicatie wel een nummer vindt probeert de applicatie het nummer te casten naar een int: ```number = int(number)```. Als deze operatie mislukt crasht de applicatie en krijgt de gebruiker een ```500 Internal Server Error```.

```
http -f POST localhost:5000/guess number=patat
```

```html
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Length: 290
Content-Type: text/html; charset=utf-8
Date: Tue, 10 Nov 2020 14:23:18 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
```
Als de applicatie eriun slaagt het nummer te casten naar een int zal er worden gekaken of het nummer zich bevindt tussen ```1``` en het ```willekeurige getal```: ```if number in range(1,r):```. Als dit zo is returnt de server het resultaat: ```result = "the number is too low!"```. Als dit niet het geval is kijkt de server of het nummer zich tussen het ```willekeurige getal + 1``` en ```20```: ```number in range (r+1,20):```. Als dit het geval is laat de server zien dat het getal te hoog is. In alle andere gevallen zegt de server dat het getal correct is. Dit betekent dat er ```3``` manieren zijn waarop de gebruiker een correct getal kan behalen:
- ```number``` is gelijk aan het willekeurige getal
- ```number``` is kleiner dan 1
- ```number``` is groter dan 20

### Evidence
Als de gebruiker een nummer opstuurt wat kleiner is dan ```1``` genereert de server de volgende response:

```
http -f POST localhost:5000/guess number=-1
```

```html
HTTP/1.0 200 OK
Content-Length: 1207
Content-Type: text/html; charset=utf-8
Date: Tue, 10 Nov 2020 14:14:44 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
<body>
<h1>Guess a number</h1>
<p></p>
<form method=POST action=/guess>
  Guess a number<br>
  <select name=number><option>1<option>2<option>3<option>4<option>5<option>6<option>7<option>8<option>9<option>10<option>11<option>12<option>13<option>14<option>15<option>16<option>17<option>18<option>19</select>                
  <input type=submit value="Guess!">
</form>

<h2>-1 is correct!</h2>
```

Als de gebruiker een getal opstuurt wat groter is dan 20 genereert de server deze response:

```
http -f POST localhost:5000/guess number=21
```

```html
HTTP/1.0 200 OK
Content-Length: 1207
Content-Type: text/html; charset=utf-8
Date: Tue, 10 Nov 2020 14:07:56 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
<body>
<h1>Guess a number</h1>
<p></p>
<form method=POST action=/guess>
  Guess a number<br>
  <select name=number><option>1<option>2<option>3<option>4<option>5<option>6<option>7<option>8<option>9<option>10<option>11<option>12<option>13<option>14<option>15<option>16<option>17<option>18<option>19</select>                
  <input type=submit value="Guess!">
</form>

<h2>21 is correct!</h2>
```

### Mitigation
Het eerste probleem, het niet kunnen casten van het versuurde nummer, kan worden opgelost door een try-catch blok. Op deze manier wordt de exception op een nette manier afgehandeld en zal de server niet crashen.

Het tweede probleem, het goedkeuren van foute nummers kan op verschillende manieren worden opgelost:
- De applicatie kan eerst valideren of het nummer wel in de toegestande range zit voordat het nummer gecontroleerd wordt op juistheid.

```python
if number > 0 and number < 21:
    # Controleer juistheid
```

- De applicatie kan de structuur van de check aanpassen door de kijken of het nummer gelijk, groter of kleiner is dan het willekeurige getal

```python
if number == r:
    # return "Juiste antwoord"
else if number < r:
    # return "Nummer is te laag"
else if number > r:
    # return "Nummer is te hoog"
```

In het tweede voorbeeld maakt het niet uit dat de gebruiker een nummer invoert wat niet in de willekeurige range zit. De applicatie zal alleen maar zeggen dat het nummer juist is als het juiste nummer is ingevoerd. Het liefst zou de oplossing een combinatie zijn van de bovenstaande oplossingen.

## New Password

```python
@app.route('/password', methods=['GET','POST'])
def password():
    if request.method == 'GET':
        return render_template('password.html')

    password = request.form.get('password')
    new = request.form.get('new')
    repeat = request.form.get('repeat')

    if ( password is not None and pwhash != argon2.argon2_hash(password=str(password), salt="XQEXFggkPcw9BtuGkn4ELm4a7r7MUKTjBW2fjaVv6ou8mJ9ZrfEQBYhiGqQ6LzRz", t=16, m=8, p=1, buflen=128, argon_type=argon2.Argon2Type.Argon2_i) ):
         return render_template('password.html', result = 'Password is not correct')
    if len(new) < 16:
        return render_template('password.html', result = 'Passwords length is too short (min 16 characters)')
    if ( new != repeat ):
        return render_template('password.html', result = 'New passwords are not the same')
    
    return render_template('password.html', result = 'Password changed succesfully!')
```
### Problem
Als er een ```GET``` request wordt gestuurd naar ```/password``` zal de pagina worden gerenderd. Als er een ```POST``` request wordt gestuurd zullen eerst het wachtwoord, het nieuwe wachtwoord en het herhaalde wachtwoord worden opgehaald uit de form data:

```python
password = request.form.get('password')
new = request.form.get('new')
repeat = request.form.get('repeat')
```

Hierna zal worden gekeken of:
- Het wachtwoord niet ```None``` is.
- Het gehashte wachtwoord niet overeenkomt met de bestaande hash.

```python
if ( password is not None and pwhash != argon2.argon2_hash(password=str(password), salt="XQEXFggkPcw9BtuGkn4ELm4a7r7MUKTjBW2fjaVv6ou8mJ9ZrfEQBYhiGqQ6LzRz", t=16, m=8, p=1, buflen=128, argon_type=argon2.Argon2Type.Argon2_i) ):
```

Als beide stellingen ontjuist zijn zal server de foutmelding returnen: ```result = 'Password is not correct'```. Dus als het wachtwoord None is, zal het ```if``` statement niet naar true evalueren en zal de caller dus niet de foutmelding ontvangen. De rest van de checks zijn makkelijk te omzeilen. De caller moet enkel ```new``` sturen met een lengte groter dan ```16``` en een ```repeat``` die gelijk is aan ```new```.

### Evidence
Bij het sturen van de volgende request:
```
http -f POST localhost:5000/password new=qwertyuiopasdfgh repeat=qwertyuiopasdfgh
```
Zal het commando:
```python
request.form.get('password')
```
worden geëvalueerd naar ```None```.
Dit betekent dat als de check
```python
if password is not None #......
```
wordt uitgevoerd, dit niet voldoet aan de gestelde eisen.
Om deze rede krijgt de caller de volgende response:
```html
HTTP/1.0 200 OK
Content-Length: 1565
Content-Type: text/html; charset=utf-8
Date: Tue, 10 Nov 2020 19:57:55 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
<body>
<h1>Change your password</h1>

<p>The form below can be used to change your password.</p>
<form method=POST action=/password>
  <table border=0>
    <tr><td>Current password <td>: <td><input name=password value=>
    <tr><td>New password <td>: <td><input name=new value=>
        <tr><td>Repeat new password <td>: <td><input name=repeat value=>
            <tr><td> <td> <td><input type=submit value="Change password">
                </table>
</form>

<h2>Password changed succesfully!</h2>
```

### Mitigation
Ook hier zijn er een snelle en een nette oplossing. De snelle oplossing is om het ```if``` statement zo aan te passen dat er staat dat het wachtwoord ```None``` is of de hash niet overeen komt met de bestaande hash:
```python
if password is None or hash != #hash functie()
```
Dit zou betekenen dat als het password ```None``` is, de functie direct de foutmelding rendert.

De nette oplossing zou altijd de foutmelding returnen en alleen in het geval dat alles goed is, het wachtwoord veranderen:

```python
password = request.form.get('password')
new = request.form.get('new')
repeat = request.form.get('repeat')

if len(new) < 16:
    return render_template('password.html', result = 'Passwords length is too short (min 16 characters)')
if (new != repeat):
    return render_template('password.html', result = 'New passwords are not the same')
if (password is not None and pwhash == hash_function(password))):
    return render_template('password.html', result = 'Password changed')

return render_template('password.html', result = 'Password incorrect')
```
Op deze manier is er maar ```1``` mogelijkheid waarop dit werkt in plaats van een onbekend aantal mogelijkheden waarop dit werkt.

## File Upload

```python
@app.route('/upload', methods=['GET','POST'])
def upload():
    if request.method == 'GET':
        return render_template('upload.html')

    f = request.files['file']

    if f.content_type == 'image/jpeg' or f.content_type == 'image/gif':
        f.save('static/' + f.filename)
        return render_template('upload.html', result = 'File uploaded succesfully.')
            
    return render_template('upload.html', result = "File upload not accepted")
```

### Problem
Als de caller een ```POST```request stuurt met een file en een ```Content-Type: image/jpeg``` Header zal de file in de map ```static/ + filenaam``` worden opgeslagen. Hierbij gaat de server er vanuit dat de file ook daadwerkelijk van het type is wat er in de header staat.  Als de caller een bestand stuurt wat geen ```jpeg``` of ```gif``` is, maar wel een header stuurt die voldoet aan:
```python
if f.content_type == 'image/jpeg' or f.content_type == 'image/gif':
```
zal de server dit alsnog goedkeuren en de file opslaan.

### Evidence
Een bestand ```payload.conf``` kan worden gepost met het type ```image/jpeg```.

```
http -f localhost:5000/upload file@"payload.conf;type=image/jpeg"
```

De server geeft dan de volgende response:
```html
HTTP/1.0 200 OK
Content-Length: 1000
Content-Type: text/html; charset=utf-8
Date: Fri, 13 Nov 2020 15:08:13 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
  <body>
    <h1>Upload a file</h1>
    <p>The following form is used to upload an image to the server. Only jpeg en gif images are allowed.</p>
      <form action = "/upload" method = "POST" 
         enctype = "multipart/form-data">
         <input type = "file" name = "file" />
         <input type = "submit"/>
      </form>

      <h1>                                                                                                        
        File uploaded succesfully.                                                                                
      </h1>
```

### Mitigation
Dit probleem kan wroden verholpen door in de code niet alleen te kijken naar het data type in de header maar ook daadwerkelijk te controleren of het bestand van een toegestaan type is.

Een library genaamd ```imghdr``` kan het type een image bepalen.
```python
import imghdr

imghdr.what('/tmp/bass')
```

Dit zal ```'jpeg'``` of ```'gif'``` terug geven. Op deze manier kan er worden bepaald of het bestand daadwerkelijk een foto is.