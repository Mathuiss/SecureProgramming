# Lesson 6

## Uncaught Exceptions
```python
# Get block
number = request.form.get('number')
if number == None:
    return render_template('guess.html',result = "Your input is ignored")
else:
    number = int(number)
# Else block
```
### Probleem
In de bovenstaande functie is het mogelijk om een 500 internal server error te genereren. Dit gebeurt wanneer er een nummer uit de request word gehaald en vervolgens naar een ```int``` word geparsed. Als dit niet mogelijk is omdat ```request.form.get("nmumber")``` eigenlijk een string is genereert de server een error met een stack trace.

### Evidence
Als de caller een request doet zoals:
```bash
http -f POST localhost:5000/guess number=a
```
Dan genereert de server de volgende response:
```html
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Length: 290
Content-Type: text/html; charset=utf-8
Date: Tue, 19 Jan 2021 12:36:21 GMT
Server: Werkzeug/1.0.1 Python/3.9.1rc1

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
```

### Mitigation
Dit probleem kan worden verholpen door simpelweg een ```try```-```except``` blok te maken. Dit kan worden gedaan op de volgende manier:

```python
try:
    number = int(number)
    # Andere code die mogelijk fout kan gaan
except ParseException:
    # Wat de server moet doen als dit fout gaat
```
Momenteel wordt de ```ParseException``` opgevangen, maar dit kan voor verschillende soorten exceptions gedaan worden zodat er voor elke fout een specifieke oplossing kan worden geïmplementeerd.

## Base64 Encoding
```python
@app.route("/get-document/<documentid>")
def get_document(documentid):
    try:
        fileid = base64.b64decode(documentid).decode()
        return send_from_directory("./documents", filename=fileid, as_attachment=True, attachment_filename=documentid)
    except FileNotFoundError:
        abort(404)
```
### Problem
In het bovenstaande voorbeeldprobeert de server het document id te decoderen met ```base64```. Er is wel een ```try```-```catch``` blok maar die vangt de ```binascii.Error``` niet op. Als een caller een ```documentid``` opstuurt die niet ```base64``` gecodeerd is crasht de server.

### Evidence
Als een caller een request doet met een niet gecodeerde string:
```bash
http GET localhost:5000/get-document/x
```
Dan genereert de server de volgende response:
```html
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Length: 290
Content-Type: text/html; charset=utf-8
Date: Tue, 19 Jan 2021 13:31:54 GMT
Server: Werkzeug/1.0.1 Python/3.9.1rc1

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
```

### Mitigation
Als oplossing zou onder het ```except``` blok een nieuw ```except``` blok moeten komen voor het geval de gebruiker een foutieve codering opgeeft. Om andere onvoorziene problemen op te vangen kan er ook nog een generieke ```except:``` worden neergezet als laatste redmiddel.

## God Mode
```python
@app.route('/mode', methods=['GET','POST'])
def mode():
    mode = 'Running in normal mode'
    result = None
    if request.method == 'GET':
        return render_template('mode.html',result=None, mode=mode)
    if request.method == 'POST':
        input = request.form.get('input');
        if input != None and re.match("^[A-Za-z0-9]+$",input):
            result = os.system("./util %s" % input)
            if ( result == 512 ):
                mode = 'Running in god mode'
            else:
                time.sleep(3)
            result = len(input)
        else:
            result = None
        return render_template('mode.html',result=result, mode=mode)
```
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)  
{
    volatile int debug;
    char buffer[64];

    // change debug to enable debugging in god mode
    debug = 0x00;

    // one argument is required
    if ( argc == 2 ) {
    strcpy(buffer, argv[1]);
    } else {
    exit(-1);
    }

    fprintf(stderr,"debug = %08x\n",debug);

    if(debug == 0x50 ) {
    exit(2);
    } else {
    exit(1);
    }
}
```
### Problem
De web applicatie luistert naar een een request met daarin een input. Deze input word als argument meegegeven aan een util binary die geschreven is in c. In deze util wordt een ```buffer[64]``` gedeclareerd. Vervolgens word een debug byte aangemaakt. De input wordt weggeschreven naar de buffer, waarna er word gekken naar de debug byte om te kijken of het programma in godmode moet runnen. Als de debug byte gelijk is aan ```0x50``` gaat het programma in god mode, anders in normal mode. De input word niet op lengte gecheckt en er kan dus een bufferoverflow ontstaan. Hierdoor is het mogelijk om de debug byte aan te passen en het programma in god mode te krijgen.

### Evidence
Bij het uitvoeren van het c programma is er een output zoals dit:
```bash
./util input
debug = 00000000
```

Vervolgens kan er net zo ver over de buffer worden geschreven totdat de debug variabele word aangetast.
```bash
./util xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
debug = 00000078
```

Nu dat het duidelijk is dat de debug byte kan worden aangepast naar ```0x78``` moet er een manier gevonden worden om de byte op ```0x50``` te krijgen.
De ASCII waarde ```0x50``` is gelijk aan de hoofdletter ```P```.

```bash
./util xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxP
debug = 00000050
```

Dus bij het opsturen van de request geeft de server de godmode response:
```
http -f POST localhost:5000/mode input=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxP
```
```html
HTTP/1.0 200 OK
Content-Length: 1776
Content-Type: text/html; charset=utf-8
Date: Wed, 30 Dec 2020 11:10:23 GMT
Server: Werkzeug/1.0.1 Python/3.9.1rc1

<h1>God mode</h1>
```

### Mitigation
Een oplossing zou kunnen zijn om de lengte van de input te vergelijken met de lengte van de buffer. Als de input groter is dan past in de buffer moet de applicatie stoppen.
```c++
if (strlen(input) <= sizeof(buffer)) {
    strcpy(buffer, argv[1]);
} else {
    exit(3);
}
```