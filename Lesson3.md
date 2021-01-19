# Lesson 3

## Online Banking
```python
def getUserFromSession(session):
    username = ''

    for user in users:
        if session == user['session']:
            username = user['username']
            break

    return username

@app.route('/transfer', methods=['POST'])
def transfer():
    to_value = request.form.get('to')
    amount_value = request.form.get('amount')
    description_value = request.form.get('description')

    session_cookie = request.cookies.get('session-cookie')
    current_user = getUserFromSession(session_cookie)

    if current_user != to_value and (to_value == 'henk' or to_value == 'oma'):
        transactions.append({'to':to_value,
            'from':current_user,
            'amount':amount_value,
            'description':description_value})

    return redirect("/onlinebanking")
```
### Probleem
Als een user word opgehaald uit de sessie wordt username geïnitialiseerd als ```username = ''```. Als er in de loop geen username word gevonden zal de functie ```''``` returnen.
```python
username = ''
for user in users:
    if session == user['session']:
        username = user['username']
        break

return username
```
Bij de onderstaande check zal de ```to_value``` dus ongelijk zijn aan ```current_user```.
```python
if current_user != to_value and (to_value == 'henk' or to_value == 'oma'):
        transactions.append({'to':to_value,
            'from':current_user,
            'amount':amount_value,
            'description':description_value})
```
Hierdoor zal de transactie dus wel doorgaan.

### Evidence
Als de gebruiker de volgende call maakt:
```
http -f POST localhost:5000/transfer to=henk amount=1337 description=patat
```
En vervolgens inlogt met henk ziet hij deze transactie:
```html
<!DOCTYPE html>
<html>
<body>
<h1>Bank dashboard</h1>
<p>Current user: 'henk'</p>
  <h2>Transactions</h2>
  <table>
    <thead>
      <tr>
        <th>To</th>
        <th>From</th>
        <th>Amount</th>
        <th>Description</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>henk</td>
        <td></td>
        <td>1337</td>
        <td>patat</td>
      </tr>
```

### Mitigation
De server zou een ```Exception``` moeten raisen als de sessie niet correspondeert met een gebruiker. Het ```if``` statement zou moeten worden opgedeeld in 3 stappen:

```python
if to_value == current_user:
    # raise Exception

if !database.validate(to_value, current_user):
    # raise Exception

if validate_amount(amount):
    # render_template()

# raise Exception
```

## Shop
Basket:
```python
basket = {
    '1':{
        'description':'Apple Juice',
        'quantity':0
    },
    '2':{
        'description':'Grapefruit Juice',
        'quantity':0
    },
    '3':{
        'description':'Carrot Juice',
        'quantity':0
    },
    '4':{
        'description':'Orange Juice',
        'quantity':0
    },
    '5':{
        'description':'Beet root juice',
        'quantity':0
    }
}
```
Function:
```python
@app.route('/shop', methods=['GET'])
def shop():
    if request.method == 'GET':
        #
        if request.args.get("action") == "checkout":
            baskettotal = 0
            order = {}
            for item in basket:
                order[item] = { 'description':basket[item]['description'],
                    'quantity':basket[item]['quantity'] }                            
                baskettotal = baskettotal + basket[item]['quantity']
                basket[item]['quantity'] = 0
                
            return render_template('checkout.html', basket = order)
                        
    return render_template('shop.html', basket = basket, credits=10, baskettotal=0)
```
### Problem
In de shop vind er geen validatie plaats van het aantal items in de ```basket```. De server rendert de knop naar de ```/checkout``` url alleen als er 10 of minder items in de basket zit. Dit betekent echter niet dat de ```/checkout``` url niet bestaat. Een gebruiker kan gemakkelijk navigeren naar de url terwijl er meer items in de basket zitten.

### Evidence
```
http GET localhost:5000/shop action==buy item==1
```
```html
HTTP/1.0 200 OK
Content-Length: 3040
Content-Type: text/html; charset=utf-8
Date: Mon, 30 Nov 2020 18:15:10 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
<body>
This shop sells a limited selection of high quality fruit juices. Come in and buy some!
<table border=0>
  //
</table>
There are 10 credits available to buy Juice!
<p>Insufficient credits available. Remove items from your basket!</P>
```
Maar we kunnen toch naar ```/checkout``` navigeren.
```
http GET localhost:5000/shop? action==checkout
```
```html
HTTP/1.0 200 OK
Content-Length: 492
Content-Type: text/html; charset=utf-8
Date: Mon, 30 Nov 2020 18:30:55 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<!DOCTYPE html>
<html>
<body>
<h1>Web shop</h1>

<p>Thanks for buying, visit us again or by some more <a href="/shop">here</a>.</p>

<h2>Order summary</h2>
<table border=1>
  <tr><td><b>Item</b></td><td><b>Amount</b></td></tr>
  <tr><td>Apple Juice</td><td>12</td></tr>
```

### Mitigation
De validatie van de basket zou moeten platsvinden op het ```/checkout endpoint```.
```python
if baskettotal > credits:
    raise Exception("The total exceeds your balance.")
```

## Document Download
```python
@app.route("/get-document/")
def get_document(documentid):
    try:
        fileid = base64.b64decode(documentid).decode()
        return send_from_directory("./documents", filename=fileid,
            as_attachment=True, attachment_filename=documentid)
    except FileNotFoundError:
        abort(404)
```

```html
<ul>
  <li><a href="/get-document/MjE3NjEy">Document 1</a>
  <li><a href="/get-document/MjE3NjEz">Document 2</a>
  <li><a href="/get-document/MjE4NTI0">Document 3</a>
  <li><a href="/get-document/MjE2NTMy">Document 4</a>
</ul>
```

```bash
oot@kali:~/Development/SecureProgramming/Lesson3# ll documents/
total 28
-rw-r--r-- 1 root root 1816 Nov 10 14:33 216532 216532
-rw-r--r-- 1 root root 1824 Nov 10 14:33 216533
-rw-r--r-- 1 root root  474 Nov 10 14:33 217612 217612
-rw-r--r-- 1 root root  922 Nov 10 14:33 217613 217613
-rw-r--r-- 1 root root 1822 Nov 10 14:33 217614
-rw-r--r-- 1 root root 1369 Nov 10 14:33 218524 218524
-rw-r--r-- 1 root root 1822 Nov 10 14:33 218525
```
### Problem
Het commando ```fileid = base64.b64decode(documentid).decode()``` is verantwoordelijk voor het coderen van de file naam. Het base64 algoritme is cryptografisch niet willekeurig genoeg om veilig te zijn voor het downloaden van bestanden. Men kan namelijk een verboden bestand base64 encoden en alsnog een geheim bestand downloaden.

### Evidence
```
http GET localhost:5000/get-document/MjE2NTMz
```

```
TTP/1.0 200 OK
Cache-Control: public, max-age=43200
Content-Disposition: attachment; filename=MjE2NTMz
Content-Length: 1824
Content-Type: application/octet-stream
Date: Mon, 30 Nov 2020 16:41:07 GMT
ETag: "1605015228.0-1824-1534002994"
Expires: Tue, 01 Dec 2020 04:41:07 GMT
Last-Modified: Tue, 10 Nov 2020 13:33:48 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

This is a confidential document. 

Lorem ipsum dolor sit amet...
```

### Mitigation
Er zou een vertalingstabel moeten worden bijgehouden waarin de file namen staan, gekoppeld aan cryptografisch willekeurige ID's. Op die manier kan de gebruiker niet achterhalen wat de namen of hashes zijn van de bestanden. Ook zou validatie aan de server kant moeten plaatsvinden waarin word gechekt welke gebruiker rechten heeft op welk bestand.